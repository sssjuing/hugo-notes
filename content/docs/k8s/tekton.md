---
title: Tekton
date: "2024-09-30"
---

每个 Task 在各自单独的 Pod 中执行，使用相同的硬盘，通常执行简单的工作，如执行一个 test 或 lint，或者创建一个 Kaniko 缓存。默认情况下，一个 Pipeline 中的 Task 不共享 data ，因此，用户必须显式配置 Task，使其 outputs 作为下一个 Task 的 inputs。

## Tekton 编译前端项目并上传到镜像仓库

### 用于编译前端镜像的 Dockerfile

```dockerfile
FROM docker.io/joseluisq/static-web-server:2
COPY ./dist ./public
ENV SERVER_FALLBACK_PAGE="./public/index.html"
EXPOSE 80
ENTRYPOINT [ "./static-web-server"]
```

{{<callout type="info">}}
注意：可以使用 `docker build -t <image_name> .` 命令直接在本地构建镜像，之后对构建好的镜像使用 `docker tag  <image_name> <image_name_with_username_and_tag>` 命令修改镜像的 tag 后，使用 `docker push <image_name_with_username_and_tag>` 推送到镜像仓库。P.S.: `docker push myname/imagename:0.0.1`
{{</callout>}}

### pnpm task

将如下代码保存为 pnpm.yaml 文件，随后输入 `kubectl apply -f pnpm.yaml` 将其输入 k8s 集群中保存为一个 Task。

```yaml
# pnpm.yaml

apiVersion: tekton.dev/v1
kind: Task
metadata:
  name: pnpm
  labels:
    app.kubernetes.io/version: "0.1"
  annotations:
    tekton.dev/pipelines.minVersion: "0.64.0"
    tekton.dev/categories: Build Tools
    tekton.dev/tags: build-tool
    tekton.dev/platforms: "linux/amd64,linux/s390x,linux/ppc64le"
spec:
  description: >-
    This task can be used to run pnpm goals on a project.

    This task can be used to run pnpm goals on a project
    where package.json is present and has some pre-defined
    pnpm scripts.
  workspaces:
    - name: source
  params:
    - name: PATH_CONTEXT
      type: string
      default: "."
      description: The path where package.json of the project is defined.
    - name: ARGS
      type: array
      default: ["version"]
      description: The pnpm goals you want to run.
    - name: IMAGE
      type: string
      default: "guergeiro/pnpm:20-9-alpine"
      description: The node image you want to use.
  steps:
    - name: pnpm-run
      image: $(params.IMAGE)
      command:
        - "pnpm"
      args:
        - $(params.ARGS)
      workingDir: $(workspaces.source.path)/$(params.PATH_CONTEXT)
      env:
        - name: CI
          value: "true"
```

### Pipeline

将如下代码保存为 pipeline.yaml 文件，随后输入 `kubectl apply -f pnpm.yaml` 将其输入 k8s 集群中保存为一个 Task。

```yaml
# pipeline.yaml

apiVersion: tekton.dev/v1
kind: Pipeline
metadata:
  name: pnpm-build
spec:
  description: |
    This pipeline clones a git repo, then echoes the README file to the stdout.
  params:
    - name: repo-url
      type: string
      description: The git repo URL to clone from.
    - name: path-context
      type: string
      description: The path where target package.json of the project is defined.
    - name: image-reference
      type: string
      description: "Name (reference) of the image to build. e.g.: docker.io/sujingclg/monorepo-boilerplate"
    - name: tag
      type: string
  workspaces:
    - name: shared-data
      description: |
        This workspace contains the cloned repo files, so they can be read by the
        next task.
    - name: git-credentials
      description: My ssh credentials
    - name: docker-credentials
      description: My docker credentials
  tasks:
    - name: fetch-repository
      taskRef:
        name: git-clone
      workspaces:
        - name: output
          workspace: shared-data
        - name: ssh-directory
          workspace: git-credentials
      params:
        - name: url
          value: $(params.repo-url)
    - name: install-dependencies
      runAfter:
        - fetch-repository
      taskRef:
        name: pnpm
      workspaces:
        - name: source
          workspace: shared-data
      params:
        - name: ARGS
          value: ["i"]
    - name: run-build
      runAfter: ["install-dependencies"]
      taskRef:
        name: pnpm
      workspaces:
        - name: source
          workspace: shared-data
      params:
        - name: PATH_CONTEXT
          value: $(params.path-context)
        - name: ARGS
          value:
            - build:docker
    - name: build-push-image
      taskRef:
        name: buildah
      runAfter: ["run-build"]
      workspaces:
        - name: source
          workspace: shared-data
        - name: dockerconfig
          workspace: docker-credentials
      params:
        - name: IMAGE
          # value: docker.io/sujingclg/monorepo-boilerplate:$(params.tag)
          value: $(params.image-reference):$(params.tag)
        - name: CONTEXT
          value: "$(params.path-context)/docker"
        - name: FORMAT
          value: "docker"
```

#### 从 git 上拉取代码

Tekton 需要从 git 上拉取代码到本地，可以使用 Tekton Hub 中的 **git-clone** task。如果是私有仓库，则需要创建一个 k8s 的 secret，将 ssh 私钥和 known_hosts 做 base64 转换后传入 git-clone。以下代码用于创建 secret。

```yaml
# git-ssh-secret.yaml

apiVersion: v1
kind: Secret
metadata:
  name: git-credentials
data:
  # cat ~/.ssh/id_rsa | base64 -w0
  id_rsa: AS0tLS...
  known_hosts: AG033S...
  # config: GS0FFL...
```

之后，需要在 pipeline.yaml 中添加如下代码，并将上述 secret 的 name 作为 ssh-directory 参数传入 git-clone。其中 output 及其指向的 **shared-data** workspace 用于将拉取到的代码传入下一个 task。

```yaml
workspaces:
  - name: shared-data
    description: |
      This workspace contains the cloned repo files, so they can be read by the
      next task.
  - name: git-credentials
    description: My ssh credentials
tasks:
  - name: fetch-source
    taskRef:
      name: git-clone
    workspaces:
      - name: output
        workspace: shared-data
      - name: ssh-directory
        workspace: git-credentials
```

#### 上传镜像产物到仓库

上传镜像产物到仓库一般需要用户的账号信息，在上述 pipeline.yaml 中对应为名为 docker-credentials 的 secret。其大致格式如下：

```yaml
# docker-secret.yaml

apiVersion: v1
kind: Secret
metadata:
  name: docker-credentials
data:
  config.json: GS0FFL...
```

其中 config.json 对应的值为 ～/.docker/config.json 经过 base64 编码后的值，由于 docker login 一般会使用操作系统的外部凭据存储功能，因此 config.json 默认没有包含用户的登录凭据，因此需要使用如下代码直接获取包含凭据的 config.json 的 base64 编码。

```bash
kubectl create secret docker-registry docker-regcred --dry-run=client \
--docker-server=https://index.docker.io/v1/ \
--docker-username=xxx \
--docker-password=xxx \
-o yaml > docker-secret.yaml
```

### Pipeline Run

pipeline run 中包含了运行对应的 pipeline 所需要的信息，包括给 workspaces 提供 pvc 存储，git 仓库的地址，git 和镜像仓库的凭据等。输入 `kubectl create -f pipelinerun.yaml` 执行 pipeline。注意此处是 create 而非 apply。

```yaml
# pipelinerun.yaml

apiVersion: tekton.dev/v1
kind: PipelineRun
metadata:
  generateName: pnpm-build-run-
spec:
  pipelineRef:
    name: pnpm-build
  workspaces:
    - name: shared-data
      volumeClaimTemplate:
        spec:
          accessModes:
            - ReadWriteOnce
          resources:
            requests:
              storage: 1Gi
          storageClassName: topolvm-provisioner
    - name: git-credentials
      secret:
        secretName: git-credentials
    - name: docker-credentials
      secret:
        secretName: docker-credentials
  params:
    - name: repo-url
      value: git@gitee.com:sujingclg/monorepo-boilerplate.git
    - name: path-context
      value: "apps/web"
    - name: image-reference
      value: "docker.io/sujingclg/monorepo-boilerplate"
    - name: tag
      value: "0.0.2"
```

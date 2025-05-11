---
title: OpenWrt LEDE
date: "2024-09-22"
---

---

## 编译

### LEDE 仓库准备

- 首先 Fork `https://github.com/coolsnowwolf/lede
- 其次，参考 https://github.com/KFERMercer/OpenWrt-CI，上传`openwrt-ci.yml`和`merge-upstream.yml`到刚才Fork的`/.github/workflows/`下。
- 由于 Fork 的`coolsnowwolf/lede`没有 PassWall 组件，需要参考 https://github.com/kenzok8/openwrt-packages，添加相关package到自己的仓库。

### 编译虚拟机准备

使用 Ubuntu Server 20.04 LTS x64 虚拟机，注意创建时设置 80G 硬盘，且在安装步骤中的 Storage Configuration 页面手动调整 ubuntu-lv 的硬盘大小不小于 60G。配置软件源为http://mirrors.aliyun.com/ubuntu。详细过程可参考[Ubuntu 20.04 live server 版安装(详细版)](https://www.cnblogs.com/mefj/p/14964416.html)。

虚拟机创建完成后输入以下命令，参考 https://github.com/coolsnowwolf/lede。

```shell
sudo apt-get update
sudo apt-get -y install build-essential asciidoc binutils bzip2 gawk gettext git libncurses5-dev libz-dev patch python3 python2.7 unzip zlib1g-dev lib32gcc1 libc6-dev-i386 subversion flex uglifyjs git-core gcc-multilib p7zip p7zip-full msmtp libssl-dev texinfo libglib2.0-dev xmlto qemu-utils upx libelf-dev autoconf automake libtool autopoint device-tree-compiler g++-multilib antlr3 gperf wget curl swig rsync
# 初始化git
ssh-keygen
git config --global user.name "<username>"
git config --global user.email "<username>@***.***"
git clone <上一步准备好的lede仓库>
# 准备配置菜单
./scripts/feeds update -a
./scripts/feeds install -a
make menuconfig
```

推荐主题：ifit、Argon、Netgear、Edge。**LuCI => Applications**详细配置可参考[OpenWrt 编译 LuCI -> Applications 添加插件应用说明-L 大](https://www.right.com.cn/forum/thread-344825-1-1.html)、[OpenWRT 编译 make menuconfig 配置及 LUCI 插件说明.xlsx](https://docs.google.com/spreadsheets/d/1e1pvol-9QK6HgkbqRRmtqlOUaeVr_SxIiopd4foi6q4/edit#gid=18734276)。

如果是本地编译，则按照 lede 的说明进行操作即可。如果需要使用 Github 的 Action 进行在线编译，则参考 https://github.com/KFERMercer/OpenWrt-CI 进行。

注意如果在使用 Github 的 Action 时需要定制化 lede，按如下步骤进行。

```shell
# https://github.com/coolsnowwolf/lede/issues/2288
make defconfig
./scripts/diffconfig.sh > seed.config
```

之后将 seed.config 中的内容复制粘贴到`/.github/workflows/openwrt-ci.yml`的指定位置中，然后提交。

```yaml
          #
          # 在 cat >> .config <<EOF 到 EOF 之间粘贴你的编译配置, 需注意缩进关系
          # 例如:
          cat >> .config <<EOF
          # 将seed.config的内容粘贴在这里
          EOF
          #
          # ===============================================================
          #
```

注意：编译过程中可能会报错 `No space left on device`，显示编译空间不足，可以参考[这里](https://github.com/coolsnowwolf/lede/issues/11887)，通过在 steps 中的最前面添加以下脚本删除无用的软件或文件。

```yaml
- name: Before freeing up disk space
  run: |
    echo "Before freeing up disk space"
    echo "=============================================================================="
    df -hT
    echo "=============================================================================="

- name: "Optimize Disk Space"
  uses: "hugoalh/disk-space-optimizer-ghaction@v0.8.1"
  with:
    operate_sudo: "True"
    general_include: ".+"
    general_exclude: |-
      ^GCC$
      ^G\+\+$
      Clang
      LLVM
    docker_include: ".+"
    docker_prune: "True"
    docker_clean: "True"
    apt_prune: "True"
    apt_clean: "True"
    homebrew_prune: "True"
    homebrew_clean: "True"
    npm_prune: "True"
    npm_clean: "True"
    os_swap: "True"

- name: Freeing up disk space
  uses: easimon/maximize-build-space@master
  with:
    root-reserve-mb: 2048
    swap-size-mb: 1
    remove-dotnet: "true"
    remove-android: "true"
    remove-haskell: "true"
    remove-codeql: "true"
    remove-docker-images: "true"

- name: Free up disk space complete
  run: |
    echo "Free up disk space complete"
    echo "=============================================================================="
    df -hT
    echo "=============================================================================="
```

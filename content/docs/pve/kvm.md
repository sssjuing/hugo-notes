---
title: KVM
date: "2024-10-15"
---

---

使用 `qemu-img` 转换镜像格式

```bash
qemu-img convert -p -f qcow2 -O raw /folder/image_name.qcow2 /folder/image_name.raw
```

参数说明：

- -p: presenting the conversion progress

- -f: format of the source image

- -O: format of the target image

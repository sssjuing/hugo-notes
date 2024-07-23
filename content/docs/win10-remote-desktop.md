---
title: Win10 远程桌面连接
---

---

## Windows10 支持远程桌面连接

1. 首先在这里下载最新版的 RDP Wrapper Library，解压到程序目录下。

![](https://pic2.zhimg.com/80/v2-3c478f9315809399348186683cfc4c29_720w.webp)

2. 右键以管理员身份运行 install.bat

3. 在[这里](https://github.com/sebaxakerhtc/rdpwrap.ini)获取到最新的 rdpwrap.ini 文件，复制到此目录下

4. 在此目录下打开命令行，输入如下命令

```powershell
rdpwinst -u -k
rdpwinst -i
```

4. 右键以管理员身份运行 RDPConf.exe

![](https://pic3.zhimg.com/80/v2-b4858462e722ad9591c5b1a855b9fb12_720w.webp)

如无问题，如上图所示，就可以远程连接了

---

### 参考文章

- https://github.com/loyejaotdiqr47123/rdpwrap/releases

- https://zhuanlan.zhihu.com/p/445216327

- https://github.com/stascorp/rdpwrap/issues/2346

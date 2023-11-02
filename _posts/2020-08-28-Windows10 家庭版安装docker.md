---
categories: [tool]
tags: [windows, docker]
---

> #### Windows10 家庭版升级到20H1（19041）版本后，可以直接安装


#### 本机环境
![image.png](https://raw.githubusercontent.com/vinceDa/image-host/main/img/1598580128521-943eeaaf-1fc5-4813-ba8d-b8e29814ce79-20231102085441082.png)

#### 开启Hyper-V

> Docker for Windows需要开启Hyper-V，而windows10家庭版是默认不能开启Hyper-V的


- 新建hyperv.cmd文件

  ```bash
  pushd "%~dp0"
  
  dir /b %SystemRoot%\servicing\Packages\*Hyper-V*.mum >hyper-v.txt
  
  for /f %%i in ('findstr /i . hyper-v.txt 2^>nul') do dism /online /norestart /add-package:"%SystemRoot%\servicing\Packages\%%i"
  
  del hyper-v.txt
  
  Dism /online /enable-feature /featurename:Microsoft-Hyper-V-All /LimitAccess /ALL
  ```

  
- 以管理员身份运行该文件，可能会要求重启，选Y便是
- 在控制面板->程序和功能->启用或关闭Windows功能打开Hyper-V，如下图

#### 伪装成win10专业版

- 以管理员身份运行cmd
- 在cmd窗口中运行以下命令
```bash
REG ADD "HKEY_LOCAL_MACHINE\software\Microsoft\Windows NT\CurrentVersion" /v EditionId /T REG_EXPAND_SZ /d Professional /F
```

#### 安装docker

**以上环境都准备好了之后就可以开始安装docker了**

- 官网下载docker windows版exe
- 下载到本地后双击.exe文件安装即可

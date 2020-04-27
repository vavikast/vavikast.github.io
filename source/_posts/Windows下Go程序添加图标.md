---
title: Windows下Go程序添加图标
catalog: true
date: 2020-04-27 17:55:52
subtitle:
header-img: "/img/article_header/article_header.png"
tags:
- golang
catagories:
- golang
updateDate: 2020-04-27 17:55:52
​---
---

计划使用go语言编译一系列实用工具，提高自己的工作效率。发现编译后的.exe文件没有图标，甚是难看，所以找了windows平台下添加Go程序图标的方法。

#### 1. 查找ico图标

​		查找一个符合程序气质的图标，下载备用。

​		ico链图标下载： [easyicon](https://www.easyicon.net/)

#### 2.生成syso文件

​        rsrc是在Windows的Go程序中嵌入.ico和manifest资源的工具。

##### 		2.1 下载安装rsrc

```
go get github.com/akavel/rsrc
```

##### 		2.2  生成程序描述文件ico.manifest

```
<?xml version="1.0" encoding="UTF-8" standalone="yes"?>
<assembly xmlns="urn:schemas-microsoft-com:asm.v1" manifestVersion="1.0" xmlns:asmv3="urn:schemas-microsoft-com:asm.v3">
	<assemblyIdentity version="1.0.0.0" processorArchitecture="*" name="SomeFunkyNameHere" type="win32"/>
	<dependency>
		<dependentAssembly>
			<assemblyIdentity type="win32" name="Microsoft.Windows.Common-Controls" version="6.0.0.0" processorArchitecture="*" publicKeyToken="6595b64144ccf1df" language="*"/>
		</dependentAssembly>
	</dependency>
	<asmv3:application>
		<asmv3:windowsSettings xmlns="http://schemas.microsoft.com/SMI/2005/WindowsSettings">
			<dpiAware>true
		</asmv3:windowsSettings>
	</asmv3:application>
</assembly>
```

##### 		2.3  生成go程序嵌入文件

```
rsrc.exe -manifest ico.manifest -o myapp.syso -ico myapp.ico
```

##### 		2.4  提示

- go get下载的rsrc.exe记得添加到环境变量PATH中，以免无法执行rsrc.exe。
- go get 如果下载较慢，可以通过go mod 方式下载。
- 生成的myapp.syso记得放到源码文件目录中。			

#### 3.编译Go程序

​		将myapp.syso文件放到相应go程序下，然后直接运行go build .即可。

​		golang已经可以自动寻找子目录下的 syso 文件。

​		例如我的程序。

![image.png](https://i.loli.net/2020/04/27/5bn8GMFfchrXSel.png)



#### 	4. 参考

[教你为Win下的Go程序添加图标](https://studygolang.com/articles/7980)

[RSRC](
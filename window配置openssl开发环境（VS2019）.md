---
title: window配置openssl开发环境（VS2019）
date: 2020-07-12 09:52:14
tags:
- 环境配置
- openssl
- vs2019
comments: true
categories: 程序开发
---
*基础环境信息 ：*  
Window 10 专业版 1909  
Microsoft Visual Studio Community 2019（16.5.5）  
Win64 OpenSSL v1.1.1g  
(https://slproweb.com/products/Win32OpenSSL.html)  
<!-- more -->
*配置步骤：*  
1、下载安装已经编译好的openssl安装包  
位数版本根据想要开发出的软件版本选择，而不是你的操作系统版本  
如果需要开发双版本不要安装在同一目录下  
![1](1.png) 
最好将dll放在openssl目录下，方便使用时调用  
2、在VS中配置项目属性  
新建一个解决方案  
![2](2.png) 
更改项目设置  
平台根据你要开发的项目位数选择  
![3](3.png) 
在包含目录中添加openssl安装目录下的include文件夹  
![4](4.png) 
![5](5.png) 
在包含库目录中添加openssl安装目录下的lib  
![6](6.png) 
![7](7.png) 
3、复制dll文件  
将OpenSSL安装目录下bin文件夹中的“libcrypto-1_1-x64.dll”和“libssl-1_1-x64.  dll”（名字后面的版本号可能因更新而不同）复制到工程目录下  
![8](8.png) 

4、添加lib文件  
可以直接使用代码引入  
#pragma comment(lib,"libssl.lib")  
#pragma comment(lib,"libcrypto.lib")  
或在项目属性-链接器-输入-附加依赖项中添加  
![9](9.png) 
![10](10.png) 
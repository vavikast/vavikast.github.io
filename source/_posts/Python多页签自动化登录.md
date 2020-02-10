---
title: Python多页签自动化登录
catalog: true
date: 2019-12-30 12:32:03
subtitle:
header-img: "/img/article_header/article_header.png"
tags:
- python
catagories:
- python
updateDate: 2019-12-30 12:32:03
---
#### Python多页签自动化登录	


​		自己管理了好几个系统，虽然实现自动监控报警，但是还还想要人工检查。为了提高效率，现在写了一个脚本实现多个系统的自动化登录。

##### 		脚本选择：

- 开始想用bat实现，发现走不通，账号和密码登录认证的方式无法解决。有方法的小伙伴可以推荐。
- 后面使用python实现，主要是方案成熟，可参考案例多啊。

##### 		浏览器选择

​		chrome浏览器：因为习惯了。	

##### 		事前准备

- 安装python： 机器已装python3.6.2

- 安装selenium： pip install selenium 

- 安装webdriver插件：选择chrome版本对应的webdriver( http://chromedriver.chromium.org/downloads )，解压至相关目录下。

  ##### 目的

  - 自动输入账号和密码认证，实现自动登录。

  - 同时打开多个系统，在一个chrome浏览器下打开多页签。

  ##### 脚本实现：

  ```
  # -*- coding: utf-8 -*-
  import  os
  from  selenium import  webdriver
  from selenium.webdriver.common.keys import Keys
  chromedriver = "I:\webdriver\chromedriver.exe"
  os.environ["webdriver.chrome.driver"] = chromedriver
  driver = webdriver.Chrome(chromedriver)       # 声明浏览器对象
  
  username = "admin"
  username1 = "root"
  password = "xxxxyyyy1111"
  password1 = "xxxxyyyy2222"
  
  #1.管理系统
  driver.get("https://192.168.21.6/login/login.htm")
  driver.find_element_by_id("username").send_keys(username)  //driver.find_element_by_id("username") 查找id方式
  driver.find_element_by_id("password").send_keys(password2)
  driver.find_element_by_xpath('//*[@id="form"]/form/div[5]/input').click() //driver.find_element_by_xpath 查找xpath方式
  #2.管理系统1
  driver.execute_script("window.open();")
  # switch to the new window which is second in window_handles array
  driver.switch_to.window(driver.window_handles[1])
  driver.get("https://192.168.21.7/zh_cn/")
  driver.find_element_by_xpath('//*[@id="hs_login_tbl"]/tbody/tr[1]/td[2]/input').send_keys(username1)
  driver.find_element_by_xpath('//*[@id="hs_login_tbl"]/tbody/tr[2]/td[2]/input').send_keys(password1)
  
  
  ```

  ​			注解：

  ```
  driver.execute_script("window.open();")
  # switch to the new window which is second in window_handles array
  driver.switch_to.window(driver.window_handles[1])
  handles[] 中的数字代表打开第几个页签，如果后面还有管理系统，填写handles[2]。从0开始计数，代表打开第三个页签。
  ```

  

  ##### 重点说明：

  - xpath的使用

    ​	每个网站使用的框架不同，但是xpath很容易确定路径，解决问题。

    ​	基本说明下：

    ​	1.打开网页，按F12调出开发者工具，选到Elements页面。

    ​	2.点击页面中的输入框，此时开发者页面定为到所在代码行。

    ​	3.右键代码选择COPY-选择copy xpath。

    ​	4.复制粘贴到代码即可。

  - chrome多页面的打开

    请参考“文档参考”

    先打开了一个chrome浏览器，自动输入账号和密码，再打开一个新的页签，切换到新的页签，自动输入账号和密码，以此往复。

  ##### 文档参考：

  [大型网站模拟登录](https://github.com/Kr1s77/awesome-python-login-model)

  [chrome中打开多页签](https://stackoverflow.com/questions/31559365/selenium-new-tab-in-chrome-browser-by-python-webdriver/52240407)

  [使用python+selenium实现浏览器自动登录](https://www.jianshu.com/p/d7a966ec1189)

​				

​	

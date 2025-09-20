---
date : '2025-03-26T08:53:10+08:00'
draft : true
title : 'Mac效率提升工具'
tags : ["Mac"]
categories: ["工具"]
---
# 动图工具
[licecap](https://www.cockos.com/licecap/)  
应该没有比 licecap 更加傻瓜式的动图录制工具了吧！
![](https://github.com/BoomChao/boomchao.github.io/blob/main/content/posts/rd-tools/picture/mac/licecap.png?raw=true)

# 截图软件
[Snipaste](https://www.snipaste.com/)  
支持截图，贴图等功能（贴图yyds👍）
![](https://github.com/BoomChao/boomchao.github.io/blob/main/content/posts/rd-tools/picture/mac/snipaste.gif?raw=true)


# 切屏软件
[AltTab](https://alt-tab-macos.netlify.app/)  
有了它，对于多开窗口的同学真的效率提升好多，再也不用鼠标移动到任务栏去找应用了！  
自己配置多应用切换款快捷键以及活跃应用切换款快捷键
![](https://github.com/BoomChao/boomchao.github.io/blob/main/content/posts/rd-tools/picture/mac/alttab.png?raw=true)

# 粘贴板工具
[raycast](https://www.raycast.com/)  
这个非常强大，其命令记录功能很好用  
鼠标选中对应的功能 Ctrl+c 便可将内容复制到面贴板，配置快捷键唤起 raycast 直接选择粘贴板内容即可
![](https://github.com/BoomChao/boomchao.github.io/blob/main/content/posts/rd-tools/picture/mac/raycast.gif?raw=true)

# 终端配置
无理由推荐 zsh + oh-my-sh !  
1. 下载 [Item2](https://iterm2.com/index.html)   
2. 安装 [homebrew](https://brew.sh/)   
3. 安装zsh  以及 [oh-my-zsh](https://github.com/ohmyzsh/ohmyzsh/wiki/Installing-ZSH)   
- 安装 [zsh-autosuggestions](https://github.com/zsh-users/zsh-autosuggestions/blob/master/INSTALL.md)，自动提示历史命令非常方便     
- 安装命令缩写提示符 [you-should-use](https://github.com/MichaelAquilina/zsh-you-should-use)   
在 .zshrc 里面配置下面这几款插件即可  
``` bash
# Which plugins would you like to load?
# Standard plugins can be found in $ZSH/plugins/
# Custom plugins may be added to $ZSH_CUSTOM/plugins/
# Example format: plugins=(rails git textmate ruby lighthouse)
# Add wisely, as too many plugins slow down shell startup.
plugins=(git golang python zsh-autosuggestions aliases alias-finder zsh-you-should-use docker zsh-syntax-highlighting)  
``` 
4. 安装命令搜索器 [fzf](https://github.com/junegunn/fzf)   
看看我的 shell 配置
![](https://github.com/BoomChao/boomchao.github.io/blob/main/content/posts/rd-tools/picture/mac/zsh.gif?raw=true)


# 研发工具
[He3](https://he3.app/zh/)  
json format、 md5 加密、日期计算等这些，我相信没有 He3 处理不了的小事情，内置非常多的小工具提升效率  
![](https://github.com/BoomChao/boomchao.github.io/blob/main/content/posts/rd-tools/picture/mac/he3.png?raw=true)


# 浏览器插件
## 工作区区分
![](https://github.com/BoomChao/boomchao.github.io/blob/main/content/posts/rd-tools/picture/mac/workspace.png?raw=true)

## 广告拦截
![](https://github.com/BoomChao/boomchao.github.io/blob/main/content/posts/rd-tools/picture/mac/ublock.png?raw=true)

# vscode 插件
让你的文件目录变的更加好看  
![](https://github.com/BoomChao/boomchao.github.io/blob/main/content/posts/rd-tools/picture/mac/code_icon.png?raw=true)

拼写检查（建议安装：帮你避免代码单词拼错的低级错误！）   
![](https://github.com/BoomChao/boomchao.github.io/blob/main/content/posts/rd-tools/picture/mac/code_spell.png?raw=true)

plantUML 画图工具  
![](https://github.com/BoomChao/boomchao.github.io/blob/main/content/posts/rd-tools/picture/mac/plantUML.png?raw=true)
``` bash
@startuml
actor User
participant "Web Application" as WebApp
participant "Backend Service" as Backend
database "Database" as DB

User -> WebApp : Login
WebApp -> Backend : Authenticate(User Credentials)
Backend -> DB : Query User
DB --> Backend : User Data
Backend --> WebApp : Authentication Result
WebApp --> User : Login Success

User -> WebApp : Request Data
WebApp -> Backend : Fetch Data
Backend -> DB : Query Data
DB --> Backend : Data
Backend --> WebApp : Data
WebApp --> User : Display Data
@enduml
```
注意文件后缀为 .wsd 文件  
![](https://github.com/BoomChao/boomchao.github.io/blob/main/content/posts/rd-tools/picture/mac/demo.png?raw=true)

# 画图工具
人家说意图胜千言，平常工作中也会接触到各种各样的画图软件，这里就将自己使用最多的几款介绍下

## 飞书画板
因为平常记笔记什么的都是在飞书画板上，这应该是我用的最多也感觉最舒服的一款画图工具了

## [excalidraw](https://github.com/excalidraw/excalidraw)  
一款手绘的白板,能绘制出很好看的字体

## [mermaid](https://github.com/mermaid-js/mermaid)  


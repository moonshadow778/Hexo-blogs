title: Hexo环境迁移到Mac
date: 2014-12-20 14:13:24
categories: Others
tags:
---
作为时下比较流行的Blog搭建环境，Hexo适合程序员的口味，结合Github，有利于提升逼格。唯一让人感到不方便的地方就是如果想换个机器，需要把本地的一些文件（最重要的就是source目录）同步到新机器的环境里。
Windows的Hexo环境搭建比较简单，我个人的Blog之前一直是在单位的Windows机器上整的，感觉还好。最近想在家里的MacBook上移植一把，遇到了一些坑儿，在本篇文章里试着整理一下。 
##Hexo安装 
1. **Git**安装，这个没什么可说的。建议安装最新版的，Mac自带的版本稍微老一些。完成安装后确保新安装的git的路径在**$PATH**里在旧版本的路径之前；  
2. 为本机Mac生成**SSH Key**并加入Github；  
3. **Node.js**安装。我是从官网直接下的Node.js for Mac OS X的pkg安装包。直接安装即可；   
4.  安装好Node.js自动就带了npm，执行
```
sudo npm install -g hexo
```
如果不使用sudo，会报错。我遇到了此步hexo安装时抛出Warning后hang住的情况，Google了若干解决方案都不适用，直到尝试按照某个[贴子](http://stackoverflow.com/questions/15633029/npm-no-longer-working)修改了/usr/local的权限后
```
sudo chown -R $USER /usr/local
```
安装成功.
##迁移Hexo环境
5.  执行
```
hexo init
```
生成hexo环境，copy备份的source文件覆盖本地source目录。  
6.  安装theme并修改当前hexo工作目录下的_config.yml配置文件。我的Windows和Mac的Hexo版本不同，所以不能直接使用备份好的_config.yml文件直接覆盖。  
7.  修改下载的theme的配置。至此，可以在我的Mac上继续使用Hexo了。  
使用Markdown编写Blog的时候，Mac上推荐一款免费的编辑软件Mou，非常好用。

##多台机器共享    
而为了在不同机器的Hexo环境中共享写好的内容（因为部署上去的hexo生成的blog都是静态的，每次有新的文章都需要重新生成静态页面），基于我使用的light theme，我把以下内容备份到了Github上：  
1. source目录  
2. 根目录_config.yml文件  
3. themes／light／_config.yml  
4. themes/light/layout/_widget/blogroll.ejs  

   
用到的git命令主要有：  
```
git add <dir or file path>
```  
```
git commit -m "<comment>"
```  
```
remote add origin git@github.com:<my github name>/<my hexo git repo>.git
```  
```
git push -u origin master
```   
```
git rm -r -f --cached <dir of file path to remove from stage area>
```   
  


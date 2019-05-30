---
title: 使用hexo和git实现多地更新和配置博客源文件
date: 2018-06-03 13:03:48
categories: hexo
tags: 
- Git
- Hexo
---
使用hexo写博客的一个问题就是源文件都是在本地的，如果换了电脑需要更新博客时就会比较麻烦。目前，觉得比较靠谱的办法就是用github来管理了。

 - 主要的思路是 利用git分支实现
 - hexo生成的静态博客文件默认放在master分支上。
 - hexo的源文件（部署环境文件）可以都放在hexo分支上（可以新创建一个hexo分支），换新电脑时，直接git clone hexo分支

## 一、hexo搭建博客原理

 - hexo帮助把博客发送到github，同时把md文件转换成网页文件。
 - hexo目录下的文件和github上的文件是不同的，public文件夹的文件通过hexo d 上传到github去了，其他的文件则留在本地目录下。
##    二、搭建hexo服务器端电脑设置
### 1.准备工作
首先确保自己已经使用hexo在github搭建好了自己的个人博客。
### 2.对username.github.io仓库新建hexo分支，并克隆
 - 在Github的username.github.io仓库上新建一个xxx分支，并切换到该分支，并在该仓库->Settings->Branches->Default branch中将默认分支设为xxx，save保存；然后将该仓库克隆到本地，进入该username.github.io文件目录。

完成上面步骤后，在当前目录使用Git Bash执行git branch命令查看当前所在分支，应为新建的分支xxx：
### 3.将本地博客的部署文件拷贝进username.github.io文件目录

 - 如题，先将本地博客的部署文件（Hexo目录下的全部文件）全部拷贝进username.github.io文件目录中去。

接下来，进入username.github.io文件目录下，将该目录下的全部文件提交到xxx分支，提交之前需注意：
 - 将themes目录以内中的主题的.git目录删除（如果有），因为一个git仓库中不能包含另一个git仓库，提交主题文件夹会失败。
 - 可能有人会问，删除了themes目录中的.git不就不能git pull更新主题了吗，很简单，需要更新主题时在另一个地方git clone下来该主题的最新版本，然后将内容拷到当前主题目录即可。
### 4.提交hexo分支
 - 执行git add .、git commit -m 'back up hexo files'（引号内容可改）、git push即可将博客的hexo部署环境提交到GitHub个人仓库的xxx分支。

现在可以在GitHub上的username.github.io仓库看到两个分支的差异了。
master分支和xxx分支各自保存着一个版本，master分支用于保存博客静态资源，提供博客页面供人访问；xxx分支用于备份博客部署文件，供自己维护更新，两者在一个GitHub仓库内互不冲突。
## 三、其他任何一台电脑
至此，你的博客已经可以在其他电脑上进行同步的维护和更新了，方法很简单：

 - 将新电脑的生成的ssh key添加到GitHub账户上
 - 在新电脑上克隆username.github.io仓库的xxx分支到本地，此时本地git仓库处于xxx分支
 - 切换到username.github.io目录，执行npm install(由于仓库有一个.gitignore文件，里面默认是忽略掉 node_modules文件夹的，也就是说仓库的hexo分支并没有存储该目录[也不需要]，所以需要install下)
 - 到这里了就可以开始在自己的电脑上写博客了！
 
编辑、撰写文章或其他博客更新改动：
 - 依次执行git add .、git commit -m 'back up hexo files'（引号内容可改）、git push指令，保证xxx分支版本最新
 - 执行hexo d -g指令（在此之前，有时可能需要执行hexo clean），完成后就会发现，最新改动已经更新到master分支了，两个分支互不干扰！
## 四、回到hexo服务器端电脑更新并提交博客
 **注意： 每次换电脑进行博客更新时，不管上次在其他电脑有没有更新，最好先git pull**
 - 按照之前的方法写自己博客

 - 然后将目录切换下username.github.io下，此时需要安装一下npm install， 最后执行hexo g、hexo s、hexo d等命令即可提交成功.

 


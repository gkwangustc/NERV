---
title: 最完美的Hexo多电脑同步方法
date: 2018-05-04 20:08:06
---

原文链接：https://ricky.moe/2017/01/13/hexo-perfect-synchronize/

经常我们有一个场景：需要在公司或者家庭多个电脑完成Hexo的博客撰写和发布工作。这就涉及到Hexo多电脑的同步问题。

<!-- more -->

网上的方案基本上都是多分支方案。也即，在同一个仓库创建两个分支：

1. Hexo分支 – 用来保存所有Hexo的源文件
master分支 – 用来保存Hexo生成的博客文件
2. 在创建GitHub Pages或者Coding Pages时，以master分支为pages分支。

Hexo的deploy指向master分支部署pages，git的管理指向Hexo分支。

但是这里有一个巨大的问题，就是多分支的方案一定是让完整的Hexo源文件暴露在公开的仓库了。这对一些Hexo博客采用的`leancloud阅读次数管理`、`多说评论`等服务的私有secret key也暴露在公开仓库分支了。如果对这些配置的`_config.yml`进行单独管理的话，又不能在另一台电脑直接`git pull`同步，非常的麻烦。

所以**Hexo最完美的多电脑同步方法**是，创建两个仓库：

1. Hexo私有仓库 – 用来保存所有Hexo的源文件
2. master公开仓库 – 用来保存Hexo生成的博客文件

下面来具体讲讲实现方法。

### 建立仓库

这里假设读者已经建立起了Hexo的博客系统了，实现了比方说：

1. 利用hexo d 直接deploy Hexo博客
2. 实现了Hexo的GitHub和Coding国外和国内的同时发布
3. 自行定义了例如next的第三方主题

#### 创建私有仓库
注册一个Coding账号，然后创建一个私有项目，名称为Hexo-my

#### 建立本地git仓库
进入你现有的Hexo文件夹，删除第三方主题的git配置，如对`next`主题

```
rm -fr ./themes/next/.git/
```

然后建立本地的git仓库

```
git init
```

创建一个`.gitignore`文件，并放在Hexo的根目录，内容为：

```
.DS_Store
Thumbs.db
db.json
*.log
node_modules/
public/
.deploy*/ 
```

push到私有仓库

```
git remote add origin https://git.coding.net/<yourname>/Hexo-my.git
git add .
git commit -m "my first private hexo"
git push -u origin master
```

至此，就完成了本地Hexo源码的全备份

### 在另一台电脑进行Hexo写作
上面已经完成了Hexo的全备份，那么如果在另一台电脑进行Hexo编辑呢。
当然首先你也要完成`node`/`npm`/`hexo`/`git`等环境的搭建和配置。

#### Hexo拉取
```
git clone https://git.coding.net/<yourname>/Hexo-my.git
```

这样你就拥有了你的所有Hexo源文件

#### Hexo编写和发布
尽管拉取下来了，还需要建立一下Hexo的环境，这里需要格外注意的一点是：
千万不要用`hexo init`命令。原因是当前目录已经建立了git仓库环境, `hexo init`会覆盖到当前的git环境，重建一个新的，这样和我们的私有Hexo源码仓库脱离了联系。

正确的做法是：

```
npm install
```

因为`package.json`里面已经保存了`hexo`的必备资源包信息，`npm install`后Hexo环境就建立起来了。

接下来就进行正常的编写和发布就好。
本地预览的命令还是：
```
hexo g
hexo s
```
Hexo的发布命令是`hexo d`。

最后执行`git status`把更改的新文件`git add`和`git commit`，最后`git push`到私有仓库，又会完成Hexo源码仓库的同步。

#### Hexo仓库更新
下次进行Hexo仓库拉取时执行：

```
git fetch --all #将git上所有文件拉取到本地
git reset --hard origin/master  #强制将本地内容指向刚刚同步git云端内容
```
reset 对所拉取的文件不做任何处理，此处不用 pull 是因为本地尚有许多文件，使用 pull 会有一些版本冲突，解决起来也麻烦，而本地的文件都是初始化生成的文件，较拉取的库里面的文件而言基本无用，所以直接丢弃。

END
从此，世界是如此的美好。



---
title: 笔记迁移到云服务配合Github workflow自动部署
date: 2022-07-18
categories:
  - 技术杂谈
    笔记
tags:
  - ci/cd
  - github action
---

# Github workflow配合云服务器自建一个笔记服务

## 前言

> 前半部分主要是记录选型的一些想法和考虑，干货可以直接看第三小节

> 整这个主要是学习的目的，并不是说自建的笔记服务比起已经稳定的商业化笔记软件有多大优势，就是瞎折腾

> 如果有任何错误之处，感谢您的指出~ :nail_care:

自己搭一个笔记服务的起因是因为发现`wolai`不开通VIP没办法递归导出目录，只能一个一个导出笔记，考虑到如果迁移笔记的话，以后会造成很大的麻烦。

考虑到对我个人而言笔记的使用场景就是记录一些`技术相关的知识点`、`菜谱`等，但是又懒得写成博客的内容。所以干脆全部丢在服务器上跑。

### 方案选择

接下来就是分析我对笔记的使用场景和需求：

1. 单篇文章为最小单位

2. 可以通过分类或标签归类

3. 由于是总结类型的内容，仅在电脑编辑，手机上使用其他软件做备忘或者日记，不考虑移动端编辑的场景

4. 可以随时查看
5. 拒绝公开访问

#### github page

本来打算通过github page托管项目page的方式来部署(当时也没考虑访问权限的事)，结果发现因为我的仓库是`私有的`(笔记肯定不能乱传啦),github提示我需要公开仓库或者打钱，所以就pass了这条选项 

> Upgrade or make this repository public to enable Pages

#### gitbook

gitbook之前记录刷题的时候有用过，所以github page流产以后尝试了gitbook.

然后因为gitbook需要自己写summary加上政治倾向过于明显，所以就放弃了gitbook

####  github workflow+ 云服务器

最初我的博客其实就是挂在薅来的腾讯云服务器上，通过jenkins做的ci，后来跟朋友耍七日杀要建服重装了服务器，就一直没往云服务器丢东西了。

这次考虑了前两手方案都会有缺陷，加上平时也没怎么整过github workflow，刚好趁这个机会练练手。

### 写作环境

作为一个程序猿写文章最顺手的还是markdown了，但是个人以为vscode的markdown编辑体验真的非常一般，写写README还行，写文章的话体验就不是那么流畅。尤其我对侧边预览真的是够够的了。

### 实施方案

由于博客的hexo是现成的，所以就copy了一份，把posts里的文件全部删掉后期替换成笔记的md内容就可以直接编译了,所以笔记的迁移和实现还是比较easy。对github action + github page托管博客感兴趣的也可以看[github action配合hexo持续集成博客](https://blog.sawyersven.xyz/2022/07/16/git/github%20action%E9%85%8D%E5%90%88hexo%E6%8C%81%E7%BB%AD%E9%9B%86%E6%88%90/)这篇文章

#### 文章管理

既然使用Typora直出，我希望文件目录只有markdown文件，所以选择了将Hexo和文章的目录拆成两个repo来管理(也可以用一个repo的两个分支来管理)。

> 因为我的实现方案是两个repo，所以下文都会以两个repo为出发点，`note`repo用来存放_posts的内容，`note-hexo`用来存放hexo的配置，两个repo用git submodule关联。

#### CI

思路是: `note`每次push的时候触发workflow,调用github api触发`note-hexo`的 workflow; `note-hexo`的workflow主要实现拉取`note`的内容然后生成静态文件。然后想办法同步到云服务器即可。

#### 文件同步到云服务器

整个过程中我尝试了两种方案：

##### 通过ssh上传到云服务器

`note-hexo`构建完成后，通过ssh的方式将打包后的public文件夹放入云服务器nginx指定的note服务的文件目录.

通过账号密码或者私钥，ssh到服务器然后上传文件到nginx的指定目录即可。

但是这种方式需要将私钥或者服务器账号密码上传到github。虽然action secrets不会明文展示，但是把服务器的“大门钥匙”放在github上总归是让人有点膈应(有人把root账户的账号密码放在了github)，所以这个方案pass。

并且如果你是国内云服务商的话，每次部署代码，大概率你的服务商会疯狂的给你报警异常登录。

##### 每次构建后通过webhook触发服务器的拉取

服务器配置webhook执行构建脚本，`note-hexo`action打包静态资源并push到gh-pages分支，然后触发webhook，服务器直接拉取`note-hexo@gh-pages`的代码到nginx配置的网站目录。

这种方案不会把安全信息上传到github，并且通过对webhook加basic auth可以避免出现恶意请求。

## 创建和关联Hexo和Note仓库

### Hexo

首先，安装hexo-cli

```shell
yarn global add hexo-cli -g
```

然后新建Hexo文件夹

```shell
hexo init note-hexo # 最后的名字随意
cd note-hexo
yarn install
```

然后根据Hexo[官方文档](https://hexo.io/zh-cn/docs/setup)配置_config.yml的内容，安装自己感兴趣的[主题](https://hexo.io/themes/),

新建完成以后，文件的目录结构应该如下：

```
.
├── _config.yml
├── package.json
├── scaffolds
├── source
|   ├── _drafts
|   └── _posts
└── themes
```

然后我们将_posts文件夹直接删掉，新建一个github的同名repo

![image-20220718224632170](C:/Users/Administrator/AppData/Roaming/Typora/typora-user-images/image-20220718224632170.png)

因为我们的笔记不打算公开，所以repo可以选择Private.

将note-hexo的内容push到远端

```shell
git add .
git commit -m "first commit"
git branch -M main
git remote origin git@github.com:<git-username>/note-hexo.git
git push -u origin main
```

恭喜你，第一步已经完成啦！

---



### note

在任何地方新建一个note目录(也可以直接建成\_posts目录，仓库也叫\_posts，这样就可以省去改名的操作了)

```shell
mkdir _posts
echo Hello world > hello.md
```

这时我们的_posts目录下有一个hello.md文件，内容是Hello world

同样的新建一个github的同名repo

![image-20220718225257292](C:/Users/Administrator/AppData/Roaming/Typora/typora-user-images/image-20220718225257292.png)

将内容push到远端，这部分代码跟上边一样，就不赘述了。

---

### 使用git submodule关联两个仓库

这个时候我们的书写仓库和发布仓库就完成了，然后使用git submodule将两个仓库关联起来

```shell
cd note-hexo/source

git submodule add git@github.com:xxx/_posts.git  # 这里的分支就是我们的远端分支

```

执行完这段后就会发现note-hexo的source的目录变成了这样

```
.
├── source
|   ├── _drafts
|   └── _posts
|   |	└── hello.md
.
```

并且根目录下多出了一个`.gitmodules`的文件，这个文件里记录了submodule的path和url

```
[submodule "source/_posts"]
	path = source/_posts
	url = git@github.com:SawyerSven/_posts.git

```

使用submodule status命令可以查看submodule指向的commit id和分支

```shell
> git submodule status 
> +42c6bef05185d9e4de95739930b1e452c9cbd0a2 source/_posts (remotes/origin/HEAD)
```

关于submodule的内容，感兴趣的可以[点击](https://www.jianshu.com/p/2d74a6f41d07)了解。

> Congratulations!:tada:,你已经完成了项目的初始化工作~

## 设置git workflow

我们自动化工作流程应当如下：

![未命名白板 (4)](C:/Users/Administrator/Downloads/%E6%9C%AA%E5%91%BD%E5%90%8D%E7%99%BD%E6%9D%BF%20(4).jpg)

按照上边的流程图，我们大致分解为五步：

1. `_post` repo 每次push代码后，触发`note-hexo`的action
2. 在`note-hexo`的action中，我们同步`_posts`的内容到source/_posts中
3. Hexo generate将Hexo打包成静态html文件
4. 为了保存静态文件，将打包后的文件发布到`note-hexo`的`gh-pages`中
5. 同步到云服务器

思路整理好以后，代码反而成最简单的部分:



### _post的action配置

```shell
cd _post
mkdir -p ./github/workflows  # 创建git workflows目录
touch ./github/workflows/trigger-note-build.yml  #创建文件
```

编辑`trigger-note-build.yml`文件：

```yaml
name: Trigger Note Build

on:
  push:
    branches:
      - main
      - master # 可以不要
      
jobs: 
  trigger_note_build:
  	runs-on: ubuntu-latest
  	
  	steps:
  	  - name: Trigger note-hexo workflow
  	    uses: satak/webrequest-action@master  # 这里可以替换成任何request的库，也可以用curl.我懒得写所以随便找了个库
  	    with:
  	      url: https://api.github.com/repos/<OWNER>/<REPO>/actions/workflows/<WORKFLOW_ID>/dispatches # 大写单词均为变量
  	      methos: POST
  	      payload: '{"ref":"main"}'
  	      headers: '{"Accept":"application/vnd.github+json","Authorization":"token ${{secrets.GIT_TOKEN}}"}'
```

这段其实比较简单，主要在发送的trigger请求里:

> 记得将下边的变量改为你自己的仓库相关的信息哦~

OWNER: 仓库拥有者的用户名，即你的github用户名

REPO: 仓库的名字

WORKFLOW_ID： 工作流的ID，也可以是工作流的名字

payload里的ref是一个**必传参数**，工作流的git引用，可以是一个分支或者tag的name

${{secrets.GIT_TOKEN}}是我们自定义的一个secret变量，马上会讲到

关于这个请求的文档可以看[这里](https://docs.github.com/cn/rest/actions/workflows#create-a-workflow-dispatch-event)

### Github Action 的secrets上下文

> `secrets` 上下文包含可用于工作流程运行的机密的名称和值。 `secrets` 上下文不可用于复合操作。

简单的理解就是secrets上下文就是由于github仓库通常是public的，所以workflow文件也是可以被别人看到的，所以有一些*敏感的内容，不想被其他人看到的，设置为secrets变量就可以避免在workflow文件中明文传输*了。

#### 创建github access token

access token是我们调用`Github Api`的令牌，有了它我们才可以通过api操作我们的仓库

我们首先在[这里](https://github.com/settings/tokens)，选择`Generate new token`

![image-20220719000606312](C:/Users/Administrator/AppData/Roaming/Typora/typora-user-images/image-20220719000606312.png)

在打开的新界面中，输入一个**自己方便区分**的名字,然后勾选workflow的权限即可

因为这个token只有我们在workflow中使用，不会用到其他地方，过期时间选择`No expiration`即可。

![image-20220719000827786](C:/Users/Administrator/AppData/Roaming/Typora/typora-user-images/image-20220719000827786.png)

> 生成成功以后马上复制生成的token，如果忘了没复制刷新过页面后就再也看不到了，这个时候就只能删掉这个token再重新新建一个token了

#### 创建repo的secret

进入_post的仓库，选择`settings`，然后选择 `Security`-`Actions`-`New repository secret`

![image-20220719001237927](C:/Users/Administrator/AppData/Roaming/Typora/typora-user-images/image-20220719001237927.png)

![image-20220719001257429](C:/Users/Administrator/AppData/Roaming/Typora/typora-user-images/image-20220719001257429.png)

Name中填GIT_TOKEN (这个就是我们在workflow文件中定义的secret.GIT_TOKEN)

Value中复制我们刚才copy的access token  (如果你没有复制的话，恭喜你 ，可以重新生成一个access token了:cake:)

到此，我们**对_posts这个仓库的所有操作都结束了，你已经完成了30%的工作啦** :flags:

### note-hexo的action配置

> 这部分我不知道有没有更好的办法，希望大佬们可以不吝赐教，尤其是设置ssh私钥那一步有没有替代方案

现在，我们忘掉_posts,将上下文切换到`note-hexo`中来，开始为`note-hexo`写workflow文件

同样的，我们在在根目录下创建一个`.github/workflow/main.yml`的文件，内容如下：

```yaml
name: Hexo Note CI

on: workflow_dispatch # 我们期望更新了_posts再重新构建，所以这里仅使用workflow_dispatch的方式触发

jobs:
  build:
    runs-on: ubuntu-latest  

    steps:
      - name: Checkout Respository master branch  # 切换到当前仓库的默认分支中
        uses: actions/checkout@master

      - name: Setup Node.js 14.x    # 因为构建hexo需要Node环境，所以我们设置node环境
        uses: actions/setup-node@master
        with:
          node-version: '14.17.3'

      - name: Setup Deploy Private Key # 为了拉取submodule，需要设置私钥
        env:
          HEXO_DEPLOY_PRIVATE_KEY: ${{secrets.HEXO_DEPLOY_PRIVATE_KEY}}
        run: |
          mkdir -p ~/.ssh/
          echo "$HEXO_DEPLOY_PRIVATE_KEY" > ~/.ssh/id_rsa
          chmod 600 ~/.ssh/id_rsa
          ssh-keyscan github.com >> ~/.ssh/know_hosts

      - name: Get Submodule  # 拉取子模块, --remote是为了拉取到最新的远端修改,不能忽略
        run: |
          git submodule init
          git submodule update --remote  

      - name: Setup Hexo Dependencies # 安装依赖 (可以省略下文的第一步,把BuildHexo的hexo命令改为package.json中的对应scripts)
        run: |
          npm install hexo-cli -g 
          npm install

      - name: Build Hexo # 生成静态文件
        run: |
          hexo clean
          hexo generate

      - name: Deploy to Server # 将生成的静态文件推送到gh-pages分支
        uses: jamesives/github-pages-deploy-action@v4.3.4
        with:
          folder: public
          branch: gh-pages
          
         
```



有了_posts的配置经验，相信上边的配置文件对你来说已经理解起来肯定很easy了.

我们就只看一下特殊的这一块：

```yaml
  - name: Setup Deploy Private Key # 为了拉取submodule，需要设置私钥
    env:
      HEXO_DEPLOY_PRIVATE_KEY: ${{secrets.HEXO_DEPLOY_PRIVATE_KEY}}
    run: |
      mkdir -p ~/.ssh/
      echo "$HEXO_DEPLOY_PRIVATE_KEY" > ~/.ssh/id_rsa
      chmod 600 ~/.ssh/id_rsa
      ssh-keyscan github.com >> ~/.ssh/know_hosts
```

这部分内容在我的[Github Action配合Hexo持续集成博客](https://blog.sawyersven.xyz/2022/07/16/git/github%20action%E9%85%8D%E5%90%88hexo%E6%8C%81%E7%BB%AD%E9%9B%86%E6%88%90/)里有讲解，感兴趣的可以参考。

我们马上会生成一对密钥对，然后把私钥配置在secrect中,公钥放在github中。



#### 配置密钥对

使用[ssh-keygen](https://www.ssh.com/academy/ssh/keygen)生成一对密钥对(强烈不推荐使用已经在使用的密钥对,请生成一对仅用本仓库的密钥对)

将生成的`公钥`(带pub的那个)复制到[这里](https://github.com/settings/ssh/new)，Title依旧起一个方便自己区分的即可

![image-20220719004416249](C:/Users/Administrator/AppData/Roaming/Typora/typora-user-images/image-20220719004416249.png)

将`私钥`(不带pub的那个) 粘贴到`note-hexo`的`secrets`-`actions`里:

![image-20220719004707829](C:/Users/Administrator/AppData/Roaming/Typora/typora-user-images/image-20220719004707829.png)

:tada: Congratulations, 你已经完成了本文的70%内容，现在先放下要继续写代码的心情，来看看我们目前的进展吧



## 阶段成果

我们修改一下`_posts`目录下的`hello.md`文件，然后把它push到远端的main分支~

然后我们打开github的_posts仓库，点击`actions`，应该会看到一条新的workflow,类似下图：

![image-20220719005535054](C:/Users/Administrator/AppData/Roaming/Typora/typora-user-images/image-20220719005535054.png)

当这调workflow执行完毕后，我们去`note-hexo`中查看`actions`:

![image-20220719005649309](C:/Users/Administrator/AppData/Roaming/Typora/typora-user-images/image-20220719005649309.png)

会看到有workflow也在执行，等到这条workflow执行完毕后，查看`note-hexo`的分支，会发现多了一个gh-pages分支，里面就是打包后的Hexo静态资源

![image-20220719005931507](C:/Users/Administrator/AppData/Roaming/Typora/typora-user-images/image-20220719005931507.png)

## Congratulations

截止目前，你已经拥有了一个可以自动部署到gh-pages的笔记系统，如果你的仓库是public，现在就可以选择将它作为项目page托管到github，然后就可以通过`xxx.github.io/hexo-note`访问了~



## 后续

由于文章篇幅原因，剩下的部分写在下篇文章中吧，剩余内容包括：

- 通过webhook同步gh-pages的文件到服务器
- Nginx Basic Auth限制webhook的调用和note的访问
- 剩下的想到哪算哪吧..



好久不写文章，有任何建议都可以提; 技术菜鸡，有任何错误的地方欢迎大佬指正;



## 相关文档

[深入理解git submodules]()

[Github REST API Doc](https://docs.github.com/cn/rest/actions/workflow-runs#re-run-a-job-from-a-workflow-run)

[Github Action](https://docs.github.com/cn/actions)

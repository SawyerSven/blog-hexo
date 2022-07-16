---
title: github action配合hexo持续集成博客
---

# Github Action配合Hexo持续集成博客

通过Hexo写博客，根据Hexo文档安装hexo并且找一个自己喜欢的主题，部署在GitPage上，节省我们自己的服务器资源

## 新建Github仓库

新建github仓库，如`hexo-blog` ，仓库如果要部署gitpage不能选择private。

## 新建Hexo目录 

```bash
hexo init hexo-blog
```

正常配置hexo和hexo主题并开始写作，写作完成后:

> 如果你的主题文件夹theme中有.git文件夹，请将其删除后再push代码

```shell
git init
git add .
git commit -m "first commit"
git branch -M main
git remote add origin git@github.com:xxxx/hexo-blog (xxx是你的github用户名)
git push -u origin main
```



## 新建Git Page仓库

在Github中新建一个仓库,仓库名为:

```
<你的Git用户名>.github.io    //用户名

如: sawyersven.github.io
```

至此准备工作完成，`hexo-blog`仓库存放博客的源码，xxx.github.io仓库存放打包生成的静态资源，下边开始配置CI

## 配置hexo deploy

添加hexo的git deploy依赖：

```shell
npm install hexo-deployer-git   / yarn add hexo-deployer-git
```

在hexo-blog根目录的`_config.yml`文件中找到deploy字段并添加:

type为git(必须是git)

repo是要发布的repo的地址

```yaml
...

deploy:
	type: 'git'
	repo: 'git@github.com:<gitusername>/<gitusername>.github.io.git'
...
```

## 配置Github Action

GithubAction的[文档](https://docs.github.com/cn/actions)

### 配置workflows

在`hexo-blog`项目根目录下创建一个`.github\workflows`结构的目录，然后新建一个yml文件:

```bash
cd path/hexo-blog
mkdir -p .github/workflows
cd ./github/workflows 
touch main.yml
```

然后我们编辑main.yml:

```yaml
name: Deploy  #随便取

on:
  push:
  	branches:			# 当main或者master分支被push到远端仓库的时候，触发ci
  		- main			
  		- master
 jobs:
  build:
    runs-on: ubuntu-latest			# 在ubuntu下执行后续的操作

    steps:
      - name: Checkout Respository master branch		#actions/checkout checkout到$GITHUB_WORKSPACE下的repo  默认为当前repo
        uses: actions/checkout@master

      - name: Setup Node.js 14.x	# 安装node14
        uses: actions/setup-node@master
        with:
          node-version: '14.17.3'

      - name: Cache node modules  # 缓存依赖增加工作流的执行速度
        uses: actions/cache@v1
        id: cache
        with:
          path: node_modules
          key: ${{ runner.os }}-node-${{ hashFiles('**/[package-lock|yarn.lock].json') }}  # 这里根据用的包管理工具切换lock文件，hashFile会为匹配的文件计算SHA-256哈希值
          restore-keys: |
            ${{runner.os}}-node-

      - name: Setup Hexo Dependencies # 安装hexo-cli 并且安装依赖
        run: |
          npm install hexo-cli -g
          npm install

      - name: Setup Deploy Private Key  # 给ubuntu环境添加ssh的私钥,先按这里填，后边单独讲
        env:
          HEXO_DEPLOY_PRIVATE_KEY: ${{secrets.HEXO_DEPLOY_PRI}}
        run: |
          mkdir -p ~/.ssh/
          echo "$HEXO_DEPLOY_PRIVATE_KEY" > ~/.ssh/id_rsa
          chmod 600 ~/.ssh/id_rsa
          ssh-keyscan github.com >> ~/.ssh/know_hosts

      - name: Setup Git Infomation   # hexo deploy必须有git user.name和git user.email才能运行
        run: |
          git config --global user.name "sawyersven"
          git config --global user.email "kormondor@gmail.com"

      - name: Deploy Hexo
        run: |
          hexo clean
          hexo generate
          hexo deploy


 	
```



## 配置SSH key

因为是，在`hexo-blog`的仓库中提交代码然后构建到了`xxx.github.io`的仓库中，所以需要把SSH key配置在执行action的ubuntu环境中

首先在命令行中执行：

> 设置filename的时候建议设置为github_id_rsa或者其他名称，以便区分

```bash
ssh-keygen   # 生成的key默认在~/.ssh/github_id_rsa,windows在C:/User/<username>/.ssh/github_id_rsa
```



然后将公钥上传到github的ssh key中：[点我直达配置页](https://github.com/settings/keys)

选择`New SSH key`

title : 任意填

key: 用记事本或编辑器打开`github_id_rsa.pub`,将里面的文本粘贴进去.



## 配置Action Secrets

由于workflow的文件是在代码仓库中，脚本中的内容是对仓库可见的，如果我们将一些隐私内容直接明文写在文件中，会被泄露。所以github提供了Action secrect来帮我们存放敏感信息。在Actions中通过$\{{secrets.<keyword>\}}即可获取

进入`blog-hexo`的repo中，点击`settings`，然后在侧边栏选择`Secrets`-`Actions`

然后选择`New repository secret`

name中填写HEXO_DEPLOY_PRI

value将`github_id_rsa`的内容复制后粘贴进去即可



至此github Action的配置就完成了，可以尝试更新`hexo-blog`的代码并push到github，然后观察Action中job的执行情况.

然后访问xxx.github.io即可查看效果。

> 这里不建议将本机正在使用的ssh密钥对上传到github中，尤其是私钥.

> 到这里配置就完成了，后文是一些知识点的补充，可选择阅读



## 补充阅读

### main.yml中的Setup Deploy Private Key

```yaml
  - name: Setup Deploy Private Key  # 给ubuntu环境添加ssh的私钥,先按这里填，后边单独讲
        env:
          HEXO_DEPLOY_PRIVATE_KEY: ${{secrets.HEXO_DEPLOY_PRI}}  # 设置环境变量
        run: |
          mkdir -p ~/.ssh/    
          echo "$HEXO_DEPLOY_PRIVATE_KEY" > ~/.ssh/id_rsa
          chmod 600 ~/.ssh/id_rsa
          ssh-keyscan github.com >> ~/.ssh/know_hosts
```

#### 拆分理解

```shell
mkdir -p ~/.ssh/
```

创建.ssh文件夹

```shell
 echo "$HEXO_DEPLOY_PRIVATE_KEY" > ~/.ssh/id_rsa
```

读取环境变量$HEXO_DEPLOY_PRIVATE_KEY的值，并将其写入id_rsa中 , 其实这步操作约等于我们在本地生成ssh key并将公钥上传到github的ssh key中,只是顺序对调了而已。

```shell
chmod 600 ~/.ssh/id_rsa
```

设置文件权限为拥有者可读写，其他人不可读写执行

```shell
ssh-keyscan github.com >> ~/.ssh/know_hosts
```

收集github.com的公钥并将其写入know_hosts，避免被openSSH警告

### 将Gitpage解析到自己的域名

在购买域名的服务商中添加新的CNAME解析，解析值填为:xxx.github.io (你的gitpage的访问地址)

进入gitpage的仓库,并点击settings

![image-20220715224453906](https://s2.loli.net/2022/07/15/6uzl3wtkZcVgCiS.png)

在custom domain中填写刚才解析的地址，github会校验域名的解析是否是gitpage,如果是就会successful.

这个时候访问你解析的地址，就可以看到你的Github Page被解析过去了。

### CNAME文件

发现每次提交以后设置的Customer domain会被清除掉

所以使用CNAME文件维护

在hexo-blog的source文件夹下创建一个CNAM的文件,内容如下：

```bash

blog.sawyersven.xyz  # 只有一行代码，就是你要解析的自定义域名

```

然后每次deploy后就不用手动调整custom domain了
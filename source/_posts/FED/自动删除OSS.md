---
title: 自动更新并删除过期的OSS静态资源(上)
date: 2022-01-05 15:00:29
tags: 
  - OSS
  - CI/CD
  - 常用脚本
---

使用了OSS托管前端静态资源的项目，在每次构建之后通过脚本自动上传静态资源到OSS中，避免人工操作可能出现的低级失误和节省开发人员精力


# 自动上传OSS资源

首先，需要实现一个OSS上传的功能,使用阿里云官方的npm包-[ali-oss](https://github.com/ali-sdk/ali-oss)

## 自动上传脚本

```js
/* eslint-disable no-console */
const path = require('path')
const fs = require('fs')
const slash = require('slash')
const OSS = require('ali-oss')

const formattedVersion = require('../package').version.replace(/\./g, '-')

const ossConfig = require('./config').ossConfig

const buildPath = path.join(__dirname, '../.nuxt/dist/client')

function AliOss (buildPath, config = ossConfig) {
  const client = new OSS(config)
  const targetPath = `${formattedVersion}/static`
  // 上传
  function uploadResource () {
    const filesArray = fileDisplay(buildPath, [])
    const length = filesArray.length
    for (let i = 0; i < length; i++) {
      console.log('Oss client put file progress: ', filesArray[i], Math.floor((i + 1) / length * 100), '%')
      client.put(`${targetPath}${slash(filesArray[i].replace(buildPath, ''))}`, filesArray[i])
    }
  }
  // 递归所有的file
  function fileDisplay (filePath, filesArray) {
    const files = fs.readdirSync(filePath)

    for (let i = 0; i < files.length; i++) {
      const fileDir = path.join(filePath, files[i])
      if (fs.statSync(fileDir).isFile()) {
        filesArray.push(fileDir)
      } else {
        fileDisplay(fileDir, filesArray)
      }
    }
    return filesArray
  }
  uploadResource()
}

AliOss(buildPath)

```

上边的代码中，`fileDisplay`用来递归查找目标路径下所有的静态资源文件,即`path/to/.nuxt/dist/client`下的所有文件。调用阿里云的put方法，将我们所有的静态资源上传到OSS的pucket中。

为了便于区分版本对应的oss文件,我们通过追加了当前的版本号`formattedVersion`到OSS的文件"路径"中。

`ossConfig`的内容如下：

```js
module.exports = {
  ossConfig: {
    region: 'xxxxxxxxx',
    accessKeyId: 'Your access Key Id',
    accessKeySecret: 'your Access Key Secret',
    bucket: 'bucket name'
  },
}
```

## 避免版本号重复

为了避免版本号重复导致的静态资源被勿删或冲突，在上传前我们校验版本号并且在版本号已存在的情况中终止进程

增加一个validateVersion的函数:

```js
 function validateVersion () {
    return client.listV2({
      delimiter: '/',
      'max-keys': 1000
    }).then((res) => {
      console.log('version'.res)
      const alreadyExistVersion = res.prefixes || []
      const isConflictVersion = alreadyExistVersion.some(item => item.replace('/', '') === formattedVersion)
      if (isConflictVersion) {
        console.log(`已有重复版本${formattedVersion}的OSS文件，请检查package.json中的版本号是否已更新`)
        process.exit(1)
      }
    })
  }
```

在uploadResource前先检查版本号

```js
  validateVersion().then(() => {
    console.log('版本确认正常，开始上传...')
    uploadResource()
  }).catch((error) => { console.log('upload error: ', error) })
```
 

### client.list方法

`client.list()`方法实际上是阿里云OSS的`GetBucket`的封装，可以参考[GetBucket](https://help.aliyun.com/document_detail/187544.html)


**delimiter**: 对Object名字进行分组的字符。

**max-keys**: 指定返回Object的最大数,默认为100，取值 0 < max-keys <= 1000

>从文档可以看出，max-keys的取值最大只能取到1000条,且默认值只取`100`条数据
>未避免版本迭代导致的版本号增加，版本检查失效，所以可以在上传之后通过脚本清理历史数据。

## 自动执行OSS上传脚本

我们不会希望在构建代码之后还要手动执行OSS上传的工作，因为人工操作的不可预期性，比如构建完代码但是忘记上传OSS就发布到了uat甚至是生产环境。所以我们期望在完成构建操作之后自动将代码上传到OSS中

### npm钩子

npm脚本提供了`pre`和`post`两个钩子，分别代表操作前和操作后。相关内容可以参考[阮一峰-npm scripts使用指南](http://www.ruanyifeng.com/blog/2016/10/npm_scripts.html)

我们期望在build之后进行oss的上传操作，所以可以修改package.json:

```json
{
   "scripts": {
    "postbuild": "yarn oss",
    "oss": "node ./scripts/oss.js",
  },
}
```

`oss.js`即上文中的代码

![](/images/oss.png)

这样，每次我们build完代码后即可通过脚本自动将当前版本的静态资源上传到OSS上了
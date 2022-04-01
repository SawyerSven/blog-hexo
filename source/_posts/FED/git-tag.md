---
title: 半自动化维护Git Tag和package.json的版本号
categories: 
  - javascript
tags: 
  - CI/CD
  - 常用脚本

---

项目的静态资源使用CDN加速，在Jenkins中，`build`完成后会自动执行OSS上传的脚本，为了避免多分支/多版本的静态资源互相影响，项目组在每次需要发布版本的时候需要更新package.json的版本号，并且静态资源根据版本号作为路径区分，避免不同版本的静态资源冲突。

> 本文基于Brembo项目，各项目组实现方式可能各有不同，仅供参考。

## 自动更新package版本

### 捕获用户输入

使用`inquirer`提供捕获用户输入的功能

```js
inquirer.prompt([
  {
    type: 'confirm',
    name: 'needUpdateVersion',
    prefix: `${projectInfo}-`,
    suffix: `(当前版本号:${chalk.yellow(currentVersion)})`,
    message: `是否需要更新版本号?`,
    default: false
  },
  {
    type: 'input',
    name: 'version',
    default: getNextDefaultVersion(),
    prefix: `${projectInfo}-`,
    suffix: `(当前版本号:${chalk.yellow(currentVersion)})  : \n`,
    message: `请输入版本号:`,
    when (question) {
      return question.needUpdateVersion
    },
    validate (version) {
      if (!regVersion.test(version)) {
        console.log(chalk.yellow('输入的版本号格式未通过检查,请检查格式(eg:1.0.0,1.1.0)'))
        return false
      }
      return true
    }
  }]
)
```

上述代码中,基于`when`方法第二个object会在一个问题的答案为`true`的时候被激活，`validate`方法会验证用户的输入的版本号格式是否符合规范

```js
// getNextDefaultVersion()
const getNextDefaultVersion = () => {
  const numbers = currentVersion.split('.')  // currentVersion是从package.json读取的version
  const lastNumber = Number(numbers.pop())
  return numbers.concat(lastNumber + 1).join('.')
}
```


执行脚本会提示:


![123](http://47.95.214.156:3001/uploads/big/6a412b4e21f1aceda3caabfa9e81a871.png)



### 处理版本号并提交commit

在inrequirer获取到用户的输入后，将用户输入传递给下一步进行处理：

```js
 if (newVersion && newVersion !== currentVersion) {
    let spinner
    try {
      await command(`yarn version --no-git-tag-version --new-version ${newVersion}`)
      console.log(chalk.green(`\n ${projectName} package.json版本号更新成功,当前版本为:${newVersion}`))
      spinner = ora('git commit中...请等待git hooks执行..').start()
      await command(`git add package.json && git commit -m "chore(package.json): 更新项目版本号为：${newVersion}"`)
    } catch (err) {
      err ? spinner.fail('git执行失败，请手动提交package.json') : spinner.succeed('git commit成功，请直接push代码')
      console.log(chalk.red(err))
      console.log('\n')
      process.exit(1)
    }

function command (cmd, options = {}) {
  return new Promise((resolve, reject) => {
    exec(cmd, options, (err, stdout, stderr) => {
      if (err) {
        reject(err)
      } else {
        console.log(stdout)
        resolve(true)
      }
    })
  })
}
```

由于更新代码的脚本放在了pre-push的git hook中，所以并不是每一次push代码我们都需要进行版本更新，所以第一个问题我们问了是否需要更新版本号.如果不需要更新版本号就退出脚本。

脚本中使用了node.child_process的exec执行git命令。


## 为git打tag

为了便于跟踪版本，可以在每次更新版本的时候同时打上git的tag,便于使用git checkout随时切换到指定版本的代码


### 增加捕获输入

```js
...something // 更新版本号的捕获
inquirer.prompt([
 {
    type: 'confirm',
    name: 'needAddTag',
    message: `${projectInfo}(${chalk.yellow(currentVersion)})-是否为当前版本添加Tag?`,
    default: false,
    when ({ needUpdateVersion }) {
      return needUpdateVersion
    }
  },
  {
    type: 'list',
    name: 'operationType',
    message: '请选择本次更新的类型',
    choices: ['fix', 'feat', 'chore', 'build', 'doc'],
    when (question) {
      return question.needAddTag
    }
  },
  {
    type: 'input',
    name: 'versionDescr',
    message: '请输入版本描述*',
    when (question) {
      return question.needAddTag
    },
    validate (desc) {
      return !!trim(desc)
    }
  }]
)
```

上边的代码中增加了一个确认输入，一个选择输入，一个文本输入。通过确认来确定是否要更新git tag（推荐在需要发版的版本添加git tag而不是每个小版本都添加git tag，有利于git tags
干净整洁）, 更新的类型符合commit type规范即可，版本描述填写本版本主要更新内容

### 处理新的输入

``` js
 if (needAddTag) {
      try {
        spinner.start('正在对本次提交打tag,请耐心等待...')
        await command(`git tag -a v${newVersion} -m "${operationType}-${versionDescr}"`)
        spinner.start('正在push tag，请耐心等待...')
        await command(`git push origin --tags`)
      } catch (err) {
        console.log(err)
        consola.error('git Tag更新失败,请手动同步git tag')
        process.exit(1)
      }
    }
    await command(`git checkout develop`)
    console.log(chalk.green(`\n 校验通过，分支已切换回develop`))
```

如果需要添加tag，对当前版本进行 git tag操作，并且自动push tag到远程仓库.

> 第13行，由于Brembo项目是在release分支做发布，所以为避免发布过后忘记切换到develop分支导致在release做提交，所以在提交后自动切换回develop分支。


## 使用

### 自动执行脚本

可以基于`husky`自动执行本脚本

`husky4`的配置方式:

```json package.json
{
  "husky": {
    "hooks": {
      "pre-push": "yarn [script-file-name]"
    }
  }
}
```

### 单分支发布

如果项目有多个分支，但我们只使用特定的发布分支做发布操作，可以在脚本最前方定义如下，以release分支做发布为例：

```js check-version.js

const getGitBranch = execSync('git name-rev --name-only HEAD', { encoding: 'utf-8' }).replace(regLineBreak, '')

if (!/release/.test(getGitBranch)) {
  consola.warn('仅release分支用于版本更新,check-version脚本不会在其余分支运行，请勿手动修改package.json')
  process.exit(0)
}

```

以上是实现自动更新代码和自动打git tag的主要逻辑，基本可以解决目前项目中的需求.

## 完整代码

```js check-version.js
/* eslint-disable no-console */

const { exec, execSync } = require('child_process')
const inquirer = require('inquirer')
const chalk = require('chalk')
const ora = require('ora')
const consola = require('consola')
const { trim } = require('lodash')
const { name: projectName, version: currentVersion } = require('../package')

const regVersion = /^[1-9]{1}\d*\.\d+\.\d+$/

const regLineBreak = /\s/

const getGitBranch = execSync('git name-rev --name-only HEAD', { encoding: 'utf-8' }).replace(regLineBreak, '')

const projectInfo = `(${chalk.yellowBright(getGitBranch)})`

const getNextDefaultVersion = () => {
  const numbers = currentVersion.split('.')
  const lastNumber = Number(numbers.pop())
  return numbers.concat(lastNumber + 1).join('.')
}

if (!/release/.test(getGitBranch)) {
  consola.warn('仅release分支用于版本更新,check-version脚本不会在其余分支运行，请勿手动修改package.json')
  process.exit(0)
}

console.log('\n')

inquirer.prompt([
  {
    type: 'confirm',
    name: 'needUpdateVersion',
    prefix: `${projectInfo}-`,
    suffix: `(当前版本号:${chalk.yellow(currentVersion)})`,
    message: `是否需要更新版本号?`,
    default: false
  },
  {
    type: 'input',
    name: 'version',
    default: getNextDefaultVersion(),
    prefix: `${projectInfo}-`,
    suffix: `(当前版本号:${chalk.yellow(currentVersion)})  : \n`,
    message: `请输入版本号:`,
    when (question) {
      return question.needUpdateVersion
    },
    validate (version) {
      if (!regVersion.test(version)) {
        console.log(chalk.yellow('输入的版本号格式未通过检查,请检查格式(eg:1.0.0,1.1.0)'))
        return false
      }
      return true
    }
  },
  {
    type: 'confirm',
    name: 'needAddTag',
    message: `${projectInfo}(${chalk.yellow(currentVersion)})-是否为当前版本添加Tag?`,
    default: false,
    when ({ needUpdateVersion }) {
      return needUpdateVersion
    }
  },
  {
    type: 'list',
    name: 'operationType',
    message: '请选择本次更新的类型',
    choices: ['fix', 'feat', 'chore', 'build', 'doc'],
    when (question) {
      return question.needAddTag
    }
  },
  {
    type: 'input',
    name: 'versionDescr',
    message: '请输入版本描述*',
    when (question) {
      return question.needAddTag
    },
    validate (desc) {
      return !!trim(desc)
    }
  }
]).then(async (answers) => {
  const { version: newVersion, versionDescr, operationType, needAddTag } = answers
  if (!answers.needUpdateVersion) { process.exit(0) }
  if (newVersion && newVersion !== currentVersion) {
    let spinner
    try {
      await command(`yarn version --no-git-tag-version --new-version ${newVersion}`)
      console.log(chalk.green(`\n ${projectName} package.json版本号更新成功,当前版本为:${newVersion}`))
      spinner = ora('git commit中...请等待git hooks执行..').start()
      await command(`git add package.json && git commit -m "chore(package.json): 更新项目版本号为：${newVersion}"`)
    } catch (err) {
      err ? spinner.fail('git执行失败，请手动提交package.json') : spinner.succeed('git commit成功，请直接push代码')
      console.log(chalk.red(err))
      console.log('\n')
      process.exit(1)
    }
    if (needAddTag) {
      try {
        spinner.start('正在对本次提交打tag,请耐心等待...')
        await command(`git tag -a v${newVersion} -m "${operationType}-${versionDescr}"`)
        spinner.start('正在push tag，请耐心等待...')
        await command(`git push origin --tags`)
      } catch (err) {
        console.log(err)
        consola.error('git Tag更新失败,请手动同步git tag')
        process.exit(1)
      }
    }
    await command(`git checkout develop`)
    console.log(chalk.green(`\n 校验通过，分支已切换回develop`))
    spinner.stop()
    process.exit(0)
  } else {
    console.log(chalk.green(`本次未修改版本号,version:${newVersion} ! \n`))
  }
})

function command (cmd, options = {}) {
  return new Promise((resolve, reject) => {
    exec(cmd, options, (err, stdout, stderr) => {
      if (err) {
        reject(err)
      } else {
        console.log(stdout)
        resolve(true)
      }
    })
  })
}

```

## 参考

[BremboFront](https://git.baozun.com/brembo/brembo-frontend/-/blob/develop/scripts/check-version.js)
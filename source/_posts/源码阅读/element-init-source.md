---
title: Element-UI初始化组件源码
date: 2022-02-11 11:14:04
tags:
---
## 环境记录


- windows10
- node v14.17.3
- yarn 1.22.4



由于make new执行的target是node build/bin/new.js，所以关注点直接切到new.js文件(其实是公司的windows电脑不支持make命令)


在源码解读过程中，约定执行node new.js的入参为：**node new.js new-comp 新组件**


## new.js


```javascript
if (!process.argv[2]) {
  console.error('[组件名]必填 - Please enter new component name');
  process.exit(1);
}
```


### process.argv
根据nodejs官方文档，process.argv为一个字符串数组,其中：


- 第一个元素为**process.execPath**
- 第二个元素是**正在执行的Javascript文件的路径**
- 其余元素是任何其他命令行参数



如：


```javascript
// test.js
console.log(process.argv)
```


执行


```
node test.js one two=true three:123
```


会输出:


```
[
  'NODE_HOME/node.exe',
  '__dirname/test.js',
  'one',
  'two=true',
  'three:123'
]
```


> process.argv的第一个元素是**process.execPath**所以根据文档描述，execPath属性返回启动node.js进程的可执行文件的绝对路径。



#### 结论


```javascript
// 如果执行node build/bin/new.js时没有传入第三个参数，则代表用户未输入组件名
if (!process.argv[2]) {
  // 打印错误日志
  console.error('[组件名]必填 - Please enter new component name');
  // 退出node进程
  process.exit(1);
}
```


### 依赖及变量


```javascript
// 文件保存
const fileSave = require('file-save'); 
// 将字符串转为大驼峰
const uppercamelcase = require('uppercamelcase'); 
// 传入的组件名
const componentname = process.argv[2];  // test-comp
// 中文名，如果未传入则会用componentname
const chineseName = process.argv[3] || componentname; // 新组建
// 将组件名转为大驼峰的形式
const ComponentName = uppercamelcase(componentname);  // TestComp
// 要保存组件的文件路径
const PackagePath = path.resolve(__dirname, '../../packages', componentname); // PROJECT_PATH/element/packages/new-comp

const Files = [...]  // 要生成的文件内容
```


> 上边依赖的的file-save已经超过7年没有维护了，而且原作者已经删除了代码仓库.. uppercamelcase依赖camelcase。而且自己的代码也只有短短3行，通过camelcase转为小驼峰后再将第一个字符转为大写然后拼接到原字符串的第二位前



### Files内容


Files生成的文件目录较长，拆分开单独看会比较清晰:


#### 生成packages/new-comp/index.js


```javascript
 {
    filename: 'index.js',
    content: 
    `import ${ComponentName} from './src/main';
    /* istanbul ignore next */
    ${ComponentName}.install = function(Vue) {
      Vue.component(${ComponentName}.name, ${ComponentName});
    };

    export default ${ComponentName};`
  },
```


> **istanbul ignore next** 作用是istanbul统计覆盖率的时候忽略install方法。
istanbul是土耳其最大城市伊斯坦布尔的命名，因为土耳其地毯世界文明，而地毯是用来覆盖的



#### 生成packages/new-comp/src/main.vue


```javascript

  {
    filename: 'src/main.vue',
    content: 
    `<template>
        <div class="el-${componentname}"></div>
      </template>

      <script>
      export default {
        name: 'El${ComponentName}'
      };
      </script>`
  },
```


#### 生成不同语言的文档


```javascript
 {
    filename: path.join('../../examples/docs/zh-CN', `${componentname}.md`),
    content: `## ${ComponentName} ${chineseName}`
 },
 {
    filename: path.join('../../examples/docs/en-US', `${componentname}.md`),
    content: `## ${ComponentName}`
  },
  {
    filename: path.join('../../examples/docs/es', `${componentname}.md`),
    content: `## ${ComponentName}`
  },
  {
    filename: path.join('../../examples/docs/fr-FR', `${componentname}.md`),
    content: `## ${ComponentName}`
  },
```


> 文档通过markdown-it转成html后直接展示在页面上,开启本地调试后，可以访问本地文档查看修改后的内容



#### 生成单元测试文件


```javascript
{
    filename: path.join('../../test/unit/specs', `${componentname}.spec.js`),
    content: 
    `import { createTest, destroyVM } from '../util';
     import ${ComponentName} from 'packages/${componentname}';

     describe('${ComponentName}', () => {
       let vm;
       afterEach(() => {
         destroyVM(vm);
       });
 
       it('create', () => {
         vm = createTest(${ComponentName}, true);
         expect(vm.$el).to.exist;
       });
     });
     `
  }
```


#### 生成样式


```javascript
  {
    filename: path.join('../../packages/theme-chalk/src', `${componentname}.scss`),
    content: 
    `@import "mixins/mixins";
     @import "common/var";

     @include b(${componentname}) {
     }`
  },
```


[**@include **](/include )** b() **是sass中mixins的引入方式：查看theme-chalk/src/mixins.scss可以找到b的定义：


```scss
@mixin b($block) {
  $B: $namespace+'-'+$block !global;

  .#{$B} {
    @content;
  }
}
```


其中$namespace在config.scss中被定义为**el-**,**#{}**是一个插值语句，用#{}可以在属性名或选择其中使用变量。


> [**@content **](/content )** **可以的效果有点类似slot,[@include ](/include ) b(new-comp){...attrs},attrs中的内容将被替换到@content的位置(不是非常确定) 



可以理解为生成的CSS文件如下：


```css

.el-new-comp{

}
```


#### 关于mixins中定义的BEM


在看mixins的时候发现其中对BEM的处理很不错，让我学到了之前一直没有想到的用法，在此记录:


```scss
/* BEM
 -------------------------- */
@mixin b($block) {
  $B: $namespace+'-'+$block !global;

  .#{$B} {
    @content;
  }
}

@mixin e($element) {
  $E: $element !global;
  $selector: &;
  $currentSelector: "";
  @each $unit in $element {
    $currentSelector: #{$currentSelector + "." + $B + $element-separator + $unit + ","};
  }

  @if hitAllSpecialNestRule($selector) {
    @at-root {
      #{$selector} {
        #{$currentSelector} {
          @content;
        }
      }
    }
  } @else {
    @at-root {
      #{$currentSelector} {
        @content;
      }
    }
  }
}

@mixin m($modifier) {
  $selector: &;
  $currentSelector: "";
  @each $unit in $modifier {
    $currentSelector: #{$currentSelector + & + $modifier-separator + $unit + ","};
  }

  @at-root {
    #{$currentSelector} {
      @content;
    }
  }
}
```


如果我们在new-comp组件中遵守BEM的规范,定义一个paragraph元素:


```html
<template>
  <div class="el-new-comp">
    <p class="el-new-comp__paragraph">This is new component</p>
    <p class="el-new-comp__paragraph--dark">This is new component,but my color is dark</p>
  </div>
</template>
```


在new-comp.scss中定义样式:


```scss
@import 'mixins/mixins';
@import 'common/var';

@include b(new-comp) {
  @include e(paragraph) {
    font-size: 12px;
    color: #ff0000;
    @include m(dark){
      color:#000;
    }
  }
}
```


new-comp.scss中的内容会被编译成：


```css
.el-new-comp .el-new-comp__paragraph {
  font-size: 12px;
  color: #ff0000;
}
.el-new-comp .el-new-comp__paragraph--dark {
  color: #000;
}
```


在页面查看:


![](https://www.hualigs.cn/image/61a72c7218e14.jpg#crop=0&crop=0&crop=1&crop=1&id=nnA6m&originHeight=353&originWidth=1015&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)


#### 生成d.ts文件


```javascript

 {
    filename: path.join('../../types', `${componentname}.d.ts`),
    content: `import { ElementUIComponent } from './component'

/** ${ComponentName} Component */
export declare class El${ComponentName} extends ElementUIComponent {
}`
  }
```


### 将新组件添加到components.json


```javascript
// 添加到 components.json
const componentsFile = require('../../components.json'); //element/components.json
// 如果components中已经存在同名组件
if (componentsFile[componentname]) { 
  // 控制台打印错误: eg. new-comp 已存在.
  console.error(`${componentname} 已存在.`);
  // 退出进程
  process.exit(1);  // exit方法的code:0为成功，1为失败
}
// componentsFile添加新的属性new-comp,值为new-comp的路径
componentsFile[componentname] = `./packages/${componentname}/index.js`; // ./packages/new-comp/index.js
// 保存序列化后的componentsFile
fileSave(path.join(__dirname, '../../components.json'))
  .write(JSON.stringify(componentsFile, null, '  '), 'utf8')
  .end('\n');
```


### 将新组件的样式文件添加到index.scss


```javascript
// 添加到 index.scss
const sassPath = path.join(__dirname, '../../packages/theme-chalk/src/index.scss'); // 获取index.scss的路径
// 更新index.scss的内容，在文件末尾增加new-comp.scss
const sassImportText = `${fs.readFileSync(sassPath)}@import "./${componentname}.scss";`;
// 重新写入index.scss
fileSave(sassPath)
  .write(sassImportText, 'utf8')
  .end('\n');
```


### 追加新组件的d.ts到element-ui.d.ts


```javascript
const elementTsPath = path.join(__dirname, '../../types/element-ui.d.ts');

// 在element-ui.d.ts文件的最后追加 NewComp的class
let elementTsText = `${fs.readFileSync(elementTsPath)}
/** ${ComponentName} Component */
export class ${ComponentName} extends El${ComponentName} {}`;

// 找到第一个export字符的位置,index即是字符e的下标
const index = elementTsText.indexOf('export') - 1;
// 定义new-comp的import语句
const importString = `import { El${ComponentName} } from './${componentname}'`;
// 将importString插入到export前
elementTsText = elementTsText.slice(0, index) + importString + '\n' + elementTsText.slice(index);
// 重新吸入element-ui.d.ts
fileSave(elementTsPath)
  .write(elementTsText, 'utf8')
  .end('\n');
```


### 创建组件在package的内容


```javascript
// 遍历files
Files.forEach(file => {
  // packagePath 新组件在packages中的目录即：element/packages/new-comp
  // file.filename 文件路径和名称
  // 通过path.join拼接文件路径
  fileSave(path.join(PackagePath, file.filename))
  // 写入对应的文件内容
    .write(file.content, 'utf8')
    .end('\n');
});
```


### 将新组件添加到路由中


```javascript
// 添加到 nav.config.json
//  nav.config.json的路径
const navConfigFile = require('../../examples/nav.config.json');

Object.keys(navConfigFile).forEach(lang => {
  // navConfigFile中每种语言的第5个元素是定义的数组,所以navConfigFile[lang][4].group即插入组件的数组
  let groups = navConfigFile[lang][4].groups;
  // 在最后一个对象的list中插入新组件:即Others分组
  groups[groups.length - 1].list.push({
    path: `/${componentname}`,
    // 中文language下，title会同时显示中英文两种，其他只language只显示组件名
    title: lang === 'zh-CN' && componentname !== chineseName
      ? `${ComponentName} ${chineseName}`
      : ComponentName
  });
});
// 重新写入nav.config.json
fileSave(path.join(__dirname, '../../examples/nav.config.json'))
  .write(JSON.stringify(navConfigFile, null, '  '), 'utf8')
  .end('\n');
```


> 花了部分时间阅读从nav.config到route.config然后生成路由的逻辑，搭配生成文件和生成路由，可以在自己的项目里进一步减少工作量



## 总结


- [x] 复习了node的path和process的部分属性和方法以及之间的区别
- [x] 源码中mixins.scss的部分内容很有启发性，eg.BEM的mixins实现，@content的使用，scss function的使用,when配合$state-prefix的使用等.
- make和makefile由于阅读源码在windows平台，所以没有做过多了解



自动生成文件的内容和阅读前想象的基本差不多，都是通过**预定义模板动态插入内容**及固定位置插入文件的形式实现自动生成模板。之前在项目中研究过基于plop生成新的页面及组件，调整逻辑后更适用于业务层面的项目，但增加了额外依赖和hbs的学习成本。element-ui的new.js感觉更加适合组件库这种同级别组件的开发。


## 疑问


关于new.js中一开始的:
​

```javascript
console.log();
process.on('exit', () => {
  console.log();
});

```
感觉没什么用，不是很明白保留这段代码的理由
## 参考文献


[代码覆盖率工具 Istanbul 入门教程 - 阮一峰](http://www.ruanyifeng.com/blog/2015/06/istanbul.html)


[Sass教程 Sass中文文档 | Sass中文网](https://www.sass.hk/docs/)


[NodeJs 14.x documentation](https://nodejs.org/docs/latest-v14.x/api/index.html)

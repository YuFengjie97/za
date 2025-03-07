## 烦人的命令
`nr` `npm run`的简写,来自`@antfu/ni`
```
"build": "nr js:build && nr ui:build && nr -C test/fixtures/vite build",
```
说明: build命令: npm run js:build 命令, 然后npm run ui:build命令, 然后 -C 来切换到test/fixtures/vite 目录,npm run build命令

## 是如何监测正则
  复写原型上exec方法.
  猜测正则的大部分动作地层都是运行exec

## 原型被复写,但是要保证在目标运行前注册,在哪里注册的
  ```
    "build:doctor": "node --import regex-doctor/register ./node_modules/nuxi/bin/nuxi.mjs build",
  ```
  `node --import`在目标库动作之前注册正则监听模块
  ```json
  {
    "name": "regex-doctor",
    "exports": {
      "./register": {
          "types": "./dist/register.d.ts",
          "import": "./dist/register.mjs",
          "require": "./dist/register.cjs"
        }
    }
  }
  ```
  在`package.json`中指明了模块
  ## 一般使用
  ```
  node --import regex-doctor/register path/to/your/script.js
  ```
  ```
  npx regex-doctor
  ```
  npx运行的是package.json中的bin关键项
  ```
  "bin": "./bin.mjs",
  ```
## 命令
```
"scripts": {
    "build": "nr js:build && nr ui:build && nr -C test/fixtures/vite build",
    "start": "nr js:build && nr -C ui build:doctor && nr -C test/fixtures/vite build && nr dev",
    "dev": "nr -C ui dev",
    "js:build": "unbuild",
    "ui:build": "nr -C ui build",
    "lint": "eslint .",
    "prepublishOnly": "nr build",
    "release": "bumpp && npm publish",
    "test": "vitest",
    "typecheck": "pnpm -r typecheck",
    "prepare": "simple-git-hooks"
  },
```
- "js:build": "unbuild",
  unbuild一个基于rollup的打包库,传说有更好的配置体验
  ```
  // /build/config.ts
  import { defineBuildConfig } from 'unbuild'

  export default defineBuildConfig({
    entries: [
      'src/index',
      'src/register',
      'src/cli',
      'src/dirs',
    ],
    declaration: true,
    clean: true,
    rollup: {
      emitCJS: true,
    },
  })

  ```
- "ui:build": "nr -C ui build",
  切换到ui目录,npm run build
- "dev": "nr -C ui dev",
  运行ui,展示列表
- "start"
  开发这个库,关键的命令吧
  - 用unbuild打包regex-doctor这个库,最后应该是编译到了dist目录
  - 然后切换到ui子项目,通过node --import注册regex-doctor的正则检验,目标检验./node_modules/nuxi/bin/nuxi.mjs 的build动作, build动作就是函数在运行,只要运行过并使用过正则,就会被记录数据

## 正则检测得到的数据存放到了哪里
查看ni目录下server/api/playload.json.ts,发现`new URL('../../.regex-doctor/output.json', import.meta.url),`

存放在此`ui/.regex-doctor/output.json`

## 在一般使用中,最后一步`npx regex-doctor`发生了什么
`npx regex-doctor`执行了包下的`bin.mjs`

## 在打包成库后,npm包是如何指定`npx regex-doctor`是执行`bin.mjs`的
`package.json`中配置了`  "bin": "./bin.mjs",`

## bin.mjs指向了哪里
bin.mjs没有经过打包,而且只有1行命令,`await import('./dist/cli.mjs')`运行了打包后的dist目录下的cli.mjs,而cli.mjs是由`src/cli`打包得到

## src/cli 干了什么
以脚本命令行的方式启动了服务器

## 入口文件在哪里
命令行的入口文件(指 node --import regex-doctor/registor)
脚本的入口文件(index.ts)
分别对应package.json中export关键字中的`./registor` `.`

## regex-doctor/src的完整逻辑
### src/index.ts 
入口文件
- 导出所有类型
- 导出RegexDoctor类from ./doctor
- 导出startRegexDoctor
### src/doctor.ts
RegexDoctor 关键类
- `map = new Map<RegExp, RecordRegexInfo>()`
  public属性,记录正则信息,key为正则, value为对应info信息
- duration
  耗时,这里的耗时是doctor实例在分析所有正则(没有stop之前)耗费时间
- timeStart
  正则分析的开始时间
- constructor
  传入带public关键字的options配置,这样会自动在类上添加public options.(这样比较简洁,一般写法,在类上定义public options, 有默认值或没有,然后在constructor传入options参数,再在constructor内部赋值,写起来确实麻烦)
- 私有方法saveDuration
  控制正则分析时间是否记录, duration timeStart的控制逻辑在此
- start
  saveDuration(true)开始记录开始时间
  hijack, 正则原型exec复写
  [Symbol.dispose],实例离开作用域(无人使用),后自动销毁
- stop
  saveDuration(false)停止记录
  listener删除regexdoctor实例
  hijack模块内全局变量重置restore,但是这个是套娃空函数

## doctor中的类直接对hijack模块中的变量和方法直接调用,看起来像是把hijack模块当作了"实例"使用一样.
这种写法我还从来没用过,模块的使用,我也只导出导入,这种看起来像是"模块的闭包"/"模块的全局变量"

## hijack中的restore是想干什么,最后是个套娃的空函数
hijack函数在运行时对该模块的全局变量_restore赋值.将被改写的的Regex重置为原始Regex,原型也重新绑定为原来的,同时将_restore变为一个空函数

## 对Regex复写有几步
1. globalThis.RegExp = MyRegExp,全局下的regex被改写为myregexp
2. 将regex原型exec复写为自己的exec

## dynimic是如何标记的
正则使用有两种方式,1是构造函数,这种一般来被用作动态生成正则的方式,也就是MyRegExp中表示了dynaimic.2是//正则字面量.是静态的

## 正则原型复写为什么只有exec,正则匹配不是涉及了很多方法吗?
正则的大部分匹配方法,底层都是exec方法

## listener是干嘛的?
字面意思,监听.类型是RegExpDoctor的一个set,存储了不重复的实例对象.
对他的设置(对listener.map的初始化和calls的添加):在劫持正则构造中,对exec原型改写中.


## listener是Set<RegexDoctor>,为什么?难道说在分析正则的过程中,会创建多个regexDoctor实例来分析吗

## 在哪里触发存储(dump)数据的
1. src/register.ts exitHook(), 
2. doctor.stop后 显式调用dump写入文件

## 如何存储
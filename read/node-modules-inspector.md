## 10分钟限制
- 多项目项目
- pnpm run prepare,报错
- 没有贡献文档

## pnpm run prepare报错 'D:\workspace\node-modules-inspector\packages\node-modules-inspector\runtime\webcontainer-server.mjs'. 看样子,是缺失一个运行时的环境,根据mjs后缀,以及本身目录不存在,猜测为打包的产物,

## 验证
执行完build命令,产生了runtime/webxxxx.mjs,再执行prepare,成功
但是dev命令失败,提示src目录不存在,猜测,命令不应该在顶层运行,
切换到子项目目录执行dev,依然失败

## pnpm run dev, 提示src目录不存在
1. "dev": "pnpm -C packages/node-modules-inspector run dev",根目录的dev命令,-C切换到子项目目录,运行了子项目的dev命令
2. "dev": "pnpm run -r stub && cd src && nuxi dev",子项目dev命令,-r是什么? -r是多包项目中,递归的运行命令(所有的子包都运行)但是这里子项目里没有再有其他子子项目. 
3. 删除掉cd src可以启动,但是只有一个首页,启动了什么东西
4. "stub": "unbuild --stub",--stub是什么, 创建类型存根文件占位,防止引用失败,并加速

## 多项目项目中,在顶层定义的命令,是怎么穿透到packages中的子项目中去的

## dev环境是什么样子的
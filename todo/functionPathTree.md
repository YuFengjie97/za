## 我想要什么?
- 树结构
- 函数的树
- 在库中定义的函数
- 可能会引用到node_modules中的api,设置层级来限制

## 函数信息包括哪些
- 函数名
- 函数位置(定义位置? 调用位置?)
- 上级函数

## 怎么获取这些信息
- 劫持函数,重写Function.portotype类似的方法
  - 普通函数声明
  - 函数变量声明
  - 箭头函数
  - 普通调用
  - call,apply,bind调用
  - 类实际上也是函数,会不会也走底层这些
- 手动解析js/ts模块文件
  - 一个入口文件
  - 模块规范(es/cjs)解析import/require语句
  - 怎么查找入口函数,和引用函数,正则匹配整个js文件?
  - 效率低下

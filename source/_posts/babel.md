---
layout: page
title: babel
date: 2018-06-30 23:04:36
tags: 构建工具
---

### 基本使用与配置方式

#### .babelrc 配置文件 

```json
{
  // 表示一组 plugin  
  "presets": [ "es2015" ],
  "plugins": []
}
```

---

#### 转换命令

转换 文件 -> 文件 `$ babel example.js -o compiled.js`

转换 目录 -> 目录 `$ babel src -d lib`

---

#### 预设 preset

就是一系列 plugins 的集合

官方 preset:

1. babel-preset-es2015	// 已被弃用

2. `babel-preset-env` 包含了 babel-preset-es2015 ~ babel-preset-es2017

可以根据你想要支持的浏览器版本设置 polyfills 支持的内容，使打包的时候更小

```json
{
    "presets": [[
    "env",
    {
        "targets": {
            // The % refers to the global coverage of users from browserslist
            "browsers": [ ">0.25%", "not ie 11", "not op_mini all"]
        }
    }
    ]]
}
```

3. babel-polyfill	// 需要单独安装

import "babel-polyfill"	// 只需在根文件引入一次

4. babel-preset-react	// 转换 react

5. babel-preset-stage-0 ~ 4	// 最全的最新提案，包含 1~4，4 代表审核通过，所以没必要设置 4

---

#### 基于环境自定义 babel

```json
 {
    "presets": ["env"],
    "plugins": [],
+   "dev": {
+     "development": {
+       "plugins": [...]
+     },
+     "pro": {
+       "plugins": [...]
+     }
    }
  }
```

可以根据不同的环境变量使用不同的插件

1. 当前环境可以使用 process.env.BABEL_ENV 来获得

2. 如果 BABEL_ENV 不可用，将会替换成 NODE_ENV

3. 缺省值是 "development"

---

#### 关于 eslint

配置一些规则，检查代码格式的工具

```json
{
  "parser": "babel-eslint",	// 不填写，默认是官方的 解析器
  "parserOptions": {
    "ecmaVersion": 2015
  },
  "rules": {
    "indent": ["error"],
    "linebreak-style": ["error", "unix"],
    "quotes": ["error", "single"],
    "semi": ["error", "never"]
  }
}
```

解析器的作用是将代码转换为 AST，并且让 eslint 校验格式，babel 为了兼容 eslint 写了一个解析器，兼容 eslint 接口，可以生成 AST 使用 eslint 代码格式化

---

### 编写 babel 插件

#### 抽象语法树 AST

这个站点可以看到每个语法的抽象结构 https://astexplorer.net/

语法树每一层都拥有相同的结构

```js
{
  type: "FunctionDeclaration",
  id: {...},
  params: [...],
  body: {...}
}
```

```js
{
  type: "Identifier",
  name: ...
}
```

```js
{
  type: "BinaryExpression",
  operator: ...,
  left: {...},
  right: {...}
}
```

每一个节点都有如下所示的接口：

```
interface Node {
  type: string;
}
```

这样的每一层结构也被叫做 节点（Node），一个 AST 可以由单一的节点或是成百上千个节点构成

---

#### Babel 的处理步骤

解析（parse）：接收代码并输出 AST

转换（transform）：接收 AST 并对其进行遍历，在此过程中对节点进行添加、更新及移除等操作

生成（generate）：深度优先遍历整个 AST，然后构建可以表示转换后代码的字符串

---

#### 访问者

1. 当我们谈及`进入`一个节点，实际上是说我们在访问它们

2. 访问者是一个用于遍历 AST 的模式

3. 简单的说它们就是一个对象，定义了用于在一个树状结构中获取具体节点的方法

例如：

```js
const MyVisitor = {
  ExportNamedDeclaration(path){}
};
```

```js
const MyVisitor = {
  Identifier: {
   // 遍历 ast 时，深度遍历，会先进入后退出
    enter() {
      console.log("Entered!");
    },
    exit() {
      console.log("Exited!");
    }
  }
};
```

`MyVisitor` 是一个简单的访问者，把它用于遍历中时，每当在树中遇见一个 `Identifier` 的时候会调用 `Identifier` 内的方法

你还可以把方法名用 `|` 分割成 `Idenfifier | MemberExpression` 形式的字符串，把同一个函数应用到多种访问节点

```js
const MyVisitor = {
  'ExportNamedDeclaration|Flow' (path) {}
}
```

---

#### Paths（路径）

表示两个节点之间连接的对象

例如有个节点及其子节点

```js
{
  type: "FunctionDeclaration",
  id: {
    type: "Identifier",
    name: "square"
  },
  ...
}
```

将子节点 Identifier 表示为一个路径（Path）的话，看起来是这样的：

```js
{
  "parent": {
    "type": "FunctionDeclaration",
    "id": {...},
    ....
  },
  "node": {
    "type": "Identifier",
    "name": "square"
  }
}
```

路径对象还包含添加、更新、移动和删除节点有关的其他很多方法

---

#### Paths in Visitors

当你有一个 Identifier() 成员方法的 Visitor 时，你实际上是在访问路径而非节点。

---

#### API

babylon 解析代码为 ast

```js
import * as babylon from "babylon";
const code = `function square(n) {
  return n * n;
}`;
babylon.parse(code, {
sourceType: "module", // default: "script"
  plugins: ["jsx"] // default: []
});
```

sourceType 可以是 "module" 或者 "script"，它表示 Babylon 应该用哪种模式来解析。

"module" 将会在严格模式下解析并且允许模块定义，"script" 则不会。

sourceType 的默认值是 "script" 在 import 或 export 时产生错误。 使用 scourceType: "module" 来避免这些错误。

```js
// Node {
//   type: "File",
//   start: 0,
//   end: 38,
//   loc: SourceLocation {...},
//   program: Node {...},
//   comments: [],
//   tokens: [...]
// }
```

---

#### babel-traverse      

维护了整棵树的状态

```js
import * as babylon from "babylon";
import traverse from "babel-traverse";
const code = `function square(n) {
  return n * n;
}`;
const ast = babylon.parse(code);
traverse(ast, {
  enter(path) {
    if (
      path.node.type === "Identifier" &&
      path.node.name === "n"
    ) {
      path.node.name = "x";
    }
  }
});
```

---

#### babel-types  

工具库，类似 Lodash，方便编写 ast 逻辑

```js
import traverse from "babel-traverse";
import * as t from "babel-types";
traverse(ast, {
  enter(path) {
    if (t.isIdentifier(path.node, { name: "n" })) {
      path.node.name = "x";
    }
  }
});
```

---

#### babel-generator  

读取AST并将其转换为代码和源码映射

```js
import * as babylon from "babylon";
import generate from "babel-generator";
const code = `function square(n) {
  return n * n;
}`;
const ast = babylon.parse(code);
generate(ast, {}, code);
```

---

#### 来搞一个插件

目标: `a === a  -->  sebmck === dork`

./plugin.js

```js
// 返回访问者对象
// t 就是 babel-types  工具库
export default function({ types: t }) {
  return {
    visitor: {
      BinaryExpression(path) {
        if (path.node.operator !== "===") {
          return;
        }

        path.node.left = t.identifier("sebmck");	// 生成节点
        path.node.right = t.identifier("dork");
      }
    }
  };
}
```

配置文件

```json
{
  "presets": ["env"],
  "plugins": [
    "./plugin.js"
  ],
  "env": {
    "development": {
      "plugins": []
    },
    "production": {
      "plugins": []
    }
  }
}
```

---

#### 获取节点

1. path.node.xx 即可

2. path.get('left')  // 通过 get

```js
BinaryExpression(path) {
  path.node.left;
  path.node.right;
  path.node.operator;
}
```

---

#### 节点检查

```js
BinaryExpression(path) {
  if (t.isIdentifier(path.node.left)) {
    // ...
  }
}
```

---

#### 你同样可以对节点的属性们做浅层检查：

```js
BinaryExpression(path) {
  if (t.isIdentifier(path.node.left, { name: "n" })) {
    // ...
  }
}
```

---

#### 等价于

```js
BinaryExpression(path) {
  if (
    path.node.left != null &&
    path.node.left.type === "Identifier" &&
    path.node.left.name === "n"
  ) {
    // ...
  }
}
```

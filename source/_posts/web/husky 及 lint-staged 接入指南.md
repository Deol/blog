title: "husky 及 lint-staged 接入指南"
date: 2017-06-07 21:20:19
categories: Web
---

一、接入流程

 1. [npm 包安装](#npm包安装)
 2. [新建 .eslintrc，继承基础规则包](#必须在package.json同路径下新建.eslintrc，继承基础规则包)
 3. [修改 package.json 配置，设置 precommit 和 lint-staged](#修改package.json配置，设置precommit和lint-staged)
<!-- more -->

二、 [拓展阅读](#拓展阅读)

## 接入流程

### 1. npm 包安装

```bash
$ npm i -D eslint eslint-config-kaola husky lint-staged
```

#### 安装包说明：

 - [eslint](https://www.npmjs.com/package/eslint): 进行 JavaScript 代码检查的基础包；
 - [eslint-config-standard](https://github.com/standard/eslint-config-standard)：ESLint 基础规则包，也可以引用其他规则包或者自己维护一份；
 - [husky](https://www.npmjs.com/package/husky)：在 .git/hooks 中写入 pre-commit 等脚本激活[钩子](https://git-scm.com/book/zh/v2/%E8%87%AA%E5%AE%9A%E4%B9%89-Git-Git-%E9%92%A9%E5%AD%90)，在 Git 操作时触发；
 - [lint-staged](https://www.npmjs.com/package/lint-staged)：参考 Git 中 staged 暂存区概念，在每次提交时只检查本次提交的文件。

---

### 2. 必须在 package.json 同路径下新建 .eslintrc，继承基础规则包

需要忽略检查的文件夹及文件通过新建 .eslintignore 进行配置（同 .gitignore）。

 - ES5 引入：

```
{
  "extends": "standard",
  "globals":{
    // 在这里写系统特殊需要的全局变量，不需要则不写 globals
  },
  "rules": {
    // 在这里写覆盖规则，不需要则不写 rules
  }
}
```

---

### 3. 修改 package.json 配置，设置 precommit 和 lint-staged

一句话解释：husky 利用钩子探测到 commit 大巴并拦住后，lint-staged 接手并对坐在 commit 校车里面那堆暂存区 js 小弟逐个用 ESLint 扫描仪进行安检，并帮这帮小弟一个个把无伤大雅的安检问题解决掉，但只要出现一个解决不掉的问题安检就不通过，这辆 commit 大巴就别想过去。

回到正题，设置 lint-staged 分两种情况：

 - 当 package.json 不在项目根目录中（如位于子目录）：

    **利用 gitDir 找到相对 package.json 所在处的项目根目录（即 .git 文件夹所在目录）**，利用 linters 对与文件过滤路径匹配的每个暂存区文件，依次执行后面配置的任务。

```
{
  "scripts": {
    "precommit": "lint-staged"
  },
  "lint-staged": {
    "gitDir": "../",
    "linters":{
      "fed/src/**/*.js": ["eslint --fix", "git add"]
    }
  }
}
```

 - 当 package.json 位于项目根目录中：

    **在 lint-staged 中可以省略 gitDir 及 linters 这一层。**

```
{
  "scripts": {
    "precommit": "lint-staged"
  },
  "lint-staged": {
    "src/**/*.js": ["eslint --fix", "git add"]
  }
}
```

#### 文件过滤路径说明

指相对于项目根目录（即包含 .git 文件夹）的路径，利用此路径对暂存区中的文件进行过滤，仅对匹配文件执行命令。

```
{
    "*.js": "工程下所有的 js 文件",
    "**/*.js": "工程下所有的 js 文件",
    "src/*.js": "src 目录中所有的 js 文件",
    "src/**/*.js": "src 文件夹下所有的 js 文件"
}
```

> #### 钩子触发流程说明
> 
> 当开发者执行 `git add` 操作将代码提交到暂存区后，再执行 `git commit` 操作时：
> 
> 1. 由于 husky 在 .git/hooks 中写入了 pre-commit 钩子，该钩子在 `git commit` 执行时被触发，执行 `npm run precommit` 脚本（即 lint-staged 命令）；
> 2. lint-staged 利用配置的文件过滤路径，对暂存区文件一个个进行匹配，匹配成功时，运行 eslint --fix 并自动将修改添加到暂存区：
> ```bash
> $ eslint --fix path/[name].js
> $ git add path/[name].js
> ```
> 3. 此时如果有报错，则流程终止，无法执行 commit 操作。

## 拓展阅读

 - [用 husky 和 lint-staged 构建超溜的代码检查工作流](https://zhuanlan.zhihu.com/p/27094880)
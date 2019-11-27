---
title: vscode代码自动提示利器eslint
date: 2019-11-22 11:45:29
tags:
  - vscode
  - vuejs
  - eslint
categories:
  - tools
---

## eslint安装
eslint：Find and fix problems in your JavaScript code
+ 打开Teminal，进入到你的项目根目录后，输入下面命令
```
npm install eslint --save-dev
```
+ vscode安装eslint插件
如上面截图的logo，请安装对应的eslint插件
+ 验证安装成功
```
./node_modules/.bin/eslint -v
```
+ 生成eslintrc.js配置文件
```
./node_modules/.bin/eslint --init

// 下面是安装流程提示文字
? How would you like to use ESLint? 
To check syntax and find problems
? What type of modules does your project use? 
JavaScript modules (import/export)
? Which framework does your project use? 
Vue.js
? Does your project use TypeScript? 
No
? Where does your code run? 
Browser
? What format do you want your config file to be in? 
JavaScript
The config that you've selected requires the following dependencies:
```
+ 最终生成eslintrc.js文件
```
module.exports = {
    "env": {
        "browser": true,
        "es6": true
    },
    "extends": [
        "eslint:recommended" // eslint
    ],
    "globals": {
        "Atomics": "readonly",
        "SharedArrayBuffer": "readonly"
    },
    "parserOptions": {
        "ecmaVersion": 2018,
        "sourceType": "module"
    },
    "plugins": [
        "vue"
    ],
    "rules": {
        "semi": ["error", "always"],
        "quotes": ["error", "double"],
    }
};
```
> eslintrc配置规则（rules）
"off" or 0 - turn the rule off  
"warn" or 1 - turn the rule on as a warning (doesn’t affect exit code)
"error" or 2 - turn the rule on as an error (exit code will be 1)
+ 安装eslint-plugin-vue
Official ESLint plugin for Vue.js.
This plugin allows us to check the template and script of .vue files with ESLint.
Finds syntax errors.
Finds the wrong use of Vue.js Directives.
Finds the violation for Vue.js Style Guide.
```
npm install --save-dev eslint-plugin-vue
```
++ 修改eslintrc.js文件
```
// ... other options
extends: [
    "plugin:vue/essential" //eslint规则
    'plugin:vue/recommended' //eslint-plugin-vue规则
  ],
```
+ 安装eslint-loader
```
npm install eslint-loader eslint-friendly-formatter --save-dev
```
++ 修改webpack配置
这样你的 .vue 文件会在开发期间每次保存时自动检验。
```
// webpack.base.config.js
module.exports = {
  // ... other options
  module: {
    rules: [
      {
        test: /\.(js|vue)$/,
        loader: "eslint-loader",
        exclude: /node_modules/
        enforce: "pre",
        include: [resolve("src"), resolve("test")],
        options: {
          formatter: require("eslint-friendly-formatter")
        }
      }
    ]
  }
}
```
+ 修改.vscode/settings.json
You have to configure the eslint.validate option of the extension to check .vue files because the extension targets only *.js or *.jsx files by default.
```
"editor.formatOnSave": true, //保存时自动格式化
"eslint.validate": [
  "javascript",
  "javascriptreact",
  {
    "language": "html",
    "autoFix": true
  },
  {
    "language": "vue",
    "autoFix": true
  }
],
"eslint.autoFixOnSave": true

```

## 总结
首先推荐使用vscode这个编辑器来开发vuejs项目，最优秀的轻量级编辑器，迅速加载项目，即使你的项目里有数百个node_modules。  
其实eslint可以帮助你解决日常开发中遇到的代码bug，有友好的提示效果。
vue官方推荐使用eslint-plugin-vue插件来兼容eslint对于vue代码的检查。
另外想要关闭eslint的检查，可以在文件顶部加入
```
You may use special comments to disable some warnings.
Use // eslint-disable-next-line to ignore the next line.
Use /* eslint-disable */ to ignore all warnings in a file.
```
参考地址如下：
https://vue-loader-v14.vuejs.org/zh-cn/workflow/linting.html
https://eslint.org/docs/user-guide/getting-started
https://eslint.vuejs.org/user-guide/#faq
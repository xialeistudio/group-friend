# 用NodeJs构建命令行工具

分享者|分享日期
:-:|:-:|
Aturan|2018-04-11

## 工具说明

* [inquirer.js](https://github.com/SBoudrias/Inquirer.js)：一个封装了常用命令行交互的node.js模块，通过该模块可以很方便地构建一个新的命令行应用。

* [shell.js](https://github.com/shelljs/shelljs)：跨平台的unix shell命令模块。

* Node版本：由于**inquirer.js**的异步方法默认返回Promise，建议使用node.js>=8。

## 目标

工作中有大量项目上线前最后一步需要执行测试、编译、更新版本号、提交，甚至执行的命令都是一样，在这里我们通过命令行工具将这些步骤一键自动化，同时进行预检查，防止错漏。

## 准备

1. 创建一个新的Node.js项目。
2. 创建文件bin/my-cli.js，node.js项目通常会把cli入口放在bin目录下，其他模块放在lib目录下。
3. 在bin/my-cli.js文件头部添加`#!/usr/bin/env node`。
4. 添加 `"bin": {"my-cli": "./bin/my-cli.js"}`到package.json，声明我们要使用的命令。
5. 项目根目录下执行`npm link`，创建一个全局命令`my-cli`。

稍微修改下`my-cli.js`，添加代码`console.log("I am a cli tool!")`，然后打开控制台运行`my-cli`命令，如果看到控制台输出`I am a cli tool!`就表示成功。

## 安装依赖

首先安装主要依赖的两个模块（关于这两个模块的使用请参考官方文档）

`npm install inquirer shelljs`

## 构建发布流程自动化

接下来首先实现测试、更新版本号、构建、自动提交发布的自动化

```js
const pkg = JSON.parse(fs.readFileSync(pkgPath, 'utf8'));

const { version } = await inquirer.prompt([
  {
    type: 'list',
    name: 'version',
    message: '版本号更新方式：',
    choices: [
      {
        name: `v${semver.inc(pkg.version, 'patch')}: Fix Bugs / Patch`,
        value: 'patch'
      },
      {
        name: `v${semver.inc(pkg.version, 'minor')}: Release New Version`,
        value: 'minor'
      },
    ]
  }
]);
// 拉取最新版本
shelljs.exec('git pull');
// 运行测试
shelljs.exec('npm test');
//通过npm version更新版本号，但不自动添加git tag，而是在构建完成后由cli工具添加
shelljs.exec(`npm version ${version} --no-git-tag-version`);
// 构建
shelljs.exec('npm run build');
// 提交发布代码
const nextVersion = semver.inc(pkg.version, version);
shelljs.exec('git add . -A');
shelljs.exec(`git commit -m "build: v${nextVersion}"`)
shelljs.exec(`git tag -a v${nextVersion} -m "build: ${nextVersion}"`);
shelljs.exec("git push")
shelljs.exec("git push --tags");
```

## 添加新功能：配置检查

接下来给`my-cli`添加一个功能:

当检查到package.json的`my-cli`对象的`check-baidu-id`属性为`true`时，检查项目的`config.json`是否存在`baidu-id`属性

```js
if (pkg['my-cli'] && pkg['my-cli']['check-baidu-id']) {
  const configPath = path.join(process.cwd(), 'config.json');
  if (!fs.existsSync(configPath)) {
    shelljs.echo('找不到config.json');
    shelljs.exit(1);
  }
  const config = JSON.parse(fs.readFileSync(configPath, 'utf8'));
  if (!config['baidu-id']) {
    shelljs.echo('config.json缺少属性[baidu-id]');
    shelljs.exit(1);
  }
```

## 最后一步

这样一个简单的cli程序就实现完毕了，它自动化了构建发布流程，构建发布之前还进行了配置检查。

在实际项目中，为了提高程序的稳定性，还需要添加检查当前项目是否存在package.json，防止json解析出错、执行前确认等功能，具体见示例代码。

## 示例代码
地址：[https://github.com/Aturan/node-cli-example](https://github.com/Aturan/node-cli-example)

## 结语

虽然上述功能使用shell也可以实现，但代码编写就没那么方便快速，而且一旦碰到更复杂的问题，用shell实现就很麻烦，维护也是一个问题。

PS. 其实也可以用python，对于Ubuntu，系统自带Python是一个优势，在服务器不需要安装环境就可以直接使用，再加上Python也有Inquirer模块。
---
title: 利用patch修改依赖源码
---

## 背景
在用formily的ArrayTable进行二次封装时遇到一个问题：在点击表头的排序按钮后，本应触发的`onChange`事件并没有发生。

在花了大半天时间排查到底是不是自己这边出了问题后看了一遍[源码](https://github.com/alibaba/formily/commit/11e14a39#diff-66bda78b571cd2999c798e18c00373deac87952ea720c7e0559dded1f955786eR280)后才发现原来是开发者为了防止事件冒泡在组件源头把`onChange`给拦截了。

这种源码级别的拦截很难在外部用代码绕过，所以这里要借用patch的能力修改源码。
## 操作
有多种方式可以实现patch，本次用的是yarn的patch功能，指令很简单，输入`yarn patch [需要修改的包名]`即可。

这时候能看到一个文件夹地址，这就是需要修改的源码地址。
![[Pasted image 20240930165237.png]]
一般来说一个npm包都有**dist**、**esm**、**lib**三部分的内容。主要修改后两部分的内容，分别对应ESM和CJS两种JS的模块引入方式。

修改完后执行相应的`yarn patch-commit`指令即可报错这次修改，patch文件可以在`.yarn/patches`文件夹下找到，只要重新运行一遍`yarn install`，项目就会应用上这些修改。
## 注意事项
在使用VSCode打开源码修改文件夹的时候，VSCode会自动创建`.vscode`文件夹，这会影响patch结果。

每次patch都是全量修改，所以如果一个包有多次修改，每一次都要把之前改过的地方再做一次。

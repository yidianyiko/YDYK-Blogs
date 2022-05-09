---
date: 2021-11-16
title: 【源码第六期】检测npm名称源码分析
tags:
  - 源码
describe: 浅析 validate-npm-package-name源码
---

### 1. 这是什么？



[validate-npm-package-name](https://github.com/npm/validate-npm-package-name)的作用就是：检验npm包的名称是否符合命名标准。

官网中也列出这个工具应该如何使用且举例了正确的包名称

```
var validate = require("validate-npm-package-name")

validate("some-package")
validate("example.com")
validate("under_score")
validate("123numeric")
validate("@npm/thingy")
validate("@jane/foo.js")
```

###

### 2.学习目标

粗略的看了一下源码的内容，能大致知道通过本次阅读能学习到的知识有：

-   知道正确的包名称应该怎么命名
-   简单理解正则表达式



### 3.阅读源码



整个源码分为两个函数

-   validate()，接收一个参数——包名称
-   done()，接收两个参数——警告数组、错误数组

#### 3.1 validate()

这部分主要是一些字符串判断的校验

```
var validate = module.exports = function (name) {
  var warnings = []
  var errors = []

  // 不能为null
  if (name === null) {
    errors.push('name cannot be null')
    return done(warnings, errors)
  }
	
  // 不能为undefined
  if (name === undefined) {
    errors.push('name cannot be undefined')
    return done(warnings, errors)
  }
	
  // 必须是一个字符串
  if (typeof name !== 'string') {
    errors.push('name must be a string')
    return done(warnings, errors)
  }

  // 名称不能为空
  if (!name.length) {
    errors.push('name length must be greater than zero')
  }

  // 名称不能以.开头
  if (name.match(/^./)) {
    errors.push('name cannot start with a period')
  }

  // 名称不能以下划线开头
  if (name.match(/^_/)) {
    errors.push('name cannot start with an underscore')
  }

  // 名称尾部不能有空格
  if (name.trim() !== name) {
    errors.push('name cannot contain leading or trailing spaces')
  }

  /* var blacklist = [
      'node_modules',
      'favicon.ico'
		]
  */
  // 不能出现黑名单中的名字
  // No funny business
  blacklist.forEach(function (blacklistedName) {
    if (name.toLowerCase() === blacklistedName) {
      errors.push(blacklistedName + ' is a blacklisted name')
    }
  })

  
  // 不能是node内置模块的名称，这里引入了一个包，包里是node内置模块的数组
  // core module names like http, events, util, etc
  builtins.forEach(function (builtin) {
    if (name.toLowerCase() === builtin) {
      warnings.push(builtin + ' is a core module name')
    }
  })

  // 名称不可以很长很长很长很长
  if (name.length > 214) {
    warnings.push('name can no longer contain more than 214 characters')
  }

  // 大小写不能同时出现
  if (name.toLowerCase() !== name) {
    warnings.push('name can no longer contain capital letters')
  }
	
  // 不能包含()~ ! *等符号
  if (/[~'!()*]/.test(name.split('/').slice(-1)[0])) {
    warnings.push('name can no longer contain special characters ("~'!()*")')
  }

  // 有些内容可能会出现歧义，所以需要encode一下，比如出现/，或@baidu/anti-spam
  // 如果encode前后内容一致，则会通过
  // 否则会报错
  if (encodeURIComponent(name) !== name) {
    // Maybe it's a scoped package name, like @user/package
    var nameMatch = name.match(scopedPackagePattern)
    if (nameMatch) {
      var user = nameMatch[1]
      var pkg = nameMatch[2]
      if (encodeURIComponent(user) === user && encodeURIComponent(pkg) === pkg) {
        return done(warnings, errors)
      }
    }

    errors.push('name can only contain URL-friendly characters')
  }

  return done(warnings, errors)
}
```

这一部分已经将所有包命名规则都校验了一遍，如果出现不合理的情况时，则会往`errors、warnings`数组中push错误信息

#### 3.2 done()

这一部分则是对校验完的参数进行处理

```
var done = function (warnings, errors) {
  var result = {
    // 如果没有错误和警告，就为true
    validForNewPackages: errors.length === 0 && warnings.length === 0,
    // 如果没有错误就为true
    validForOldPackages: errors.length === 0,
    warnings: warnings,
    errors: errors
  }
  // 检查errors、warnings数组的长度，如果有空，则删除该属性
  if (!result.warnings.length) delete result.warnings
  if (!result.errors.length) delete result.errors
  
  // 返回一个校验后的最终结果对象
  return result
}
```

### 4. 总结

这次的源码较其他期比较简单，但也能学习到如果需要制定一个公共的约定时，需要考虑的判断因素会比较多，各方面都需要进行考虑。因此在项目开发中，对复杂的业务场景进行处理时，也应该考虑各种情况的出现，避免代码出现问题
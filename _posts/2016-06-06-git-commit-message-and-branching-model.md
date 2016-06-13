---
layout: post
title: Git提交记录和分支模型
date: 2016-06-06 16:20
author: CatTail
comments: true
---

> 文章首发于[个人博客](https://cattail.me/tech/2016/06/06/git-commit-message-and-branching-model.html)

两年前编写的文章[Git Style](https://cattail.me/tech/2013/08/21/git-style.html)，是参考业界实践对Git提交记录格式和分支模型所做的总结。**本文在Git Style基础上，再次描述提交记录的格式和分支模型，并介绍两个工具commitizen和gitflow，分别处理维护提交记录格式和分支切换的工作**。

## Commit Message

在Git Style中已经介绍了提交记录（Commit Message）的格式，但是没有说明为什么要遵循这样的约定。事实上，这个格式参考了[AngularJS's commit message convention](https://github.com/angular/angular.js/blob/f3377da6a748007c11fde090890ee58fae4cefa5/CONTRIBUTING.md#commit)，而AngularJS制定这样的约定是出于几个目的

* 自动生成CHANGELOG.md
* 识别不重要的commit
* 为浏览Commit历史时提供更好的信息

后面简称AngularJS's commit message convention为conventional message。

### 格式

**Conventional message格式**是这样的

	<type>(<scope>): <subject>
	<BLANK LINE>
	<body>
	<BLANK LINE>
	<footer>

`subject`是对变更的简要描述。

`body`是更为详细的描述。

`type`则定义了此次变更的类型，只能使用下面几种类型

* feat：增加新功能
* fix：问题修复
* docs：文档变更
* style：代码风格变更（不影响功能）
* refactor：既不是新功能也不是问题修复的代码变更
* perf：改善性能
* test：增加测试
* chore：开发工具（构建，脚手架工具等）

`footer`可以包含Breaking Changes和Closes信息。

### 应用

[node-trail-agent](https://github.com/open-trail/node-trail-agent)项目遵循conventional message，这里以这个项目举例，演示其应用场景。

**CHANGELOG**

通过`git log`可以生成CHANGELOG.md中版本间新增功能，

	$ git log v0.4.0..v1.1.2 --grep feat --pretty=format:%s

	feat: add "type" tag to distinguish client and server span
	feat: instrument via Module._load hook

也可以同时获取修复的问题，

	$ git log v0.4.0..v1.1.2 --grep 'feat\|fix' --pretty=format:%s

	fix: wrapper function not returned
	feat: add "type" tag to distinguish client and server span
	feat: instrument via Module._load hook

**定位错误**

使用`git bisect`可以定位引入问题的提交，通过`type`可以快速辨别不会引入bug的提交，

	(master) $ git bisect start
	(master) $ git bisect bad
	(master) $ gco v0.1.0
	(7f04616) $ git bisect good
	
	Bisecting: 15 revisions left to test after this (roughly 4 steps)
	[433f58f26a53b519ab7fc52927895e67eb068c64] docs: how to use agent
	
	(433f58f) $ git bisect good
	
	Bisecting: 7 revisions left to test after this (roughly 3 steps)
	[02894fb497f41eecc7b21462d3b020687b3aaee2] docs(CHANGELOG): 1.1.0
	
	(02894fb) $ git bisect good
	
	Bisecting: 3 revisions left to test after this (roughly 2 steps)
	[9d15ca2166075aa180cd38a3b4d3be22aa336575] chore(release): 1.1.1
	
	(9d15ca2) $ git bisect good
	
	Bisecting: 1 revision left to test after this (roughly 1 step)
	[f024d7c0382c4ff8b0543cbd66c6fe05b199bfbc] fix: wrapper function not returned
	
	(f024d7c) $ npm test
	...

因为docs或chore不会引入bug，所以可以直接执行`git bisect good`。

使用`git bisect skip`可以直接过滤掉这些提交，

	$ git bisect skip $(git rev-list --grep 'style\|docs\|chore' v0.1.0..HEAD)

### Commitizen

命令行工具[commitizen](https://github.com/commitizen/cz-cli)帮助开发者生成符合conventional message的提交记录。

成功安装并初始化commitizen后，通过调用`git cz`来提交代码，

	$ git cz

	Line 1 will be cropped at 100 characters. All other lines will be wrapped after 100 characters.

	? Select the type of change that you're committing: (Use arrow keys)
	❯ feat:     A new feature
	  fix:      A bug fix
	  docs:     Documentation only changes
	  style:    Changes that do not affect the meaning of the code
	            (white-space, formatting, missing semi-colons, etc)
	  refactor: A code change that neither fixes a bug or adds a feature
	  perf:     A code change that improves performance

提交后会按照convertional message格式化提交记录，

	commit f024d7c0382c4ff8b0543cbd66c6fe05b199bfbc
	Author: zhongchiyu <zhongchiyu@gmail.com>
	Date:   Mon May 30 14:49:17 2016 +0800
	
	    fix: wrapper function not returned

除此之外，commitizen还依据conventional message，创建起一个生态

* [conventional-changelog-cli](https://github.com/conventional-changelog/conventional-changelog-cli)：通过提交记录生成CHANGELOG.md
* [conventional-github-releaser](https://github.com/conventional-changelog/conventional-github-releaser)：通过提交记录生成github release中的变更描述
* [conventional-recommended-bump](https://github.com/conventional-changelog/conventional-recommended-bump)：根据提交记录判断需要升级[Semantic Versioning](http://semver.org/)哪一位版本号
* [validate-commit-msg](https://github.com/kentcdodds/validate-commit-msg)：检查提交记录是否符合约定

使用这些工具可以简化npm包的发布流程，

	#! /bin/bash

	# https://gist.github.com/stevemao/280ef22ee861323993a0
	# npm install -g commitizen cz-conventional-changelog trash-cli conventional-recommended-bump conventional-changelog-cli conventional-commits-detector json
	trash node_modules &>/dev/null;
	git pull --rebase &&
	npm install &&
	npm test &&
	cp package.json _package.json &&
	preset=`conventional-commits-detector` &&
	echo $preset &&
	bump=`conventional-recommended-bump -p angular` &&
	echo ${1:-$bump} &&
	npm --no-git-tag-version version ${1:-$bump} &>/dev/null &&
	conventional-changelog -i History.md -s -p ${2:-$preset} &&
	git add History.md &&
	version=`cat package.json | json version` &&
	git commit -m"docs(CHANGELOG): $version" &&
	mv -f _package.json package.json &&
	npm version ${1:-$bump} -m "chore(release): %s" &&
	git push --follow-tags &&
	npm publish

运行上述脚本会更新CHANGELOG.md、升级版本号并发布新版本到npm，所有这些操作都基于提交记录自动处理。

## Branching Model

[Vincent Driessen](http://nvie.com/about/)的分支模型（Branching Model）介绍Git分支和开发，部署，问题修复时的工作流程，

![workflow]({{ site.remoteurl }}/assets/git-workflow/workflow.png)

> Author: Vincent Driessen Original blog post: http://nvie.com/posts/a-succesful-git-branching-model License: Creative Commons BY-SA

在整个开发流程中，始终存在master和develop分支，其中master分支代码和生产环境代码保持一致，develop分支还包括新功能代码。

**分支模型主要涉及三个过程：功能开发，代码发布和问题修复**。

**功能开发**

1. 从develop创建一个新分支（feature/\*）
2. 功能开发
3. 生产环境测试
4. Review
5. Merge回develop分支

**代码发布**

需要发布新功能到生产环境时

1. 从develop创建新分支（release/\*）
2. 发布feature分支代码到预上线环境
3. 测试并修复问题
4. Review
5. 分别merge回develop和master分支
6. 发布master代码到生产环境

**问题修复**

当生产环境代码出现问题需要立刻修复时

1. 从master创建新分支（hotfix/\*）
2. 发布hotfix代码到预上线环境
3. 修复问题并测试
4. Review
5. 分别merge会develop和master分支
6. 发布master代码到生产环境



该分支模型值得借鉴的地方包括，

* **规范的分支命名**
* **将分支和代码运行环境关联起来**

分支和代码运行环境的关系是这样的，

- master => 生产环境
- release/\*，hotfix/\* => 预上线环境
- feature/\*，develop => 开发环境

### gitflow

Vincent Driessen的分支模型将开发流程和Git分支很好的结合起来，但在实际使用中涉及复杂的分支切换，[gitflow](https://github.com/nvie/gitflow)可以简化这些工作。

安装并在代码仓库初始化gitflow后，就可以使用它完成分支工作流程，

	$ git flow init

	No branches exist yet. Base branches must be created now.
	Branch name for production releases: [master]
	Branch name for "next release" development: [develop]
	
	How to name your supporting branch prefixes?
	Feature branches? [feature/]
	Release branches? [release/]
	Hotfix branches? [hotfix/]
	Support branches? [support/]
	Version tag prefix? []

**功能开发**

开始开发时

	(develop) $ git flow feature start demo

	Switched to a new branch 'feature/demo'

	Summary of actions:
	- A new branch 'feature/demo' was created, based on 'develop'
	- You are now on branch 'feature/demo'

完成开发后

	(feature/demo) $ git flow feature finish demo

	Switched to branch 'develop'
	Already up-to-date.
	Deleted branch feature/demo (was 48fbada).
	
	Summary of actions:
	- The feature branch 'feature/demo' was merged into 'develop'
	- Feature branch 'feature/demo' has been removed
	- You are now on branch 'develop'

**代码发布**

发布代码前

	(develop) $ git flow release start demo

	Switched to a new branch 'release/demo'

	Summary of actions:
	- A new branch 'release/demo' was created, based on 'develop'
	- You are now on branch 'release/demo'

测试完成准备上线时

	(release/demo) $ git flow release finish demo

	Switched to branch 'master'
	Deleted branch release/demo (was 48fbada).
	
	Summary of actions:
	- Latest objects have been fetched from 'origin'
	- Release branch has been merged into 'master'
	- The release was tagged 'demo'
	- Release branch has been back-merged into 'develop'
	- Release branch 'release/demo' has been deleted

发布master代码到线上环境

**问题修复**

发现线上故障时，

	(master) $ git flow hotfix start demo-hotfix

	Switched to a new branch 'hotfix/demo-hotfix'

	Summary of actions:
	- A new branch 'hotfix/demo-hotfix' was created, based on 'master'
	- You are now on branch 'hotfix/demo-hotfix'

修复问题后

	(hotfix/demo-hotfix) $ git flow hotfix finish demo-hotfix

	Deleted branch hotfix/demo-hotfix (was 48fbada).

	Summary of actions:
	- Latest objects have been fetched from 'origin'
	- Hotfix branch has been merged into 'master'
	- The hotfix was tagged 'demo-hotfix'
	- Hotfix branch has been back-merged into 'develop'
	- Hotfix branch 'hotfix/demo-hotfix' has been deleted

### 相关链接

* [commitizen](https://github.com/commitizen/cz-cli)
* [gitflow](https://github.com/nvie/gitflow)
* [AngularJS Git Commit Message Conventions](https://docs.google.com/document/d/1QrDFcIiPjSLDn3EL15IJygNPiHORgU1_OOAqWjiDU5Y/edit)
* [Git Style](https://cattail.me/tech/2013/08/22/git-style.html)
* [node-trail-agent](https://github.com/open-trail/node-trail-agent)
* [A successful Git branching model](http://nvie.com/posts/a-successful-git-branching-model/)
* [Using git-flow to automate your git branching workflow](http://jeffkreeftmeijer.com/2010/why-arent-you-using-git-flow/)

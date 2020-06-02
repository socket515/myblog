---
title: cicd
date: 2020-06-02 20:42:45
tags:
- 软件工程
- cicd
categories:
- cicd
---
<meta name="referrer" content="no-referrer" />


# CI/CD 介绍

## 概述:
 软件开发周期中需要一些可以帮助开发者提升速度的自动化工具。其中工具最重要的目的是促进软件项目的*持续集成*(Continuous integration,CI)与*持续部署*(continuous deployment,CD)。通过CI/CD工具，开发团队可以保持软件更新并将其迅速的投入实践中。CI/CD也被认为是敏捷开发的最重要实践之一。

## 1.持续集成
  持续集成包含以下几个部分
  
  - 自动化构建Continuous Build
  - 自动化测试Continuous Test
  - 自动化集成Continuous Integration

  自动化构建包含以下部分
  
  - 将源码编译成为二进制码
  - 打包二进制码
  - 运行自动化测试
  - 生成文档
  - 生成分发媒体（例如：Debian DEB、Red Hat RPM或者Windows MSI文件）
  
  自动化构建关键三部分: 版本控制工具、构建工具、CI服务器
 
### 版本控制工具
  版本控制工具又称配置管理（SCM）所以版本控制工具同时也是配置管理工具。其中较为出名版本控制工具是SVN和GIT.

### 构建工具
  构建工具是持续集成的核心，它对源代码进行自动化编译、测试、代码检查，以及打包程序、部署（发布）到应用服务器上。从配置管理工具上下载最新源代码后，所有的后续工作几乎都可以通过构建工具完成。比如JAVA构建工具是Ant、Maven、Gradle,PHP构建工具是Phing。而golang的编译比较方便快捷,go tool有一系列编译代码检查工具也很方便快捷。golang官方没有类似像MAVEN一样构建工具，项目组使用Makefile做代码编译，代码生成等。

### CI服务器
  CI服务器的主要作用就是提供一个平台，用于整合版本控制和构建工作，并管理、控制自动化的持续集成。其中比较出名为Jenkins、CruiseControl、Continuum、gitlab ci。项目组使用gitlab做git项目仓库 能方便使用gitlab ci。
  
  自动化测试
  
  自动化测试是持续集成必不可少的一部分，基本上，没有自动化测试的持续集成，都很难称之为真正的持续集成。我们希望持续集成能够尽早的暴露问题。只有真正用心编写了较为完整的测试用例，并一直维护它们，持续集成才能孜孜不倦地运行测试并第一时间报告问题。自动化测试还包括单元测试、集成测试、系统测试、验收测试、性能测试等，在不同的场景下，它们都能为软件开发带来极大的价值

## 2. 持续部署和持续交付
 持续交付（Continuous Delivery, CD）是一种软件工程的手段，让软件在短周期内产出，确保软件随时可以被可靠地发布。其目的在于更快、更频繁地构建、测试以及发布软件。通过加强对生产环境的应用进行渐进式更新，这种手段可以降低交付变更的成本与风险。一个简单直观的与可重复的部署过程对于持续交付来说是很重要的。

 持续部署与持续交付之间的差异就是前者将部署自动化了。在持续交付的实践中，交付的目标是QA，但是实际上，软件最终是要交付到客户手上的。在SaaS领域里，持续部署采用得比较广泛，因为服务比较容易做到静默升级。采用持续部署的前提是自动化测试的覆盖率足够高。采用持续部署的好处是能减少运维的工作量，缩短新特性从开发到实际交付的周期。

# Gitlab CI 使用及介绍

## GitLab
 GitLab 是一个利用Ruby on Rails开发的开源应用程序，实现一个自托管的 Git 项目仓库，可通过 Web 界面进行访问公开的或者私人项目。它拥有与GitHub类似的功能，能够浏览源代码，管理缺陷和注释。可以管理团队对仓库的访问，它非常易于浏览提交过的版本并提供一个文件历史库。
 
## GitLab CI/CD
 GitLab CI/CD 是GitLab Continuous Integration（Gitlab持续集成）的简称。GitLab 自GitLab 8.0开始提供了持续集成的功能，且对所有项目默认开启。只要在项目仓库的根目录添加.gitlab-ci.yml文件，并且配置了Runner（运行器），那么每一次push或者合并请求（Merge Request）都会触发CI Pipeline。
 
## GitLab Runner
 GitLab Runner是一个开源项目，可以运行在 GNU / Linux，macOS 和 Windows 操作系统上。是运行CI/CD任务的软件。每次push的时候 GitLab CI 会根据.gitlab-ci.yml配置文件运行流水线（Pipeline）中各个阶段的任务（Job），并将结果发送回 GitLab。GitLab Runner 是基于 Gitlab CI 的 API 进行构建的相互隔离的机器（或虚拟机）。GitLab Runner 不需要和 Gitlab 安装在同一台机器上，且考虑到 GitLab Runner 的资源消耗问题和安全问题，也不建议这两者安装在同一台机器上。
 
 Gitlab Runner 分为三种：
 
 - 共享Runner(Shared runners)
 - 专享Runner(Specific runners)
 - 分组Runner(Group Runners)

## Pipeline
 Pipeline即是流水线,是分阶段执行的构建任务。如：安装依赖、运行测试、打包、部署开发服务器、部署生产服务器等流程。每一次`push`或者`Merge Request`都会触发生成一条新的Pipeline。

## Stages
 Stages表示构建阶段，可以理解为上面所说“安装依赖”、“运行测试”等环节的流程。我们可以在一次 Pipeline 中定义多个 Stages，这些 Stages 会有以下特点：
 
 - 所有 Stages 会按照顺序运行，即当一个 Stage 完成后，下一个 Stage 才会开始（当然可以在.gitlab-ci.yml文件中配置上一阶段失败时下一阶段也执行）
 - 只有当所有 Stages 完成后，该构建任务 (Pipeline) 才会成功
 - 如果任何一个 Stage 失败，那么后面的 Stages 不会执行，该构建任务 (Pipeline) 失败

## Jobs
 Jobs 表示构建的作业（或称之为任务），表示某个 Stage 里面执行的具体任务。我们可以在 Stages 里面定义多个 Jobs，这些 Jobs 会有以下特点：
 - 相同 Stage 中的 Jobs 无执行顺序要求，会并行执行
 - 相同 Stage 中的 Jobs 都执行成功时，该 Stage 才会成功
 - 如果任何一个 Job 失败，那么该 Stage 失败，即该构建任务 (Pipeline) 也失败（可以在.gitlab-ci.yml文件中配置允许某 Job 可以失败，也算该 Stage 成功）

## .gitlab-ci.yml
 GitLab 中默认开启了 Gitlab CI/CD 的支持，且使用YAML文件.gitlab-ci.yml来管理项目构建配置。该文件需要存放于项目仓库的根目录（默认路径，可在 GitLab 中修改），它定义该项目的 CI/CD 如何配置。所以，我们只需要在.gitlab-ci.yml配置文件中定义流水线的各个阶段，以及各个阶段中的若干作业（任务）即可。
 
 下面是.gitlab-ci.yml文件的一个简单的Hello World示例：
 
 ```yaml
 # 定义 test 和 package 两个 Stages
 stages:
   - test
   - package
 
 # 定义 package 阶段的一个 job
 package-job:
   stage: package
   script:
     - echo "Hello, package-job"
     - echo "I am in package stage"
 
 # 定义 test 阶段的一个 job
 test-job:
   stage: test
   script:
     - echo "Hello, test-job"
     - echo "I am in test stage"
 ```
 
 以上配置中，用 stages 关键字来定义 Pipeline 中的各个构建阶段，然后用一些非关键字来定义 jobs。每个 job 中可以可以再用 stage 关键字来指定该 job 对应哪个 stage。job 里面的script关键字是每个 job 中必须要包含的，它表示每个 job 要执行的命令

## 配置gitlab-ci.yml

 开始构建之前.gitlab-ci.yml文件定义了一系列带有约束说明的任务。这些任务都是以任务名开始并且至少要包含script部分，.gitlab-ci.yml允许指定无限量 jobs。每个 jobs 必须有一个唯一的名字，且名字不能是下面列出的保留字段：
 
 - image
 - services
 - stages
 - types
 - before_script
 - after_script
 - variables
 - cache

 下列关键字用来定义job的行为
 
 | Keyword    | Required | Description |
 | ------- | -------- | -------- |
 | script  |   yes      |  Runner执行的脚本, 即是这个job要执行的脚本 |
 | extends |   no      |  定义此作业将继承的配置条目 |
 | images |   no      |  定义运行这个job使用的docker镜像 |
 | services |   no      |  定义运行这个job使用的docker镜像 |
 | stage    |   no      |  定义这个stage的名字，默认是test|
 | variables |  no      |  定义运行job的环境变量, 在执行脚本时候可能需要一些预先设定环境变量 |
 | only |  no      | 限定某些分支才创建job,支持正则。 |
 | except |  no      | 与only相反,限定某些分支不创建job,支持正则。 |
 | tags |  no      | 定义某些tag的runner来运行job,支持正则(gitlab runner 需要预先设定好tag) |
 | allow_failure |  no      | 允许job失败。失败的job不影响commit状态 |
 | when |  no      | 定义何时执行job,可选值: on_success(前面的stage都成功才执行)、on_failure(前面任一个stage失败时候才执行)、always(无论怎么样都执行)、manual(手动执行) |
 | dependencies	 |  no      | 定义job依赖关系, 被依赖的job完成后此job才执行，这样他们就可以互相传递artifacts, 可以使用artifacts中的文件 |
 | artifacts |  no      | 用于指定成功后应附加到 job 的文件和目录的列表, 可以让被依赖的job使用附加的文件 |
 | cache |  no      | 定义应在后续运行之间缓存的文件列表 |
 | before_script  |   no      |  定义job运行前的脚本 |
 | after_script  |   no      |  定义job运行后的脚本 |
 | environment  |   no      |  定义此作业完成部署的环境名称 |
 | coverage  |   no      |  定义给定作业的代码覆盖率设置 |
 | etry  |   no      |  定义在发生故障时可以自动重试作业的时间和次数 |
 | parallel  |   no      |  定义应并行运行的作业实例数 |
 
### extends

 `extends`定义了一个使用`extends`的作业将继承的条目名称。它是使用YAML锚点的替代方案，并且更加灵活和可读：
 
 ```yaml
.tests:
  script: rake test
  stage: test
  only:
    refs:
      - branches

rspec:
  extends: .tests
  script: rake rspec
  only:
    variables:
      - $RSPEC
 ```

  GitLab 将根据键执行反向深度合并,把继承的job内容递归复制到此job中,已经存在的键不合并, 实际会和以下配置一样。

```yaml
rspec:
  script: rake rspec
  stage: test
  only:
    refs:
      - branches
    variables:
      - $RSPEC
```

### cache
  使用`path`使用paths指令选择要缓存的文件或目录。也可以使用通配符。如果 cache 定义在 jobs 的作用域之外，那么它就是全局缓存，所有 jobs 都可以使用该缓存。下面例子缓存`binaries`和`.config`中的所有文件：
  ```yaml
rspec:
  script: test
  cache:
    paths:
    - binaries/
    - .config
  ```
  
  `untracked` 表明缓存不被git跟踪的文件
  ```yaml
rspec:
  script: test
  cache:
    untracked: true
  ```
  
  缓存`binaries`目录下不给git跟踪的文件
  ```yaml
rspec:
  script: test
  cache:
    untracked: true
    paths:
    - binaries/
  ```
  
  job 中优先级高于全局的。下面这个rspec job中将只会缓存binaries/下的文件：
  ```yaml
cache:
  paths:
  - my/files

rspec:
  script: test
  cache:
    key: rspec
    paths:
    - binaries/
  ```
  
  `key`指令允许我们定义缓存的作用域(亲和性)，可以是所有 jobs 的单个缓存，也可以是每个 job，也可以是每个分支或者是任何你认为合适的地方。它也可以让你很好的调整缓存，允许你设置不同 jobs 的缓存，甚至是不同分支的缓存。缓存是在 jobs 之前进行共享的。如果你不同的 jobs 缓存不同的文件路径，必须设置不同的`cache:key`，否则缓存内容将被重写。
  
  `key`可以使用[预定义变量](https://docs.gitlab.com/ee/ci/variables/predefined_variables.html)
  
  缓存每一个job
  ```yaml
cache:
  key: "$CI_JOB_NAME"
  untracked: true
  ```
  缓存每个分支：
  ```yaml
cache:
  key: "$CI_COMMIT_REF_NAME"
  untracked: true
  ```
  缓存每个 job 且每个分支：
  ```yaml
cache:
  key: "$CI_JOB_NAME/$CI_COMMIT_REF_NAME"
  untracked: true
  ```
  缓存每个分支且每个stage：
  ```yaml
cache:
  key: "$CI_JOB_STAGE/$CI_COMMIT_REF_NAME"
  untracked: true
  ```

### artifacts
 `artifacts`用于指定成功后应附加到 job 的文件和目录的列表。只能使用项目工作间内的文件或目录路径。在job成功完成后artifacts将会发送到GitLab中，同时也会在 GitLab UI 中提供下载。
 
 发送binaries和.config中的所有文件：
 ```yaml
artifacts:
  paths:
  - binaries/
  - .config
 ```
 
 发送所有没有被Git跟踪的文件：
 ```yaml
artifacts:
  untracked: true
 ```
 
 发送没有被Git跟踪和binaries中的所有文件：
 ```yaml
artifacts:
  untracked: true
  paths:
  - binaries/
 ```
 
 要在job直接传递`artifacts`,可以使用`dependencies`。定义了`dependencies`的job在开始时候会下载被依赖job的`artifacts`文件。下面例子是先编译后执行单元测试。

```yaml
build:
 stage: build
 artifacts:
  paths:
   - build
 script:
  - go build build/unittest unittest.go
  - go test -coverpkg=project/... -c -tags test project"
  - cp project.test build/

test:
 stage: test
 dependencies:
  - build
 artifacts:
  paths:
   - coverage
 script:
  - ./build/project.test
  - pid=$!
  - ./build/unittest
  - kill -15 $pid
  - gocovmerge *.out > coverage/all.out
  - touch coverage/coverage.html
  - go tool cover -html=coverage/all.out -o coverage/coverage.html
```

## GitLab Pages
 在持续集成中自动化构建的一部分是自动化生成page, 可以生成项目对应使用文档。在每次项目更改时候都能通过持续集成来生成对应文档。其中用得比较多是生成覆盖率文档。在每次提交代码后进行单元测试覆盖率计算生成覆盖率文档。
 
 GitLab Pages是用于托管静态文件的服务。可以用于托管自动化生成的文档。`pages`是一个特殊的job1,用于生成静态页面上传到GitLab。可用于为您的网站提供服务。它有特殊的语法，因此必须满足以下两个要求：
 
 - 任何静态内容必须放在public/目录下
 - artifacts必须定义在public/目录下
 
 下面例子生成单元测试覆盖率页面, 依赖的test job中生成关于单元测试报告页面，cp到public目录下。
 ```yaml
pages:
  stage: deploy
  dependencies:
   - test
  script:
   - mkdir .public
   - mv coverage/coverage.html public/
  artifacts:
    paths:
    - public
  only:
  - master
 ```

## 相关文档
 [gitlab ci 官方文档](https://docs.gitlab.com/ce/ci/yaml/README.html)

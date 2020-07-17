---
layout: post
title: 使用 gitlab-runner 持续集成
---

gitlab-runner 是 Gitlab 推出的与 [Gitlab CI](https://about.gitlab.com/stages-devops-lifecycle/continuous-integration/) 配合使用的持续集成工具。当开发人员在 Gitlab 上更新代码之后，Gitlab CI 服务能够检测到代码更新，此时可以触发一些动作，比如代码测试、编译和部署等，下图简单示意了这个流程。
![](/assets/gitlab-ci-sample.png)

那么，Gitlab 触发动作的时候是怎么找到对应的工作机的呢，这时 gitlab-runner 就出场了。我们需要在工作机上用 gitlab-runner 配置好与之对应的 Gitlab 服务。

首先需要在工作机上安装 gitlab-runner，这里以 Ubuntu 操作系统为例。
```shell
# 下载
curl -LJO https://gitlab-runner-downloads.s3.amazonaws.com/latest/deb/gitlab-runner_amd64.deb
# 安装
dpkg -i gitlab-runner_amd64.deb
```

安装完成之后就可以在工作机上注册 Gitlab。
```shell
gitlab-runner register
```
按照提示，填入 gitlab url 和 token。
![gitlab-runner register](/assets/gitlab-runner-register.png)

在 gitlab 项目页面依次点击 「Settings」->「CI/CD」->「Runners Expand」，就可以看到需要填入的 gitlab url 和 token 了。
![查看 gitlab url 和 token](/assets/gitlab-url-and-token.png)
![查看 gitlab url 和 token](/assets/show-url-and-token.png)

至此，就在工作机上完成了 Gitlab 的注册，那么 Gitlab 服务器就能找到对应的工作机了。至于 Gitlab 服务器会触发工作机执行哪些动作，需要在项目的根目录下配置 `.gitlab-ci.yml` 文件，说明具体的执行命令。配置规则可以参考[官方文档](https://docs.gitlab.com/ee/ci/yaml/)，下面是一个简单的例子，当 master 分支上有代码更新的时候会在工作机上打印执行当前命令的用户名。

```yml
stages:
  - test
    
job_test:
  stage: test
  script: 
    - echo '开始测试...'
    - whoami
    - echo '测试完成'
  only:
    - master
```

如果按照上述步骤操作之后，更新代码依然不能触发工作机执行动作，有可能是下面两个原因：

**一、gitlab-runner 注册的 CI 服务默认不处理没有打 tag 的 commit**。

更新代码之后 CI 就一直处于 pending 状态，并提示「This job is stuck, because the project doesn't have any runners online assigned to it.」这时需要在 Gitlab 项目页面修改一下 Runner 的配置。

![进入 Runner 配置](/assets/enter-runner-config.png)
![勾选可以处理未打 tag 的 commit](/assets/commit-without-tags.png)

**二、以普通用户身份注册的 Gitlab 服务不会在后台运行，此时需要手动执行 `gitlab-runner run` 命令**

如果是超级用户用 `sudo gitlab-runner register` 注册的服务，会在后台运行，不需要执行上述命令。
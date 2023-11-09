---
title : "使用您自己的aws账号"
weight : 12
---

::alert[仅当您自己运行workshop时才完成此部分。如果您参加*AWS*举办的活动（例如 re\:Invent、Kubecon、Immersion Day 等），请转到 [使用aws活动提供的账号](/account/aws-event-account)]{header="Warning" type="warning"}

您的账户必须有启动EC2 权限。

* 如果您还没有 AWS 账户：[单击此处立即创建一个](https://aws.amazon.com/getting-started/)
* 拥有 AWS 账户后，可以创建一个具有 AWS 账户管理员访问权限的 IAM 用户身份执行剩余的步骤： [创建新的 IAM 用户](https://console.aws.amazon.com/iam/home?#/users$new)
* 输入用户详细信息：
  ![account_own_iam](/static/account_own_iam.png)
* 附加 AdministratorAccess IAM 策略：
  ![account_own_policy](/static/account_own_policy.png)
* 点击创建新用户：
  ![account_own_user](/static/account_own_user.png)
* 记下登录 URL 并保存：
  ![account_own_login](/static/account_own_login.png)
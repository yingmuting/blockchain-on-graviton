---
title : "Ton"
weight : 34
---

# 选择EC2机型

参考 https://docs.ton.org/participate/run-nodes/full-node 运行Ton Full Node，使用 mytonctrl。

硬件要求：
* RAM: at least 64 GB
* CPU: at least 8 cores
* Storage: at least 512 GB SSD

本次实验使用r7g.4xlarge（arm架构，16 core，128GB），如果对应的region没有r7g，选择r6g.4xlarge，对比 r6i.4xlarge（x86架构），价格更低。

* r7g.4xlarge
* Ubuntu 22.04, 800GB, GP3

public IP

创建一个EC2 SSH密钥，比如ton.pem，下载到本机，并执行 
:::code{showCopyAction=true showLineNumbers=true}
chmod 400 ton.pem
ssh -i ton.pem ubuntu@public IP
:::

# 安装软件
:::code{showCopyAction=true showLineNumbers=true}
sudo apt update
wget https://raw.githubusercontent.com/ton-blockchain/mytonctrl/master/scripts/install.sh
sudo bash install.sh -m full
弹出界面选择 <ok> 即可
重启
sudo reboot
重新登录服务器
ssh -i ton.pem ubuntu@public IP
:::

:::code{showCopyAction=true showLineNumbers=true}
mytonctrl
参考 https://github.com/ton-blockchain/mytonctrl/blob/master/docs/en/manual-ubuntu.md 查看常见命令
status
查看节点当前运行状态，同步进度
:::

![ton_status](/static/ton_status.png)

关注 Mytoncore status，Local validator status，Local validator out of sync（开始会比较高，低于20s后同步，再次查询status会很快，因为直接查询当前节点）

# 监控

通过aws console 查看ec2的监控指标
---
title : "Ethereum"
weight : 33
---

# 选择EC2机型

参考 https://ethereum.org/zh/developers/docs/nodes-and-clients/run-a-node/ 运行ETH Node ，本实验使用eth-docker。

硬件要求：https://eth-docker.net/Usage/Hardware

* RAM: 32 GB
* CPU: 4 core CPU
* Storage: 2TB SSD (make sure it is extendable)

本次实验使用m7g.2xlarge，如果当前region还没有m7g.2xlarge，选择m6g.2xlarge

* m7g.2xlarge
* Ubuntu 22.04, 2TB, GP3, 16000 IOPS, 250MB/s

public IP, 安全组打开 ports 30303 and 9000 and 22

创建一个EC2 SSH密钥，比如eth.pem，下载到本机，并执行 
:::code{showCopyAction=true showLineNumbers=true}
chmod 400 eth.pem
ssh -i eth.pem ubuntu@public IP
:::

根据不同ETH客户端和性能要求的场景，进行Graviton完整的性能测试，参见blog：https://aws.amazon.com/cn/blogs/database/choose-aws-graviton-and-cloud-storage-for-your-ethereum-nodes-infrastructure-on-aws/

# 安装软件

## 安装geth
:::code{showCopyAction=true showLineNumbers=true}
sudo add-apt-repository ppa:ethereum/ethereum
sudo apt-get update -y
sudo apt-get upgrade -y
sudo apt-get install ethereum -y
弹出过期包时，选择 cancel 即可
geth -version
:::

## 使用geth创建一个ETH钱包（钱包address仅用于本次实验）
:::code{showCopyAction=true showLineNumbers=true}
geth account new
随意输入一个密码
复制钱包 Public address of the key，用于后续步骤（以 0x开头的字符）
:::

## 安装docker-ce
:::code{showCopyAction=true showLineNumbers=true}
sudo apt-get update
sudo apt-get -y install ca-certificates curl gnupg lsb-release whiptail
sudo mkdir -p /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo echo \
"deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
$(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt-get update
sudo apt-get install -y docker-ce docker-ce-cli containerd.io docker-compose-plugin
:::

## 安装eth-docker
:::code{showCopyAction=true showLineNumbers=true}
cd ~ && git clone https://github.com/eth-educators/eth-docker.git && cd eth-docker
./ethd install  输入 yes，yes
source ~/.profile
exit 退出，重新ssh服务器
:::

## 配置eth-docker
:::code{showCopyAction=true showLineNumbers=true}
./ethd config
键盘上下选择Gorli Testnet，回车
选择 Ethereum RPC node - consensus and execution client，回车
选择 Lighthouse (Rust) - consensus client，回车
选择 Geth (Go)，回车
Checkpoint Sync 选择Yes，回车
Configure checkpoint URL 直接回车
MEV Boost，键盘左右选择No，回车
Grafana 选择 Yes，回车
Configure fee recipient 复制前面步骤中geth创建的钱包address，右键paste，回车
:::


## 启动eth-docker
:::code{showCopyAction=true showLineNumbers=true}
./ethd up
查看日志
./ethd logs -f --tail 50
:::


# 监控

## 本机访问eth-docker安装的Grafana Dashboard
:::code{showCopyAction=true showLineNumbers=true}
通过ssh 隧道，本机直接访问dashboard，建议生产环境使用proxy方式
ssh -i eth.pem -L 8888:localhost:3000 ubuntu@public IP
:::

1.本机浏览器访问：http://localhost:8888 ，使用admin，admin登录，设置密码

2.点击 dashboard，打开 geth dashboard，查看blockchain同步进度

![eth1-dashboard](/static/eth1.png)


# 机型对比

**先说重点**，机型性价比从高到低推荐： m7g >  m6g > m6i > m5 ，生产环境建议优先选择m7g进行节点部署。
* m7g.2xlarge(arm graviton3), m6g.2xlarge(arm graviton2), m6i.2xlarge(6代x86), m5.2xlarge(5代x86)

cloudwatch监控（从cpu，network性能上看出，m7g>m6i,m6g>m5）：

m7g:
![m7g-cw](/static/eth1-cw.png)

m6g:
![m6g-cw](/static/eth2-cw.png)

m6i:
![m6i-cw](/static/eth3-cw.png)

m5:
![m5-cw](/static/eth4-cw.png)


geth dashboard监控（blockchain区块保持同步，从execution，validation，commit上看avg耗时，m7g>m6i>m6g,m5）：

m7g:
![m7g-dashboard](/static/eth1.png)

m6g:
![m6g-dashboard](/static/eth2.png)

m6i:
![m6i-dashboard](/static/eth3.png)

m5:
![m5-dashboard](/static/eth4.png)

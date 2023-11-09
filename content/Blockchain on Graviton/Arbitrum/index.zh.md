---
title : "Arbitrum"
weight : 35
---

# 选择EC2机型

参考 https://developer.arbitrum.io/node-running/how-tos/running-a-full-node 运行Arbitrum Full Node。

硬件要求：
* RAM: 4-8 GB
* CPU: 2-4 core CPU
* Storage: Minimum 1.2TB SSD (make sure it is extendable)
  Estimated Growth Rate: around 3 GB per day

本次实验使用c7g.xlarge（arm架构，4 core，8GB），如果对应的region没有c7g，选择c6g.xlarge，对比 c6i/c6a/t3（x86架构），性价比更高。

* c7g.xlarge
* 操作系统 Amazon linux 2023, EBS 1500GB, GP3, 6000IOPS, 500Mbps 

public IP

创建一个EC2 SSH密钥，比如arbitrum.pem，下载到本机，并执行 
:::code{showCopyAction=true showLineNumbers=true}
chmod 400 arbitrum.pem
ssh -i arbitrum.pem ec2-user@public IP
:::

# 安装软件
:::code{showCopyAction=true showLineNumbers=true}
sudo yum update
sudo yum -y install docker
sudo service docker start
sudo usermod -a -G docker $USER
sudo chkconfig docker on

sudo curl -L https://github.com/docker/compose/releases/latest/download/docker-compose-$(uname -s)-$(uname -m) -o /usr/local/bin/docker-compose
sudo chmod +x /usr/local/bin/docker-compose
docker-compose version

重启
sudo reboot
重新登录服务器
ssh -i arbitrum.pem ec2-user@public IP
:::

:::code{showCopyAction=true showLineNumbers=true}
cd /home/ec2-user
mkdir arbitrum
chmod -fR 777 /home/ec2-user/arbitrum

运行节点
docker run -it  -v /home/ec2-user/arbitrum:/home/user/.arbitrum -p 0.0.0.0:8547:8547 -p 0.0.0.0:8548:8548 -d --restart unless-stopped offchainlabs/nitro-node:v2.0.14-2baa834 --l1.url https://eth.llamarpc.com --l2.chain-id=42161 --http.api=net,web3,eth,debug --http.corsdomain=* --http.addr=0.0.0.0 --http.vhosts=* --init.url="https://snapshot.arbitrum.foundation/arb1/nitro-pruned.tar" --log-level=5
其中参数参考官方文档，重点参数如下：
--l2.chain-id :
从 https://developer.arbitrum.io/node-running/node-providers#rpc-endpoints 获取
--l1.url：
使用自己的eth节点，或者使用节点提供商的节点，本次实验使用免费的节点（不保证稳定性），从 https://chainlist.org/chain/1 获取
:::

查看日志：
:::code{showCopyAction=true showLineNumbers=true}
docker ps
docker logs -f --tail 100 $containerName
:::

# 监控

查看区块同步进度：
:::code{showCopyAction=true showLineNumbers=true}
curl 'http://localhost:8547/' \
--header 'Content-Type: application/json' \
-d '{
"jsonrpc":"2.0",
"method":"eth_syncing",
"params":[],
"id":1
}'
:::

通过aws console 查看ec2和ebs的监控指标
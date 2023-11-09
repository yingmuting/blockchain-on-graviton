---
title : "Polygon"
weight : 32
---

# 选择EC2机型

polygon的硬件要求，参考 https://wiki.polygon.technology/docs/operate/technical-requirements
* RAM: 16-32 GB
* CPU: 8-16 core CPU
* Storage: 8TB+ SSD (make sure it is extendable)
硬件要求建议定期关注更新，尤其是Storage。

本次实验使用基于Graviton的ARM架构的m7g/m6g，性价比对比t3, m5有40%左右提升。

参考Polygon DevOps Director在AWS Graviton Customers中对从x86-based迁移到Graviton-based的描述 ：https://aws.amazon.com/cn/ec2/graviton/customers/

> "Our blockchain nodes, web applications, and automation are built using Golang, Nodejs, Python, and Rust. Most of our workloads are processing large scale transaction volumes. As early as 2021, we’ve seen large growth in transaction volumes, so we needed to increase processing performance while maintaining costs. When we benchmarked our applications on Graviton-based Amazon EC2 C7g, C6g, and R6g instances against comparable x86-based Amazon EC2 instances, we observed 30% higher performance and throughput, along with 30% lower costs. We have custom blockchain nodes software that extract performance from the CPU architecture. The availability of arm64 based images for our software resulted in a seamless migration to Graviton-based instances. From benchmarking to production, migration took us about 2 months. We also use Graviton-based instances across many workloads and AWS services, including Amazon RDS, Amazon OpenSearch Service, and Amazon DocumentDB. We are planning to scale all of our upcoming services and applications to run primarily on Graviton."

**Vince Reed, DevOps Director, Polygon Technology**

# 使用快照加速节点同步

参考官方文档下载快照：https://wiki.polygon.technology/docs/pos/reference/snapshot-instructions-heimdall-bor/

节点下载快照阶段使用大机型m7g.8xlarge，如果对应region没有则使用m6g.8xlarge，可以大幅缩短快照处理时间，快照处理结束后，在线修改实例类型为2xlarge，节省成本。
* m6g.8xlarge 32core 128GB
* 10TB gp3 16000 iops 1000M/s
* ubuntu 22.04
* public IP, 安全组打开 ports 22，30303，26656

## 1.下载快照（mainnet主网预计需要一天时间）

ssh 连接到EC2

修改脚本
:::code{showCopyAction=true showLineNumbers=true}
curl -o snapdown.sh https://snapshot-download.polygon.technology/snapdown.sh
vi snapdown.sh
# 根据需要调整脚本中aria2c的参数 -x16 -s16，增加参数--max-concurrent-downloads=16
aria2c -x16 -s16 --max-tries=0 --save-session-interval=60 --save-session=$client-$network-failures.txt --max-concurrent-downloads=16 --retry-wait=3 --check-integrity=$checksum -i $client-$network-parts.txt
aria2c -x16 -s16 --max-tries=0 --save-session-interval=60 --save-session=$client-$network-failures.txt --max-concurrent-downloads=16 --retry-wait=3 --check-integrity=$checksum -i $client-$network-failures.txt
:::
下载快照
:::code{showCopyAction=true showLineNumbers=true}
# 下载heimdall快照
nohup bash snapdown.sh --network mainnet --client heimdall --extract-dir ~/snapshots/heimdall &
tail -30f nohup.out
# 下载bor快照，可以和heimdall快照同时下载
nohup bash snapdown.sh --network mainnet --client bor --extract-dir ~/snapshots/bor &
tail -30f nohup.out
# 如果日志显示下载中断，删除已经下载的目录，调整aria2c下载参数
:::
如果日志显示join分片未结束，可以修改脚本，注释掉 aria2c下载部分，直接执行最后两部分的join和tar

## 2.（可选）对已经下载快照的EC2制作镜像AMI

通过制作镜像AMI，可以方便后续从AMI启动新机器，减少下载快照的时间。（制作AMI时间预计6小时）

从以上共享的AMI启动新的EC2，选择VPC，公有子网，安全组。

## 3.修改EC2实例类型
由于快照已经下载完成，降低实例类型以节省成本，登录EC2控制台，停止实例，修改实例类型为 m7g.2xlarge或m6g.2xlarge，启动实例，重新ssh登录。

# 运行Full Node
polygon的官方wiki会更新一些内容，并且频率不低，请定期关注官方wiki以获取最新资讯。

https://wiki.polygon.technology/docs/pos/operate/node/full-node-deployment/

## 安装Heimdall和bor
:::code{showCopyAction=true showLineNumbers=true}
sudo apt-get update
sudo apt-get install build-essential
curl -L https://raw.githubusercontent.com/maticnetwork/install/main/heimdall.sh | bash -s -- v0.3.4 mainnet sentry 
heimdalld version --long
# bor
curl -L https://raw.githubusercontent.com/maticnetwork/install/main/bor.sh | bash -s -- v0.4.0 mainnet sentry
bor version
:::

按照官方文档处理快照（https://wiki.polygon.technology/docs/pos/reference/snapshot-instructions-heimdall-bor/）
:::code{showCopyAction=true showLineNumbers=true}
sudo rm -rf /var/lib/heimdall/data
sudo rm -rf /var/lib/bor/data/bor/chaindata
sudo mv /home/ubuntu/snapshots/heimdall /home/ubuntu/snapshots/data
sudo mv /home/ubuntu/snapshots/bor /home/ubuntu/snapshots/chaindata

sudo ln -s /home/ubuntu/snapshots/data /var/lib/heimdall/data
sudo mkdir -pv /var/lib/bor/data/bor/chaindata
sudo ln -s /home/ubuntu/snapshots/chaindata /var/lib/bor/data/bor
:::

:::code{showCopyAction=true showLineNumbers=true}
# 从网站查找可用的seed：https://www.allthatnode.com/polygon.dsrv
sudo sed -i 's|^seeds =.*|seeds = "f4f605d60b8ffaaf15240564e58a81103510631c@159.203.9.164:26656,4fb1bc820088764a564d4f66bba1963d47d82329@44.232.55.71:26656"|g' /var/lib/heimdall/config/config.toml
sudo chown -R heimdall /var/lib/heimdall

sudo sed -i 's|.*\[p2p.discovery\]|  \[p2p.discovery\] |g' /var/lib/bor/config.toml
sudo sed -i 's|.*bootnodes =.*|    bootnodes = ["enode://b8f1cc9c5d4403703fbf377116469667d2b1823c0daf16b7250aa576bacf399e42c3930ccfcb02c5df6879565a2b8931335565f0e8d3f8e72385ecf4a4bf160a@3.36.224.80:30303", "enode://8729e0c825f3d9cad382555f3e46dcff21af323e89025a0e6312df541f4a9e73abfa562d64906f5e59c51fe6f0501b3e61b07979606c56329c020ed739910759@54.194.245.5:30303"]|g' /var/lib/bor/config.toml
sudo chown -R bor /var/lib/bor

# Update service config user permission
sudo sed -i 's/User=heimdall/User=root/g' /lib/systemd/system/heimdalld.service
sudo sed -i 's/User=bor/User=root/g' /lib/systemd/system/bor.service
:::

## 启动服务
:::code{showCopyAction=true showLineNumbers=true}
sudo systemctl daemon-reload
sudo service heimdalld start
:::
查看log
:::code{showCopyAction=true showLineNumbers=true}
journalctl -u heimdalld.service -f
:::
curl localhost:26657/status
:::code{showCopyAction=true showLineNumbers=true}
...
  "sync_info": {
    "latest_block_hash": "AFB8F4C669FC03DB41BFD35CFA328612B2127D4975FF72C82233D8BCE5D8AAE6",
    "latest_app_hash": "B028554EFFF37C6576EA101493533E450760E8DD6B06836849A6A92FA864534E",
    "latest_block_height": "15370413",
    "latest_block_time": "2023-08-30T00:21:36.042505001Z",
    "catching_up": true
  }
...
:::
查看当前区块高度，当在输出中，catch_up 值为 false，Heimdall 同步后，才能运行以下命令启动bor：
:::code{showCopyAction=true showLineNumbers=true}
sudo service bor start
:::
查看log
:::code{showCopyAction=true showLineNumbers=true}
journalctl -u bor.service -f
:::
查看区块高度
:::code{showCopyAction=true showLineNumbers=true}
bor status

curl 'localhost:8545/' \
--header 'Content-Type: application/json' \
-d '{
"jsonrpc":"2.0",
"method":"eth_syncing",
"params":[],
"id":1
}'
:::
返回已同步的 currentBlock 以及 highestBlock。如果节点已经同步，我们应该返回 false。
:::code{showCopyAction=true showLineNumbers=true}
{"jsonrpc":"2.0","id":1,"result":false}
:::

# 监控EC2

heimdall和bor启动时，查看cloudwatch监控cpu。

监控heimdall和bor的区块同步延迟（仅供参考，ubuntu系统会有区别）：
:::code{showCopyAction=true showLineNumbers=true}
对于操作系统 Amazon Linux 2023，启动crond服务
sudo yum install cronie -y
sudo systemctl enable crond.service
sudo systemctl start crond.service
sudo systemctl status crond.service

sudo yum install -y jq

# 通过脚本上传自定义指标到cloudwatch，注意需要给ec2关联IAM Role，赋予 CloudWatchFullAccess 权限
1.创建IAM Role，服务选择ec2，添加权限选择 CloudWatchFullAccess，输入角色名称，创建角色
2.修改ec2的IAM Role，选择刚创建的角色

vi /home/ec2-user/cw.sh

#!/bin/bash
date=`date`
echo $date >> /home/ec2-user/cw.log
ec2InstanceId=$(ec2-metadata --instance-id | cut -d " " -f 2)
ec2InstanceType=$(ec2-metadata --instance-type | cut -d " " -f 2)
curl localhost:26657/status > heimdall.json
latest_block_height=$(cat heimdall.json | jq '.result.sync_info.latest_block_height' --raw-output)
catching_up=$(cat heimdall.json | jq '.result.sync_info.catching_up' --raw-output)
catched_up=0
if $catching_up
then
  catched_up=0
else
  catched_up=1
fi
aws cloudwatch put-metric-data --metric-name heimdall-latest_block_height --namespace polygon --unit Count --value $latest_block_height --dimensions InstanceId=$ec2InstanceId,InstanceType=$ec2InstanceType
aws cloudwatch put-metric-data --metric-name heimdall-catched_up --namespace polygon --unit Count --value $catched_up --dimensions InstanceId=$ec2InstanceId,InstanceType=$ec2InstanceType

curl 'localhost:8545/' \
--header 'Content-Type: application/json' \
-d '{
"jsonrpc":"2.0",
"method":"eth_syncing",
"params":[],
"id":1
}' > bor.json
sync_result=$(cat bor.json | jq '.result' --raw-output)
synced=0
if [ "$sync_result" = "false" ]
then
  synced=1
else
  synced=0
fi
aws cloudwatch put-metric-data --metric-name bor-synced --namespace polygon --unit Count --value $synced --dimensions InstanceId=$ec2InstanceId,InstanceType=$ec2InstanceType

# 赋予cw.sh执行权限
chmod 775 cw.sh
crontab -e
# Example of job definition:
# .---------------- minute (0 - 59)
# |  .------------- hour (0 - 23)
# |  |  .---------- day of month (1 - 31)
# |  |  |  .------- month (1 - 12) OR jan,feb,mar,apr ...
# |  |  |  |  .---- day of week (0 - 6) (Sunday=0 or 7) OR sun,mon,tue,wed,thu,fri,sat
# |  |  |  |  |
# *  *  *  *  * command to be executed
*/1 * * * * /bin/sh /home/ec2-user/cw.sh

:::

监控heimdall的当前同步的区块高度：
![ec2-metrics-2](/static/cw-polygon-heimdall-block.png)

监控heimdall是否保持和最新的区块同步：
![ec2-metrics-2](/static/cw-polygon-heimdall-catched-up.png)

监控bor是否保持和最新的区块同步：
![ec2-metrics-2](/static/cw-polygon-bor-synced.png)

通过cloudwatch设置一个 polygon dashboard：
![ec2-metrics-2](/static/cw-polygon-dashboard.png)


# 运行Archive Node

参考官方文档：https://wiki.polygon.technology/docs/pos/operate/node/archive-node/

本次实验使用基于Graviton2的ARM架构的r7g.xlarge，性价比相比r5.xlarge有提升。

* 实例类型 r7g.xlarge
* 存储 10TB, GP3, 16000 iops, 1000M/s
* public IP, 安全组打开 ports 30303 and 26656

:::code{showCopyAction=true showLineNumbers=true}
sudo mkdir -p /mnt/data/polygon/
sudo yum install -y git
sudo yum install -y gcc
sudo yum install -y go
从https://github.com/ledgerwatch/erigon/releases上查询版本，由于本次实验选择EC2实例类型r6g（基于Graviton的arm架构），下载名字中带有linux_arm64的压缩包。
wget https://github.com/ledgerwatch/erigon/releases/download/v2.43.0/erigon_2.43.0_linux_arm64.tar.gz
tar -xvf erigon_2.43.0_linux_arm64.tar.gz
./erigon --version

Archive Node建议也使用snapshot开始，从 https://snapshots.polygon.technology/ 下载最新的archive快照
后台运行wget，本次实验使用mumbai测试网络，请替换快照url为最新日期的：
wget -bqc https://snapshot-download.polygon.technology/erigon-mumbai-archive-snapshot-2023-02-14.tar.gz
下载完成后：
sudo nohup tar -xzvf erigon-mumbai-archive-snapshot-2023-02-14.tar.gz -C /mnt/data/polygon/ &

启动Polygon mumbai测试网络，可以指定相关参数。
主网：--chain=bor-mainnet；
--bor.heimdall= 参考 https://wiki.polygon.technology/docs/operate/erigon-client 中的tips，生产环境使用localhost。
sudo nohup ./erigon --chain=mumbai --datadir="/mnt/data/polygon/" --bor.heimdall="https://heimdall-api-testnet.polygon.technology" --http=false &
:::

## 加快Archive Node同步的技巧
- 使用具有高 IOPS 和 RAM 的机器以获得更快的初始同步，比如选择r7g.2xlarge，存储 8TB, io2，IOPS 16000及以上
- 建议使用内存优化节点以实现更快的同步

# 机型对比
机型性价比从高到低推荐： m7g >  m6g > m6i > m5 ，生产环境建议优先选择m7g进行节点部署。可以在EC2控制台查看机型的价格，通过监控发现CPU利用率。

---
title : "BNBChain"
weight : 31
---

## BNB Chain

在实验中，我们会有两个主要内容

* 在ARM架构的EC2上安装BNB Smart Chain的客户端
* 在ARM架构的EC2上部署BSC Smart Chain的全节点

## ARM机型上的全节点部署

在这个实验中，我们会基于ARM架构的EC2，运行一个BSC Smart Chain的全节点

### BNB Smart Chain全节点部署

1. 机型选择

官方网站对于机型的推荐是
* EC2: m5zn.3xlarge
* Storage: 2 TB of free disk space, solid-state drive(SSD), gp3, 8k IOPS, 250 MB/S throughput, read latency <1ms. (if node is started with snap sync, it will need NVMe SSD)

我们选择以下机型以及配置

* EC2: is4gen.xlarge (4c 24GiB 3750GiB Nitro SSD)
* EBS: 200 GB 

2. 挂载磁盘

因为is4gen.xlarge自带了3.4TB的nvme磁盘，需要挂载后才能使用，所以这一步我们需要先使得nvme磁盘可用
::alert[nvme磁盘拥有高性能的io能力，但是EC2关机或者重启后就会丢失所有数据。使用这种磁盘类型是为了能够追赶最新的区块，行业里面也会使用m5zn.3xlarge配合io2的方案做全节点。对于机型选择需要权衡成本和高可用]{header="Warning" type="warning"}

* 查看磁盘
```
lsblk
```

>你会看到以下输出

```
NAME          MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
nvme0n1       259:0    0  200G  0 disk
|-nvme0n1p1   259:2    0  200G  0 part /
`-nvme0n1p128 259:3    0   10M  0 part /boot/efi
nvme1n1       259:1    0  3.4T  0 disk
```

* 执行以下指令检查磁盘状态，并且创建文件系统

```
sudo file -s /dev/nvme1n1
sudo mkfs -t xfs /dev/nvme1n1
```
* 挂载磁盘
```
sudo mkdir /data
sudo mount /dev/nvme1n1 /data
```
* 给文件夹权限
```
sudo chmod -R 777 /data
```
* 检查
```
lsblk
```
>你会看到以下输出
```
NAME          MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
nvme0n1       259:0    0  200G  0 disk
|-nvme0n1p1   259:2    0  200G  0 part /
`-nvme0n1p128 259:3    0   10M  0 part /boot/efi
nvme1n1       259:1    0  3.4T  0 disk /data
```

3. GETH部署

我们可以直接使用下载ARM版本的方式来安装

```
cd ~/
wget $(curl -s https://api.github.com/repos/bnb-chain/bsc/releases/latest |grep browser_ |grep geth-linux-arm64 |cut -d\" -f4)
mv geth-linux-arm64 geth
chmod -v u+x geth
```

也可以下载源代码编译

```
# compile and install geth
sudo yum install git make go g++ -y
git clone https://github.com/bnb-chain/bsc.git
cd bsc
make geth
cp ~/bsc/build/bin/geth ~/
cd ~/
```

执行以下命令验证geth已经安装完毕
```
./geth --help
```

4. 下载配置文件并解压

```
wget $(curl -s https://api.github.com/repos/bnb-chain/bsc/releases/latest |grep browser_ |grep mainnet |cut -d\" -f4)
unzip mainnet.zip
```

5. 下载历史数据快照安装

推荐使用bnb48的数据源， https://github.com/48Club/bsc-snapshots

* 安装aria2
```
wget https://github.com/q3aql/aria2-static-builds/releases/download/v1.36.0/aria2-1.36.0-linux-gnu-arm-rbpi-build1.tar.bz2
tar jxvf aria2-1.36.0-linux-gnu-arm-rbpi-build1.tar.bz2
cd aria2-1.36.0-linux-gnu-arm-rbpi-build1/
sudo make install
aria2c -h
cd /data
```
* 下载快照数据
下载过程预计在2小时左右
```
aria2c -s4 -x4 -c -k1024M https://snapshots.48.club/geth.local.28656700.tar.lz4 -o local.tar.lz4
```
* 解压数据
```
lz4 -cd local.tar.lz4 | tar xf -
```

6. 启动全节点

```
cd ~/
./geth --config config.toml --datadir /data --cache 8000 --rpc.allow-unprotected-txs --txlookuplimit 0
```

7. （可选）从创世块开始同步数据

* 初始化状态
从创世区块开始，我们假设上述的机器已经配置完毕，然后我们选择使用 /data目录
```
cd ~/
./geth --datadir /data/node-new init genesis.json
```
你会看到和以下类似的输出
```
INFO [05-10|05:22:09.361] Maximum peer count                       ETH=50 LES=0 total=50
INFO [05-10|05:22:09.361] Smartcard socket not found, disabling    err="stat /run/pcscd/pcscd.comm: no such file or directory"
INFO [05-10|05:22:09.361] Set global gas cap                       cap=50,000,000
INFO [05-10|05:22:09.361] Allocated cache and file handles         database=/data/node-new/geth/chaindata cache=16.00MiB handles=16
INFO [05-10|05:22:09.372] Persisted trie from memory database      nodes=15 size=2.32KiB time="31.754µs" gcnodes=0 gcsize=0.00B gctime=0s livenodes=1 livesize=0.00B
INFO [05-10|05:22:09.372] Successfully wrote genesis state         database=chaindata                              hash=0d2184..d57b5b
INFO [05-10|05:22:09.372] Allocated cache and file handles         database=/data/node-new/geth/lightchaindata cache=16.00MiB handles=16
INFO [05-10|05:22:09.382] Persisted trie from memory database      nodes=15 size=2.32KiB time="27.266µs" gcnodes=0 gcsize=0.00B gctime=0s livenodes=1 livesize=0.00B
INFO [05-10|05:22:09.382] Successfully wrote genesis state         database=lightchaindata                         hash=0d2184..d57b5b
```
* 启动全节点
```
./geth --config ./config.toml --datadir /data/node-new --cache 8000 --rpc.allow-unprotected-txs --txlookuplimit 0
```
通过观察 node-new 目录下的bsc.log可以观察到全节点数据同步的相关日志
```
tail -f 10 node-new/bsc.log
```

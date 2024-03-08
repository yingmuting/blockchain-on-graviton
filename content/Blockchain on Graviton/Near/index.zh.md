---
title : "Near"
weight : 36
---

# 选择EC2机型

参考 https://near-nodes.io/rpc 运行Near RPC Node。

本次实验使用m7g.2xlarge（arm架构），如果对应的region没有m7g，选择m6g.2xlarge，相比文当中提到的aws m5.2xlarge（x86架构），价格低20%左右。

* m7g.2xlarge
* RAM: 32 GB
* CPU: 8 cores
* Storage: at least 1000 GB SSD
* Ubuntu 22.04, 1000GB, GP3

public IP

创建一个EC2 SSH密钥，比如near.pem，下载到本机，并执行 
:::code{showCopyAction=true showLineNumbers=true}
chmod 400 near.pem
ssh -i near.pem ubuntu@public IP
:::

# 安装软件
按照文档https://near-nodes.io/rpc/run-rpc-node-without-nearup进行软件安装，本次实验选择testnet
:::code{showCopyAction=true showLineNumbers=true}
# Get Rust in linux and MacOS
curl --proto '=https' --tlsv1.2 https://sh.rustup.rs -sSf | sh
source $HOME/.cargo/env
# Add the wasm toolchain
rustup target add wasm32-unknown-unknown
# developer tools
sudo apt update
sudo apt install -y git binutils-dev libcurl4-openssl-dev zlib1g-dev libdw-dev libiberty-dev cmake gcc g++ python3 docker.io protobuf-compiler libssl-dev pkg-config clang llvm cargo awscli
# 重启
sudo reboot
重新登录服务器
ssh -i near.pem ubuntu@public IP
# clone code
git clone https://github.com/near/nearcore
cd nearcore
git fetch origin --tags
git checkout tags/1.37.0 -b mynode
make release
# init workdir
./target/release/neard --home ~/.near init --chain-id testnet --download-genesis --download-config --boot-nodes ed25519:4k9csx6zMiXy4waUvRMPTkEtAS2RFKLVScocR5HwN53P@34.73.25.182:24567,ed25519:4keFArc3M4SE1debUQWi3F1jiuFZSWThgVuA2Ja2p3Jv@34.94.158.10:24567,ed25519:D2t1KTLJuwKDhbcD9tMXcXaydMNykA99Cedz7SkJkdj2@35.234.138.23:24567,ed25519:CAzhtaUPrxCuwJoFzceebiThD9wBofzqqEMCiupZ4M3E@34.94.177.51:24567
rm ~/.near/config.json
wget https://s3-us-west-1.amazonaws.com/build.nearprotocol.com/nearcore-deploy/testnet/config.json -P ~/.near/
# get data backup
aws s3 --no-sign-request cp s3://near-protocol-public/backups/testnet/rpc/latest .
LATEST=$(cat latest)
aws s3 --no-sign-request cp --no-sign-request --recursive s3://near-protocol-public/backups/testnet/rpc/$LATEST ~/.near/data
# run node
./target/release/neard --home ~/.near run
# 节点正在运行，您可以在控制台中看到日志输出。

:::

# 监控

通过aws console 查看ec2的监控指标
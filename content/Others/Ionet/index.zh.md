---
title : "Ionet GPU"
weight : 51
---


# 选择EC2机型

参考 https://developers.io.net/docs/ignition-rewards-program 官网。

本次实验使用T4单卡，g4dn.xlarge。

* 操作系统 Ubuntu 20.04/22.04, EBS 40GB, GP3

public IP

安全组开放port：
TCP 443 25061 5432 80

UDP 80 443 41641 3478

创建一个EC2 SSH密钥，比如ionet.pem，下载到本机，并执行

chmod 400 ionet.pem

ssh -i ionet.pem ubuntu@public IP


# 安装软件

lsblk

sudo mkfs -t xfs /dev/nvme1n1

sudo mkdir /data

sudo mount /dev/nvme1n1 /data

sudo apt update -y

curl -L https://github.com/ionet-official/io-net-official-setup-script/raw/main/ionet-setup.sh -o ionet-setup.sh

chmod +x ionet-setup.sh && ./ionet-setup.sh

自动 reboot，重新登陆

chmod +x ionet-setup.sh && ./ionet-setup.sh

sudo bash -c 'cat <<EOF > /etc/docker/daemon.json
{
"runtimes": {
"nvidia": {
"path": "nvidia-container-runtime",
"runtimeArgs": []
}
},
"exec-opts": ["native.cgroupdriver=cgroupfs"]
}
EOF'



sudo systemctl restart docker



从官网复制device id和 user id，替换后运行：

curl -L https://github.com/ionet-official/io_launch_binaries/raw/main/launch_binary_linux -o launch_binary_linux

chmod +x launch_binary_linux

./launch_binary_linux —device_id=*5cee341ba417 —user_id=e3d0d* —operating_system="Linux" —usegpus=true --device_name=aws-device-*



查看进程：

docker ps


通过官网 https://developers.io.net/docs/troubleshooting-guide 排查问题。在 https://cloud.io.net/worker/devices 上查看设备状态。

注意，如果要获得奖励，必须按照官网步骤设置一个solana 钱包地址。目前奖励规则参考官网，请注意风险。

# 监控

通过aws console 查看ec2监控指标

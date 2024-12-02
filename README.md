# sd-web-ui-ec2

Run Stable Diffusion WebUI on an EC2 instance

## Quick Start

```
MYIP=$(curl ifconfig.me)
terraform init
terraform apply -var="my_ip=$MYIP/32"
```

## Quick Start with a custom AMI (skipping some steps)

```
MYIP=$(curl ifconfig.me)
MYAMI="SOMEAMITHATYOUHAVE" # change this value
terraform init
terraform apply -var="my_ip=$MYIP/32" -var="custom_ami=$MYAMI"
```

Get the IP address of the EC2 instance: 

```
EC2IP=$(terraform output -raw ip_address)
```

SSH to the instance:

```
ssh -i ~/.ssh/ec2.pem ec2-user@$EC2IP
```

## Steps to get working on the instance

### Install Grid Driver 

AL2

https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/install-nvidia-driver.html#nvidia-GRID-driver

```
sudo yum install -y gcc make
sudo yum update -y
sudo reboot
sudo yum install -y kernel-devel-$(uname -r)
aws s3 cp --recursive s3://ec2-linux-nvidia-drivers/latest/ .
chmod +x NVIDIA-Linux-x86_64*.run
# AL2023 /usr/bin/gcc (not 10-cc)
sudo CC=/usr/bin/gcc10-cc ./NVIDIA-Linux-x86_64*.run --accept-license
nvidia-smi -q | head
sudo touch /etc/modprobe.d/nvidia.conf
echo "options nvidia NVreg_EnableGpuFirmware=0" | sudo tee --append /etc/modprobe.d/nvidia.conf
sudo reboot
```

AL2023

https://repost.aws/articles/ARwfQMxiC-QMOgWykD9mco1w/how-do-i-install-nvidia-gpu-driver-cuda-toolkit-and-optionally-nvidia-container-toolkit-in-amazon-linux-2023-al2023

```
sudo dnf config-manager --add-repo https://developer.download.nvidia.com/compute/cuda/repos/amzn2023/x86_64/cuda-amzn2023.repo
sudo dnf module install -y nvidia-driver:latest-dkms
sudo dnf install -y cuda-toolkit
sudo reboot
```

Docker in AL2023 for pinning

```
#!/bin/bash
dnf check-release-update
sudo dnf update -y
sudo dnf install -y dkms kernel-devel kernel-modules-extra
sudo systemctl enable dkms

cd /tmp
if (arch | grep -q x86); then
  sudo dnf install -y nvidia-release
  sudo dnf install -y nvidia-driver
  sudo dnf install -y cuda-toolkit
else
  sudo dnf install -y vulkan-devel libglvnd-devel elfutils-libelf-devel xorg-x11-server-Xorg
  curl -L -O https://developer.download.nvidia.com/compute/cuda/12.6.3/local_installers/cuda_12.6.3_560.35.05_linux_sbsa.run
  chmod +x ./cuda*.run
  sudo ./cuda_*.run --driver --toolkit --tmpdir=/var/tmp --silent
fi

if (! dnf search nvidia | grep -q nvidia-container-toolkit); then
  sudo dnf config-manager --add-repo https://nvidia.github.io/libnvidia-container/stable/rpm/nvidia-container-toolkit.repo
fi
sudo dnf install -y nvidia-container-toolkit

sudo dnf install -y docker
sudo systemctl enable docker
sudo usermod -aG docker ec2-user
sudo usermod -aG docker ssm-user

sudo nvidia-ctk runtime configure --runtime=docker
sudo systemctl restart docker

sudo reboot
```

ami-0453ec754f44f9a4a

[root@ip-10-0-3-113 ~]# docker run -it --rm --runtime=nvidia --gpus all -v /tmp/snapshot.yaml:/snapshot.yaml ghcr.io/kjenney/cm-pkg:v0.11


### Install Python 3.10

```
sudo yum -y update
sudo yum -y groupinstall "Development Tools"
sudo yum remove -y openssl openssl-devel xz-devel sqlite-devel
sudo yum -y install bzip2-devel libffi-devel openssl11 openssl11-devel
wget https://www.python.org/ftp/python/3.10.0/Python-3.10.0.tgz
tar xzf Python-3.10.0.tgz
cd Python-3.10.0
sed -i 's/PKG_CONFIG openssl /PKG_CONFIG openssl11 /g' configure
sudo ./configure --enable-optimizations
sudo yum install -y lzma xz-devel
sudo make altinstall
```

### Install webui

```
sudo yum install -y wget git gperftools-libs libglvnd-glx 
wget https://raw.githubusercontent.com/AUTOMATIC1111/stable-diffusion-webui/master/webui.sh
chmod +x webui.sh
set use_venv to 0 for package issue
pip3 install --upgrade pip
pip3 install packaging
./webui.sh --xformers --share --listen --enable-insecure-extension-access
```

## Using the Stable Diffusion WebUI

Open the Gradio URL that is provided in the script. This is the web UI that you just installed!

Install extensions using the UI:

https://github.com/Mikubill/sd-webui-controlnet

## Using ComfyUI

You can deploy ComfyUI in much the same way on the EC2 instance.

You may need to run: `pip3.10 install pylzma`.

To start: `python3.10 main.py --listen 0.0.0.0`



## Cleaning up 

```
terraform destroy 
```

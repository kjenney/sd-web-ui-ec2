# sd-web-ui-ec2

Run Stable Diffusion WebUI on an EC2 instance. Connect to the EC2 instance using Systems Manager Session Manager.

## Quick Start

```
terraform init
terraform apply
```

## Quick Start with a custom AMI (skipping some steps) and custom instance type

```
INSTANCE_TYPE="g4dn.2xlarge"
MYAMI="SOMEAMI" # change this value to the name of an AMI - can be regex
terraform init
terraform apply -var="ami_lookup=$MYAMI" -var="instance_type=$INSTANCE_TYPE"
```

## Connecting to the instance

Use Session Manager to connect to the instance: https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/connect-with-systems-manager-session-manager.html 

## Steps to get working on the instance

### Install Grid Driver 

https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/install-nvidia-driver.html#nvidia-GRID-driver

```
sudo yum install -y gcc make
sudo yum update -y
sudo reboot
sudo yum install -y kernel-devel-$(uname -r)
aws s3 cp --recursive s3://ec2-linux-nvidia-drivers/latest/ .
chmod +x NVIDIA-Linux-x86_64*.run
sudo CC=/usr/bin/gcc10-cc ./NVIDIA-Linux-x86_64*.run --accept-license
nvidia-smi -q | head
sudo touch /etc/modprobe.d/nvidia.conf
echo "options nvidia NVreg_EnableGpuFirmware=0" | sudo tee --append /etc/modprobe.d/nvidia.conf
sudo reboot
```

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

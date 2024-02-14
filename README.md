# ec2 

Run Stable Diffusion models on an EC2 instance

```
MYIP=$(curl ifconfig.me)
terraform init
terraform apply -var="my_ip=$MYIP/32"
```

ssh -i ~/.ssh/ec2.pem ec2-user@44.211.214.201

## Steps to get working on the instance

### Install NVIDIA Driver
yum update -y
yum install -y gcc make git gcc-c++
yum install -y kernel-devel-$(uname -r)
aws s3 cp --recursive s3://ec2-linux-nvidia-drivers/latest/ .
chmod +x NVIDIA-Linux-x86_64*.run
sh CC=/usr/bin/gcc10-cc ./NVIDIA-Linux-x86_64*.run --accept-license
sudo touch /etc/modprobe.d/nvidia.conf
echo "options nvidia NVreg_EnableGpuFirmware=0" | sudo tee --append /etc/modprobe.d/nvidia.conf
reboot

### Install Python 3.10
sudo yum -y update
sudo yum -y groupinstall "Development Tools"
sudo yum remove -y openssl openssl-devel
sudo yum -y install openssl-devel bzip2-devel libffi-devel openssl11 openssl11-devel
wget https://www.python.org/ftp/python/3.10.0/Python-3.10.0.tgz
tar xzf Python-3.10.0.tgz
cd Python-3.10.0
sed -i 's/PKG_CONFIG openssl /PKG_CONFIG openssl11 /g' configure
sudo ./configure --enable-optimizations
sudo make altinstall

### Install Python 3.11 - Use Pyenv
curl https://pyenv.run | bash
echo "export PYENV_ROOT=\"$HOME/.pyenv\"" >> ~/.bash_profile
echo "[[ -d $PYENV_ROOT/bin ]] && export PATH=\"$PYENV_ROOT/bin:$PATH\"" >> ~/.bash_profile
echo "eval \"$(pyenv init -)\"" >> ~/.bash_profile

pyenv local 3.11.6

### Install webui
sudo yum install wget git python3 gperftools-libs libglvnd-glx 
wget https://raw.githubusercontent.com/AUTOMATIC1111/stable-diffusion-webui/master/webui.sh
chmod +x webui.sh
set use_venv to 0 for package issue
override python_cmd to use pyenv
./webui.sh --xformers --share --listen

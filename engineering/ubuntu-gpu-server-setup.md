---
description: 2020/06/20
---

# Ubuntu GPU Server Setup

A GPU server is notoriously difficult to set up mainly due to two things: installing the OS itself, and the Nvidia driver installation.

However, there's actually a very easy way to do it - that is to use a **Ubuntu Server** as opposed to Ubuntu Desktop. This forgoes the GUI, which is unnecessary for a server anyway, and in doing so simplifies the Nvidia driver installation.

In fact, this can all be done quickly and easily and I've set up GPU servers numerous times, so here's the guide:

Estimated time: &lt; 1 hour

1. Download the “alternative” **Ubuntu Server** image from Ubuntu: [Alternative downloads \| Ubuntu](https://ubuntu.com/download/alternative-downloads).
2. [Create a bootable USB stick on macOS \| Ubuntu](https://ubuntu.com/tutorials/tutorial-create-a-usb-stick-on-macos?_ga=2.258246282.1660341004.1581753207-683286975.1581753207#1-overview)
3. Go to BIOS, disable `secure boot`. Then boot UEFI. Install Ubuntu, overwrite full partition, add SSH Server. Finish installation and login.
4. You can now `ssh` in with password. Login and install nvidia driver. Since secure boot is disable, nvidia installation should go smoothly.

```bash
# if you install ubuntu server no GUI, ok
sudo add-apt-repository ppa:graphics-drivers
sudo apt-get update
sudo apt install ubuntu-drivers-common
ubuntu-drivers devices
# this will show a list of drivers
sudo apt-get install nvidia-driver-440
# reboot required later

# install Docker
sudo apt install apt-transport-https ca-certificates curl software-properties-common
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu focal stable"
sudo apt update
apt-cache policy docker-ce
sudo apt install docker-ce
sudo usermod -aG docker ${USER}

# install nvidia-container-toolkit to run Docker with GPU
distribution=$(. /etc/os-release;echo $ID$VERSION_ID)
curl -s -L https://nvidia.github.io/nvidia-docker/gpgkey | sudo apt-key add -
curl -s -L https://nvidia.github.io/nvidia-docker/$distribution/nvidia-docker.list | sudo tee /etc/apt/sources.list.d/nvidia-docker.list

sudo apt-get update && sudo apt-get install -y nvidia-container-toolkit
sudo systemctl restart docker
```

1. [Configure WiFi Connections \| NetworkManager documentation](https://docs.ubuntu.com/core/en/stacks/network/network-manager/docs/configure-wifi-connections).

```bash
# if you have wifi
sudo apt-get install network-manager
sudo /etc/init.d/network-manager restart
nmcli d
nmcli r wifi on
nmcli d wifi list
nmcli d wifi connect my_wifi password <password>
```

1. setup ssh keys, authorized keys, sshd\_config:

```bash
ssh-keygen
nano ~/.ssh/authorized_keys
chmod 400 ~/.ssh/authorized_keys
sudo nano /etc/ssh/sshd_config
# set: PasswordAuthentication no
sudo systemctl restart sshd
```

1. [install zsh](https://github.com/ohmyzsh/ohmyzsh/wiki/Installing-ZSH#ubuntu-debian--derivatives-windows-10-wsl--native-linux-kernel-with-windows-10-build-1903) and change shell \(optional\):

```bash
sudo apt install zsh
chsh -s $(which zsh)
# then restore your dotfiles from git
```

1. Install libraries \(optional\):

```bash
# xvfb, roboschool, orca dependencies
sudo apt-get install xvfb libpcre16-3 libgtk2.0-0 libxss1 libgconf2-4 libnss3
# install glances
curl -L https://bit.ly/glances | /bin/bash
```

1. Reboot:

```bash
sudo reboot now
```


---
layout: post
title: "Reducing memory and disk usage for Jetson Nano"
date: 2022-03-17 18:34:23 -0600
categories: Jetson
excerpt_separator: <!--more-->
---
The Jetson nano, for reducing the memory footprint and disk usage along with increasing the memory through a swap partition and allow you to perform better in memory intensive tasks. 

<!--more-->

Type the following in the terminal

```bash
sudo apt update
sudo apt autoremove -y
sudo apt clean
```

Remove LibreOffice and Thunderbird
```bash
sudo apt remove thunderbird libreoffice-* -y
```

Remove Ubuntu Desktop and GUI
```bash
sudo apt remove --purge ubuntu-desktop
sudo apt-get purge gnome-shell ubuntu-wallpapers-bionic light-themes chromium-browser* libvisionworks libvisionworks-sfm-dev -y
sudo apt-get autoremove -y
sudo apt clean -y
```

Bonus: create swap file.

```bash
sudo fallocate -l 4.0G /mnt/4G.swap
sudo chmod 600 /mnt/4G.swap
sudo mkswap /mnt/4G.swap
sudo swapon /mnt/4G.swap
```

To create swap memory on every reboot

```bash
sudo vim /etc/fstab
```

Add the following line.

```bash
/mnt/4G.swap swap swap defaults 0 0
```
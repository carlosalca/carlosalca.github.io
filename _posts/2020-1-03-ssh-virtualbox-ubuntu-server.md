# Setting an Ubuntu Server virtual machine and VirtualBox for SSH connection from host machine

The other day I wanted to prepare a set up to start playing with Ansible. For this purpose I installed a couple of Ubuntu Servers on VirtualBox as the machines which were going to be controlled by the ansible controller. Ansible controls the managed machines through a SSH connection on port 22, so I needed to be able to SSH the to recently installed Ubuntu Servers. In this post I am going to explain what I did to configure Ubuntu and VirtualBox to be able to connect them via SSH and, thus, Ansible. 

*Everything explained in this post has been done on a Ubuntu 18.04.3 LTS, VirtualBox 6.1 and Ubuntu Server 18.04.3*

## Setting up the VirtualBox

As I installed VirtualBox only for this, its configuration was clear. So the first thing I had to do was to create o host network for the servers. This can be done in *File > Host Network Manager (Ctrl + H)* and by clicking on the *Create* button.

Once there is a host network created, it can be configured as a secondary o main network interface for the server. So, with the virtual machine turnned off, enter the network settings (*Right click on the virtual machine > Settings (Ctrl + S) > Network*). Then, click on the desired adapter to configure and make sure it is enabled, and select the previously created Host-only Adapter and clikc on Ok. 

Now, you can start the virtual machines with a host network adapter.

## Setting up the Ubuntu Server

The first thing to do is to see which network interfaces are being used with `ifconfig`

![_config.yml]({{ site.baseurl }}/images/2020-1-03-ssh-virtualbox-ubuntu-server/ifconfig-1.png)

The interface shown on the screeshot is the default interafce after installing the machine with NAT configuration. So, we need to identify the other interface for the just added adapter. For doing this, execute this command and you will see a list of the used or not interfaces: 

```bash
carlos@node:~$ ls -l /sys/class/net/
total 0
lrwxrwxrwx 1 root root 0 Jan 10 21:45 enp0s3 -> ../../devices/pci0000:00/0000:00:03.0/net/enp0s3
lrwxrwxrwx 1 root root 0 Jan 10 21:45 enp0s8 -> ../../devices/pci0000:00/0000:00:08.0/net/enp0s8
lrwxrwxrwx 1 root root 0 Jan 10 21:45 lo -> ../../devices/virtual/net/lo

```

Once the interface needed is located, note its name and modify the file in charge of the network configuration. I used to go to the */etc/network/interfaces*, but in this system this seems it has been replaced by */etc/netplan* configuration files. You can modify the existing one or create a new one, I decided to add the new interface to the existing one: 

![_config.yml]({{ site.baseurl }}/images/2020-1-03-ssh-virtualbox-ubuntu-server/netplan.png)


Finally, the only thing left to do is to apply the new changes in the network configuration file:

```bash
sudo netplan generate && sudo netplan apply
```

Is that all ? No, you still need to install the SSH server if you did not do it before: 

```bash
sudo apt update
sudo apt install openssh-server
```

And now yes, it is all. You only have to reboot your system to see the changes applied and be able to connect to it via SSH. 


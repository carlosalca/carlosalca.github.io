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


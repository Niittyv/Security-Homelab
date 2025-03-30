# Security-Homelab

I've decided to build a VirtualBox internal network to simulate a corporate LAN network environment. The VM devices inside the internal network can only communicate between each other and the PFSense gateway VM which acts as a firewall and a router allowing communication with external devices. Inside the internal network there is a Windows Workstation VM and an Ubuntu VM with Splunk installed to gather logs from the gateway and Windows devices. The initial more simple homelab won't include Windows Active Directory Domain Controller VM due to limited processing resources but maybe later I will include such a VM to better simulate an organization network environment.

As an external adversary I've decided to use VirtualBox with Kali Linux VM installed on my Thinkpad X230 laptop. This external VM connects to the simulated corporate network through the PFSense gateway VM via Bridged connection. I want the Kali VM to act as an independent host capable of interacting with surrounding hosts on the same LAN network, that is why I chose my Kali VM to run in a bridged networking mode. The bridged mode allows my Kali VM to have its own IP-address on my home network so that it can interact with other hosts on the same network, simulating an attacker who tries to infiltrate into a corporate network. As a note to myself, I have to be careful though not to execute malware downloaded from the internet on this bridged VM because that malware can in the worst scenario spread to other hosts in my home network including my PC.

![homelab](https://github.com/user-attachments/assets/3f43d5e2-f1e6-4518-928a-8cdee3c37ada)

## Setting up PFSense Gateway VM

PFSense runs on FreeBSD which is a lightweight operating system, thus suitable for my use case. I chose PFSense as a virtual gateway device for its low resource requirements in terms of CPU and memory. I'll be running multiple virtual machines on the same host which makes it important to save resources wherever possible.

On VirtualBox I built a new Debian 64bit VM. I allocated 1024MB of RAM, 1 CPU core and 8gb of dynamic disc space. I set the boot order to 1. hard drive and 2. optical. For now the hard drive is empty so we will boot from PFSense .ISO image I downloaded from netgate. In the networking settings I will enable two network adapters:

Adapter 1 is for internal network to connect my other VMs to the PFSense gateway. I selected AMD PCnet-FAST III (Am79C973) as the virtualized network adapter because it is supported by almost all quest operating systems.

Adapter 2 is for external network. I chose bridged adapter because that allows other hosts on the same home network to reach me through the NIC of my host PC that is running VirtualBox. Selecting NAT-adapter wouldn't allow me to interact with other hosts on the same home network without doing port forwarding through the host PC that is running VirtualBox. Even then the network access would be limited to selected services only. NAT-adapter by default would only allow my VMs to access internet but not other devices on my home network. I don't need to access internet on my VMs so there's no reason to choose NAT-adapter. 

I booted up the VM with the .ISO installation disk attached. I selected my VM's internal network as LAN and the external bridged network as WAN. I had the option to choose between ZFS and UFS for a filesystem. I decided to go with UFS because it is more suitable for my lightweight firewall/router setup which doesn't require the extra features provided by ZFS. UFS is more simpler and uses less memory compared to ZFS. 

After the installation was complete I configured LAN-interface IP-addresses. I set the LAN host IP-addresses to range between 10.10.10.1-50 (255.255.255.0 subnet mask). I didn't find any reason to configure IPv6-addresses for my LAN. I enabled DHCP-server for the LAN. Here is a screenshot of the PFSense console:

![kuva](https://github.com/user-attachments/assets/411948e8-35e7-47cf-b794-a91ecfeedfe6)

I later configured the PFSense WAN-interface to use my personal computer's NIC connected to the internet. I only did this to later download some essential packages on Ubuntu VM from the internet. After downloading the packages I can configure the PFSense WAN-interface to use another NIC on my PC without internet connection and will connect to the Kali machine on the same LAN instead.

## Setting up Ubuntu VM

The idea of this machine is to simulate organization's IT-security team workstation. This machine will ingest logs from other hosts on the same internal network and analyze network packet traffic. I decided to run Splunk and malware analysis tools on Linux OS because it is more lightweight and customizable compared to Windows OS. I built a new Ubuntu 64bit VM on VirtualBox. I allocated the virtual machine 8192MB of RAM, 2 CPU cores and 160gb of dynamic disk space. I selected the same network adapter as with my gateway VM and set the adapter mode as internal network. I set the boot order to 1. hard drive and 2. optical. I downloaded Ubuntu Server 24.04.2 .ISO image from Ubuntu.com and booted the VM from the .ISO image. The Ubuntu installer started and I selected to install OpenSSH for future Kali Linux connections. The installer was not able to configure IP-address from DHCP-server, so I need to set a temporary static IP-address manually for now and later access FPSense webConfigurator on Ubuntu VM's web browser to configure the DHCP-server. I set the IPv4-address for my Ubuntu VM to be 10.10.10.1/24. The Ubuntu server doesn't come with a graphical user interface so I need to install the GUI-package first to access the web browser. To start the static IP-address configuration process I first need to find the name of my ethernet interface: 

![kuva](https://github.com/user-attachments/assets/6ea1b300-7eca-40ad-884b-6ef80e1b4e5b)

Then I need to set a new static IPv4 address to my ethernet interface:
```
sudo ip addr add 10.10.10.11/24 dev enp0s3
```
To set the link up:
```
sudo ip link set dev enp0s3 up
```
To confirm that the link is up:
```
sudo ip address show dev enp0s3 
```
![kuva](https://github.com/user-attachments/assets/bbd4400e-9e44-4c0d-8a8b-43782da40947)

To set the default gateway address to my PFSense gateway:
```
sudo ip route add default via 10.10.10.254 
```
To confirm that the correct default gateway is set:
```
sudo ip route show 
```
![kuva](https://github.com/user-attachments/assets/d880ab79-c67e-4bfe-9450-3c73d2829e52)

To ping the PFSense gateway device to confirm that the connection is established:
```
ping -c 3 10.10.10.254
```
![kuva](https://github.com/user-attachments/assets/22a2a64f-1049-42af-b6de-e1e8b60387bc)

All ICMP-packets were able to reach the PFSense gateway device so it seems that the connection was succesfully established. Next I need to download the Ubuntu GUI-package using commands:
```
sudo apt-get update
sudo apt-get install ubuntu-desktop
```
After running these commands it seems that fetching the files from Ubuntu server failed due to problems with address resolution. I haven't yet set up client DNS nameservers so I had to open /etc/resolv.conf and add "nameserver 8.8.8.8" and "nameserver 8.8.4.4" to the list of nameservers. Now when I run the commands above I'm able to reach Ubuntu server and fetch the files from there. Installing Ubuntu desktop took quite a while but is finally complete.

Now it is time to access the PFSense WebConfigurator on Ubuntu VM's web browser to configure DHCP-server on https://10.10.10.254/. It seems that the IPv4 configuration type is set to Static IPv4, this explains why my Ubuntu VM was not automically assigned an IPv4-address from the DHCP-server during installation. I choose to keep the IPv4 confguration type as static for my LAN and assigned 10.10.10.254 as the DNS-server for the LAN.


While we're at it let's check the firewall rules for the LAN. There are three LAN firewall rules by default. The first one is the Anti-lockout rule, which prevents locking out an administrator from the web interface. The second and the third rule are there to allow all traffic on the LAN for IPv4 and IPv6 protocols respectively. I'll return to firewall rules later.

## Setting up Windows 10 workstation VM

This device simulates a company employee's workstation. I downloaded a Windows 10 media creation tool from Microsoft's official web server. I use this tool to create an installation .ISO image. I built a Windows VM using the .ISO image on VirtualBox and allocated 2048MB of RAM, 1 CPU core and 50gb of dynamic disc space. I connected the VM to the same internal network with the two previous virtual machines and used the same network adapter. I booted the VM and started the Windows 10 Pro installation process.

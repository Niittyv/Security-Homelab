# Security-Homelab

I've decided to build a VirtualBox internal network to simulate a corporate LAN environment. The VM devices inside the internal network can only communicate between each other and the PFSense gateway VM which acts as a firewall and a router allowing communication with external devices. Inside the internal network there is a Windows Workstation VM, an AD domain controller VM and an Ubuntu VM with Splunk installed to gather logs from the gateway and Windows devices.

As an external adversary I've decided to use VirtualBox with Kali Linux VM installed on my Thinkpad X230 laptop. This external VM connects to the simulated corporate network through the PFSense gateway VM via Bridged connection. I want the Kali VM to act as an independent host capable of interacting with surrounding hosts on the same LAN, that is why I chose my Kali VM to run in a bridged networking mode. The bridged mode allows my Kali VM to have its own IP-address on my home network so that it can interact with other hosts on the same network, simulating an attacker who tries to infiltrate into a corporate network. As a note to myself, I have to be careful though not to execute malware downloaded from the internet on this bridged VM because that malware can in the worst scenario spread to other hosts in my home network including my PC.

![homelab](https://github.com/user-attachments/assets/f7298791-ae1f-4924-b34c-c824ee7413be)


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

The idea of this machine is to simulate organization's IT-security team workstation. This machine will ingest logs from other hosts on the same internal network and analyze network packet traffic. 

I decided to run Splunk and malware analysis tools on Linux OS because it is more lightweight and customizable compared to Windows OS. I built a new Ubuntu 64bit VM on VirtualBox. I allocated the virtual machine 8192MB of RAM, 2 CPU cores and 160gb of dynamic disk space. I selected the same network adapter as with my gateway VM and set the adapter mode as internal network. 

I downloaded Ubuntu Server 24.04.2 .ISO image from Ubuntu.com and booted the VM from the .ISO image. The Ubuntu installer started and I selected to install OpenSSH for future Kali Linux connections. The installer was not able to configure IP-address from DHCP-server, so I needed to set a temporary static IP-address manually for now and later access FPSense webConfigurator on Ubuntu VM's web browser to configure the DHCP-server. I set the IPv4-address for my Ubuntu VM to be 10.10.10.1/24. The Ubuntu server doesn't come with a graphical user interface so I need to install the GUI-package first to access the web browser. To start the static IP-address configuration process I first need to find the name of my ethernet interface on the Ubuntu terminal: 

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
After running these commands it seems that fetching the files from Ubuntu server failed due to problems with address resolution. I haven't yet set up client DNS nameservers so I had to open /etc/resolv.conf and add "nameserver 8.8.8.8" and "nameserver 8.8.4.4" to the list of nameservers. I have to note that this is only a temporary solution because the resolv.conf file will reset after reboot, I will add the DNS servers to my PFSense DNS serverlist later when I can access the configuration page on my browser. Now when I run the commands above I'm able to reach Ubuntu server and fetch the files from there. Installing Ubuntu desktop took quite a while but is finally complete.

Now it is time to access the PFSense WebConfigurator on Ubuntu VM's web browser to configure DHCP-server on https://10.10.10.254/. It seems that the IPv4 configuration type is set to Static IPv4, this explains why my Ubuntu VM was not automically assigned an IPv4-address from the DHCP-server during installation. I choose to keep the IPv4 confguration type as static for my LAN and assigned 10.10.10.254, 8.8.8.8 and 8.8.4.4 as the DNS-servers for the LAN.

![kuva](https://github.com/user-attachments/assets/aaf570f3-cfff-4394-b123-43c5769eedb5)

While we're at it let's check the firewall rules for the LAN. There are three LAN firewall rules by default. The first one is the Anti-lockout rule, which prevents locking out an administrator from the web interface. The second and the third rule are there to allow all traffic on the LAN for IPv4 and IPv6 protocols respectively. I'll return to firewall rules later.

![firewall](https://github.com/user-attachments/assets/a22ae55a-88f2-445b-93d7-cae70af7cb44)

### Setting up Splunk on Ubuntu VM

First I'll run
'''
sudo apt-get update && upgrade
'''
To update all the installation packages. 

## Setting up Windows 10 workstation VM

This device simulates a company employee's workstation. I downloaded a Windows 10 media creation tool from Microsoft's official web server. I use this tool to create an installation .ISO image. I built a Windows VM using the .ISO image on VirtualBox and allocated 2048MB of RAM, 1 CPU core and 50gb of dynamic disc space. I connected the VM to the internal network with and selected AMD PCnet-FAST III (Am79C973) as the network adapter. I booted the VM and started the Windows 10 Pro installation process.

The Windows 10 installation was succesful but the operating system was not able to detect the network interface connected to the internal network. I had to change the network adapter to Intel PRO/1000 MT Desktop (82540EM) and now the network interface popped up in network connections:

![kuva](https://github.com/user-attachments/assets/27eaef9c-9461-4221-8254-ceb3a4430a22)

I opened CMD and checked the configuration for network interfaces. It seems that the DHCP-server is up, configured properly and able to lease IP-addresses.

![kuva](https://github.com/user-attachments/assets/4e89bf6d-0599-47ec-826c-82b1b75718f1)

Let's check connectivity to the PFSense gateway device using ICMP-packets. All packets made it to the destination and received a reply: 

![kuva](https://github.com/user-attachments/assets/560394d4-cec4-4659-a578-161306225c9d)

Out of curiosity I wanted to ping the Windows workstation from the PFSense gateway VM and got the following result: 

![kuva](https://github.com/user-attachments/assets/7df25e2f-51f9-42e4-afc5-9d8657305388)

It seems that the gateway device is not able to reach the windows device with ICMP-packets. My intuition tells me that this is because Windows firewall is blocking the inbound ICMP-protocol traffic but I have to make sure of that. I opened the Windows firewall settings and disabled Windows firewall for public networks.

![kuva](https://github.com/user-attachments/assets/47fdd7f0-a28a-41d1-a188-99827d4a6441)

Now the ICMP-packets are not dropped by the firewall:

![kuva](https://github.com/user-attachments/assets/4e757428-73bd-492f-b973-dca8c277aad0)

## Setting up Windows AD Domain Controller

I downloaded Windows server 2022 trial ISO from Microsoft official web page. I created a new virtual machine with 2048MB RAM, 1 CPU core and 30GB disc space. I connected the VM to internal network with Intel PRO/1000 MT Desktop (82540EM) adapter just like with the Windows 10 machine. I started the VM with Windows server 2022 ISO mounted. I want to install Windows server with full graphical environment:

![kuva](https://github.com/user-attachments/assets/53e89670-51e4-4150-bbdd-900c1ca79c7e)

Windows AD domain controller VM is up and running. Now I'm going to configure Active Directory Domain Services (AD DS).

![kuva](https://github.com/user-attachments/assets/3976b2c2-961e-417a-b4ea-a107a363fb2b)

I need to promote this server as the domain controller. I select "Add a new forest" and add .local suffix to my root domain name.

![kuva](https://github.com/user-attachments/assets/ea506bf1-361c-4923-aca9-53135d18d1a7)

I selected Windows Server 2016 for Forest/Domain functional level. I disabled AD DNS server because PFSense takes care of the DNS server. I proceed to complete the installation.

![kuva](https://github.com/user-attachments/assets/c07a920d-09dc-43a3-a229-fe34db9cde80)

I have now configured AD domain and domain controller. Next I will connect the Windows 10 user to AD domain. First I'll disable the windows server firewall (windows 10 firewall is already disabled) to make sure that the firewalls don't block the connection. Oh and before I forget, I'll have to make sure that the DNS server for both Windows 10 and Windows server host are configured to 10.10.10.254 (the IP address of PFSense host). I will disable IPv6 for both Windows hosts as it is not needed.

![kuva](https://github.com/user-attachments/assets/e5891997-19df-4955-846c-c0c71c1e0271)

I allowed remote connections on Windows server VM. I opened system properties on Windows 10 VM and connected the client host to the newly created domain.

![kuva](https://github.com/user-attachments/assets/8677398b-3de5-4778-9535-d6fd6f5b5186)

The Windows 10 client was unable to reach NVPorg.local domain because there seems to be a problem with the DNS configuration. Only the PFSense IP address is listed as a DNS server. I made an error in judgement, maybe I'll need to enable the AD DNS server.

![kuva](https://github.com/user-attachments/assets/6ad1f503-e8e2-4956-a91b-75bec0b63b9c)




## Setting up Linux Kali VM on my Thinkpad X230 laptop

I installed VirtualBox on my laptop but forgot to run the installation file as an administrator. This led to VirtualBox not being allowed to utilize CPU resources for virtualization, even though virtualization is enabled on BIOS. I reinstalled VirtualBox as an administrator and now I'm allowed to access CPU resources and run virtual machines. 

I downloaded Linux Kali prebuild VM from the Linux Kali official website. I added the prebuild machine to VirtualBox and connected the VM to bridged network using AMD PCnet-FAST III (Am79C973) virtual network adapter. I left the VM settings as default. This means that the VM is allocated with 2048MB of RAM, 2 CPU cores and 80gb of dynamic disc space.

I boot up the VM and the operating system is ready to go. No installation is required.

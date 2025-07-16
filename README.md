# Security-Homelab

I've decided to build a VirtualBox internal network to simulate a corporate LAN environment. The VM devices inside the internal network can only communicate between each other and the PFSense gateway VM which acts as a firewall and a router allowing communication with external devices. Inside the internal network there is a Windows Workstation VM, an AD domain controller VM and an Ubuntu VM with Splunk installed to gather logs from the gateway and Windows devices.

As an external adversary I've decided to use VirtualBox with Kali Linux VM installed on my Thinkpad X230 laptop. This external VM connects to the simulated corporate network through the PFSense gateway VM via Bridged connection. I want the Kali VM to act as an independent host capable of interacting with surrounding hosts on the same LAN, that is why I chose my Kali VM to run in a bridged networking mode. The bridged mode allows my Kali VM to have its own IP-address on my home network so that it can interact with other hosts on the same network, simulating an attacker who tries to infiltrate into a corporate network. As a note to myself, I have to be careful though not to execute malware downloaded from the internet on this bridged VM because that malware can in the worst scenario spread to other hosts in my home network including my PC.

![homelab](https://github.com/user-attachments/assets/f7298791-ae1f-4924-b34c-c824ee7413be)

Here are the intended host IP-addresses:

10.10.10.0/24:

PFSense VM: 10.10.10.254

Ubuntu VM: 10.10.10.1

Windows Server 2022 (Active Directory Domain controller): 10.10.10.14


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

First I need to register on splunk.com and activate free enterprise trial for 60 days. After my trial expires I can switch to Splunk free with limited features.

I run the following command to update all installation packages. 
```
sudo apt-get update && upgrade
```

#### Enabling shared clipboard

I copied the .deb wget link from splunk.com to download and install splunk on my Linux VM. What annoys me currently though is that the clipboard between my Windows desktop and my Ubuntu VM are not shared so I can not copy & paste the wget link to my Ubuntu VM. First I have to enable the shared bidirectional clipboard on VirtualBox and then I need to install VirtualBox quest additions from an .ISO that is provided by VirtualBox. I execute the .run file and it says that the system is currently not set up to build kernel modules:

![kuva](https://github.com/user-attachments/assets/4f0366f0-71f0-42c4-bc27-dd20798363a5)

I had to run

```
sudo apt-get install build-essential gcc make perl dkms
```

Now I'm able to install VirtulBox quest additions and after the installation was complete my bidirectional clipboard became functional and now I'm able to copy & paste the wget link to download Splunk.

![kuva](https://github.com/user-attachments/assets/31c2219e-9334-4188-8c45-0fc8aa9ea0e3)

I installed Splunk:

![kuva](https://github.com/user-attachments/assets/18bea00d-cfce-4665-bc35-e88e608d77b8)

Splunk is located at /opt. I will start Splunk using the command

```
sudo /opt/splunk/bin/splunk start
```

After needing to accept the license I set my administrator username to NVP. The installation is complete. I can access Splunk web interface at http://ubuntuserver:8000.

![kuva](https://github.com/user-attachments/assets/913f7c98-12cb-4f7d-9cf6-af1e386873ec)

In server settings I want to enable SSL to make the HTTP secure.

![kuva](https://github.com/user-attachments/assets/ef46b975-a039-47c6-a004-cf44019e215c)

### Universal receiver/indexer

I need to configure universal receiver on my Splunk VM to collect logs from endpoints. First I want to make sure that port 9997 is not in use:

![kuva](https://github.com/user-attachments/assets/ce3ed740-3642-4439-9cb1-b69f39e5022b)

The command above returned nothing which means that port 9997 is available for listening activity. I'll assign this port to universal receiver.

![kuva](https://github.com/user-attachments/assets/f94e2ec8-bcbf-4da9-b06d-774d4c3f4bb6)

The Ubuntu ufw (uncomplicated firewall) is disabled by default, so I don't need to create a firewall rule to allow inbound traffic coming from Splunk application or TCP/UDP port 9997.


## Setting up Windows 10 workstation VM

This device simulates a company employee's workstation. I downloaded a Windows 10 media creation tool from Microsoft's official web server. I use this tool to create an installation .ISO image. I built a Windows VM using the .ISO image on VirtualBox and allocated 2048MB of RAM, 1 CPU core and 50gb of dynamic disc space. I connected the VM to the internal network with and selected AMD PCnet-FAST III (Am79C973) as the network adapter. I booted the VM and started the Windows 10 Pro installation process.

The Windows 10 installation was succesful but the operating system was not able to detect the network interface connected to the internal network. I had to change the network adapter to Intel PRO/1000 MT Desktop (82540EM) and now the network interface popped up in network connections:

![kuva](https://github.com/user-attachments/assets/27eaef9c-9461-4221-8254-ceb3a4430a22)

I opened CMD and checked the configuration for network interfaces. It seems that the DHCP-server is up, configured properly and able to lease IP-addresses. The Windows 10 host is leased with IP-address 10.10.10.2.

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

Windows AD domain controller VM is up and running. This host is configured automatically with IP-address of 10.10.10.3 by DHCP-server. I pinged PFSense gateway and made sure the connection is established. Now I'm going to configure Active Directory Domain Services (AD DS).

![kuva](https://github.com/user-attachments/assets/3976b2c2-961e-417a-b4ea-a107a363fb2b)

I need to promote this server as the domain controller. I select "Add a new forest" and add .local suffix to my root domain name.

![kuva](https://github.com/user-attachments/assets/ea506bf1-361c-4923-aca9-53135d18d1a7)

I selected Windows Server 2016 for Forest/Domain functional level. I disabled AD DNS server because PFSense takes care of the DNS server. I proceed to complete the installation.

![kuva](https://github.com/user-attachments/assets/c07a920d-09dc-43a3-a229-fe34db9cde80)

I have now configured AD domain and domain controller. Next I will connect the Windows 10 user to AD domain. First I'll disable the windows server firewall (windows 10 firewall is already disabled) to make sure that the firewalls don't block the connection. Oh and before I forget, I'll have to make sure that the DNS server for both Windows 10 and Windows server host are configured to 10.10.10.254 (the IP address of PFSense host). I will disable IPv6 for both Windows hosts as it is not needed.

![kuva](https://github.com/user-attachments/assets/e5891997-19df-4955-846c-c0c71c1e0271)

I allowed remote connections on Windows server VM. I opened system properties on Windows 10 VM and connected the client host to the newly created domain.

![kuva](https://github.com/user-attachments/assets/8677398b-3de5-4778-9535-d6fd6f5b5186)

The Windows 10 client was unable to reach NVPorg.local domain because there seems to be a problem with the DNS configuration. Only the PFSense gateway IP-address is listed as a DNS server.

![kuva](https://github.com/user-attachments/assets/6ad1f503-e8e2-4956-a91b-75bec0b63b9c)

I made an error in judgement when I misconfigured the DNS server to PFSense gateway DNS server instead of AD DNS server. I didn't install DNS server on my AD domain, I'll install it now. Before installing DNS server for my AD domain, I'll set my AD domain IP as static so that the clients will always be able to reach it consistently using the same IP-address. I'll use the IP-address of 10.10.10.14/24 for my AD domain controller. I use the AD domain controller IP-address as the DNS server address.

![kuva](https://github.com/user-attachments/assets/8f9a8496-d293-4714-8875-fcfc108a3f9b)

![kuva](https://github.com/user-attachments/assets/523276e7-91c7-45f6-a117-7b6c672f5b79)

I'm able to ping the AD domain controller with my Windows 10 client, I'm still not able to connect to the domain though. I do nslookup and see that despite of the client being physically able to reach the domain, the name server can not be resolved properly.

![kuva](https://github.com/user-attachments/assets/8467e97b-c5d9-4a2c-9b8b-0c918777328d)

I run the commands on my Windows 10 VM:

```
ipconfig /flushdns
ipconfig /registerdns
```

And now I'm finally able to connect to the domain with my Windows10 client. I'll now create a user for Windows 10 client on AD domain controller.

![kuva](https://github.com/user-attachments/assets/6d74c651-d07a-499d-a65c-6addcd6caa6a)

I'll login to the domain on my Windows 10 client:

![kuva](https://github.com/user-attachments/assets/6ab7cf03-0170-4db1-84f8-036b355e7589)
![kuva](https://github.com/user-attachments/assets/ced4764d-c8f0-4f68-9b34-46f6d1eebfa0)

## Universal forwarders on Windows VMs

I need to set up universal forwarders on Windows 10 and Windows server 2022 VMs to send system logs from Windows machines to Splunk on Ubuntu VM. I'll download the Splunk universal forwarder installer file from Splunk's website to my host PC and transfer the file to both Windows VMs. 

![kuva](https://github.com/user-attachments/assets/272cd05f-62a7-409f-8273-03a4fff98e6a)

Before I can transfer the universal forwarder installation files to Windows VMs I need to first install Guest Additions on both Windows VMs so that I can share files between my host PC and virtualbox machines. 

After I have Installed Guest Additions on both Windows VMs, I'll create a shared folder so that I can send the Splunk universal forwarder installer file from my host PC to Windows VMs. Because I have Guest Additions installed, I can check "Auto-mount" and the shared folder should appear as a drive-letter on my Windows VMs.

![kuva](https://github.com/user-attachments/assets/19619d80-b8b6-4953-b8e5-8cb858e80f5b)

Now after booting my Windows server 2022 VM, I can find the shared folder drive. 

![kuva](https://github.com/user-attachments/assets/04c4c16a-0551-468a-a569-bcdee75589d8)

The Splunk universal forwarder installation file is located inside the shared folder.

![kuva](https://github.com/user-attachments/assets/25551612-bb9b-44c0-9fe5-4506d412045f)

When installing universal forwarder, I'll need to specify the receiving indexer's IP-address and the listening port so that the forwarder knows where to send the log data. In my case I use my Ubuntu VM's IP-address and port 9997 which I reserved earlier for Splunk universal receiver on Ubuntu VM. 

NOTE: you don't need to specify port if you're using default port 9997 for your listener so you can leave the field blank.

![kuva](https://github.com/user-attachments/assets/f1f70e45-1799-4aca-b18e-94ff6f5ce86d)

The installation is complete. With Windows firewall enabled, I would need to set an outbound rule to allow any outgoing traffic from Splunk application or TCP/UDP port of 9997. I have the Windows firewall disabled on both Windows machines so I don't need to worry about changing firewall rules for now. Later I will set up the Windows firewalls and I need to create outbound rules to allow outgoing traffic from universal forwarders.

#### Universal forwarders configuration files on Windows VMs

The universal forwarder config files reside in C:\Program Files\SplunkUniversalForwarder\etc\apps\SplunkUniversalForwarder\local. There are two config files that I need to set up:
1. inputs.conf. This configuration file will tell the universal forwarder where to gather the logs from. I plan to gather logs from Windows Event Log (winevtlog) located in C:\Windows\System32\winevt\Logs.
2. outputs.conf. This file defines where the collected logs should be forwarded to. It should include the IP-address and port number of the universal indexer.

Before I continue configuring my universal forwarders I want to explore my options. The simplest option would be to edit universal forwarder inputs.conf and outputs.conf files directly to gather logs from winevtlog and send them to the indexer. However, I will need the Splunk Add-on for Windows if I want a proper field extraction and better search experience in Splunk. Splunk Add-on for windows will also generate a neat inputs.conf template for my universal forwarder.

I've decided to download Splunk Add-on for Windows. I need to download the Add-on for both the indexer on Ubuntu VM and for the universal forwarders on Windows VMs. I can download Splunk Add-on for Windows from Splunk website on my home PC and then share the Add-on folder to VMs by utilizing shared folder. I have Guest Additions installed on all VMs so I can easily share files via shared folder. 

<img width="942" height="209" alt="kuva" src="https://github.com/user-attachments/assets/1427f90d-bb3e-483c-82e2-93a35594ff42" />

I inserted Splunk Add-on for Windows into the shared folder. I will now access the shared folder on my Ubuntu VM.


#### Installing Splunk Add-on for Windows for the indexer (Splunk on Ubuntu VM)

<img width="884" height="544" alt="kuva" src="https://github.com/user-attachments/assets/26e9264e-e7c6-4e99-8440-05b4326ca813" />

I don't have the permission to access shared folder on my Ubuntu VM via GUI. I need to access the shared folder by using terminal and give myself root rights. The shared folder is located in /media/.

<img width="805" height="267" alt="kuva" src="https://github.com/user-attachments/assets/bc6cab5f-9fbd-42cb-9381-dcf1ee5443ae" />

I copied the Add-on folder from shared folder to /home/jasper/. Now I need to open the Splunk application on my Ubuntu VM so that I can integrate the Add-on to the indexer.

<img width="1446" height="753" alt="kuva" src="https://github.com/user-attachments/assets/04cf926e-8a20-471c-8956-f98d3063525f" />

On my Ubuntu VM Splunk instance I navigated to Apps and chose "Install app from file". This way I can integrate the Windows Add-on to Splunk from local files without needing internet connection. 

<img width="1435" height="759" alt="kuva" src="https://github.com/user-attachments/assets/f40443d2-a6e9-48dd-be65-895328f58eaa" />

The Add-on folder needs to be in .spl or .tar.gz format. I need to archive and compress Splunk_TA_windows folder into .tar.gz format. 

<img width="857" height="49" alt="kuva" src="https://github.com/user-attachments/assets/464aefcf-fc39-498f-bb3b-5ece6f2ae8c8" />

I tried to install the Add-on via web UI but it doesn't seem to work. Whenever I press "Upload" the page just refreshes and nothing happens. I'll try to use CLI to install the Add-on.

<img width="1168" height="111" alt="kuva" src="https://github.com/user-attachments/assets/f3afb3ce-db45-4ca6-b63f-1471dde686e0" />

Running the command through CLI did the trick. I can now find the Add-on listed on Apps list:

<img width="1454" height="782" alt="kuva" src="https://github.com/user-attachments/assets/1bb70485-faff-4bd4-9200-053a14a8fd12" />

Maybe there was an issue with the browser web session because I couldn't install the Add-on via web UI. Fortunately the CLI doesn't depend on web sessions. The size limit of installations through web UI is 50MB while the Add-on file is just 211KB so the size can't be the issue either.


#### Installing Splunk Add-on for Windows for the universal forwarders (Windows VMs)

<img width="1142" height="530" alt="kuva" src="https://github.com/user-attachments/assets/1f02b506-d0ff-4bd0-9777-7c20230a89e2" />

On my Windows Server 2022 VM, I copied the Add-on from shared folder and pasted it to SplunkUniversalForwarder/etc/apps. 

<img width="973" height="559" alt="kuva" src="https://github.com/user-attachments/assets/5dde04a7-56dd-42ca-813a-1b2f5120c46d" />

The output.conf file is located in SplunkUniversalForwarder/etc/system/local. 

<img width="786" height="251" alt="kuva" src="https://github.com/user-attachments/assets/e459cb65-a0d2-47f7-95e9-7bccf58d110d" />

The output.conf file includes the server address where the logs are being forwarded to.

<img width="784" height="393" alt="kuva" src="https://github.com/user-attachments/assets/644450f0-b358-4e29-9584-22fb1893f5b4" />

In Splunk_TA_windows/default I can find a template for inputs.conf. I will not edit it here directly but instead I'll create a new Splunk_TA_windows/local path and paste the input.conf there.

<img width="796" height="489" alt="kuva" src="https://github.com/user-attachments/assets/453f9d5a-ccf2-48f3-8269-7639baa939a3" />

Here is a screenshot of the OS Logs portion of local/inputs.conf file. For the purpose of later filtering practise on Splunk indexer I will enable the gathering of all WinEvtLogs. 

<img width="922" height="583" alt="kuva" src="https://github.com/user-attachments/assets/49723b73-2a93-45fb-adf7-e0a9cfc3cc05" />

Next I need to restart SplunkForwarder service for the changes to take effect.


#### Receiving and filtering logs on the indexer (Splunk Ubuntu VM)

<img width="1452" height="596" alt="kuva" src="https://github.com/user-attachments/assets/a5dac9e5-9278-4242-9d8e-1ac08a776a70" />

First let's make sure that the indexer is listening to port 9997.

<img width="794" height="225" alt="kuva" src="https://github.com/user-attachments/assets/96cbb39b-b428-4125-8e0e-9eae3f3ac7df" />
<img width="778" height="271" alt="kuva" src="https://github.com/user-attachments/assets/89b82d22-eb0a-487b-a8a4-42b67a34fc61" />

Looking at the "Search and reporting" tab on Splunk I can see that the logs are being received from the Windows Server 2022 machine and the indexer is being able to capture all WinEvtLogs that we enabled in inputs.conf file. Great success!

Now I will start looking at the parsing and filtering of log data on the indexer.

<img width="1494" height="789" alt="kuva" src="https://github.com/user-attachments/assets/5d30de76-a11c-4589-b158-3622e81f8750" />

Looking at the format of WinEvtLogs events, they are not easily readable. I need to change the renderXML of OS Logs in inputs.conf file from "true" to "false". Remember to restart the SplunkForwarder service on the Windows VM for the changes to take effect!

<img width="1568" height="906" alt="kuva" src="https://github.com/user-attachments/assets/838e5afc-5885-4b6f-a738-6410677ca6de" />

There! The events are now easily readable.

Next up I will repeat the process of installing universal forwarder and Splunk Add-on for Windows on my Windows 10 VM.

## Setting up Linux Kali VM on my Thinkpad X230 laptop

I installed VirtualBox on my laptop but forgot to run the installation file as an administrator. This led to VirtualBox not being allowed to utilize CPU resources for virtualization, even though virtualization is enabled on BIOS. I reinstalled VirtualBox as an administrator and now I'm allowed to access CPU resources and run virtual machines. 

I downloaded Linux Kali prebuild VM from the Linux Kali official website. I added the prebuild machine to VirtualBox and connected the VM to bridged network using AMD PCnet-FAST III (Am79C973) virtual network adapter. I left the VM settings as default. This means that the VM is allocated with 2048MB of RAM, 2 CPU cores and 80gb of dynamic disc space.

I boot up the VM and the operating system is ready to go. No installation is required.

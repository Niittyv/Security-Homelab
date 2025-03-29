# Security-Homelab

I've decided to build a VirtualBox internal network to simulate a corporate LAN network environment. The VM devices inside the internal network can only communicate between each other and the PFSense gateway VM which acts as a firewall and a router allowing communication with external devices. Inside the internal network there is a Windows Workstation VM and an Ubuntu VM with Splunk installed to gather logs from the gateway and Windows devices. The initial more simple homelab won't include Windows Active Directory Domain Controller VM due to limited processing resources but maybe later I will include such a VM to better simulate an organizational network environment.

As an external adversary I've decided to use VirtualBox with Kali Linux VM installed on my Thinkpad X230 laptop. This external VM connects to the simulated corporate network through the PFSense gateway VM via Bridged connection. I want the Kali VM to act as an independent host capable of interacting with surrounding hosts on the same LAN network, that is why I chose my Kali VM to run in a bridged networking mode. The bridged mode allows my Kali VM to have its own IP-address on my home network so that it can interact with other hosts on the same network, simulating an attacker who tries to infiltrate into a corporate network. I have to be careful though not to execute malware downloaded from the internet on this bridged VM because that malware can in the worst scenario spread to other hosts in my home network including my PC.

![homelab](https://github.com/user-attachments/assets/7c5b5112-d4a8-4fa1-aac3-c78e0ba4ce76)

## Setting up PFSense Gateway VM

PFSense runs on FreeBSD which is a lightweight operating system, thus suitable for my use case. I chose PFSense as a virtual gateway device for its low resource requirements in terms of CPU and memory. I'll be running multiple virtual machines on the same host which makes it important to save resources wherever possible.

On VirtualBox I built a new Debian 64bit VM. I allocated 1024MB of RAM, 1 CPU core and 8gb of dynamic disc space. I set the boot order to 1. hard drive and 2. optical. For now the hard drive is empty so we will boot from PFSense .ISO image I downloaded from netgate. In the networking settings I will enable two network adapters:

Adapter 1 is for internal network to connect my other VMs to the PFSense gateway. I selected AMD PCnet-FAST III (Am79C973) as the virtualized network adapter because it is supported by almost all quest operating systems.

Adapter 2 is for external network. I chose bridged adapter because that allows other hosts on the same home network to reach me through the NIC of my host PC that is running VirtualBox. Selecting NAT-adapter wouldn't allow me to interact with other hosts on the same home network without doing port forwarding through the host PC that is running VirtualBox. Even then the network access would be limited to selected services only. NAT-adapter by default would only allow my VMs to access internet but not other devices on my home network. I don't need to access internet on my VMs so choosing NAT-adapter is not a valid option for me. 

I booted up the VM with the .ISO installation disk attached. I selected my VM's internal network as LAN and the external bridged network as WAN. I selected DHCP-server to configure IP-addresses for both LAN and WAN. I had the option to choose between ZFS and UFS for a filesystem. I decided to go with UFS because it is more suitable for my lightweight firewall/router setup which doesn't require the extra features provided by ZFS. UFS is more simpler and uses less memory compared to ZFS. The installation is complete and here is the PFSense console:

![kuva](https://github.com/user-attachments/assets/017299dd-6f82-4118-a549-606bc933cfe4)

# Security-Homelab

I've decided to build a VirtualBox internal network to simulate a corporate LAN network environment. The VM devices inside the network can only communicate with each other and the PFSense gateway VM which acts as a firewall and a router allowing communication with external devices. Inside the internal network there is a Windows Workstation VM and an Ubuntu VM with Splunk installed to gather logs from the gateway and Windows devices. 

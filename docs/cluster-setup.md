# Raspberry Pi Cluster Setup
This setup uses instructions and code  from [geerlingguy (Jeff Geerling) · GitHub](https://github.com/geerlingguy). I just merged all the instructions into a one pager and added additional steps that I find helped the process for me. Clone the following repo to your local machine.
```
git clone https://github.com/geerlingguy/raspberry-pi-dramble.git
```

## Image Raspberry Pis
The latest images for both 32-bit and 64-bit can be found below. 64-bit images are not directly advertised on their website as it is a beta image still. If you have a the Raspberry PI 4 Model B, I would recommend getting the 64-bit image. Other wise opt for the 32-bit image using the links below to source the latest version.

[Index of /raspios_arm64/images](https://downloads.raspberrypi.org/raspios_arm64/images/)

All 64-bit lite images: [Index of /pub/raspberrypi/raspios_lite_arm64/images](http://ftp.jaist.ac.jp/pub/raspberrypi/raspios_lite_arm64/images/)

All 32-bit lite images: [Index of /pub/raspberrypi/raspbian_lite/images](http://ftp.jaist.ac.jp/pub/raspberrypi/raspbian_lite/images/)

Download the [Raspberry PI Imager ](https://www.raspberrypi.org/software/) software for Mac or PC and follow the instructions  on the website to get Raspian OS installed on your Raspberry PI.

## Enable SSH
*Note: you may need to unplug and plug the SD card back in as the imager will eject it after the image is written.*

Run `diskutil list`  to find the disk name. Should look similar to the following output:
```
/dev/disk4 (external, physical):
   #:                       TYPE NAME                    SIZE       IDENTIFIER
   0:     FDisk_partition_scheme                        *64.1 GB    disk4
   1:             Windows_FAT_32 boot                    268.4 MB   disk4s1
   2:                      Linux                         63.8 GB    disk4s2
```

Then run the following command to enable ssh when the Raspberry PI boots the image…
```
touch /Volumes/boot/ssh
```

The card can now be used in the Raspberry PI.

## Networking Prerequisites
Plug the Pis into your network and follow the steps to find the IPs and MAC addresses for each Pi.
```
arp -an
```

This will display the MAC address and IP address pairs for all devices on your network. The output will look like:
```
? (192.168.0.1) at 80:20:da:9:70:fe on en0 ifscope [ethernet]
? (192.168.0.11) at 1e:b6:66:9c:8:f6 on en0 ifscope [ethernet]
? (192.168.0.81) at dc:a6:32:eb:56:7a on en0 ifscope [ethernet]
? (192.168.0.82) at dc:a6:32:f8:ff:46 on en0 ifscope [ethernet]
? (192.168.0.107) at (incomplete) on en0 ifscope [ethernet]
? (192.168.0.200) at dc:a6:32:eb:56:71 on en0 ifscope [ethernet]
? (192.168.0.255) at ff:ff:ff:ff:ff:ff on en0 ifscope [ethernet]
? (192.168.196.255) at ff:ff:ff:ff:ff:ff on feth2471 ifscope [ethernet]
? (224.0.0.251) at 1:0:5e:0:0:fb on en0 ifscope permanent [ethernet]
? (239.255.255.250) at 1:0:5e:7f:ff:fa on en0 ifscope permanent [ethernet]
? (255.255.255.255) at ff:ff:ff:ff:ff:ff on en0 ifscope [ethernet]
```

All Raspberry Pis have a MAC address that starts with either `dc:a6:32` for the 64-bit models or `b8:27:Eb` for the 32-bit models.

If you have your Pis stacked like me and want to know where each device is located based on its IP address, you can run a ping command but lower the interval to about 0.1 or 0.2 seconds. This will cause one of the switch ports to blink a lot faster than the others and allows you to make a physical mapping of IP address to a specific Pi.

*Note: Sudo is required to execute a ping command at lower intervals*
```
sudo ping -i 0.1 <IP_ADDRESS>
```

Make note of the information in order to map out which Pi in the cluster has which information. Something like the following. It will be needed later on.
```
**Kube-1:** 
	- IP Address: 192.168.0.81
	- MAC address: dc:a6:32:eb:3d:93
**Kube-2:** 
	- IP Address: 192.168.0.82
	- MAC address: dc:a6:32:eb:56:71
**Kube-3:** 
	- IP Address: 192.168.0.83
	- MAC address: dc:a6:32:f8:ff:46
**Kube-4:** 
	- IP Address: 192.168.0.84
	- MAC address: dc:a6:32:eb:56:7a
```

## Copy SSH Key to Pis
Take a look at your public key to see what it looks like:
```
cat ~/.ssh/id_rsa.pub
```

Using the computer which you will be connecting from, append the public key to your authorized_keys file on the Raspberry Pi by sending it over SSH:
```
ssh-copy-id pi@<IP-ADDRESS>
```

## Network the Raspberry Pis
There is an included playbook (inside setup/networking) which will set up all the Pi networking configuration following the below network layout:

	- kube1.phronesis.cloud (192.168.0.81)
	- kube2.phronesis.cloud (192.168.0.82)
	- kube3.phronesis.cloud (192.168.0.83)
	- kube4.phronesis.cloud (192.168.0.84)

To use it, you will need to know the IP addresses and MAC addresses for all four Pis as they are currently set up. Copy example.vars.yml to vars.yml, and example.inventory to inventory. Map each MAC address to the new IP addresses you want to set in vars.yml, and add all the *current* Pi IP addresses under the [pis] group inside the networking inventory file. Then run (within the setup/networking directory):
```
ansible-playbook -i inventory main.yml
```

Assuming everything went well, the Pis should switch over to their new IP addresses quickly; if they don’t, you can forcefully reboot them with the command:
```
ansible all -i inventory -m shell -a "sleep 1s; shutdown -r now" -b -B 60 -P 0
```

## Install Kubernetes
This section provides instructions for setting up Kubernetes on the Raspberry Pis.


Source: https://github.com/geerlingguy

Once all the Pis are online and operational, you need to copy the included example.config.yml file to config.yml and modify it to suit your needs (especially consider changing salt and password-related variables!).

Run the following commands to install kubernetes
```
ansible-galaxy install -r requirements.yml --force
```
```
pip install openshift
```
```
ansible-playbook -i inventory main.yml
```

After 10-20 minutes, you should see a summary of the completed tasks:
```
PLAY RECAP *******************************************************
kube1             : ok=118  changed=59   unreachable=0    failed=0   
kube2             : ok=73   changed=41   unreachable=0    failed=0   
kube3             : ok=73   changed=41   unreachable=0    failed=0   
kube4             : ok=73   changed=41   unreachable=0    failed=0
```

# Managing the Kubernetes Cluster
You can SSH into the Kubernetes master (192.168.0.81 by default) and run kubectl by switching to the root user (sudo su). For example:
```
kubectl get nodes
kubectl get pods
```

The Ansible playbook also copies the config file locally, so you can run kubectl locally if you have it installed. Just export the path to the file (e.g. 

`export KUBECONFIG=~/.kube/config`, then you can run kubectl commands.

```
root@kube1:/home/pi# kubectl get nodes
NAME    STATUS   ROLES    AGE   VERSION
kube1   Ready    master   42m   v1.19.7
kube2   Ready    <none>   42m   v1.19.7
kube3   Ready    <none>   42m   v1.19.7
kube4   Ready    <none>   42m   v1.19.7
```

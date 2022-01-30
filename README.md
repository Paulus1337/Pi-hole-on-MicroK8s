# Pi-hole on MicroK8s
I have watched a lot of tutorials on running Pi-hole in a MicroK8s setup but none of them resulted in a fully working solution that offered all the features in Pi-hole unless you do some tricks with a load balancer and you have a router/firewall that supports DHCP Relay.

I am sharing my YAML for those who have a normal home network and are looking for a redundant and light weight version of our favorite DNS sinkhole with DHCP enabled.
## Requirements
* Some experience and understanding of DNS and DHCP.
* Give device(s) for this tutorial a static IP. (as device is responsible to lease addresses first make sure it gets an address outside the DHCP range)

Preferable
* At least 2 devices that support a linux distro.
## Setup MicroK8s
Install Snapcraft.
```
#Debian
sudo apt install snapd
#Redhat
sudo yum install snapd
```
Install MicroK8s (Optional if your like me and want to play with the latest toys add --edge)
```
sudo snap install microk8s --classic
```
Wait for install to complete.
```
microk8s status --wait-ready
```
Optional install plugins. I myself believe in standardization and GUI's help keep things to a certain degree of standard.
```
microk8s enable portainer
```
MicroK8s is setup and if you want to access the Portainer GUI just go to <http:// Device IP:30777>.

## Setup Pi-hole
Create a **StorageClass** or just enable the microk8s-hostpath storage using Portainer found at the bottom of **Cluster** -> **Setup**.

Create 2x Persistent Volumes Claims (PVC) these will be used to save all your Pi-hole settings.
```
microk8s kubectl apply -f https://raw.githubusercontent.com/Paulus88/Pi-hole-MicroK8s/main/dnsmasq-pvc.yaml
microk8s kubectl apply -f https://raw.githubusercontent.com/Paulus88/Pi-hole-MicroK8s/main/pihole-pvc.yaml
```
Now your ready to deploy Pi-hole. (Please note this installs in the **default** namespace)
```
microk8s kubectl apply -f https://raw.githubusercontent.com/Paulus88/Pi-hole-MicroK8s/main/pihole.yaml
```
Vola Pi-hole is now accessable on port 8053: <http:// Device IP:8053>.
## What makes this version of Pi-hole different from other tutorials?
**Affinity**
```
       affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: kubernetes.io/os
                operator: In
                values:
                - linux
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchExpressions:
              - key: app
                operator: In
                values:
                - pihole
```
This tells Kubernetes that the Pi-hole pod must run on a linux machine and 2 pods with the pihole app are not allowed to run on the same device (node) at one time.

**hostNetwork** in Docker known as --net=host.
```
      hostNetwork: true
```
This is actually the most important component of this setup, it means you do not need a DHCP Relay supported router/firewall to allow Pi-hole to do DHCP from a container/pod and it throws the pod directly onto your internal network.

**securityContext**
```
        securityContext:
          capabilities:
            add:
            - NET_ADMIN
            - NET_BIND_SERVICE
            - NET_RAW
```
All the needed capabilities to take full advantages of Pi-hole on your network.

**volumeMounts**
```
        volumeMounts:
        - mountPath: /etc/dnsmasq.d
          name: dnsmasq
        - mountPath: /etc/pihole
          name: pihole
```
The Persistent Volume mounts to save all your Pi-hole data even if pod is redeployed.
## Redundancy
You thought we were done right? Working Pi-hole, yeeey but what if one device fails?

**DNS Redundancy**

To make your DNS fully redundant, you can offer this through DHCP Options in dnsmasq, you may have noticed:

https://raw.githubusercontent.com/Paulus88/Pi-hole-MicroK8s/main/03-pihole-option-dhcp.conf
```
dhcp-option-force=6,<Device IP 1>,<Device IP 2>
dhcp-option-force=42,<NTP IP/DNS>
```
Were option 6 = all your Pi-hole DNS enabled devices and optional option 42 = NTP server(s).

These add DHCP options to your lease and send them to all internal devices, see all possible DHCP Options: https://www.iana.org/assignments/bootp-dhcp-parameters/bootp-dhcp-parameters.xhtml

Once your happy with your 03-pihole-option-dhcp.conf upload it to your pod(s):
```
microk8s kubectl cp 03-pihole-option-dhcp.conf <some-namespace>/<some-pod>:/etc/dnsmasq.d/03-pihole-option-dhcp.conf
#or directly to the microk8s-hostpath Persistent Volume.
cp 03-pihole-option-dhcp.conf /var/snap/microk8s/common/default-storage/default-dnsmasq-pvc-<bunch-of-letters-numbers>/03-pihole-option-dhcp.conf
```
Now (re)enable DHCP in Pi-hole or redeploy your Pi-hole app and dnsmasq will pick up this change.

On a Windows Device on your network preform an ipconfig /renew and with ipconfig /all you should now see both your Pi-hole devices as DNS Servers:

![image](https://user-images.githubusercontent.com/25663775/151719068-45af16f3-494e-4741-a9af-b255e1c7e5c0.png)

**Storage Redundancy**

Copy settings between Pi-hole pods, here are some possibilities.
* Use Pi-hole Teleporter from 1 device to the rest.
* Mount your Kubernetes StorageClass to an external shared storage or GlusterFS https://www.gluster.org/ solution.
* Do what I did and credits to Bart Simons https://bartsimons.me/sync-folders-and-files-on-linux-with-rsync-and-inotify/ use ssh, inotify and rsync and sync the /var/snap/microk8s/common/default-storage on every device with each other. (keep in mind root is used for ssh and optional I added the --delete option to the rsync command)
## Tips
**DHCP Range**

Keep your Pi-hole machines outside your DHCP Range.

Example: Give Pi-hole devices static IP's: 192.168.100.2 and 192.168.100.3 / Start DHCP Range at 192.168.100.10 and end at 192.168.100.254.

Why? This to avoid IP conflict that a internal device does not receive the same IP as your Pi-hole as DHCP does not recognize already assigned static IP's.

**DNS**

Best order for a static DNS is external Pi-hole device first and localhost 127.0.0.1 as the last option.

Why? because the localhost DNS service has not started yet at boot so that can cause some miner problems during startup / Why even offer localhost? fastest questions are the ones you can answer yourself right?

**Environment Variables**

You can also add variables to your applications YAML besides persisting the config in the backend, Cloudflare example:
```
      - env:
        - name: PIHOLE_DNS_
          value: 1.1.1.1;1.0.0.1;2606:4700:4700::1111;2606:4700:4700::1001
```
See https://hub.docker.com/r/pihole/pihole for all variables.

**CoreDNS**

You can also point the Kubernetes CoreDNS to your Pi-hole devices by editing the IP's using the following command:
```
microk8s kubectl -n kube-system edit configmaps coredns -o yaml
#nano
KUBE_EDITOR="nano" microk8s kubectl -n kube-system edit configmaps coredns -o yaml
```
Here you can change the IP's to your Pi-hole devices.

Special thanks to Pi-hole https://pi-hole.net/, Canonical https://snapcraft.io/ | https://microk8s.io/ and pralor https://hub.docker.com/u/pralor for even making these solutions possible!

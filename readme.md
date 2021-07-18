# Purpose:
Installing docker container for virtualbmc-for-vshpere.  This can be used by used by the ipmitool to power on/off VMs.  

# References:
https://github.com/kurokobo/virtualbmc-for-vsphere


One use case for this is for openstack director when installing openstack on a nested esxi environment.  Openstack Director requres the use of ipmi (ntelligent Platform Management Interface) as listed in  https://access.redhat.com/documentation/en-us/red_hat_openstack_platform/13/html/director_installation_and_usage/appe-Power_Management_Drivers.  When installing Openstack on bare metal, most industrial servers like the cisco ucs has ipmi capabilities built into the server.  However, in a lab environment if you wanted to install openstack over VMs spun up on esxi (managed by vCenter), then you need a method to fake out the ipmi capability for the VMs.   


# Requirements:  
VM with ubuntu and docker/docker-compose installed.  If you don't have that, please look at the bottom of this README file

# Steps to install:
1) ssh to ubuntu box and clone this repo
2) cd vBMC-docker-compose
3) vi the docker-compose file and modify / put in more entries for the port numbers based on the number of VMs you want to use ipmi for

   The 3 docker-compose port entries in this example are shown below:
 ```plain
    ports:
      - "6231:6231/udp"
      - "6232:6232/udp"
      - "6233:6233/udp"
```
3) docker-compose up --build -d ( or do docker-compose build   followed by docker-compose up -d)
4) make sure to do a docker ps to verify that container is up and running

# Steps to use:

1) Please use the vCenter administrator user and password.  I have not investigated what exact roles are needed, so at this time I'm not sure what exact privilige is needed for a non-admin vCenter user to be able to use ipmi.
 ```plain
   In this example, I have 3 VMs that I spun up from vCenter, Openstack-Controller1, OpenstackCompute1 and openstackCompute2.  My Vcenter IP is 10.1.100.60 and my vCenter credentials is administrator@vsphere.local / myvCenterPass.  Also, note, that in this case, since we are going to do nested virtualization, the VMs need to have "expose hardware assisted virtualization to guest OS" and also "enable virtualized cpu performance conters enabled" on CPU.  Also, on VM Options, please make sure to go to Boot Options and turn on EFI instead of BIOS.
   
From the base ubuntu box, where you are running the container, execute these commands:

docker-compose exec vbmc4vsphere vbmc add Openstack-Controller1 --port 6231 --viserver 10.1.100.60 --viserver-password myvCenterPass --viserver-username administrator@vsphere.local
docker-compose exec vbmc4vsphere vbmc add OpenstackCompute1 --port 6232 --viserver 10.1.100.60 --viserver-password myvCenterPass --viserver-username administrator@vsphere.local
docker-compose exec vbmc4vsphere vbmc add openstackCompute2 --port 6233 --viserver 10.1.100.60 --viserver-password myvCenterPass --viserver-username administrator@vsphere.local

To delete a mapping, let's say Openstack-Controller1, you would do: 
 docker-compose exec vbmc4vsphere vbmc delete Openstack-Controller1

```

2)  Now start the virutal BMCs:
 ```plain
docker-compose exec vbmc4vsphere vbmc start Openstack-Controller1
docker-compose exec vbmc4vsphere vbmc start OpenstackCompute1
docker-compose exec vbmc4vsphere vbmc start openstackCompute2

To stop an instance, let's say Openstack-Controller1, you would do:
 docker-compose exec vbmc4vsphere vbmc stop Openstack-Controller1

```

3)  Check to make sure the virutal BMCs are up:
 ```plain
docker-compose exec vbmc4vsphere vbmc list

To see more details on them you would do:
 docker-compose exec vbmc4vsphere vbmc show Openstack-Controller1
 docker-compose exec vbmc4vsphere vbmc show Openstack-Compute1
 docker-compose exec vbmc4vsphere vbmc show openstack-Compute2
 ```
# Steps to Test:
Now that you have installed vBMC container and enabled the mappings per VM, you need to test out that ipmi can power on/off the VMs.
First thing you need to do is install the ipmitool in the base ubuntu VM where you are running the container.  In my case, the base ubuntu box has an IP of 192.168.24.20.  Also, please note that the default password for ipmitool you need to use is admin/password as shown in the example below.

 ```plain
For the first VM, that has been configured to be managed by vBMC on port 6231, you would do the below:
ipmitool -I lanplus -H 192.168.24.20 -p 6231 -U admin -P password chassis power status
ipmitool -I lanplus -H 192.168.24.20 -p 6231 -U admin -P password chassis power on
ipmitool -I lanplus -H 192.168.24.20 -p 6231 -U admin -P password chassis power off
 ```

Once, you've verified that this works, then you are on your way to follow the redhat documentation and proceed with the install from undercloud VM.



# In case you don't have a vm with docker/docker-compose, follow these steps first to install:
--------------------------------------------------------------------------------------------
 ```plain
1) Download Ubuntu 20.04 --  https://ubuntu.com/download/desktop
2) Install Ubuntu VM minimum version
3) update/upgrade ubuntu and install basic utils: 
    sudo -i
    apt update && apt upgrade -y
    apt auto-remove
    apt install -y curl wget vim openssh-server
 4) Install docker and docker-compose:
    ssh to your Ubuntu VM on backend APIM EPG
    sudo -i
    apt-get update && apt-get upgrade -y
    echo net.ipv4.ip_forward=1 >> /etc/sysctl.conf
    sysctl -p
    sudo sysctl --system
    exit 
    sudo apt install docker.io -y
    sudo systemctl start docker
    sudo systemctl enable docker
    sudo groupadd docker
    sudo usermod -aG docker $USER
    exit # and ssh back in for this to work
    docker --version
    sudo apt install docker-compose -y

 ```




# Purpose:
Installing docker container for virtualbmc-for-vshpere.  This can be used by used by the ipmitool to power on/off VMs.  

One use case for this is for openstack director when installing openstack on a nested esxi environment.  Openstack Director requres the use of ipmi (ntelligent Platform Management Interface) as listed in  https://access.redhat.com/documentation/en-us/red_hat_openstack_platform/13/html/director_installation_and_usage/appe-Power_Management_Drivers.  When installing Openstack on bare metal, most industrial servers like the cisco ucs has ipmi capabilities built into the server.  However, in a lab environment if you wanted to install openstack over VMs spun up on esxi (managed by vCenter), then you need a method to fake out the ipmi capability for the VMs.   

# References:
https://github.com/kurokobo/virtualbmc-for-vsphere

# Requirements:  
VM with ubuntu and docker/docker-compose installed.  If you don't have that, please look at the bottom of this README file

# Steps to install:
1) ssh to ubuntu box and clone this repo:  git clone https://github.com/soumukhe/vBMC-docker-compose.git
3) cd vBMC-docker-compose
4) vi the docker-compose file and modify / put in more entries for the port numbers based on the number of VMs you want to use ipmi for

   The 3 docker-compose port entries in this example are shown below:
 ```plain
    ports:
      - "6231:6231/udp"
      - "6232:6232/udp"
      - "6233:6233/udp"
```
3) docker-compose up --build -d ( or do docker-compose build   followed by docker-compose up -d)
4) make sure to do a docker ps to verify that container is up and running
```plain
   * In case the container refuses to run with an error like so:
            vbmc4vsphere    | 2021-09-10 04:50:49,568.568 1 ERROR VirtualBMC [-] server PID #1 still running
            vbmc4vsphere exited with code 1
            
  the easist way to solve it is to remove docker all the way and reinstall:
   Docker Remove (ubuntu) Instructions:
     sudo apt-get purge -y docker-engine docker docker.io docker-ce docker-ce-cli docker-compose
     sudo apt-get autoremove -y --purge docker-engine docker docker.io docker-ce  docker-compose
     sudo rm -rf /var/lib/docker /etc/docker
     sudo rm /etc/apparmor.d/docker
     sudo groupdel docker
     sudo rm -rf /var/run/docker.sock
   Docker Install Instructions:
      See towards the end of this README.  Start from Step 4 of install
    
```

  
# Steps to use:

1) Please use the vCenter administrator user and password.  I have not investigated what exact roles are needed, so at this time I'm not sure what exact privilige is needed for a non-admin vCenter user to be able to use ipmi.
 ```plain
In this example, I have 3 VMs that I spun up from vCenter, Openstack-Controller1, OpenstackCompute1 and openstackCompute2.  
My Vcenter IP is 10.1.100.60 and my vCenter credentials is administrator@anywhere.bootcamp / myvCenterPass.  
Also, note, that in this case, since we are going to do nested virtualization, the VMs need to have "expose hardware assisted virtualization to guest OS" and also "enable virtualized cpu performance conters enabled" on CPU.  
Also, on VM Options, please make sure to go to Boot Options and turn on EFI instead of BIOS.
   
From the base ubuntu box, where you are running the container, execute these commands:

docker-compose exec vbmc4vsphere vbmc add Openstack-Controller1 --port 6231 --viserver 10.1.100.60 --viserver-password myvCenterPass --viserver-username administrator@anywhere.bootcamp
docker-compose exec vbmc4vsphere vbmc add OpenstackCompute1 --port 6232 --viserver 10.1.100.60 --viserver-password myvCenterPass --viserver-username administrator@anywhere.bootcamp
docker-compose exec vbmc4vsphere vbmc add openstackCompute2 --port 6233 --viserver 10.1.100.60 --viserver-password myvCenterPass --viserver-username administrator@anywhere.bootcamp

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
Now that you have installed vBMC container and enabled the mappings per VM, you need to test out that ipmi can power on/off the VMs targetting the port numbers you've defined in vBMC.
First thing you need to do is install the ipmitool in the base ubuntu VM where you are running the container.  

 ```plain
sudo apt-get update -y
sudo apt-get install -y ipmitool
 ```
 
Now execute the ipmitool commands to check power status, turn on/off VM

In my case, the base ubuntu box has an IP of 192.168.24.20.  Also, please note that the default password for ipmitool you need to use is admin/password as shown in the example below.

 ```plain
For the first VM, that has been configured to be managed by vBMC on port 6231, you would do the below:
ipmitool -I lanplus -H 192.168.24.20 -p 6231 -U admin -P password chassis power status
ipmitool -I lanplus -H 192.168.24.20 -p 6231 -U admin -P password chassis power on
ipmitool -I lanplus -H 192.168.24.20 -p 6231 -U admin -P password chassis power off

*** Note: ***  
The Actual commands sent by the Openstack Director during introspection and during overcloud installation are as follows: (you can verify them also)
ipmitool -I lanplus -H 192.168.24.20 -L ADMINISTRATOR -p 6231 -U admin -R 3 -N 5 -P password chassis power status 
ipmitool -I lanplus -H 192.168.24.20 -L ADMINISTRATOR -p 6231 -U admin -R 3 -N 5 -P password chassis power on
ipmitool -I lanplus -H 192.168.24.20 -L ADMINISTRATOR -p 6231 -U admin -R 3 -N 5 -P password chassis power off

The IP address and ports are defined in the Openstack Director in file instackenv.json (shown a little below)
When the "openstack overcloud install" command is issued on the director, ipmi will be used at the appropriate time to power on the nodes and pxe boot them with openstack software and configurations will be pushed to them.
 ```

Once, you've verified that this works, then you are on your way to follow the redhat documentation and proceed with the install from undercloud VM.
 ```plain
Please follow the redhat guide to make your /home/stack/instackenv.json file.
URL for guide is: https://access.redhat.com/documentation/en-us/red_hat_openstack_platform/13/html/director_installation_and_usage/appe-power_management_drivers#sect-VirtualBMC

One Note, I wanted to point out.  for the MAC please do not use the fake mac as listed from output of the vbmc show command.  Please use the real mac of the VMs.  As an example:

soumukhe@dhcp-server:~/vBMC-docker-compose$ docker-compose exec vbmc4vsphere vbmc list
+-----------------------+---------+---------+------+
| VM name               | Status  | Address | Port |
+-----------------------+---------+---------+------+
| Openstack-Controller1 | running | ::      | 6231 |
| OpenstackCompute1     | running | ::      | 6232 |
| openstackCompute2     | running | ::      | 6233 |
+-----------------------+---------+---------+------+

soumukhe@dhcp-server:~/vBMC-docker-compose$ docker-compose exec vbmc4vsphere vbmc show Openstack-Controller1
+-------------------+---------------------------------+
| Property          | Value                           |
+-------------------+---------------------------------+
| active            | True                            |
| address           | ::                              |
| fakemac           | 02:00:00:9e:1e:34               |
| password          | ***                             |
| port              | 6231                            |
| status            | running                         |
| username          | admin                           |
| viserver          | 10.1.100.60                     |
| viserver_password | ***                             |
| viserver_username | administrator@anywhere.bootcamp |
| vm_name           | Openstack-Controller1           |
+-------------------+---------------------------------+

soumukhe@dhcp-server:~/vBMC-docker-compose$ docker-compose exec vbmc4vsphere vbmc show OpenstackCompute1     
+-------------------+---------------------------------+
| Property          | Value                           |
+-------------------+---------------------------------+
| active            | True                            |
| address           | ::                              |
| fakemac           | 02:00:00:97:b7:c3               |
| password          | ***                             |
| port              | 6232                            |
| status            | running                         |
| username          | admin                           |
| viserver          | 10.1.100.60                     |
| viserver_password | ***                             |
| viserver_username | administrator@anywhere.bootcamp |
| vm_name           | OpenstackCompute1               |
+-------------------+---------------------------------+

soumukhe@dhcp-server:~/vBMC-docker-compose$ docker-compose exec vbmc4vsphere vbmc show openstackCompute2
+-------------------+---------------------------------+
| Property          | Value                           |
+-------------------+---------------------------------+
| active            | True                            |
| address           | ::                              |
| fakemac           | 02:00:00:30:a3:c8               |
| password          | ***                             |
| port              | 6233                            |
| status            | running                         |
| username          | admin                           |
| viserver          | 10.1.100.60                     |
| viserver_password | ***                             |
| viserver_username | administrator@anywhere.bootcamp |
| vm_name           | openstackCompute2               |
+-------------------+---------------------------------+

The instackenv.json file should now look like this (note 192.168.24.20 is the ubuntu box where the vBMC container is running:
Note:  The MAC addresses of the nodes are the real MACs of the VMs, not the fake mac from the output of vbmc show command.

(undercloud) [stack@undercloud ~]$ cat instackenv.json
{
  "nodes": [
    {
      "pm_type": "ipmi",
      "mac": [
        "00:50:56:a9:89:6e"
      ],
      "pm_user": "admin",
      "pm_password": "password",
      "pm_addr": "192.168.24.20",
      "pm_port": "6231",
      "name": "Openstack-Controller1"
    },
    {
      "pm_type": "ipmi",
      "mac": [
        "00:50:56:a9:e9:3a"
      ],
      "pm_user": "admin",
      "pm_password": "password",
      "pm_addr": "192.168.24.20",
      "pm_port": "6232",
      "name": "OpenstackCompute1"
    },
    {
      "pm_type": "ipmi",
      "mac": [
        "00:50:56:a9:fb:5d"
      ],
      "pm_user": "admin",
      "pm_password": "password",
      "pm_addr": "192.168.24.20",
      "pm_port": "6233",
      "name": "openstackCompute2"
    }
  ]
}

after making the /home/stack/instackenv.json file, follow from https://access.redhat.com/documentation/en-us/red_hat_openstack_platform/13/html/director_installation_and_usage/chap-configuring_basic_overcloud_requirements_with_the_cli_tools#sect-Registering_Nodes_for_the_Overcloud

 ```

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




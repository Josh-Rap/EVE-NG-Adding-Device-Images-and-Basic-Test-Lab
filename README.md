# EVE-NG Adding Device Images and Basic Test Lab
## Intro
**EVE-NG (Emulated Virtual Environment Next Generation)** is a powerful network emulation tool that allows you to create and test network configurations in a safe lab environment. I had an old gaming PC lying around that I decided to repurpose for EVE-NG.

For my CCNA studies, I primarily used Cisco's Packet Tracer with pre-made labs, but I never really experimented with building my own. If I continue on to CCNP, EVE-NG will be an invaluable tool. It’s also great for broader IT, networking, and cybersecurity learning. Some areas I plan to explore include:

- Network automation (Ansible)
- SD-WAN fundamentals
- MPLS and provider networking
- Cybersecurity labs (firewalls, secure network architectures)

**Note:** I deployed EVE-NG directly on the server (Bare Metal EVE), not in a virtual machine. This allows it to fully utilize the system’s hardware and gives better performance overall.

This write-up covers adding device images to EVE-NG and testing a basic lab setup. I use WinSCP and SSH for file transfers and configuration.

Here is a preview of the final lab topology:
![image](images/Pasted%20image%2020250404144737.png)
## Adding Device Images
To build a flexible and realistic lab environment, I added device images from several vendors, including routers, switches, firewalls, and end hosts.

**Currently added:**
- **Cisco IOS**  – Core routing and switching 
- **Cisco ASA** – Firewall testing and basic security scenarios
- **Palo Alto** – Advanced firewall/cybersecurity labs
- **Aruba Switch** – Non-Cisco switching and config testing
- **Linux Hosts** – Ubuntu and Kali for endpoint simulation

Most device images can be found online through GitHub repos, community forums or from the vendor itself.
### Adding an Image to EVE-NG:
The general steps for adding a device image are as follows:

1. Download the image from the vendor or a trusted repository  
2. Extract the contents (if the file is compressed)
3. Use a tool like WinSCP to upload the image into the appropriate directory (ex: `/opt/unetlab/addons/qemu/arubacx-version#`)  
    - Some devices require format conversion, a virtual HDD, or other device-specific steps  
    - Others may be pre-converted and ready to go  
    - EVE-NG has [official documentation](https://www.eve-ng.net/index.php/documentation/howtos/) for most supported images  
4. Fix file permissions using:
    ```
    /opt/unetlab/wrappers/unl_wrapper -a fixpermissions
    ```  
5. Launch the EVE web interface and deploy the new device

### Example: Adding a Juniper Switch
I’m planning to add Juniper, FortiGate, and other vendor images to expand device coverage. I’ll document adding the Juniper switch to EVE-NG here.

> **Note:** vJunos devices are not supported on EVE-NG installations running in a VM, Bare Metal EVE is recommended.

I first [downloaded the image from Juniper](https://support.juniper.net/support/downloads/?p=vjunos). In this case, the file was already formatted and not compressed, so I was able to move straight to placing it on the EVE-NG server.

I open a terminal and use SSH to connect to the EVE server and then create a new directory for the Juniper image:
```
mkdir /opt/unetlab/addons/qemu/vjunosswitch-24.4R1
```
The directory naming is the important part, `vjunosswitch` must be spelled exactly like that in order to work. The part after the `-` can be whatever is desired, in most cases the version number.

I use a tool like WinSCP to move the file from my laptop over to the newly created directory on the EVE server. 
![image](images/Pasted%20image%2020250404121441.png)

Then, back on SSH I go to the Juniper Switch directory, check that the file has been transferred, rename the file and fix the permissions:
```
root@eve-ng2:~# cd /opt/unetlab/addons/qemu/vjunosswitch-24.4R1
root@eve-ng2:/opt/unetlab/addons/qemu/vjunosswitch-24.4R1# ls
vJunos-switch-24.4R1.9.qcow2
root@eve-ng2:/opt/unetlab/addons/qemu/vjunosswitch-24.4R1# mv vJunos-switch-24.4R1.9.qcow2 virtioa.qcow2
root@eve-ng2:/opt/unetlab/addons/qemu/vjunosswitch-24.4R1# /opt/unetlab/wrappers/unl_wrapper -a fixpermissions

root@eve-ng2:/opt/unetlab/addons/qemu/vjunosswitch-24.4R1# ls -l
total 3940868
-rw-r--r-- 1 root root 4035444736 Apr  4 16:03 virtioa.qcow2
```

Going to my EVE-NG web interface and trying to add a node I can see the new `Juniper vEX Switch` template using the `vjunosswitch-24.4R1` image that I uploaded. I add it to the topology and am able to login to the console and go into Configuration Mode. 
![image](images/Pasted%20image%2020250404122509.png)
![image](images/Pasted%20image%2020250404132906.png)
## Basic Lab Setup Within EVE-NG
To test the new images and confirm EVE is working properly, I created a simple lab.
![image](images/Pasted%20image%2020250404133654.png)

<!-- After completing the setup, the lab topology looked like this:
![image](images/Pasted%20image%2020250404144737.png) -->
### Add Devices and Connections:
![image](images/Pasted%20image%2020250404134103.png)
### Create the VLANs on `Juniper-Switch`
- Create VLANs 10 and 20
- Assign **access** ports to clients
- Set up a **trunk** to the router

```
set vlans VLAN10 vlan-id 10
set vlans VLAN20 vlan-id 20

set interfaces ge-0/0/1 unit 0 family ethernet-switching vlan members VLAN10
set interfaces ge-0/0/2 unit 0 family ethernet-switching vlan members VLAN20
set interfaces ge-0/0/0 unit 0 family ethernet-switching interface-mode trunk
set interfaces ge-0/0/0 unit 0 family ethernet-switching vlan members all
```
### Configure `Cisco-Router` with ROAS and DHCP
- Set up **sub-interfaces** for VLAN routing
- Configure **DHCP** for each VLAN

```
interface Ethernet1/0
 no shutdown

interface Ethernet1/0.10
 encapsulation dot1Q 10
 ip address 192.168.10.1 255.255.255.0

interface Ethernet1/0.20
 encapsulation dot1Q 20
 ip address 192.168.20.1 255.255.255.0

ip dhcp pool VLAN10
 network 192.168.10.0 255.255.255.0
 default-router 192.168.10.1
 dns-server 192.168.10.1

ip dhcp pool VLAN20
 network 192.168.20.0 255.255.255.0
 default-router 192.168.20.1
 dns-server 192.168.20.1
```
### Checking Connectivity
Pinging from `Ubuntu-Server` in VLAN10 to a host (`Kali`) in VLAN20 succeeded, confirming inter-VLAN routing is functional. Both endpoints received IPs in the correct subnets, verifying DHCP is working as expected as well. 
![image](images/Pasted%20image%2020250404141821.png)

## Conclusion, Further Exploration
In this write-up, we walked through adding vendor device images to an EVE-NG server, and testing a basic VLAN and inter-VLAN routing lab. 

This setup opens the door for much more complex and valuable lab scenarios. Some further ideas I have that I may explore include:

- Deploying FortiGate and Palo Alto firewalls 
- Building a site-to-site VPN lab between different firewall vendors
- Creating an Ansible-powered network automation lab
- [Simulating MPLS and BGP core network](https://github.com/Josh-Rap/Exploring-MPLS-Networking-and-Configuration) (writeup in progress)
- Setting up an SD-WAN testbed with different WAN edge devices
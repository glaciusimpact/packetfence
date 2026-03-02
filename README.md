# PacketFence

PacketFence is an open-source Network Access Control (NAC) system. This software is free if it is self-hosted (cloud version is not free and support is not free either). With this tool we can protect a network allowing or denying network devices connecting to network devices like network switches or Wifi Access Point.

# 1 - Objective

The objective here is to install a NAC solution allowing us to use:
- MAC Authentication Bypass (MAB)

**MAC Authentication Bypass (MAB)** is used to authenticate a device just with its MAC address on the network. If this device is recognized then it can send data to the network. This is just simple as that. This security mechanism can be bypassed, but it is a fairly simple first level of security for a network architecture wanting to implement NAC, which can then evolve towards stronger authentication (802.1X or other).

# 2 - Architecture

Very simply a Network switch (from Cisco) is connected to the Internet and to a device (a Computer). The switch used here is a small SG3XX Cisco switch. If the Computer is identified then a configuration will be sent to the switch port which will allow the Computer to access the network.

```
Computer -> Network switch -> Internet
                   ^
                   |
              PacketFence
```

The authentication of the devices (PC, printer, etc.) is performed that way:
1. First the switch uses Radius protocol to send the MAC address of a device connecting to its port to PacketFence (which runs with a FreeRadius server embedded).
2. PacketFence checks if the device is registered and has a Role.
3. If the device does not have a Role then the device is not identified and the switch will apply to its port a default VLAN (called registration in the PacketFence configuration).
4. If the device has a Role then the appropriate VLAN is provided in Radius by PacketFence then applied to the switch port.

With this logic we can set the correct VLANs to the device and a default VLAN for other devices (like guest users).

# 3 - Installation

## 3 - 1 Requirements

The installation of PacketFence will be made on VirtualBox using the ISO image provided (around 900 MB) for self-hosting.

According to the documentation:

> [!NOTE]
> The ISO edition of PacketFence allows you to install PacketFence on Debian 12 with minimal effort.
>
> Download the ISO here: [https://www.packetfence.org/download.html#/releases](https://www.packetfence.org/download.html#/releases)
> 
> This setup has been tested using VMware ESXi, Proxmox VE and VirtualBox and works with any hypervisor PacketFence supports as well as bare-metal servers.
> 
> A virtual machine or server with 16 GB of RAM dedicated to machine as well as 4 CPUs is required. Allocate at least 200GB of disk space for PacketFence.

Well, well, well. In production we have to respect the requirements but for this little lab we will use a configuration with lower features since very few devices will be used:
- 4 cores
- 8 GB of RAM
- 25 GB of disk

ISO file downloaded:
- PacketFence-ISO-v15.0.0.iso

## 3 - 2 Installation with VirtualBox

Here are the parameters used by the VirtualBox VM:
- VM Name
	- PacketFence
- ISO Image
	- PacketFence-ISO-v15.0.0.iso
- Proceed with Unattended Installation
	- Unchecked!
- Base Memory
	- 8192 MB*
- Number of CPUs
	- 4
- Create a New Virtual Hard Disk
	- 25 GB**

\*: 6 GB is not enough. Only 74 MB are available in that configuration and the server is sometimes unreacheable. 8 GB is just fine for a little lab.
\*\*: 20 GB is very low since after installation the available space will be 740 MB. So 25 GB is the minimum prefered.

-> Finish button

Since I do not want to make PAT then I switch off the VM and set interface as Bridge.

VirtualBox
- Configuration
	- Network
		- Adapter 1
			- Bridge
				- OK button
- Run the VM

On the appearing menu we choose:
- "Install Debian with PacketFence"
- Hostname
	- packetfence
		- Continue
- Domain name
	- (nothing)
		- Continue
- Root password
	- (choose a good password in production)
		- Continue
- Re-enter password
	- (type same password as before)
		- Continue

"Finishing the installation" part after is quite long (around 25 minutes). "Running preseed" is staying at 14% for a moment (20 minutes) downloading packages from the Internet. To check if there is an issue type Ctrl+Alt+F4 to see the install logs (Ctrl+Alt+F1).

> [!WARNING]
> Warning: if you set a domain name without DNS then the installation will be endless...

When the installation is finished I got a prompt. 

The IP address mentioned on the login banner was wrong (probably because the server rebooted and took another IP address). To find the IP address of the Web interface of PacketFence login as root with the password used during the installation.

Then type:
- ip a
You will find the IP address of the server (eth0).

Then you can reach the web interface with a web Browser (replace the IP address with the one of your server):
https://192.168.1.68:1443/

## Step 1 - Interface

- Click on "Detect Management Interface" button
	- Close the popup window
	- Wait
		- The interface get the type "management"
- Click on "Next Step >" button

## Step 2 - Database

- Administrator
	- (choose a password for the database)
- Click on "Next Step >" button

## Step 3 - Fingerbank

- Click on "Next Step >" button

## Step 4 - Passwords

Save
- Database Root Account Password (root)
- Database User Account Password (pf)
- Administrator Account Password (admin)

- Click on "Start PacketFence" button

You can login with admin account.

# 4 - Adding an allowed Role on PacketFence

This Role will be used by all devices of the same VLAN we are going to use.

- Configuration
	- Roles
		- New Role
			- Name
				- ALLOWED_USER_LAN
			- Description
				- User allowed to access the network
			- Create button

# 5 - Adding a Cisco Switch on PacketFence

To add a switch on PacketFence we can use the online documentation:
https://www.packetfence.org/doc/PacketFence_Network_Devices_Configuration_Guide.html#_cisco_small_business_smb

However this documentation is not enough and we have to search for our own configuration.

We want:
- By default we will put a device into VLAN 1 on the switch.
- If the device is identified then it will be in VLAN 2.

Note about below parameter:
- registration: this is the default VLAN when a device is not authenticated (it is not "default" parameter).
- If you get in the PacketFence logs a message saying **"No category computed for autoreg"** and the registration VLAN not applying to your switch then it might be because of a registered device with the option "Automatically register devices" in a "Configuration/Policies and Access Control/Connection profile". After disabling this option you have also to remove the device from "Nodes".

- Configuration
	- Policies and Access Control
		- Network Devices
			- Switches
				- New Switch
					- Default
						- New switch
							- Definition
								- IP Address/MAC Address/Range (CIDR)
									- 192.168.1.254
								- Type
									- Cisco SG300
								- Mode
									- Production
								- Deauthentication Method
									- RADIUS
							- Roles
								- Role by VLAN ID
									- Default (Yes)
								- registration
									- 1
								- ALLOWED_USER_LAN
									- 2
							- RADIUS
								- Secret Passphrase
									- Hiddenp4ssphra$e
							- Save button


# 6 - Configuring Cisco Switch for NAC (Radius)

## 6 - 1 Radius configuration with PacketFence

Every switch has its own configuration. You can find the configuration of the switches to activate Radius for PacketFence here:
https://www.packetfence.org/doc/PacketFence_Network_Devices_Configuration_Guide.html

Here is the configuration to type on the switch:
```
dot1x system-auth-control
radius-server key Hiddenp4ssphra$e
radius-server host 192.168.1.68
aaa accounting dot1x start-stop group radius
```

``` pascal
switch01#conf t
switch01(config)#
switch01(config)#dot1x system-auth-control
switch01(config)#radius-server key Hiddenp4ssphra$e
switch01(config)#radius-server host 192.168.1.68
switch01(config)#aaa accounting dot1x start-stop group radius
switch01(config)#
```
## 6 - 2 Cisco switch interface configuration

In the configuration below we will configure only 1 port (Gi1) but in production it should be applied on all endpoint ports of the switch.

```
dot1x host-mode multi-sessions
dot1x reauthentication
dot1x timeout reauth-period 10800
dot1x timeout quiet-period 10
dot1x timeout server-timeout 5
dot1x timeout supp-timeout 3
dot1x authentication mac
dot1x radius-attributes vlan
dot1x port-control auto
spanning-tree portfast
switchport mode general
switchport general pvid 1
```

``` pascal
switch01(config)#
switch01(config)#int gi1
switch01(config-if)#dot1x host-mode multi-sessions
switch01(config-if)#dot1x reauthentication
switch01(config-if)#dot1x timeout reauth-period 10800
switch01(config-if)#dot1x timeout quiet-period 10
switch01(config-if)#dot1x timeout server-timeout 5
switch01(config-if)#dot1x timeout supp-timeout 3
switch01(config-if)#dot1x authentication mac
switch01(config-if)#dot1x radius-attributes vlan
switch01(config-if)#dot1x port-control auto
switch01(config-if)#spanning-tree portfast
switch01(config-if)#switchport mode general
switch01(config-if)#switchport general pvid 1
switch01(config-if)#
```

# 7 - First test: an unknown device

The test is easy. I shut/no shut the port configured then I check the logs on PacketFence and the VLAN table on the switch.

Commands on the switch:

``` pascal
switch01(config-if)#shutdown
switch01(config-if)#02-Mar-2026 15:50:55 %LINK-W-Down:  gi1

switch01(config-if)#no shutdown
switch01(config-if)#02-Mar-2026 15:51:03 %LINK-I-Up:  gi1
02-Mar-2026 15:51:04 %LINK-W-Down:  gi1
02-Mar-2026 15:51:07 %LINK-I-Up:  gi1

switch01(config-if)#do sh vlan
Created by: D-Default, S-Static, G-GVRP, R-Radius Assigned VLAN, V-Voice VLAN

Vlan       Name           Tagged Ports      UnTagged Ports      Created by
---- ----------------- ------------------ ------------------ ----------------
 1           1                             gi1,gi3-10,Po1-8        DVR
 2           2                gi10               gi2                R

switch01(config-if)#
```

-> gi1 is set to VLAN 1

Let's see the PacketFence logs:

``` pascal
root@packetfence:~# tail -f /usr/local/pf/logs/packetfence.log

[...]

2026-03-02T14:51:09.360384+00:00 packetfence httpd.aaa-docker-wrapper[3695]: httpd.aaa(7) INFO: [mac:22:22:22:22:22:22] handling radius autz request: from switch_ip => (192.168.1.254), connection_type => Ethernet-NoEAP, switch_mac => (11:00:11:00:11:00), mac => [22:22:22:22:22:22], port => 1, username => "b422000c5a7d" (pf::radius::authorize)
2026-03-02T14:51:09.398105+00:00 packetfence httpd.aaa-docker-wrapper[3695]: httpd.aaa(7) INFO: [mac:22:22:22:22:22:22] Instantiate profile default (pf::Connection::ProfileFactory::_from_profile)
2026-03-02T14:51:09.473455+00:00 packetfence httpd.aaa-docker-wrapper[3695]: httpd.aaa(7) INFO: [mac:22:22:22:22:22:22] is of status unreg; belongs into registration VLAN (pf::role::getRegistrationRole)
2026-03-02T14:51:09.486350+00:00 packetfence httpd.aaa-docker-wrapper[3695]: httpd.aaa(7) INFO: [mac:22:22:22:22:22:22] (192.168.1.254) Added VLAN 1 to the returned RADIUS Access-Accept (pf::Switch::returnRadiusAccessAccept)
2026-03-02T14:51:09.503684+00:00 packetfence httpd.aaa-docker-wrapper[3695]: httpd.aaa(7) WARN: [mac:22:22:22:22:22:22] No parameter registrationRole found in conf/switches.conf for the switch 192.168.1.254 (pf::Switch::getRoleByName)
2026-03-02T14:51:09.586540+00:00 packetfence httpd.aaa-docker-wrapper[3695]: httpd.aaa(7) INFO: [mac:22:22:22:22:22:22] Updating locationlog from accounting request (pf::api::handle_accounting_metadata)


^C
root@packetfence:~#
```

-> "Added VLAN 1 to the returned RADIUS Access-Accept": the default VLAN is set to the switch port. Good.


# 8 - Adding a known device

Now we want this device will be allowed to reach our network (VLAN 2 aka ALLOWED_USER_LAN).


We have 2 choices to do that:
- Either the device is already connected to the switch port and can be found in "Nodes/Search" or "Nodes/Online Nodes"
- or the device is not yet connected to the switch port, in which case the device is created with "Nodes/Create".

Parameters are the same where the device is connected or not. Here they are:

- Nodes
	- Create
		- MAC
			- 22:22:22:22:22:22
		- Role
			- ALLOWED_USER_LAN
	- Create button

Note: 22:22:22:22:22:22 is the MAC address of your device and must be adapted to your device.

# 9 - Second test: a know device

Like previously: I do a shut/no shut on the Gi1 port of the switch and I check the results on PacketFence log and on the switch.


``` pascal
switch01(config-if)#shut
switch01(config-if)#02-Mar-2026 16:07:44 %LINK-W-Down:  gi1

switch01(config-if)#no shut
switch01(config-if)#02-Mar-2026 16:07:53 %LINK-I-Up:  gi1
02-Mar-2026 16:07:54 %LINK-W-Down:  gi1
02-Mar-2026 16:07:56 %LINK-I-Up:  gi1

switch01(config-if)#do sh vlan
Created by: D-Default, S-Static, G-GVRP, R-Radius Assigned VLAN, V-Voice VLAN

Vlan       Name           Tagged Ports      UnTagged Ports      Created by
---- ----------------- ------------------ ------------------ ----------------
 1           1                               gi3-10,Po1-8           DV
 2           2                gi10              gi1-2               R

switch01(config-if)#
```

-> port Gi1 is now on VLAN 2 (with port Gi2)

How about the PacketFence log?

``` pascal
root@packetfence:~# tail -f /usr/local/pf/logs/packetfence.log

[...]

2026-03-02T15:07:58.596011+00:00 packetfence httpd.aaa-docker-wrapper[3695]: httpd.aaa(7) INFO: [mac:22:22:22:22:22:22] handling radius autz request: from switch_ip => (192.168.1.254), connection_type => Ethernet-NoEAP, switch_mac => (11:00:11:00:11:00), mac => [22:22:22:22:22:22], port => 1, username => "b422000c5a7d" (pf::radius::authorize)
2026-03-02T15:07:58.615323+00:00 packetfence httpd.aaa-docker-wrapper[3695]: httpd.aaa(7) INFO: [mac:22:22:22:22:22:22] Instantiate profile default (pf::Connection::ProfileFactory::_from_profile)
2026-03-02T15:07:58.658153+00:00 packetfence httpd.aaa-docker-wrapper[3695]: httpd.aaa(7) INFO: [mac:22:22:22:22:22:22] Found authentication source(s) : 'local,file1' for realm 'null' (pf::config::util::filter_authentication_sources)
2026-03-02T15:07:58.658153+00:00 packetfence httpd.aaa-docker-wrapper[3695]: httpd.aaa(7) INFO: [mac:22:22:22:22:22:22] Connection type is MAC-AUTH. Getting role from node_info (pf::role::getRegisteredRole)
2026-03-02T15:07:58.658153+00:00 packetfence httpd.aaa-docker-wrapper[3695]: httpd.aaa(7) INFO: [mac:22:22:22:22:22:22] Username was defined "b422000c5a7d" - returning role 'ALLOWED_USER_LAN' (pf::role::getRegisteredRole)
2026-03-02T15:07:58.658153+00:00 packetfence httpd.aaa-docker-wrapper[3695]: httpd.aaa(7) INFO: [mac:22:22:22:22:22:22] PID: "default", Status: reg Returned VLAN: (undefined), Role: ALLOWED_USER_LAN (pf::role::fetchRoleForNode)
2026-03-02T15:07:58.671135+00:00 packetfence httpd.aaa-docker-wrapper[3695]: httpd.aaa(7) INFO: [mac:22:22:22:22:22:22] (192.168.1.254) Added VLAN 2 to the returned RADIUS Access-Accept (pf::Switch::returnRadiusAccessAccept)
2026-03-02T15:07:58.674019+00:00 packetfence httpd.aaa-docker-wrapper[3695]: httpd.aaa(7) WARN: [mac:22:22:22:22:22:22] No parameter ALLOWED_USER_LANRole found in conf/switches.conf for the switch 192.168.1.254 (pf::Switch::getRoleByName)
^C
root@packetfence:~#
```

->

We can see:

```
 Role: ALLOWED_USER_LAN 
```
and
```
Added VLAN 2 to the returned RADIUS Access-Accept
```

The port has been configured dynamically in VLAN2 as expected. Wonderful!


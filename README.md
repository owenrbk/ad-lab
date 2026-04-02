# Active Directory Domain Services Lab
Enterprise Infrastructure Simulation of Windows Server Active Directory Domain Services, steps with images.

## Project Overview
This project involves the end-to-end architecture of a localized Enterprise Infrastructure. I implemented a dual-homed Windows Server environment to simulate real-world networking, identity management via AD DS, and automated policy enforcement through Group Policy.

## Creating the Virtual Machines
* I created two virtual machines with Microsoft Hyper-V, allocating 4GB of RAM and 80GB of virtual disk space to each. One is the domain controller (DC01) with Windows Server 2022 and the other is a client workstation (CLIENT01) with Windows 10.
![Hyper-V Setup](images/1.PNG)
* I created two virtual network interfaces, one called Internal-Lab, which connects the Client to the Domain Controller, and one called Internet-Bridge, which bridges internet connectivity from the Domain Controller to the Client. By configuring this dual-homed setup, I established a secure gateway where the Domain Controller functions as a router, providing the internal lab assets (Client) with WAN access through Network Address Translation (NAT) without exposing them directly to the public internet. We configure this fully when we set up routing (RRAS).
![Network Interfaces](images/3.png)
![Network Interface Setup, Internal Only and External Virtual Switch](images/4.png)

## Configuration of DC01
* When Windows Server OS finished installing, I entered "Administrator" for the username and a strong password I could remember.
* Upon logging in, I went to Network Connections (run ncpa.cpl) and configured a static IP

![Static IP](images/2.png)

## Configuring Active Directory
* In the Server Manager, I click Add Roles and Features under Manage in the top right corner.
* I select "Role-based or feature-based installation".
* I select DC01 as the Server.
* I select Active Directory Domain Services, DHCP Server, DNS Server, and Remote Access. Any default features I also add.
* I confirm installation and click install.

## Promoting to Domain Controller
* Once the features are done installing, I click the yellow flag in the top right corner and click promote to domain controller

![Promote to DC](images/5.PNG)

* In the AD DS Wizard, I click **Add a new forest**. For now, my root domain name is **lab.local**

![Domain Name](images/6.PNG)

* In the next steps, I leave them as default and I input a Directory Services Restore Mode password, which is something that needs to be saved in case AD is in need of recovery.
* I leave the rest of the steps as the default and ignore warnings in the Prerequisites Check, as is this a lab environment. Then I click install.

![Domain Controller Configured](images/7.PNG)

* After rebooting the machine, i'm able to log in to the Administrator account in the domain under **LAB\Administrator**
* In an elevated command prompt, I am able to see the domain's IP address using `nslookup lab.local`

## Creating Organizational Units
* In the Server Manager, I click **Tools** in the top right corner and click **Active Directory Users and Computers**.
* I right click lab.local, then **New**, then **Organizational Unit**.
* I name the OU **_Branches** with an underscore so it appears first in a list, and I make sure **Protect container from accidental delection** is checked.
* Under _Branches, I create another OU called **Raleigh**, and under Raleigh, I create three more called **Users**, **Workstations**, and **Laptops**.

![Organizational Units](images/8.PNG)

## Creating Users
* In Active Directory Users and Computers, I right click the **Users** OU I created in the last step and then **New**, then **Users**.
* I create three users (Jim Valvano, DJ Burns, and James Goodnight). A password is set for each one and I leave **User must change password at next logon** unchecked.

![Users](images/9.PNG)

## Creating Security Groups
* In Active Directory Users and Computers, I right click lab.local, then **New**, then **Organizational** **Unit**. This OU is called **_Groups**.
* When right clicking _Groups, I select **New**, then **Group**.
* Here I create 3 Groups called Accounting, HR, and IT. The group scope is **Global** and the group type is **Security**

![Security Groups](images/10.png)

* Each user I created is assigned a security group. To do this, I open the group, select the members tab, then add the user.

## Client Computer Domain Configuration
* Using the CLIENT01 Workstation created in previous steps, we will first log in with a local administrator account.
* Upon login, we will go to network connections (Windows + R, ncpa.cpl), and change the IPv4 properties of the network adapter so that the preferred DNS server is the domain controller's private IP Address. This ensures that our Active Directory Domain Names can be read by our client computer.
* Next is to join the domain. We will need to open **Settings**, **System**, **About**, then **Rename this PC (advanced)**.
* Here we can rename the computer to **client**, and make it a member of **lab.local**. It will prompt to restart the PC.

![Rename PC](images/11.PNG)

* Upon restart, we can log in to the client computer as our domain administrator account. We can see in Active Directory our new client in **lab.local** under the container **Computers**.

![Client Join](images/12.png)

* We can go back into AD Users and Computers and drag the Client Computer Account into **Workstations** under **_Branches** > **Raleigh**.

## Group Policy RDP Access & Domain User Login
* I am doing this lab with Remote Desktop Protocol into my home server. Thus, I must configure a specific permission to allow a non-administrator to log in via Remote Desktop Protocol.
* In the Server Manager, I click **Tools** in the top right corner and click **Group Policy Management**.
* I right click the **Raleigh** group, then **Create a GPO in this domain, and Link it here...**
* In the image below we see where to locate the Remote Desktop Services permission.

![GP Management Editor](images/13.png)

* Here we see the GPO linked to the Raleigh location. This GPO can now be pushed to clients in command prompt with `gpupdate /force`

![GPO](images/14.png)

* In the client computer's login screen, we select Other User, then enter in one of our users username and password. The username must be entered with the `LAB\` domain prefix. We can then identify ourselves in command prompt with `whoami /groups`.

![whoami](images/15.png)

## Setting a Password Policy
* It is important to understand common responibilities and nuances of security requirements that organizations use and how Active Directory plays a huge role. Setting a domain-wide password policy shows just this.
* In the Server Manager, I click **Tools** in the top right corner and click **Group Policy Management**.
* Under **lab.local**, right click **Default Domain Policy** and click **Edit...**
* In the drop down menu on the left panel as shown in the image below, we can make our way to **Password Policy** and set any requirements needed. In this case, I changed the maximum password age to 365 days and the minimum password length to 7 characters. I kept everything else as default.

![Password Policy](images/16.png)

## Permission Delegation
* I want to make it so everyone on the IT Team, admin or not, can perform specific permission for their job, such as password resets and forcing a change at next login. To do this, I must go into AD Users and Computers, right click **Users** under **_Branches** > **Raleigh**, and select **Delegate Control**. There I can select tasks to delegate to non-admin IT users like Help Desk.

![Task Delegation](images/17.png)

## DHCP Configuration
* When first setting up the Client Computer, I used a static IP address. Outside a lab environment with many workstations, we will need DHCP to assign IP addresses.
* In the Server Manager, I click **Tools** in the top right corner and click **DHCP**.
* I select **IPv4** on the left panel, and in the **Actions** screen on the right, I click **New Scope...**
* In this Wizard, I name the scope **Raleigh-Workstations** and I configure the last octet to use 150-250 as the DHCP range, allowing 101 devices for DHCP. this leaves anything outside that range for static IPs. I set the lease duration to 8 days.

![DHCP Scope](images/18.png)

* In the next steps, I list my Domain Controller's private IP address as the Default Gateway and the DNS Server. Then I activate the scope now. My Client Computer can now connect with DHCP. I verify in command prompt with `ipconfig /all`, where I see DHCP is enabled.

![DHCP Enabled](images/19.png)

## DNS Forwarding Configuration
* Next, we need to forward a public DNS resolver through our domain controller to our client computer. This way, switching DNS from say Google to Cloudflare is centralized by changing it once on the domain controller, which forwards it.
* In the Server Manager, I click **Tools** in the top right corner and click **DNS**.
* After right clicking **DC01** and selecting **Properties**, I can go to the **Forwarders** tab and add Google and Cloudflare. If one server goes down, it can default to the other.

![DNS Forwarding](images/20.png)

## Router Configuration
* Next, we will set up our domain controller as a router for our client computer.
* In the Server Manager, I click **Tools** in the top right corner and click **Routing and Remote Access**.
* After right clicking DC01, I select **Configure and Enable**, custom configuration, and check **NAT** and **LAN routing**
* I set my Internet-Bridge Interface as the public interface connected to the network, and my Internal-Lab as the private interface connected to private network.
* In the image below we see inbound and outbound packets for each network interface

![NAT Interfaces](images/21.png)

## Final Lab Validation
To confirm the integrity of the full stack (Networking, Identity, and Policy), I performed a final verification from the **CLIENT01** workstation:
* **Authentication:** Successfully authenticated as a non-admin 'Raleigh' user via RDP, confirming that the GPO and User Rights Assignments were active.
* **Connectivity:** Successfully reached external web resources (google.com) via the DC01 NAT Gateway.
* **Resolution:** Confirmed internal name resolution by pinging lab.local and resolving to the DC's private IP.

![Command Validation](images/22.png)

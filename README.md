# Using Active Directory (AD) and Sophos Firewall to block distracting sites
## Goal
For this quick lab, my goal is to utilize AD to block access to distracting sites such as YouTube, Reddit, Facebook, etc... across the entire domain.
## Preparations
- The AD lab is based on the one from [TCM Academy's free Practical Help Desk](https://tcm-sec.com/academy/practical-help-desk/) course, which I finished beforehand
	- VirtualBox is the hypervisor used for launching the VMs
 	- Domain controller (DC) and Windows 11 virtual machine (VM) are in the same domain
	- Verified that the Windows 11 VM doesn't have an alternate DNS server set
	- Ensured that the DC and VM only have one adapter connected to the internet (NAT network adapter)
- Did lots of research on ways I could achieve this goal. These are the methods I tried:
## First method: Modifying AD DNS
1. In the DC, on Server Manager, navigate to **Tools > DNS**:
2. Right-click on **Forward Lookup Zones**, then select **New Zone...**
3. Click Next up to **Zone Name**. Here, enter the domain to be blocked (without `www.`):

	![](/screenshots/1.png)

4. Under **Dynamic Update**, select **Do not allow dynamic updates**, then proceed with creating the zone
5. Double-click on the zone, then right-click in the white area and select **New Host (A or AAAA)...**
6. Under **IP address**, enter `0.0.0.0` (all IPs) or `127.0.0.1` (localhost address), check **Create associated pointer (PTR) record**, then click **Add Host**:
7. Click **Done**. The new host record should now appear:

	![](/screenshots/2.png)

8. Repeat steps 2-7 for as many domains to be blocked
9. Open a command prompt as administrator and run `gpupdate /force` across all machines on the domain to apply the policy changes, as well as `ipconfig /flushdns` to reset previously stored DNS caches

I then tested things by attempting to access YouTube on the DC. Sure enough, I wasn't able to:

![](/screenshots/3.png)

Same with the VM:

![](/screenshots/4.png)

While I could've stopped there, I wanted to go further and try to display a custom block page to domain users should they try to access the blocked pages.
## Second method: Integrating with Sophos Firewall
In order to get a custom block message up and running, I needed to use some third-party web filtering software (Sophos Firewall, Admin by Request) and have it connect to the Active Directory domain controller. Ultimately, I decided on Sophos Firewall for this. According to the installation guides, I needed to set this up in a separate VM with at least 4GB of RAM. After signing up for the [Free Home Edition](https://www.sophos.com/en-us/free-tools/sophos-xg-firewall-home-edition) and then failing to install the ISO file myself, I hunted down a preconfigured setup of Sophos for VirtualBox [here](https://www.sophos.com/en-us/support/downloads/firewall-installers) (under the VMWare virtual installer for Sophos Firewall OS).

### Setting up Sophos
1. I first modified the network settings in VirtualBox Manager for my Sophos VM to have adapter #1 use the `ADNetwork` NAT network that I set up during the Practical Help Desk course. Additionally, I made sure that my DC and Windows 11 VMs only have the NAT network adapter active for networking.
2. Launching the Sophos VM for the first time, I entered the password `admin` to begin the setup process
3. Under `Network configuration menu`, I chose `Yes` to `Set IPv4 Address`. For `PortA` (local area network (LAN)), modify the IP address to be within `ADNetwork`'s subnet mask (in my case, `10.5.5.0/24`):

	![](/screenshots/5.png)

4. Leave `PortB` (wide area network (WAN)) as is. Launch the DC and Windows 11 VMs, then in either one, enter `https://<the new LAN IP address from step 3>:4444` into a browser:

	![](/screenshots/6.png)
	![](/screenshots/7.png)

5. Go through the setup process:
	- Leave **Install latest firmware** checked
	- For **Serial Number**, use the one sent to your email from registering for Sophos Home
	- De-check **Enable DHCP**, and choose **This firewall** as the gateway
	- Since I'm running this firewall for the sole purpose of blocking distracting sites, leave everything under the **Network protection** section unchecked
	- Under **Notifications and Backups**, feel free to enter your actual or dummy email(s). Uncheck **Send backups every week**
6. After going through the setup process, I ran into a "Refused to connect" page, so I re-entered the URL from step 4. From there, I was able to sign-in to the admin portal:
	- Username: `admin`
	- Password: `<the one entered in the setup process throughout step 5>`
	
	![](/screenshots/8.png)
### Integrating AD with Sophos
- First, creating a new organizational unit (OU) and domain user for Sophos integration:

	![](/screenshots/9.png)

	- By default, this new user can read AD group membership of itself and other domain users:

		![](/screenshots/10.png)

1. On the Sophos Firewall admin portal, navigate to **Authentication > Servers** and **Add** a server with the following settings:

	![](/screenshots/11.png)

	- For personal purposes, I unchecked **Validate server certificate**
	- To describe the search query written here, the query searches the domain itself for all groups and OUs in it
2. Click **Test connection** and make sure connection is successful before clicking **Save**
3. Next to the AD server, click the import icon
4. Under **Select AD groups to import**, I selected the three OUs I created from doing the Help Desk course (Engineering, Management, IT)
5. Final selections:

	![](/screenshots/12.png)

6. Setting primary authentication method to Active Directory:

	![](/screenshots/13.png)

7. To add a profile for an AD user into Sophos, I needed to use their credentials on the user portal via `https://<Firewall LAN IP address>:4443`:

	![](/screenshots/14.png)
	![](/screenshots/15.png)

	Back on the admin portal, I finally see the AD user pop up:
	![](/screenshots/16.png)

8. Repeat step 7 for additional AD users to be added
### Creating block rule
1. Create the following custom category:

	![](/screenshots/17.png)

2. Add a web policy with the following settings:

	![](/screenshots/18.png)
	![](/screenshots/19.png)

	- NOTE: Quota can't be zero; must be a multiple of 10
3. A popup should appear asking to create a firewall rule for the policy. Do so, and add the following firewall rule:

	![](/screenshots/20.png)
	![](/screenshots/23.png)
	![](/screenshots/24.png)
	![](/screenshots/21.png)
	![](/screenshots/22.png)

	- Action must be `Accept` in order for the other options past **Add exclusion** to appear
	- Leave **Destination networks & Services** as `Any`; the web policy will take care of the specific sites to be blocked
	- Rule position must be at `Top`; it should be the one to process first rather than the other rules
	- For this test, I'll block distracting sites for only the Engineering and IT OUs to demonstrate the granularity of this method
#### Testing rule
- When accessing YouTube as someone in the Management OU, they can do so just fine as expected:

	![](/screenshots/25.png)

- But when accessing YouTube as someone in the Engineering OU, they can still access it just fine as well, despite not being allowed to do so
- A couple of troubleshooting steps I tried:
	- Policy tester: Policies worked fine
 	- Checking the user portal for the one shown in the previous screenshot: I noticed the bandwidth stats was 0 bytes across the board, meaning the problem is that the firewall isn't doing any networking/routing for the domain machines
##### FIX:
- In the DC, I modified the settings for the Ethernet adapter, changing the gateway & DNS server to point to the firewall's IP address:

	![](/screenshots/26.png)
	![](/screenshots/27.png)
	![](/screenshots/28.png)
	![](/screenshots/29.png)

- In Windows 11, I also changed its Ethernet adapter settings, changing the gateway to the firewall's IP address, but keeping the DC as the primary DNS server
	- A test that I tried: In the DC, I signed into the portal with an account that can access YouTube, and in Windows 11 with an account who's restricted from accessing it. After accessing YouTube in the DC, I tried the same in Windows 11, and actually got blocked as I was expecting
	- With this setup though, Windows 11 can only access the internet if both it and the DC are signed into the firewall's captive portal (on `https://<Firewall LAN IP address>:8090`)
		- As a result, I modified the Windows 11 adapter settings again, this time making the firewall the primary DNS server with the DC acting as the secondary DNS server so that Windows 11 doesn't have to depend on the DC being logged in for internet access. However, this resulted in the wallpaper group policy object (GPO) for the Engineering OU not being able to load, so I swapped the DNS servers around to get everything fixed:

			![](/screenshots/30.png)

	- Now, when attempting to browse the internet in both machines, a browser window automatically popped up for each, telling the user to sign in to the captive portal to access the internet. Accessing YouTube while logged in as someone in the Management OU allows me, while accessing as someone in the Engineering OU blocks me:

		![](/screenshots/31.png)
		![](/screenshots/32.png)

		- I also tried the same DC to Windows 11 internet access test from earlier, and I still get blocked, as expected
		- Additionally, I tried switching accounts on the captive portal with the YouTube session still up, and the web policy changes between accounts were able to take effect as well
- Despite the fixes however, no custom block page is being shown in the previous screenshot
#### Getting the custom block page up
1. Enable the page in the admin portal under **Web > User notifications > Message for block action > Use custom block message**
2. Even after enabling it, the "Your connection isn't private" screen kept coming up when accessing the blocked site. Based on the error code, I needed to push the appropriate certificate to the machine's list of trusted certificates. I was able to find the certificate I'm looking for under **Web > General settings > HTTPS decryption and scanning** in the admin portal:

	![](/screenshots/33.png)

3. With the certificate downloaded onto the DC, open up Server Manager, navigate to **Tools > Group Policy Management**, and create a new GPO on the domain that will house the certificate:

	![](/screenshots/34.png)

4. I followed the Microsoft guide on distributing certificates via GPOs from step 3 onwards to successfully import the certificate into the GPO
	- Since my certificate came in a `.pem` file, I changed the file type in the select file screen to `All files` so I can select it
5. After running `gpupdate /force` on all machines in the domain, I tested things on Windows 11. The page finally shows:

	![](/screenshots/35.png)

While I'm at it, I also took care of the "Your connection isn't private" screen for the Sophos web portals by following the Sophos documentation on removing untrusted certificate errors, albeit with a few modifications:
- Under **Subject Alternative Names**, I entered in the LAN IP address of the firewall under **IP addresses**

	![](/screenshots/36.png)
	![](/screenshots/37.png)

- Under **Admin console and end-user interaction**, I left **Use the IP address of the first internal interface** checked

From there, I was able to use the `tar -xvzf` command in PowerShell to extract the tar.gz zip file in Windows. I then pushed the default certificate throughout the domain via the same GPO used for the blocked sites and ran `gpupdate`. Even with the slight deviations from the instructions, the untrusted certificate error no longer appeared when accessing the Sophos portals, thereby accomplishing my goal for this lab.

## Helpful resources
- [YouTube: VirtualBox installation guide](https://www.youtube.com/watch?v=wQSWsFm1DHc)
- [Sophos techvid: Integrating AD with Sophos](https://techvids.sophos.com/watch/1ixxoYCGZ6dV3qzfHbZ7rN)
- [Sophos Support: Creating the site block rule](https://support.sophos.com/support/s/article/KBA-000004747?language=en_US#Blocking-YouTube-with-a-policy)
- [Microsoft Learn: Certificate distribution across machines in domain via GPOs](https://learn.microsoft.com/en-us/windows-server/identity/ad-fs/deployment/distribute-certificates-to-client-computers-by-using-group-policy)
- [Sophos Docs: Removing untrusted certificate error for Sophos portals](https://docs.sophos.com/nsg/sophos-firewall/22.0/Help/en-us/webhelp/onlinehelp/AdministratorHelp/Certificates/HowToArticles/CertificatesRemoveUntrustedCertificateError/index.html#edit-the-admin-and-user-settings)

# Using Active Directory (AD) and Sophos Firewall to block distracting sites
## Goal
For this quick lab, my goal is to utilize AD to block access to distracting sites such as YouTube, Reddit, Facebook, etc... across the entire domain.
## Preparations
- The AD lab is based on the one from TCM Academy's free Practical Help Desk course, which I finished beforehand
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
4. Under **Select AD groups to import**, I selected the three OUs created throughout the Help Desk course (Engineering, Management, IT)
5. Final selections:
![6bc4415a391c178d845e236773d8e326.png](:/ec4cc0d910574f0e923284d3a8b5add5)
6. Setting primary authentication method to Active Directory:
![8841272a0ce25a92eb577e9c9cfe2929.png](:/dc4f087312004ad4bbc1db61e5a1dba6)
	* Local authentication: Firewall's admin account to be used on firewall login screen
	* Active Directory authentication: Can use AD credentials on firewall login screen
		* With Active Directory on top, AD credentials are prioritized
7. To add a profile for our AD user into Sophos automatically, use their credentials on the user portal:
![a13807b2cfecd084b3385fab2822cfa3.png](:/b4ae6bb3f1b04dc2a18649af1d5514ce)
![1fa7b51571c974bdf69a1f7854fb6eb9.png](:/53e471f699b94f1e9624370bb7ced37f)
![68e174245dd7b882c53915e7522cd63f.png](:/76173d35547e4028a37cdfe0b1f57f01)
Back on the admin portal, we finally see Chihaya pop up:
![95aa45253383f2c8c21939d0bc52aed3.png](:/9160e87732a84f61aa91fdb58bd3c805)
8. Repeat step 7 and add the other AD users (NOTE: Not efficient in an enterprise setting; consider using mechanisms to automate the process)
### Creating Block Rules
1. Create the following custom category:
![14563d599372d9626efa90c9862fc696.png](:/eaa8e36ba9654414a8a127028a0a25c3)
2. Construct the web policy with the following settings:
![397da219f94f540cd9964f64762b707a.png](:/8382685f3303422ca4d719755f79bf20)
![f00cdc66c02a60f5072b6d4950ad1fe6.png](:/60ca0133149a4691b994e1381070c5c7)
	* NOTE: Quota can't be zero; must be a multiple of 10
3. A popup should appear asking you to create a firewall rule for the policy. Do so, and add the following firewall rule:
![a4ccf205bd28f18a1e118f20524fec26.png](:/75d2c40176cc4b9c872e8bb9aa969161)
![b4f1da3ac257e817b47fcd286b7f79ad.png](:/e813b1d5a9404916977d4ba46f9a0794)
![8193ab71e869fbe088a8691c9b7f78e7.png](:/bf034d5a4ea84909bc6232d593ffa24d)
![7a88ff5a385152770c48dc9b757a7321.png](:/b59f2e838c234e038322928c85b06db3)
![792d66a698d87ddedf6bfcfe8f473312.png](:/167fef64b4da4839bb5a5dcb793dec1c)
	* Action must be `Accept` for the other options past Add exclusion
	* Leave Destination networks & Services as `Any`; our web policy will take care of the specific sites to target
	* Rule position should be at `Top`; want it to be the one to proc rather than other ones
	* Will block social media for only the Engineering and IT OUs for our test
	* Want Use web authentication for unknown users checked so that we can use a custom blocking page
#### Testing rules
* Chihaya can still access YouTube (she's in Management group, which isn't part of the block):
![29bd75d74042bd2a44de68fb433caa1f.png](:/d66e13e482f84d089f1852b2f09db15f)
* But Miki can still access YouTube
	* After some playing around, I found out about policy tester. Policies worked fine, meaning the problem lies within the Firewall not doing the networking (checked out bandwidth stats for users after playing around with them for a bit - 0 B came out from all)
* FIX:
	* In the DC, I modified the settings for the Ethernet adapter, changing the gateway & DNS server to point to the Firewall's IP address:
	  ![2b4b15817d22f3bb7713d2cb8e8c7e9d.png](:/62580edcf74f4e2493bdbec1841cfe88)
	  ![666b152f1ccedf6054fd1b79c37475eb.png](:/059d5c2557234577ba5297215c0c0a80)
	  ![f6ff0fa078d7484de86876f1209389c7.png](:/bcbb5de743324ce796e6d28690906587)
	  ![c9879bdb75aef7e60a08b041607badde.png](:/881f45932bff4cc2afd6594a1fb3529b)
	* In the Windows 11 domain computer, also change the Ethernet adapter settings, changing the gateway to the Firewall's IP address
		* In the DC, I signed in the portal with an account that can access YouTube, while in Windows 11 I signed in with an account who's restricted. After accessing YouTube in the DC, I tried the same in Windows 11, and actually still got restricted
			* Theoretically speaking, it works this way since the Windows 11 DNS server still points to the DC's IP address, who's DNS server in turn points to the Firewall, meaning the Windows 11 VM queries the DC, which then queries the Firewall, and the wall performs the policy checks as established
			* As a result, I modified the Windows 11 adapter settings one final time, this time making the Firewall the primary DNS server (in order to establish individual internet access), with the DC acting as the secondary DNS server
				* This setup resulted in the previous wallpaper policy for Engineering OU failing to load, so I swapped the DNS servers around in the Windows 11 adapter settings to get it up:
				  ![1b15469ffc6d6cc548edbdf5752ae9ae.png](:/0365f8efe43543e89d2666a38e3ff560)
	* Afterwards, logging into both machines/attempting to browse the internet should automatically pop up a browser window for each, telling the user to sign in via the Firewall's captive portal (port 8090) to access the internet. With these fixes, each machine on the domain can access the internet after logging into the captive portal, all while the domain users can still get GPOs from the DC
	* Now, accessing YouTube while logged in as Haruka lets me, while accessing as Yayoi blocks me:
 ![f9e198e20a3e32feb66585bbcb2eff06.png](:/d83db5b97c3347d1b87e36a240e0263b)
![b6b93ad44140e3bebab246cc191cc93d.png](:/8fe8661122e6498fa8764fb0a39ecdce)
		* Additionally, after signing into an account with access to YouTube on the DC, I proceed to sign in with an account restricted from accessing YouTube on the Windows 11 machine. Then, when trying to access YouTube there, I still get blocked
		* Changing accounts also echoes the changes even when you don't close the window, though depending on the firewall specs, it'll take some time
#### However, I'd like to put in one final touch - a custom block page to show to restricted users
1. According to various sources I found on the internet about Sophos custom block pages, you enable them in the admin portal under Web > User notifications > Message for block action > Use custom block message
2. Even after enabling it, the "Your connection isn't private" screen kept coming up when accessing the blocked site. According to the error code (same as before) and the sources, the solution to stop this page from showing is to push the appropriate certificate to the machine, located under Web > General settings > HTTPS decryption and scanning:
![967532754cc37255cffab06de0f2c319.png](:/9d96953f6a254b11bb9d295a8f15b878)
3. With the certificate downloaded onto the DC, open up Server Manager to Group Policy Management and create a new GPO on the domain that will house the certificate:
![e9430754d183f58a4bf4774d22e7a222.png](:/a78ae0ae93d041caa1aad3dd86924841)
4. I followed \<this guide\> word-by-word after step 2 to successfully import the certificate file into the GPO (NOTE: Since my certificate file came in a `.pem`, I change the file type to all files in the file selection screen to select it)
5. After running `gpupdate /force` on all machines in the domain, I tested things on Windows 11 VM. The page finally shows, completing my task:
![2af11da6e80219b7bb901828e578fa13.png](:/24530611e71f40569023abd81902988a)
* And for convenience, I also decided to take care of the other "Your connection isn't private" screen that comes when trying to browse unauthenticating by following \<this guide\>, but with a few slight modifications:
	* Under Subject Alternative Names, I entered in the IP address of the firewall under IP addresses
	  ![2ee394f96f251e79bd210d4e22f72914.png](:/1597ba93eba14ed9989774e9f0d1932d)
	  ![8ec4d6eaa04d76b689395666655d7041.png](:/e3766beaccb64641b13dddae6f38ea64)
	* Under Admin console and end-user interaction, I left Use the IP address of the first internal interface checked
	* I was able to use the `tar -xvzf` command in PowerShell to unzip the tar.gz file in Windows
	* I then pushed the certificate via the same GPO used for the blocked sites and ran the `gpupdate` command on all machines in the domain
	* Even with the changes, the untrusted certificate error no longer appears when accessing the Sophos portals, thereby completing my job here for good

## Helpful resources
- https://www.youtube.com/watch?v=wQSWsFm1DHc (VirtualBox installation guide)
- https://techvids.sophos.com/watch/1ixxoYCGZ6dV3qzfHbZ7rN (Integrating AD with Sophos)
- https://www.youtube.com/watch?v=xIHNoeOEPbM (for telling me exactly where I needed to go to use the AD credentials to add AD user to firewall)
- https://support.sophos.com/support/s/article/KBA-000004747?language=en_US#Blocking-YouTube-with-a-policy (creating the site block rule)
- https://community.sophos.com/sophos-xg-firewall/f/discussions/148136/sophos-firewall-blocking-page/549319 & https://community.sophos.com/sophos-xg-firewall/f/discussions/132083/nu-custom-message-for-malware-blocked-pages (Enabling custom block pages & removing "Your connection isn't private" screen)
- https://learn.microsoft.com/en-us/windows-server/identity/ad-fs/deployment/distribute-certificates-to-client-computers-by-using-group-policy (certificate distribution across machines in domain via GPOs)
- https://docs.sophos.com/nsg/sophos-firewall/22.0/Help/en-us/webhelp/onlinehelp/AdministratorHelp/Certificates/HowToArticles/CertificatesRemoveUntrustedCertificateError/index.html#edit-the-admin-and-user-settings (removing untrusted certificate error for Sophos portals)

# Using Active Directory (AD) and Sophos Firewall to block distracting sites
## Goal
For this quick lab, my goal is to utilize AD to block access to distracting sites such as YouTube, Reddit, Facebook, etc... across the entire domain.
## Preparations
* The AD lab is based on the one from TCM Academy's free Practical Help Desk course, which I finished beforehand
	* Domain controller (DC) and Windows 11 virtual machine (VM) are in the same domain
	* Verified that the Windows 11 VM doesn't have an alternate DNS server set
	* Ensured that the DC and VM only have one adapter connected to the internet (NAT network adapter)
* Did lots of research on ways I could achieve this goal. These are the methods I tried:
## First method: Modifying AD DNS
1. In domain controller, navigate to Tools > DNS:
![c59779b28e7d8015e0ae1f0073771982.png](:/f963b33f05174b75ae91734b8bee2cce)
2. Right-click on Forward Lookup Zones, then select New Zone...
3. Click Next up to "Zone Name". Here, enter the domain to be blocked (w/o `www.`):
![87fb28e8536286a3eeb759ae25056f37.png](:/1753f70aa5ed4566ac8ed1f88be9afb9)
4. Under "Dynamic Update", select Do not allow dynamic updates, then proceed with creating the zone
5. Double-click on the newly created zone, then right-click in the white area and select New Host (A or AAAA)...
6. Under IP address, enter `0.0.0.0` (all IPs) or `127.0.0.1` (localhost address), check Create associated pointer (PTR) record, then click Add Host:
![97c57bbbb9f3ddb58fc2f2efeff59699.png](:/bc117d9d1f7d4fb88d894630f4a9571e)
7. Click Done, and now you should see the host:
![2f1a2b910be2a1c0d8559ebd36a55455.png](:/2c3278d2855b4526a83705c9e91cf75a)
8. Repeat steps 2-7 for as many domains you want to block
9. Open an admin command prompt and run `gpupdate /force` across all machines on the domain to apply the policy changes, as well as `ipconfig /flushdns` to reset previously stored DNS caches
	* Additionally, clear browser caches and/or close then reopen browser
10. Verify you can't access the blocked domains:
![1ad06a0bb0c3d4f08c3dadea95b78911.png](:/57ebdfb5ccbe40488dafa46d30a63401)
![4ff2fdbd8984412aa7571e79d3010226.png](:/74dbb4a3f85c496faf4dd5de87470224)
## However... Instead of this, I'd like a blocked page to show up indicating the site in question is blocked in the domain
* Will need to use some third-party web filtering software to pull this off (Sophos Firewall, Admin by Request). However, this necessitates (or at the very least, strongly recommended), a dedicated VM to install the software in, then have it connect to the domain controller
	* I will be using Sophos Firewall
		* Helpful resources:
			* https://www.youtube.com/watch?v=B-ZFj_Bn_-I (where to find installers for VirtualBox + installation guide)
			* https://www.youtube.com/watch?v=wQSWsFm1DHc (VirtualBox installation guide)
			* https://www.youtube.com/watch?v=xIHNoeOEPbM (for telling me exactly where I needed to go to use the AD credentials to add AD user to firewall)
			* https://support.sophos.com/support/s/article/KBA-000004747?language=en_US#Blocking-YouTube-with-a-policy (creating the site block rule)
			* https://community.sophos.com/sophos-xg-firewall/f/discussions/148136/sophos-firewall-blocking-page/549319 & https://community.sophos.com/sophos-xg-firewall/f/discussions/132083/nu-custom-message-for-malware-blocked-pages (Enabling custom block pages & removing "Your connection isn't private" screen)
			* https://learn.microsoft.com/en-us/windows-server/identity/ad-fs/deployment/distribute-certificates-to-client-computers-by-using-group-policy (certificate distribution across machines in domain via GPOs)
			* https://docs.sophos.com/nsg/sophos-firewall/22.0/Help/en-us/webhelp/onlinehelp/AdministratorHelp/Certificates/HowToArticles/CertificatesRemoveUntrustedCertificateError/index.html#edit-the-admin-and-user-settings (removing untrusted certificate error for Sophos portals)
### Setting up Sophos
1. Following video 1 to grab the installer, I then follow video 2 to set it up in VirtualBox
2. Select `Network configuration menu`, then choose `Yes` to `Set IPv4 Address`. For PortA (LAN network/AD network), change the address to be within the AD network's subnet mask:
![9c06d5fb3ced09882182bd21e19d5f91.png](:/13ac6002e0234bd6a1eaea6bf5270b1f)
3. WAN should be different from above. If so, leave as is and navigate to a VM situated in the same AD network and enter the Sophos Firewall's new IP address into a browser:
![1fc039e2603fa1ee64e416cd722281a0.png](:/7773bc0de7954b8f87ac6566c9d142f3)
![d15be765a6139c7019e03bd27a5a6a26.png](:/a348f532e3584622b8f12d7e39724137)
4. Go through the setup process:
	* Password: WorDPASS1!
	* Leave Install latest firmware checked
	* Master key: MASTERpassword#1
	* For serial number, register for Sophos Home Firewall to get one. Then, enter it under Serial Number
	* Make sure to de-check `Enable DHCP`, and choose `This firewall` as the gateway
	* As I'm running this firewall for the sole purpose of blocking distracting sites for productivity, I'll leave everything under the `Network protection` section unchecked
	* Under `Notifications and Backups`, feel free to enter your actual email or dummy email(s). Uncheck `Send backups every week`
5. After setup, I ran into a `Refused to connect` page, so I re-entered the exact URL from step 3. Then, I was able to sign-in:
	* Username: admin
	* Password: WorDPASS1!
![60d303c4069eb975ee42a3aab7a1aa54.png](:/434c2083aa3545c4adc265d43e77e592)
### Integrating AD with Sophos
* Help: https://techvids.sophos.com/watch/1ixxoYCGZ6dV3qzfHbZ7rN
* First, creating a new OU and User for Sophos integration:
![f3777144d67b3a3931bd8add6fae5b45.png](:/46004c6932ab4f9080c609441fc82b32)
	* Password: WorDPASS!!!
	* By default, this new user can read AD group membership of itself + other domain users:
	  ![d73fa6c4e1cefa39179dbe7f22457c34.png](:/f37448d6f20a42d5855117cb59a837ed)
1. On the Sophos Firewall GUI, navigate to Authentication > Servers and input the following settings:
![0be6d40978d31d9f3d75499a51aaf8fc.png](:/eb2df29955c241e6828514ec0027dc96)
	* For our personal purposes, uncheck Validate server certificate (in an enterprise setting though, leave it checked)
	* Search query: Searches the domain itself for all groups and OUs in it
2. Click Test connection and make sure connection is successful before clicking Save
3. Next to the Active Directory server, click the Import icon
4. Under Select AD groups to import, select the three OUs created throughout the lab (Engineering, Management, IT)
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

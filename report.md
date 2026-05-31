# LazySysAdmin CTF walkthrough

This small project aims to breach the vulnerable VM **[LazySysAdmin](https://www.vulnhub.com/entry/lazysysadmin-1,205/ "Download the VM")**, a Boot-to-Root challenge with full writeup available on [VulnHub](https://www.hackingarticles.in/hack-lazysysadmin-vm-ctf-challenge/ "See the original writeup"), to show how permissive configurations and bad management/operative administrative habits can lead to a deep compromise in just few minutes.  
> *Note*: at the beginning the project started following the writeup mentioned but subsequently evolved into a different approach. To see the original idea follow the link above.


## Threat model & environment

> ⚠️**Threat model**: attacker is able to communicate with the vulnerable service (eg: service has public IP & vulnerable ports open)

*Kali Linux* (attacker) vs *LazySysAdmin* (victim) within the same Host-Only VirtualBox isolated network to secure the test environment.   


## Idea of attack structure

The *LazySysAdmin* machine will be constantly waiting for interactive logon while attacker is operating beneath.

![LazySysAdmin machine during the whole attack](images/LazySysAdmin.png)
1. **Reconnaissance**: `nmap` for port scanning and service identification
2. **Discovery**: `smbclient` for exploring victim's SMB shared objects and analyzing configuration files
3. **Initial Access & Exploitation**:
    * **Valid Accounts**: abuse of admin's valid credentials trough remote SSH access
    * **Exploit Public-Facing Application**: abuse of admin DB's valid credentials to obtain a Reverse Shell trough WordPress' admin panel
4. **Execution**: RCE via Reverse Shell using Python `pty` module
5. **Privilege Escalation** & CTF: abusing permissive configuration in `/etc/sudoers` (`ALL:ALL ALL`) to obtain interactive shell as *root* & completing the writeup "capturing the flag" inside *root*'s directory   


### 1. Reconnaissance

It is implicit - by threat model - that the attacker already knows the public IP of target service (in this case due to VirtualBox configuration).  
I started the attack doing a scan of the open services of the target machine using `nmap`:

![Result of nmap on Kali](images/nmap_scan.png)  
Among all services there is Samba on port 139, so with the command:  
```bash
smbclient -L <serverIP>
```
 it is able to list all shared directories of the target host.


### 2. Discovery <a id="ch2"></a>

A secure Samba configuration does not allow a guest user to access the shared objects inside the server but with this configuration instead it does. After connecting to the *share$* folder with:  
```bash
smbclient '//<serverIP>/share$'
```
I was indeed able to access them with privileges of downloading files and traversing directories, so after searching for a while I noticed the files `deets.txt` and `wordpress/wp-config.php` in which, respectively, are contained:  
* password of host-admin account Togie (creator of the CTF) in cleartext: *12345*
* username and password of the service-admin control panel in WordPress: *Admin, TogieMYSQL12345^^*

> For the following sections (3,4,5) I allowed myself for a partial detour w.r.t. the original writeup because I noticed a simpler way of continuing the attack (follow the route "a" for SSH or "b" via Wordpress).


### 3a. Initial Access: Valid Accounts

As we previously saw from the screeshot there is also an open SSH port that I tried to use with a bunch of simple usernames combined with the password of the host-admin account, having success with the one in the following screenshot.

![Found SSH connection](images/ssh_valid_accounts.png)  
Now I'm currently logged in as *togie*. In section ([5](#ch5)) we'll see how I used this connection.


### 3b. Exploitation: Exploit Public-Facing Application

An alternative, longer and graphical way of reaching the same position starts from surfing the address `<serverIP>/wordpress/wp-admin/index.php` on the browser. This is the admin login page, in which I inserted the credentials of the admin DB found inside the file at ([2](#ch2)) and successfully entered as WordPress admin.  
With these capabilities I altered one of the *php* pages of the active template (I've chosen `404.php`) to inject a ready-to-use Reverse Shell, foundable at `/usr/share/webshells/php/php-reverse-shell.php` inside Kali and omitted for brevity and slightly modified to connect to the Kali machine (IP & port).  
I set up a listener on Kali using `netcat`:  
```bash
nc -lvnp 4444
```
and triggered the Reverse Shell visiting the *php* page just modified via browser, in my case is `<serverIP>/wordpress/wp-content/themes/twentyfifteen/404.php`


### 4. Execution (only for (b) part)

So far the Reverse Shell allowed me to obtain a Shell inside the server but it is not interactive, the account which I'm interacting with is the usual on which the Apache server runs (*www-data*). I, thus, verified that Python is installed and executed some Python commands to use the module `pty` to execute an interactive Shell:
```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
```
This Shell allowed me to impersonate *togie* using:
```bash
su togie
```


### 5. Privilege Escalation & CTF <a id="ch5"></a>

> From now on the attack follows almost the same path.  

I executed the following command to know which commands *togie* is able to execute:
```bash
sudo -l
```
The output is pretty self-explanatory: `(ALL:ALL) ALL`, meaning that this account is able to execute any command on the host.  
Executing `sudo su` I was able to become *root* and complete the challenge capturing the flag:
* via SSH
![Capturing the flag via SSH](images/ssh_CTF.png)
* using the Reverse Shell
![Capturing the flag using the Reverse Shell](images/reverse_shell_CTF.png)


## Attack observations

This attack demonstrates the effectiveness of attacks based on misconfigurations. It was not necessary to exploit any vulnerability; the whole compromise was carried out abusing overprivileges and bad practices only.  
It is indeed interesting that the SSH version installed on the server is actually vulnerabile to **CVE-2018-15473** for user enumeration. In this scenario though, the vulnerability was not exploited.


## Possible defense

The attack leverages only onto human errors, mainly violations to the *Principle of Least Privilege*.  
To secure the server it is needed to:
1. **Samba configuration change**: disable anonymous `guest` access to shared directories
2. **Password policy hardening**: use for each IT entity a different, "complex" and "reasonably long" password (eg: not the same between service and workstation)
3. **Privilege restriction**: restrict privileges inside `/etc/sudoers` in a way that normal user cannot execute any command as root without admin password

## LLM usage

I discovered the (b) attack path asking [Gemini](https://www.google.com/aclk?sa=L&ai=DChsSEwiq6O7WseOUAxVOroMHHaABLvcYACICCAEQABoCZWY&ae=2&co=1&ase=2&gclid=CjwKCAjwuO_QBhAWEiwAIkVhU1NXR5CK1vHMXhIxlYq06KJ3lVDYHgkUqjDc0mCSOIT5HQj7IYmyThoCOZgQAvD_BwE&cce=2&category=acrcp_v1_71&sig=AOD64_2SCp3Dgcu6Ck2tQGAzg5TCuYk-Qg&q&nis=4&adurl&ved=2ahUKEwi75efWseOUAxXogf0HHU19G-AQ0Qx6BAgNEAE "Have a look at the LLM") to better explain the exploit steps of the writeup and I choose to follow it, and so diverge from the writeup's path, because easier to comprehend.
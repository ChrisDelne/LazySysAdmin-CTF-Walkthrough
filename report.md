# LazySysAdmin CTF walktrough

This small project aims to breach the vulnerable VM **[LazySysAdmin](https://www.vulnhub.com/entry/lazysysadmin-1,205/ "Download the VM")**, a Boot-to-Root challenge with full writeup available on [VulnHub](https://www.hackingarticles.in/hack-lazysysadmin-vm-ctf-challenge/ "See the original writeup"), to show how permissive configurations and bad management/operative administrative habits can lead to a deep compromise in just few minutes.  
> *Note*: at the beginning the project started following the writeup mentioned but subsequently evolved into a different approach. To see the original idea follow the link above.


## Threat model & environment

> ⚠️**Threat model**: attacker is able to communicate with the vulnerable service (eg: service has public IP & vulnerable ports open)

*Kali Linux* (attacker) vs *LazySysAdmin* (victim) within the same Host-Only VirtualBox isolated network to secure the test environment.   


## Attack structure

The *LazySysAdmin* machine will be constantly waiting for interactive logon while attacker is operating beneath.
1. **Reconnaissance**: `nmap` for port scanning and service identification
2. **Discovery**: `smbclient` for exploring victim's SMB shared objects and analyzing configuration files
3. **Initial Access & Exploitation**:
    * **Valid Accounts**: abuse of admin's valid credentials trough remote SSH access
    * **Exploit Public-Facing Application**: abuse of admin DB's valid credentials to obtain RCE trough WordPress' admin panel
4. **Execution**: abitrary code execution (Reverse Shell) using Python (`pty` module)
5. **Privilege Escalation**: abusing permissive configuration in `/etc/sudoers` (`ALL:ALL ALL`) to obtain interactive shell as `root`
6. CTF: completing the writeup "capturing the flag" inside `root`'s directory   


### 1. Reconnaissance

It is implicit - by threat model - that the attacker already knows the public IP of target service (in this case due to VirtualBox configuration).  
I started the attack doing a scan of the open services of the target machine using `nmap`:
![Result of nmap on Kali](images/nmap_scan.png)  
Among all services there is Samba on port 139.


### 2. Discovery

A secure Samba configuration does not allow a guest user to access the shared objects inside the server. With this configuration instead it does. Using indeed `smbclient` I was able to access them with privileges of downloading files and traversing directories, so after searching for a while I noticed the files `deets.txt` and `wordpress/wp-config.php` in which, respectively, are contained:  
* password of admin account Togie (creator of the CTF) in cleartext: *12345*
* username and password of the admin control panel in WordPress: *Admin, TogieMYSQL12345^^*

> For the following sections (3) I allowed myself for a partial detour w.r.t. the original writeup because I noticed a simpler way of continuing


### 3a. Initial Access: Valid Accounts

<!-- Continuare da qui -->


## Attack observations

<!-- Da tradurre -->
Questo laboratorio dimostra l'efficacia degli attacchi basati su configurazioni errate. Non è stato necessario sfruttare alcuna CVE; l'intera compromissione è avvenuta abusando di permessi eccessivi e cattive pratiche.  
Notiamo che il servizio SSH è teoricamente vulnerabile alla CVE-2018-15473 per l'enumerazione degli utenti. Tuttavia, nel nostro scenario questa vulnerabilità software passa in secondo piano, poiché la pessima configurazione del servizio Samba ci ha già permesso di ottenere direttamente i nomi utente e le password in chiaro, rendendo superfluo l'utilizzo di exploit software.


## Possible defense

The attack leverages only onto human errors, mainly violations to the *Principle of Least Privilege*.  
To secure the server it is needed to:
1. **Samba configuration change**: disable anonymous `guest` access to shared directories
2. **Password policy hardening**: use for each IT entity a different, "complex" and "reasonably long" password (eg: not the same between service and workstation)
3. **Privilege restriction**: restrict privileges inside `/etc/sudoers` in a way that normal user cannot execute any command as root without admin password
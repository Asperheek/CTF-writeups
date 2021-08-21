# Crocc Crew Writeup
This is a writeup for the Crocc Crew room on TryHackMe. The difficulty of this room was "insane" which is what peaked my interest.
You can find the room at: https://tryhackme.com/room/crocccrew 

## Enumeration
### Nmap Scan
![image](https://user-images.githubusercontent.com/25471487/130314880-650a5cc4-84b3-4d47-8d58-b246045db71b.png)


### Port 80
![image](https://user-images.githubusercontent.com/25471487/130314898-aa85da55-0ce9-4420-a548-594671a5f931.png)


### /robots.txt
![image](https://user-images.githubusercontent.com/25471487/130314938-a1832ccd-1ef7-4292-b998-82b7b4597d26.png)


### /backdoor.php
![image](https://user-images.githubusercontent.com/25471487/130315362-2dd3d476-40de-47d8-a55d-2e98d4b1a1f9.png)
</br>
A php webshell - seems all commands are blocked!</br>

### /db-config.bak
![image](https://user-images.githubusercontent.com/25471487/130314946-9932dd44-4aa8-45e7-bf81-06fb39abab07.png)
</br>
Found credentials, but could this be a rabbit hole? From the Nmap scan we found out there is an AD LDAP server running. Let's investigate! </br>

### Port 445
#### RPC client
Authenticated 'Userless' SMB Session with rpcclient.</br> 
After spending a lot of time on rabbit holes, I stumbled across the service which is commonly ignored in the enumeration phase.
You can find the cheatsheet for rpcclient commands online.
Most of the rpcclient commands were denied and found out that only one worked!
![image](https://user-images.githubusercontent.com/25471487/130315436-e11bc980-ac5d-4168-8951-2f0f4ef04d7d.png)

The command 'enumprivs' revealed the privileges of the current user on the machine.
We can see that the "SeEnableDelegationPrivilege" which governs whether a user account can enable user accounts to be trusted for delegation. 
This can factor into constrained delegation.
You can read more about constrained/unconstrained delegation at: https://docs.microsoft.com/en-us/windows-server/security/kerberos/kerberos-constrained-delegation-overview


### Port 3389 [RDP]
Userless RDP session.</br>
![image](https://user-images.githubusercontent.com/25471487/130316187-182c5eec-3bef-41f9-98cf-31903eb2a493.png)

Found credentials for the guest user 'Visitor'.</br>
![image](https://user-images.githubusercontent.com/25471487/130316254-dac00b12-6e4d-414a-a545-5d299b87d1b2.png)

Trying to login as the guest user! </br>
![image](https://user-images.githubusercontent.com/25471487/130316374-8de0c61d-4f0f-4c4e-a80c-6e4fbfb93aa2.png)

Seems we cannot login remotely.</br>
![image](https://user-images.githubusercontent.com/25471487/130316392-8e6b0fc9-13bb-49a4-a04c-efbc58d12f04.png)


Using crackmapexec to verify the password for SMB.</br>
![image](https://user-images.githubusercontent.com/25471487/130316886-072f8de0-e00c-4c45-9ede-417e19807d1d.png)



Using Visitor's credentials to list SMB shares.</br>
![image](https://user-images.githubusercontent.com/25471487/130316501-b998e4e7-0f8b-4bf8-82b9-10348eba1afe.png)

Listing the contents of the 'Home' share.
![image](https://user-images.githubusercontent.com/25471487/130316512-47469511-1874-47f8-aaf2-633ab6e07870.png)
</br>
Found 'user.txt'!</br>
![image](https://user-images.githubusercontent.com/25471487/130316862-f037f8ab-22eb-484d-886d-bbed3f988206.png)



Listing other shares.</br>
![image](https://user-images.githubusercontent.com/25471487/130316584-4f2c36d3-25ba-41cf-a218-9d35af3e03d0.png)


Since, now we know this is an LDAP server. Let's run ldapsearch and find the namingcontexts and more information about the domain.</br>
![image](https://user-images.githubusercontent.com/25471487/130316605-85cc0cc9-3648-4b80-a12b-d7a14d918fe7.png)
![image](https://user-images.githubusercontent.com/25471487/130316617-a14f2d8d-11ec-4408-b3a1-e348e38e08c6.png)
Running authenticated ldapsearch and saving the result into a text file.</br>
![image](https://user-images.githubusercontent.com/25471487/130316623-4e51b45a-004d-47ca-b2ff-2359ef0cda9c.png)


Let's now look at the result of the ldapsearch.</br>
![image](https://user-images.githubusercontent.com/25471487/130316724-5e54b288-ac69-4370-96b1-fe1210e74bc6.png)
Found the user that has been planted by the admins!</br>

Upon further inspection, found out the account which has the flag for constrained delegation set.
![image](https://user-images.githubusercontent.com/25471487/130317047-65f0c857-dcdd-4885-ab25-d9a7c43b76b3.png)
msDS-AllowedToDelegateTo oakley from password-reset.


Let's also run ldapdomaindump for a more visual domain dump.</br>
![image](https://user-images.githubusercontent.com/25471487/130316935-d3df0510-7495-4a42-a169-f50ae31db099.png)



Let's look at the result of the ldapdomaindump.</br>
![image](https://user-images.githubusercontent.com/25471487/130316960-c150321d-a37a-4a87-bf27-4bbbf092803f.png)
Seems that the password-reset account has the flag 'TRUSTED_TO_AUTH_FOR_DELEGATION!' set which confirms our contrained delegation theory.</br>

Other user accounts listed. </br>
![image](https://user-images.githubusercontent.com/25471487/130317023-669bb685-e0b6-4856-9a9d-e23d4389cbd8.png)



































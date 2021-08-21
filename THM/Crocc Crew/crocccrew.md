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
A php webshell - seems that we cannot run any commands!</br>

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
We can see that the "SeEnableDelegationPrivilege" is listed along with "SeDelegateSessionUserImpersonatePrivilege"."SeEnableDelegationPrivilege" governs whether a user account can enable user accounts to be trusted for delegation. 
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
</br>
Running authenticated ldapsearch and saving the result into a text file.</br>
![image](https://user-images.githubusercontent.com/25471487/130316623-4e51b45a-004d-47ca-b2ff-2359ef0cda9c.png)


Let's now look at the result of the ldapsearch.</br>
![image](https://user-images.githubusercontent.com/25471487/130316724-5e54b288-ac69-4370-96b1-fe1210e74bc6.png)
</br>
Found the user that has been planted by the admins!</br>

Upon further inspection, found out the account which has the flag for constrained delegation set.
![image](https://user-images.githubusercontent.com/25471487/130317047-65f0c857-dcdd-4885-ab25-d9a7c43b76b3.png)
</br>
msDS-AllowedToDelegateTo oakley from password-reset. SPN of the account is also listed.</br>


Let's also run ldapdomaindump for a more visual domain dump.</br>
![image](https://user-images.githubusercontent.com/25471487/130316935-d3df0510-7495-4a42-a169-f50ae31db099.png)
</br>


Let's look at the result of the ldapdomaindump.</br>
![image](https://user-images.githubusercontent.com/25471487/130316960-c150321d-a37a-4a87-bf27-4bbbf092803f.png)
Seems that the password-reset account has the flag 'TRUSTED_TO_AUTH_FOR_DELEGATION!' set which confirms our contrained delegation theory.</br>

Other user accounts listed. </br>
![image](https://user-images.githubusercontent.com/25471487/130317023-669bb685-e0b6-4856-9a9d-e23d4389cbd8.png)


Using the impacket script for finding user SPNs and encrypted password.</br>
![image](https://user-images.githubusercontent.com/25471487/130317462-5804244c-1c1b-458e-8863-5c9e9253700b.png)
</br>
Found out that the error above indicates that my SPN script is not compatible with the version of impacket. Upgraded to the latest version of impacket to remove errors and got the encrypted password.</br>
![image](https://user-images.githubusercontent.com/25471487/130317610-479f54c6-23af-4fe8-a380-0e577b52aa12.png)


Used jTR to decrypt the password.</br>
![image](https://user-images.githubusercontent.com/25471487/130317639-c84a36cf-a558-4d1f-a1cc-f701351ede6a.png)


Using the impacket's find delegation to extract more information about the delegation.</br>
![image](https://user-images.githubusercontent.com/25471487/130317667-c9dafbf7-4391-40ab-bb37-f4dec0285850.png)


Using the impacket's getST script to impersonate and get the ticket of the Administrator user.</br>
If the account is configured with constrained delegation (with protocol transition), we can request service tickets for other users, assuming the target SPN is allowed for delegation</br>
![image](https://user-images.githubusercontent.com/25471487/130317704-51653b4c-f69d-406e-9ffc-e8528e87c88d.png)
</br>
The output of this script will be a service ticket for the Administrator user.</br>
Once we have the ccache file, set it to the KRB5CCNAME variable and use it to our advantage.</br>
![image](https://user-images.githubusercontent.com/25471487/130317775-b1ea3877-14c3-4dc4-af16-8582d9ae3b09.png)

Using the secretsdump script from impacket to dump user hashes.</br>
![image](https://user-images.githubusercontent.com/25471487/130317841-5e2b5dc9-5836-40f8-9b55-cc5dec59ad9d.png)
This did not work because we have to set the domain and the IP in out /etc/hosts file.</br>
![image](https://user-images.githubusercontent.com/25471487/130317866-7bb987a3-5661-4b13-96a2-5ca731746205.png)
After we have done that, it should successfully dump user NTLM hashes.</br>
![image](https://user-images.githubusercontent.com/25471487/130317957-de8c6db8-ddbe-419b-bfc4-bc792575f0be.png)


Now, we can use evil-winrm to login as Administrator.</br>
![image](https://user-images.githubusercontent.com/25471487/130317985-50bc0209-5bbd-4d90-b60c-a8fd8090948e.png)

Found the location of rest of the flags.</br>
![image](https://user-images.githubusercontent.com/25471487/130318012-e70d0746-4f76-460a-9dae-eee27b683550.png)
![image](https://user-images.githubusercontent.com/25471487/130318037-1dd7d667-b99d-41ac-94bc-4fae00ee7779.png)

Found the root flag!
![image](https://user-images.githubusercontent.com/25471487/130318049-3d6968dc-d8af-489f-afa9-61724485361a.png)

</br>

## Privilege Escalation
### N/A
























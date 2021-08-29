# Uranium CTF Writeup
This is a writeup for the Uranium CTF room on TryHackMe.
You can find the room at: https://tryhackme.com/room/uranium

## Enumeration
### Nmap Scan
![image](https://user-images.githubusercontent.com/25471487/131245191-6f2f2558-8445-4641-bc40-82b959bcc278.png)

</br>

Twitter account for the employee 'hakanbey'. One of the post says that "Everyone can send me application files (filename: "application") from my mail account. I open and review all applications one by one in the terminal." We can perform a phishing attack and send our payload with the filename "application" to get a reverse shell. The posts also reveal the domain name.
</br>
![image](https://user-images.githubusercontent.com/25471487/131245194-ce5373cd-1a48-453c-87f6-e53a1c174d1c.png)

Let's add the domain to our /etc/hosts file. </br>
![image](https://user-images.githubusercontent.com/25471487/131245342-fe0fc790-bb0a-4667-a875-375626051015.png)


Let's now create our malicious application.</br>
![image](https://user-images.githubusercontent.com/25471487/131245414-21042b8b-654b-4efd-ad9f-08be24b44827.png)

Since the SMTP service is up and running, we can use sendEmail tool to perform our phishing attack.</br>
![image](https://user-images.githubusercontent.com/25471487/131245458-9feaa257-1417-43a8-b278-bc23c3dc9b43.png)

After a few seconds, we get a reverse shell. </br>
![image](https://user-images.githubusercontent.com/25471487/131245467-549554e4-0c92-4080-b76d-d8eab61e5fe7.png)


Let's now make it a stable TTY shell. </br>
![image](https://user-images.githubusercontent.com/25471487/131245494-9fa5f231-f200-460d-91c6-bc8c398e14ee.png)
![image](https://user-images.githubusercontent.com/25471487/131245511-075aa4f8-747c-44a1-8ac9-98ebfd4f8dcb.png)


Found the "user_1.txt". </br>
![image](https://user-images.githubusercontent.com/25471487/131245524-af6e42b1-93b7-4b32-8461-40d63152a2e7.png)
![image](https://user-images.githubusercontent.com/25471487/131245533-bdba67cf-b289-407b-ba06-7c155e8c2533.png)

Found a binary file but it requires a password. </br>
![image](https://user-images.githubusercontent.com/25471487/131245539-d549bd3d-b361-4c2c-930c-65e80dded994.png)


Let's download linpeas.sh on the victim machine.</br>
![image](https://user-images.githubusercontent.com/25471487/131245589-a9636905-44f1-4a5a-ac5a-81bd688b41a4.png)

Found a .pcap file that may contain passwords.</br>
![image](https://user-images.githubusercontent.com/25471487/131245606-9901c8c7-82f4-4286-a87a-51b4e7ccf906.png)


Let's download the .pcap file to our host machine and use wireshark to reveal the password. </br>
![image](https://user-images.githubusercontent.com/25471487/131245630-40907c41-3a81-485a-956e-5fdba85fdf86.png)


Found the application password. </br>
![image](https://user-images.githubusercontent.com/25471487/131245912-bf4180cb-2fa3-4b82-b3d9-09318058d66e.png)



Found the user password. </br>
![image](https://user-images.githubusercontent.com/25471487/131245955-4dcff323-33a9-441f-b3cc-1014da8713c9.png)


Using "sudo -l" to find out what the user can run. </br>
![image](https://user-images.githubusercontent.com/25471487/131245982-0fc21dff-36e8-453c-82bb-92ec114b66ff.png)



### Privilege Escalation

Using what we found we can escalate our priveleges to the user "kral4". </br>
![image](https://user-images.githubusercontent.com/25471487/131246024-20807a03-3677-4fa3-8ac3-b5b5a57085ec.png)


Found user_2.txt. </br>
![image](https://user-images.githubusercontent.com/25471487/131246038-de8b8ff3-ab93-4033-88a6-53caeed4fc7c.png)


Let's look at SUID permissions for a possible privilege escalation vector. </br>
![image](https://user-images.githubusercontent.com/25471487/131246146-fc2bf60a-acde-4b2d-9415-32c87b7ff658.png)


From GTFObins, I found out that with /bin/dd we can read/write files. </br>
![image](https://user-images.githubusercontent.com/25471487/131246167-5b130ad5-6519-4766-a0d5-aaecbab747ef.png)


### Web Flag. </br>
We don't have the right permissions to read the flag. </br> 
![image](https://user-images.githubusercontent.com/25471487/131246046-a6538cbe-4d59-4e56-88c1-44921ed3fea0.png)

Let's use /bin/dd to read the web flag.</br>
![image](https://user-images.githubusercontent.com/25471487/131246212-5ef4c24a-45b0-4521-b43f-2cf289d13b35.png)


Looking at kral4's emails. </br>
![image](https://user-images.githubusercontent.com/25471487/131246082-cfe49a27-d2b1-4a3a-9b2e-fcca137ebaa5.png)

If we edit the index.html file, root user will set the SUID bit for the /bin/nano binary in our home folder. </br>
Let's copy /bin/nano to our home folder and then edit the index.html file. </br>
![image](https://user-images.githubusercontent.com/25471487/131246254-32058a48-083a-4cc0-a255-7e3dc5358ea6.png)
![image](https://user-images.githubusercontent.com/25471487/131246269-8c7f7284-feed-430a-b647-040d33a5eb73.png)
![image](https://user-images.githubusercontent.com/25471487/131246275-42c34374-742f-441e-85f1-abe950cb0af5.png)

As we can clearly see, we got an email after editing the index.html file that "Our index.html file has been attacked again". </br>
Also, the root user has set the SUID bit for the nano binary in our home folder. </br>
![image](https://user-images.githubusercontent.com/25471487/131246316-7a00bbad-6e22-49d5-9ffa-986040da657e.png)



We can now edit the /etc/sudoers file and give all access to user hakanbey, since we have his password.</br>
![image](https://user-images.githubusercontent.com/25471487/131246341-e9385441-9b11-4725-82e9-2292454f3f19.png)
![image](https://user-images.githubusercontent.com/25471487/131246345-47fd149f-6dff-44a1-a18e-a9e3d8005ddd.png)
We could've also edited the /etc/passwd file and added a new user with root privileges. </br>

Now, let's login to the hakanbey account and then gain access to the root user account since we have the privileges to do so.</br>
![image](https://user-images.githubusercontent.com/25471487/131246372-3611fb5f-080c-4235-beef-76db3b2b1fad.png)


/root/root.txt. </br>
![image](https://user-images.githubusercontent.com/25471487/131246389-4f46d91e-3b9a-4fe3-9d5e-c279cc5b6e2f.png)

















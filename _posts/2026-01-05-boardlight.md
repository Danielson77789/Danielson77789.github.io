---
layout: post
title: BroadLight
date: 2026-01-05 21:02 -0700
categories: [Hack The Box, Retired Machine]
tags: [htb]
---
# Enumeration 

## Nmap Scans
### default nmap scan - 
![Default Nmap Scan](assets/markdown_images/BoardLightMedia/Pasted image 20240605004604.png)
### Nmap service Scan - 
![Nmap Service Scan](assets/markdown_images/BoardLightMedia/Pasted image 20240605004941.png)
### Nmap All ports - 
![Nmap All Ports](assets/markdown_images/BoardLightMedia/Pasted image 20240605133745.png)
## Web enumeration 

### Portfolio endpoint found in source code `/portfolio.php`

Unable to access this portfolio endpoint, will begin looking for domain as awell as any possible subdomains to the system.

![Portfolio Endpoint](assets/markdown_images/BoardLightMedia/Pasted image 20240605005841.png)
### ffuf to discover subdomains available 

For this step I ran the ffuf command to discover subdomains of the domain, once the total word count was on a normal response I set the flag `-fw 6423` which filtered out invalid responses. With this we are able to discover the subdomain `crm.board.htb` which reveals a Dolibarr CRM login page.

![FFUF Scan](assets/markdown_images/BoardLightMedia/Pasted image 20240605013144.png)

Browsing to `http://crm.board.htb` I was able to come across the Dolibarr CRM service default login page. From there I was able to attempt to login with `admin:admin` which allowed me to the dashboard of the CRM page. From there I was allowed minimal privilege's into the CRM. 

Create a web page under the websites tab as the admin user form there we can add a new page which will five us the option to name the page and edit the html that will go into it. By editing the code of the webpage created we are able to upload a simple PHP script that will generate a reverse shell given a netcat listener is configure correctly. 

![Site Creation](assets/markdown_images/BoardLightMedia/Pasted image 20240605020205.png)

The simple PHP script below will open a the reverse shell as the user `www-data` to the configured netcat listener.

![PHP Reverse Shell](assets/markdown_images/BoardLightMedia/Pasted image 20240605020828.png)
# General Overview
## Initial Foothold to the system

1. Starting with a simple nmap scan to detect open ports as well as service versions we first are able to discover ports `22` and `80` open. Then moving to the service version scans I was able to detect a OpenSSH as well as a Apache web server which tells us this server is used as a http web server. We are also able to detect a host name of `board.htb` which I attempt to set as a domain name in the `/etc/hosts` file. 

![Default Nmap Scan](assets/markdown_images/BoardLightMedia/Pasted image 20240605004604.png)

![Nmap Service Scan](assets/markdown_images/BoardLightMedia/Pasted image 20240605004941.png)


2. Knowing the target is a simple web server using Apache httpd I was able to browse the set domain of `board.htb`, this revealed a quite simple web page that was not vulnerable to any obvious injection attacks. Looking through the source code however revealed a `portfolio.php` file I was unable to access, this is when I decided to start looking at possible subdomains of the domain using the `ffuf` tool which reveals a `crm.board.htb`.

![FFUF Scan](assets/markdown_images/BoardLightMedia/Pasted image 20240605013144.png)

3. Now pivoting are efforts to the newly discovered subdomain I was able to login to the dashboard using the default credentials of the Dolibarr CMS allowing for traversal of the CMS UI. Looking around this web page reveals the option to generate a new web page following either a template or creating a page from scratch. In this section we will generate an new page from scratch in attempts to get a PHP reverse shell to the server. By editing the HTML elements of the newly generated page I was able to insert a PHP script allowing for reverse shell access to the system when configured correctly.


![Site Creation](assets/markdown_images/BoardLightMedia/Pasted image 20240605020205.png)

![PHP Reverse Shell](assets/markdown_images/BoardLightMedia/Pasted image 20240605020828.png)

![Initial Foothold](assets/markdown_images/BoardLightMedia/Pasted image 20240605171408.png)

## Privilege escalation from `www-data`

With the simple shell accessed as `www-data` we can start poking around the system to find information that will lead to a possible escalation of privilege's to a higher level user. Below are some screen shots of information found during enumeration of the system that can come in handy. While linPeas did not come in handy we were able to find a password located in the config file of the CMS used in this web page, why using grep to find the `config.php` file we are able to locate the password used in order to access the mysql database. Attempting to login to the `larissa` user using this password discovered we are able to login to SSH using the password. This gives a SSH shell to escalate to this user giving us the user flag.

```
//
$dolibarr_main_url_root='http://crm.board.htb';
$dolibarr_main_document_root='/var/www/html/crm.board.htb/htdocs';
$dolibarr_main_url_root_alt='/custom';
$dolibarr_main_document_root_alt='/var/www/html/crm.board.htb/htdocs/custom';
$dolibarr_main_data_root='/var/www/html/crm.board.htb/documents';
$dolibarr_main_db_host='localhost';
$dolibarr_main_db_port='3306';
$dolibarr_main_db_name='dolibarr';
$dolibarr_main_db_prefix='llx_';
$dolibarr_main_db_user='dolibarrowner';
$dolibarr_main_db_pass='serverfun2$2023!!';
$dolibarr_main_db_type='mysqli';
$dolibarr_main_db_character_set='utf8';
$dolibarr_main_db_collation='utf8_unicode_ci';
// Authentication settings
$dolibarr_main_authentication='dolibarr';


```

![Larissa Login](assets/markdown_images/BoardLightMedia/Pasted image 20240605183035.png)
## Privilege escalation from larissa

With access to larissa we can attempt to run `linPeas` once again however this only gives us the same results as before with a CVE that does not work on the system due to missing packages. After checking the files found in the home directory of the user with no success and checking for any sudo permissions we can attempt to locate any SUID privilege's which is where we are able to find the `enlightenmnet` executables using the command `find . -perm /4000 2>/dev/null` this returns the output shown below. 

![SUID Files](assets/markdown_images/BoardLightMedia/Pasted image 20240605210050.png)

From here we can look into CVEs that involve the enlightenment executable where we come across CVE-2022-37706 where we are able to find a public exploit on github found at https://github.com/MaherAzzouzi/CVE-2022-37706-LPE-exploit. Uploading this exploit to the system using the python web server from are local host we are able to gain a root level shell to the server giving us access to the root flag.

![Root Flag](assets/markdown_images/BoardLightMedia/Pasted image 20240605210316.png)
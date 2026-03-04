# 🐞 Practical Bug Bounty – Full Hands-On Walkthrough

> Course: Practical Bug Bounty

> Provider: TCM Security

> Instructors: Heath Adams & Alex

> Environment: Kali Linux VM (Local Lab)

> Tools Used: Burp Suite Community, ffuf, SecLists, Docker

---

# Introduction

As someone currently learning Web Penetration Testing, this Practical Bug Bounty course was extremely informative and practical.

Learning alongside Heath Adams and Alex helped me understand not only *how* to perform attacks, but also *why* they work and what developers often overlook.

This course focuses specifically on **bug bounty hunting through web applications**, covering:

* Authentication attacks
* Authorization & Access Control
* API Security
* File Inclusion attacks
* Basic Injection concepts
* Reconnaissance & Information Gathering

Unlike theory-based courses, this one required setting up a real lab and actively exploiting vulnerabilities.

---

#  What is Web Application Security?

Before diving into exploitation, the course defined web application security and why it matters.

A web application is essentially any website you interact with:

* Google
* Facebook
* E-commerce platforms
* Banking portals

Every web application handles some form of data — often sensitive data.

## Why Web Application Security Is Important

* 🔐 **Data Protection** – Prevents theft of user information, payment details, and confidential records.
* 🏛 **Regulatory Compliance** – Avoids fines and legal consequences.
* 💰 **Financial Protection** – The average breach can cost millions.
* 🏷 **Brand Reputation** – Trust takes years to build and seconds to lose.
* 🧠 **Intellectual Property Protection** – Prevents trade secrets and proprietary code theft.
* 🪪 **Identity Theft Prevention** – Protects personal information from misuse.

As ethical hackers and bug bounty hunters, the goal is to identify weaknesses before malicious attackers exploit them.

---

# 🛠 Lab Environment Setup

I performed all labs locally in a Kali Linux virtual machine.

## Step 1 – Installing PimpMyKali

Repository:
https://github.com/Dewalt-arch/pimpmykali
![alt text for the image](images/Screenshot%202026-02-27%20022122.png)


PimpMyKali automates post-install configuration of Kali Linux.

### Installation Process

```bash
cd /opt
sudo git clone https://github.com/Dewalt-arch/pimpmykali
cd pimpmykali
sudo ./pimpmykali.sh
```
![alt text for the image](images/Screenshot%202026-02-27%20022122.png)

I accepted the default prompts and allowed the script to configure everything automatically.

This saved time and ensured my penetration testing environment was fully prepared.

---

## Step 2 – Installing Docker

```bash
sudo apt install docker.io docker-compose
```

After installation, I restarted the VM to ensure all services were running correctly.

---

## Step 3 – Setting Up the Practical Bug Bounty Lab

After downloading the lab ZIP file:

```bash
unzip bugbounty.zip
cd bugbounty
sudo docker-compose up
```

This pulled the necessary images and built the vulnerable lab.

In another terminal:

```bash
./set-permissions.sh
```

This step was important for file upload labs.

Accessed lab at:

```
http://localhost
```

If needed, database reset:

```
/init.php
```

I frequently reset the database to repeat attacks and reinforce understanding.

---

# 🔐 Authentication Attacks

Before attacking, I learned the difference between:

* **Authentication** → Who you are
* **Authorization** → What you’re allowed to do

---

## 🔓 Lab 0x01 – Brute Force

Target account: Jeremy

### Using Burp Suite

1. Intercepted login POST request.
2. Sent to Intruder (Ctrl + I).
3. Marked password field.
4. Loaded SecLists password list.
5. Analyzed response length differences.

Found valid password: **letmein**

---

### Using ffuf (Faster Method)

Saved raw request to file:

```bash
ffuf -request lab.txt -request-proto http \
-w /usr/share/seclists/Passwords/Common-Credentials/xato-net-10-million-passwords-1000.txt \
-fs 1814
```

Filtered by response size to detect anomalies.

Successfully identified valid credentials.

---

## 🔐 Lab 0x02 – MFA Logic Flaw

Target: Jessamy / pasta

After entering correct credentials, the system requested an MFA code.

Observations:

* MFA code was short.
* Username was submitted again during second step.

### Exploit

1. Intercepted MFA POST request.
2. Modified the username parameter.
3. Forwarded request.

Successfully logged in as another user.

### Vulnerability Type:

Logic flaw in multi-step authentication.

The system failed to bind MFA verification to the original authenticated session.

---

## 🔐 Lab 0x03 – Account Lockout Bypass

The application locked accounts after 5 failed attempts.

Instead of brute forcing one user:

### Strategy:

* Used Burp Intruder.
* Selected **Cluster Bomb** attack.
* Loaded username list.
* Used 4 common passwords.
* Avoided exceeding 5 attempts per account.

Successfully logged in as **admin:letmein**

This demonstrated bypassing weak lockout mechanisms.

---

# 🔑 Authorization & Access Control

Three types of access control covered:

* Vertical (admin vs user)
* Horizontal (user A vs user B)
* Context-dependent

---

## 🔓 IDOR Lab (0x04)

URL contained:

```
?account=1001
```

### Exploit

1. Sent request to Repeater.
2. Modified account ID.
3. Accessed other users’ data.

Used Intruder with generated numeric list to enumerate accounts.

Filtered results by searching for “admin”.

Discovered multiple admin accounts.

---

### Using ffuf

```bash
ffuf -u http://localhost/labs/e0x02.php?account=FUZZ \
-w num.txt -mr "admin"
```

Found valid admin IDs.

Vulnerability: **Insecure Direct Object Reference (IDOR)**
(API equivalent: Broken Object Level Authorization)

---

# 🔐 API Security & JWT Testing

APIs made up most of the lab traffic.

---

## API 0x01 – JWT Analysis

Logged in via API endpoint.

Received JWT token.

Noticed:

* Header
* Payload
* No proper signature validation

Decoded token manually using Burp Decoder (Base64 decode).

---

## Broken Function Level Authorization

Steps:

1. Logged in as Jeremy.
2. Retrieved token.
3. Updated Jeremy’s bio (expected behavior).
4. Used Jeremy’s token to update Jessamy’s bio.
5. Request succeeded.

Confirmed broken authorization.

Jeremy should not be able to modify Jessamy’s data.

---

## Autorize Extension

Installed Autorize extension in Burp.

Although it required Jython setup, I tested access control by sending requests through the extension and analyzing bypass/enforced results.

This helped automate horizontal authorization testing.

---

# 📂 File Inclusion Attacks

---

## File Inclusion 0x01 – Basic LFI

Tested:

```
../../../../etc/passwd
```

Successfully retrieved system file.

---

## File Inclusion 0x02 – Filter Bypass + RFI

Initial traversal blocked.

Bypass technique used:

```
....//....//....//etc/passwd
```

Filter removed partial sequences but payload reconstructed successfully.

Also tested:

```
https://www.google.com
```

Confirmed Remote File Inclusion capability.

---

## PHP Filter Wrapper Exploit

Used:

```
php://filter/convert.base64-encode/resource=db.php
```

Decoded response in Burp.

Extracted database credentials.

---

## API-Based LFI (0x03)

Saved API request.

Fuzzed with ffuf:

```bash
ffuf -request api-req.txt -request-proto http \
-w /usr/share/seclists/Fuzzing/LFI/LFI-Jhaddix.txt
```

Filtered by word count and response size.

Identified working payload.

Confirmed LFI via API endpoint.

---

# 🧠 What I Learned

* Always step through functionality manually first.
* Response length and word count matter.
* Logic flaws are often more dangerous than brute force.
* Broken Access Control is extremely common.
* JWT misuse is widespread.
* Filters must be recursive and properly validated.
* Resetting labs and repeating attacks improves understanding.
* Mental fatigue affects testing accuracy — breaks are important.

---

# 🎯 Final Reflection

This course significantly improved my practical web penetration testing skills.

I:

* Built the lab environment from scratch.
* Used Burp Community effectively.
* Used ffuf for faster fuzzing.
* Tested authentication and authorization thoroughly.
* Exploited API vulnerabilities.
* Performed LFI and filter bypass attacks.
* Extracted sensitive credentials.
* Understood real-world impact of vulnerabilities.

Most importantly, I didn’t just watch the walkthrough — I actively performed the attacks, reset the database, and repeated exercises until I fully understood them.

This course strengthened my bug bounty methodology and confidence in testing real-world web applications.

---

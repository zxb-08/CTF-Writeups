# Summit: TryHackMe Writeup

![Room overview](images/01-room-overview.png)

## Introduction

PicoSecure is conducting a threat simulation and detection engineering to improve malware detection capabilities.

We have been assigned to work with an external penetration tester. The tester will be attempting to execute malware samples on a simulated internal workstation, whereas we need to configure security tools to detect and prevent them from executing.

## Methodology

## Flag 1: block the hash (sample1.exe)

Sphinx starts simple. I scanned `sample1.exe` in the sandbox and read the report. It is tagged `Trojan.Metasploit.A`, Metasploit was detected under malicious behaviour, and it connects to an unusual port. The report also lists the file hashes (MD5, SHA1, SHA256).

![sample1.exe sandbox report](images/02-flag1-sample1-sandbox-report.png)

A hash identifies this exact file and nothing else, so to block it uniquely I added the MD5 hash to the blocklist:

```
MD5: cbda8ae000aa9cbe7c8b982bae006c2a
```
<br>
<br>

## Flag 2: block the IP (sample2.exe)

Sphinx reminds me not to rely on hashes alone, since changing a single bit of the file changes its hash. So I needed to see what the malware actually does.

The sandbox report for `sample2.exe` shows the same Metasploit detection, plus two new suspicious behaviours: it connects to an unusual IP address and an unusual port.

![sample2.exe behaviour analysis](images/03-flag2-sample2-behaviour-analysis.png)

The network activity shows the connections:

- `http://154.35.10.113:4444/uvLk8YI32` over HTTP (Intrabuzz Hosting Limited)
- `40.97.128.3:443` and `40.97.128.4:443` (Microsoft Corporation)

![sample2.exe network activity](images/04-flag2-sample2-network-activity.png)

The suspicious one is `154.35.10.113`. I added an egress firewall rule to deny all outgoing connections to it.

![Firewall rule](images/05-flag2-firewall-rule.png)
<br>
<br>
<br>

## Flag 3: block the domain (sample3.exe)

The IP block isn't bulletproof. Sphinx says he can get around it with a new public IP, so I looked into `sample3.exe`.

Its network activity shows repeated connections to two domains: `services.microsoft.com` and `emudyn.bresonicz.info`. The first is Microsoft's. The second resolves to `62.123.140.9` and is the one `sample3.exe` keeps calling. It also downloads `backdoor.exe` from it, which then makes its own connection to the same host.

![sample3.exe network activity](images/06-flag3-sample3-network-activity.png)

I added a DNS filter rule denying `emudyn.bresonicz.info`. Denying a domain also covers its subdomains.

![DNS filter rule](images/07-flag3-dns-filter-rule.png)
<br>
<br>

## Flag 4: detect the registry change (sample4.exe)

By now hashes, IPs and domains are blocked. Some attackers give up and find another target. Sphinx doesn't. This time I studied what `sample4.exe` runs, which processes it creates, and what it leaves behind.

The network side is a new domain and IP again (`cranes0ft.iniware.xyz`, `102.23.20.118`), so blocking by indicator would keep failing.

![sample4.exe network activity](images/08-flag4-sample4-network-activity.png)

The registry activity is more interesting. `sample4.exe` (PID 3806) writes to Windows Defender's real-time protection key:

```
Key:   HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\Windows Defender\Real-Time Protection
Name:  DisableRealtimeMonitoring
Value: 1
```

![sample4.exe registry activity](images/09-flag4-sample4-registry-activity.png)

It's trying to switch off our real-time monitoring. That maps to the MITRE ATT&CK tactic **Defense Evasion (TA0005)**.

To detect it I used the room's Sigma rule builder: Sysmon Event Logs → Registry Modifications.

![Sigma rule builder](images/10-flag4-sigma-rule-builder.png)
![Step 1: Sysmon event logs](images/11-flag4-sigma-step1-sysmon.png)
![Step 2: registry modifications](images/12-flag4-sigma-step2-registry.png)

In step 3 I entered the key, name and value from the report and picked TA0005.

![Step 3: registry conditions](images/13-flag4-sigma-registry-conditions.png)
![Generated Sigma rule](images/14-flag4-sigma-rule-validation.png)
<br>
<br>

## Flag 5: detect the beacon (sample5.exe)

A different approach this time. The instructions now come from Sphinx's back-end server, so I needed a unique behaviour of `sample5.exe` to detect it.

Sphinx sent a log of outgoing connections. Most entries are one-off connections of varying sizes, but one pattern stands out: source `10.10.15.12` sends a 97 byte packet to `51.102.10.19` on port 443 every 30 minutes (1800 seconds).

![outgoing_connections.log](images/15-flag5-outgoing-connections-log.png)

The sandbox report matches. `sample5.exe` downloads `beacon.bat` from `bababa10la.cn`, and `beacon.bat` then POSTs a keep-alive with the hostname (`WK102`) over and over. The sandbox hit 402 HTTP requests and stopped listing them.

![sample5.exe network activity](images/16-flag5-sample5-network-activity.png)
![sample5.exe connections and DNS](images/17-flag5-sample5-connections-dns.png)

So I built a Sigma rule for network connections from the host: any remote IP, any port, size 97 bytes, frequency 1800 seconds. This maps to the tactic **Command and Control (TA0011)**.

![Sigma network connection conditions](images/18-flag5-sigma-network-conditions.png)
<br>
<br>

## Flag 6: detect the exfiltration staging (sample6.exe)

For the final flag, Sphinx sends `sample6.exe` with the recorded command logs from all the previous samples. Every command appends its output to the same file in `%temp%`:

![commands.log](images/19-flag6-commands-log.png)

The sandbox confirms it. When `sample6.exe` runs, it opens `cmd.exe` with itself as the parent, and one of those `cmd.exe` processes creates `exfiltr8.log` in `%temp%`.

![sample6.exe processes and dropped files](images/20-flag6-sample6-processes-files.png)

Data is being collected into a log file in the temp folder, which maps to the tactic **Exfiltration (TA0010)**. I built a Sigma rule for file creation: path `%temp%`, file name `exfiltr8.log`.

![Sigma file creation conditions](images/21-flag6-sigma-file-creation.png)

## Results

All six samples were detected or blocked, and Sphinx gave up.

![Final mail and completed flags (flags blurred)](images/22-completion-final-mail.png)

## Discussion

Each flag is harder for the attacker to get around than the one before. Changing a hash is a one-bit edit, a new IP or domain is cheap to set up, but changing how the malware behaves means rewriting it. That's why the later detections (registry write, beacon pattern, file drop) are based on behaviour and not on indicators.

Limitations I noticed:

- The hash, IP and domain blocks only stop that exact sample setup.
- The beacon rule depends on the exact size (97 bytes) and interval (1800 seconds). If Sphinx changes either, it won't match.
- The `exfiltr8.log` rule depends on the file name and location. A renamed file or a different folder gets past it.

## Conclusion

The takeaway for me: indicators are quick to deploy but easy to dodge, so a good detection should be built on what the malware does, not what it looks like.

## References

- TryHackMe: [Summit room](https://tryhackme.com/room/summit)
- MITRE ATT&CK tactics: [TA0005 Defense Evasion](https://attack.mitre.org/tactics/TA0005/), [TA0011 Command and Control](https://attack.mitre.org/tactics/TA0011/), [TA0010 Exfiltration](https://attack.mitre.org/tactics/TA0010/)
- David Bianco, [The Pyramid of Pain](https://detect-respond.blogspot.com/2013/03/the-pyramid-of-pain.html)

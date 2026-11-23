# RedLine — Memory Forensics

## Overview

| Field | Details |
|---|---|
| Platform | CyberDefenders |
| Lab | [RedLine](https://cyberdefenders.org/blueteam-ctf-challenges/redline/) |
| Category | Endpoint Forensics |
| Difficulty | Easy |
| Completion Date | 7 November 2026 |
| Status | Completed — 7/7 |
| Environment | Kali Linux |
| Tools | Volatility 3, Strings, SHA-256, VirusTotal |

## Objective

The objective of this lab was to investigate a Windows memory dump and identify evidence of suspicious activity.

I used Volatility 3 to examine running processes, memory regions and network connections. I also used threat intelligence sources to investigate indicators that could be related to the incident.

The investigation was carried out using the evidence provided by CyberDefenders.

## Environment and Evidence Integrity

The analysis was performed on Kali Linux using Volatility 3 v2.28.2.

The provided evidence included a 2 GiB Windows memory image named `MemoryDump.mem`, extracted from `106-RedLine.zip`.

I calculated SHA-256 hashes for the downloaded ZIP file and its working copy.

```bash
sha256sum ~/Downloads/106-RedLine.zip ~/Digital-Forensics/Lab-02-Memory-Forensics/106-RedLine.zip
```

Both hashes matched, confirming that the working copy was identical to the downloaded file. However, no official reference hash was available to verify the original download independently.

**Evidence E01 — File Integrity**

![SHA-256 hash comparison](evidence/E01-redline-evidence-integrity.webp)

## Investigation

### 1. System Identification

I started by examining the operating system information stored in the memory image.

```bash
vol -f MemoryDump.mem windows.info
```

The main results were:

| Field | Result |
|---|---|
| Architecture | Windows x64 |
| Kernel version | 10.0 |
| Logical processors | 4 |
| System time | 2023-05-21 23:02:39 UTC |
| System root | `C:\Windows` |

This provided initial context for the investigation. The exact Windows edition was not independently confirmed.

**Evidence E02 — Windows System Information**

![Windows information recovered using Volatility 3](evidence/E02-redline-windows-system-information.webp)

### 2. Suspicious Process Analysis

I used `windows.pslist` and `windows.pstree` to examine the processes recovered from memory and their parent-child relationships.

```bash
vol -f MemoryDump.mem windows.pslist
vol -f MemoryDump.mem windows.pstree
```

The analysis identified two instances of `oneetx.exe`. One instance, with PID `5896`, was associated with a child process named `rundll32.exe`.

| Process | PID | PPID |
|---|---|---|
| oneetx.exe | 5896 | 8844 |
| rundll32.exe | 7732 | 5896 |
| oneetx.exe | 5480 | 448 |

`rundll32.exe` is a legitimate Windows utility, but it can also be used during malicious activity. Its relationship with the suspicious process required further investigation.

I also attempted to examine command-line arguments and loaded DLLs, but the relevant plugins did not recover useful information for these instances.

### 3. Suspicious Memory Region

I used the `windows.malfind` plugin to examine potentially suspicious memory regions associated with the process.

```bash
vol -f MemoryDump.mem windows.malfind --pid 5896
```

The results included:

| Field | Result |
|---|---|
| Process | oneetx.exe |
| PID | 5896 |
| Memory region | `0x400000–0x437fff` |
| Protection | `PAGE_EXECUTE_READWRITE` |
| Additional observation | MZ header |

The memory region allowed reading, writing and execution. Together with the MZ header, this made it worth investigating.

However, these findings alone did not prove that code injection had occurred.

**Evidence E03 — Suspicious Memory Region**

![Suspicious memory region identified using Volatility 3](evidence/E03-redline-suspicious-memory-region.webp)

### 4. Network Connection Analysis

I examined network artefacts recovered from the memory image using:

```bash
vol -f MemoryDump.mem windows.netscan
```

A network connection associated with the suspicious process was identified.

| Field | Result |
|---|---|
| Process | oneetx.exe |
| PID | 5896 |
| Local address | 10.0.85.2 |
| Remote address | 77.91.124.20 |
| Remote port | TCP/80 |
| State | CLOSED |
| Timestamp | 2023-05-21 23:01:22 UTC |

This connected the suspicious process to an external IP address.

The connection artefact did not reveal the contents of the communication, so I could not determine exactly what data had been exchanged.

Other recovered processes included `Outline.exe` and `tun2socks.exe`, which were associated with VPN functionality.

**Evidence E04 — Network Connection Analysis**

![Network artefacts recovered using Volatility 3](evidence/E04-redline-suspicious-network-connection.webp)

### 5. Threat Intelligence Investigation

I investigated the remote IP address using VirusTotal.

At the time of my investigation in November 2026, 15 out of 92 security vendors flagged the address as malicious.

Community intelligence also included references linking the IP address to RedLine-related activity and historical malware infrastructure.

These results supported the suspicion that the connection was security-relevant.

However, threat intelligence classifications can change over time. The results observed in November 2026 did not independently establish the IP address's status in May 2023.

**Evidence E05 — VirusTotal Investigation**

![Threat intelligence results for the investigated IP address](evidence/E05-redline-threat-intelligence-ip-analysis.webp)

### 6. Suspicious URL Identification

I used `strings` to search the memory image for readable text containing the external IP address.

```bash
strings -a MemoryDump.mem | grep -F '77.91.124.20'
```

The output included references to:

```text
http://77.91.124.20/store/games/index.php
```

This URL used the same IP address identified by `windows.netscan`.

The correlation provided additional evidence linking the recovered network artefact to information stored in memory.

However, I did not reconstruct a complete HTTP request or confirm the contents of any transmitted data.

**Evidence E06 — Suspicious URL**

![Suspicious URL recovered from the memory image](evidence/E06-redline-suspicious-url-memory-analysis.webp)

### 7. Executable Location

Using `windows.pstree`, I recovered the location associated with the suspicious executable.

```text
C:\Users\Tammam\AppData\Local\Temp\c3912af058\oneetx.exe
```

The executable was located in a temporary directory within the user's profile.

Running an executable from a temporary directory is not automatically malicious, but it was relevant when considered alongside the process relationships, memory permissions and network findings.

**Evidence E07 — Suspicious Process Path**

![Process information and executable path recovered using Volatility 3](evidence/E07-redline-suspicious-process-path.webp)

## Key Findings

| Finding | Observation |
|---|---|
| Suspicious executable | `oneetx.exe` |
| Related process | `rundll32.exe` |
| Memory protection | `PAGE_EXECUTE_READWRITE` |
| External connection | `77.91.124.20:80` |
| Suspicious URL | `/store/games/index.php` |
| Executable location | User profile temporary directory |
| Threat intelligence | Historical references to RedLine-related activity |

The findings were consistent with suspicious activity on the investigated Windows endpoint.

However, I did not independently analyse the executable to confirm its malware family.

## Security Impact and Recommendations

The investigation identified indicators that would justify further incident response activity in a real environment.

Recommended actions would include:

- Isolating the affected endpoint while investigating the incident.
- Examining persistence mechanisms and the origin of the suspicious executable.
- Reviewing endpoint, DNS, proxy and firewall logs.
- Validating relevant indicators before implementing blocks.
- Analysing the executable safely in an isolated environment.
- Investigating possible data exfiltration or lateral movement.

## Investigation Limitations

The available evidence did not allow me to confirm every aspect of the incident.

In particular:

- The complete infection chain was not reconstructed.
- The malware family was not independently confirmed through executable analysis.
- Some command-line arguments and DLL information could not be recovered.
- The contents of the network communication were not available.
- Possible data exfiltration was not established.

These limitations were important when interpreting the findings.

## Results and Lessons Learned

I completed all seven questions in the CyberDefenders RedLine lab.

This investigation gave me practical experience with Windows memory analysis using Volatility 3.

The main skills I practised were examining process relationships, identifying unusual memory regions, recovering network artefacts and correlating findings with threat intelligence.

I also learned that suspicious indicators should be considered together rather than treated as proof of compromise individually.

One of the most useful lessons was understanding that a missing plugin result does not necessarily mean the activity never occurred.

## References

- [CyberDefenders — RedLine](https://cyberdefenders.org/blueteam-ctf-challenges/redline/)
- [Volatility 3 Documentation](https://volatility3.readthedocs.io/)
- [Volatility 3 — GitHub](https://github.com/volatilityfoundation/volatility3)
- [VirusTotal — IP Investigation](https://www.virustotal.com/gui/ip-address/77.91.124.20)

---

*Part of the [Cybersecurity Labs](../../README.md) repository.*

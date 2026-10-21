# Security Monitoring and Detection Tests

This document records the security events generated and analyzed in my [Wazuh](https://wazuh.com/) home lab. Each test uses controlled activity and evidence collected from Wazuh.

## 1. [File Integrity Monitoring (FIM)](https://www.crowdstrike.com/en-us/cybersecurity-101/exposure-management/file-integrity-monitoring/)

**Objective:** Check whether Wazuh detects file creation, modification and deletion on a Windows endpoint.

**Endpoint:** `SOC-Windows` (agent `001`)  
**Monitored directory:** `C:\Lab-FIM`

Opened the Windows Agent configuration file in PowerShell (Administrator):

```powershell
notepad "C:\Program Files (x86)\ossec-agent\ossec.conf"
```

Added the following entry inside the existing `<syscheck>` section to enable real-time monitoring of our test directory:

```xml
<directories realtime="yes">C:\Lab-FIM</directories>
```

Saved the configuration and restarted the Wazuh Agent using PowerShell (Administrator):

```powershell
Restart-Service -Name WazuhSvc
```

### Test results

| Action | Test performed                                 | Rule ID | Level | Result   |
| ------ | ---------------------------------------------- | ------- | ----- | -------- |
| Create | Created `admin-secret.txt` and `ciso-note.txt` | `554`   | 5     | Detected |
| Modify | Added text to `ciso-note.txt` and saved it     | `550`   | 7     | Detected |
| Delete | Deleted `admin-secret.txt`                     | `553`   | 7     | Detected |

### Observations and analysis

**File creation:** Creating a text document in Windows File Explorer triggered an `added` event for `New Text Document.txt`, followed by a `deleted` event for the original path and another `added` event for the chosen filename. This occurred even when the file was named immediately during creation. The events reflect the filename change rather than an actual destructive action.

![FIM file creation events](../images/fim-file-creation.png)

**File modification:** The event for `ciso-note.txt` recorded a size change from 0 to 64 bytes, an updated modification time and different MD5, SHA-1 and SHA-256 hashes. The checksum changes confirm that the file contents changed.

![FIM file modification details](../images/fim-file-modification.png)

**File deletion:** The alert for `admin-secret.txt` included MITRE ATT&CK mappings `T1070.004` (File Deletion) and `T1485` (Data Destruction). Those mappings identify potentially relevant techniques; they do not mean this controlled deletion was malicious.

![FIM file deletion details](../images/fim-file-deletion.png)

**Result:** All three actions were detected in real time. The alerts identify file changes, but do not establish properly who made them or why.

## 2. [Windows Authentication Monitoring](https://learn.microsoft.com/en-us/windows-server/identity/ad-ds/plan/Appendix-L--Events-to-Monitor)

_Pending testing and analysis._

## 3. [SSH Authentication Monitoring](https://documentation.wazuh.com/current/proof-of-concept-guide/detect-brute-force-attack.html)

_Pending testing and analysis._

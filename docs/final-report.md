# Security Monitoring - Final Report

I built a small Wazuh home lab to practice collecting security logs, testing detections and investigating the resulting alerts. After setting up the monitoring environment, I carried out three controlled tests: **File Integrity Monitoring (FIM), Windows authentication and SSH authentication.**

The aim was to check whether Wazuh could detect these activities and to understand what the recorded events actually showed. The full test steps, event details and evidence screenshots are available in [Detection and Analysis](detection-and-analysis.md).

## Lab Environment

The lab ran in Oracle VirtualBox using a Host-only network for communication between the systems.

| System              | Role                                                        |
| ------------------- | ----------------------------------------------------------- |
| Ubuntu Server 24.04 | Wazuh Manager, Indexer and Dashboard; SSH monitoring target |
| Windows 11 Pro      | Monitored endpoint running the Wazuh Agent                  |
| Windows host        | Used to initiate SSH connections to Ubuntu                  |

The Windows endpoint was enrolled as agent `001` (`SOC-Windows`). Ubuntu's local events appeared under manager agent `000` (`wazuh-server`). The server and endpoint setup is recorded in the [Ubuntu](ubuntu-server-setup.md), [Wazuh Server](wazuh-server-setup.md) and [Windows Agent](windows-agent-setup.md) guides.

## Test Summary

| Test                      | Activity                                                                   | Outcome                                 |
| ------------------------- | -------------------------------------------------------------------------- | --------------------------------------- |
| File Integrity Monitoring | Created, modified and deleted files in a monitored Windows directory       | All three changes detected in real time |
| Windows authentication    | Entered incorrect passwords, then signed in successfully                   | Failed and successful logins detected   |
| SSH authentication        | Entered two incorrect SSH passwords, then connected successfully to Ubuntu | Failed and successful attempts detected |

All three planned tests produced the expected Wazuh alerts.

## Findings

**File monitoring:** Wazuh recorded file creation, modification and deletion, including changed file hashes when content was edited. I also noticed that naming a new text file in File Explorer could produce an added, deleted and added sequence. This showed why an alert needs context: a reported deletion can sometimes be part of a filename change rather than deliberate removal of data.

**Windows logins:** Wazuh collected Windows Security events for both failed and successful interactive logins. The failed-login event included the account name, logon type and status codes, which helped confirm that the failure was caused by an incorrect password.

**SSH logins:** Ubuntu recorded the SSH attempts in `journald`, and Wazuh detected them without a separate Linux agent. The failed attempts and later success had the same source IP, account and source port, allowing the events to be connected to one SSH session. In an actual investigation, that sequence could deserve closer attention if the activity were unexpected.

## Limitations

These were controlled tests in an isolated home lab, not real security incidents. The alerts confirmed that monitoring was working, but they did not prove malicious intent or identify the person responsible for a file change. MITRE ATT&CK mappings provided useful context, but were not evidence of an attack by themselves.

I tested basic detection and manual event analysis only. I did not measure detection response times, test large-scale attacks or evaluate automated incident response.

## Conclusion

The lab met its main objective: Wazuh detected the file and authentication events generated during all three tests, and I was able to examine the alerts against the activity that caused them.

The most useful part was learning to read the evidence rather than relying only on an alert's title or severity. File renaming, incorrect passwords and successful logins can all generate security events, but their meaning depends on the surrounding activity. This project gave me practical experience with Windows and Linux log monitoring, basic alert investigation and documenting findings from actual tests.

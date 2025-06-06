<img width="400" src="https://github.com/user-attachments/assets/44bac428-01bb-4fe9-9d85-96cba7698bee" alt="Tor Logo with the onion and a crosshair on it"/>

# Threat Hunt Report: Unauthorized TOR Usage
- [Scenario Creation](https://github.com/saulsosa1/Threat-Hunting-Scenario-Tor/blob/main/threat-hunting-scenario-tor-event-creation.md)

## Platforms and Languages Leveraged
- Windows 10 Virtual Machines (Microsoft Azure)
- EDR Platform: Microsoft Defender for Endpoint
- Kusto Query Language (KQL)
- Tor Browser

##  Scenario

Management suspects that some employees may be using TOR browsers to bypass network security controls because recent network logs show unusual encrypted traffic patterns and connections to known TOR entry nodes. Additionally, there have been anonymous reports of employees discussing ways to access restricted sites during work hours. The goal is to detect any TOR usage and analyze related security incidents to mitigate potential risks. If any use of TOR is found, notify management.

### High-Level TOR-Related IoC Discovery Plan

- **Check `DeviceFileEvents`** for any `tor(.exe)` or `firefox(.exe)` file events.
- **Check `DeviceProcessEvents`** for any signs of installation or usage.
- **Check `DeviceNetworkEvents`** for any signs of outgoing connections over known TOR ports.

---

## Steps Taken

### 1. Searched the `DeviceFileEvents` Table

Searched for any file that had the string "tor" in it and discovered what looks like the user "svillan" downloaded a TOR installer, did something that resulted in many TOR-related files being copied to the desktop, and the creation of a file called `tor-shopping-list.txt` on the desktop at `2025-05-06T23:16:29.3232184Z`. These events began at `2025-05-06T22:43:36.6269698Z`.

**Query used to locate events:**

```kql
DeviceFileEvents  
| where DeviceName == "saul-mde"  
| where InitiatingProcessAccountName == "svillan"  
| where FileName contains "tor"  
| where Timestamp >= datetime(2025-05-06T22:43:36.6269698Z)  
| order by Timestamp desc  
| project Timestamp, DeviceName, ActionType, FileName, FolderPath, SHA256, Account = InitiatingProcessAccountName
```
![image](https://github.com/user-attachments/assets/76335293-d031-4b72-834c-7bd2f7f4299e)

---

### 2. Searched the `DeviceProcessEvents` Table

Searched for any `ProcessCommandLine` that contained the string "tor-browser-windows-x86_64-portable-14.5.1.exe". Based on the logs returned, at `2025-05-06T22:53:22.3554775Z`, the user "svillan" on the "saul-mde" device ran the file `tor-browser-windows-x86_64-portable-14.5.1.exe` from their Downloads folder, using a command that triggered a silent installation.

**Query used to locate event:**

```kql
DeviceProcessEvents
| where DeviceName == "saul-mde"
| where ProcessCommandLine contains "tor-browser-windows-x86_64-portable-14.5.1.exe"
| project Timestamp, DeviceName, AccountName, ActionType, FileName, FolderPath, ProcessCommandLine, SHA256
```
![image](https://github.com/user-attachments/assets/dd186fe9-2d7e-4dd9-af58-c02c863df153)

---

### 3. Searched the `DeviceProcessEvents` Table for TOR Browser Execution

Searched for any indication that user "svillan" actually opened the TOR browser. There was evidence that they did open it at `2025-05-06T22:54:57.097374Z`. There were several other instances of `firefox.exe` (TOR) as well as `tor.exe` spawned afterwards.

**Query used to locate events:**

```kql
DeviceProcessEvents
| where DeviceName == "saul-mde"
| where FileName has_any ("tor.exe", "firefox.exe", "tor-browser.exe")
| project Timestamp, DeviceName, AccountName, ActionType, FileName, FolderPath, ProcessCommandLine, SHA256
| order by Timestamp desc
```
![image](https://github.com/user-attachments/assets/aca7e2b1-bfa6-4d4c-be56-38064b5f722d)

---

### 4. Searched the `DeviceNetworkEvents` Table for TOR Network Connections

Searched for any indication the TOR browser was used to establish a connection using any of the known TOR ports. At `2025-05-06T22:55:15.5353294Z`, user "svillan" on the "saul-mde" device successfully established a connection to the remote IP address `107.189.8.181` on port `443`. The connection was initiated by the process `tor.exe`, located in the folder `c:\users\svillan\desktop\tor browser\browser\torbrowser\tor\tor.exe`. There were a couple of other connections over ports `443` and `9150`.

**Query used to locate events:**

```kql
DeviceNetworkEvents  
| where DeviceName == "saul-mde"  
| where InitiatingProcessAccountName != "system"  
| where InitiatingProcessFileName in ("tor.exe", "firefox.exe")  
| where RemotePort in ("9001", "9030", "9040", "9050", "9051", "9150", "80", "443")  
| project Timestamp, DeviceName, InitiatingProcessAccountName, ActionType, RemoteIP, RemotePort, RemoteUrl, InitiatingProcessFileName, InitiatingProcessFolderPath  
| order by Timestamp desc
```
![image](https://github.com/user-attachments/assets/fa7b0cfc-e08f-48d7-96f1-e7be57bc3b81)

---

## Chronological Event Timeline 

### 1. File Download - TOR Installer

- **Timestamp:** `2025-05-06T22:43:36.6269698Z`
- **Event:** The user "svillan" downloaded a file named `tor-browser-windows-x86_64-portable-14.5.1.exe` to the Downloads folder.
- **Action:** File download detected.
- **File Path:** `C:\Users\svillan\Downloads\tor-browser-windows-x86_64-portable-14.5.1.exe`

### 2. Process Execution - TOR Browser Installation

- **Timestamp:** `2025-05-06T22:53:22.3554775Z`
- **Event:** The user "svillan" executed the file `tor-browser-windows-x86_64-portable-14.5.1.exe` in silent mode, initiating a background installation of the TOR Browser.
- **Action:** Process creation detected.
- **Command:** `tor-browser-windows-x86_64-portable-14.5.1.exe /S`
- **File Path:** `C:\Users\svillan\Downloads\tor-browser-windows-x86_64-portable-14.5.1.exe`

### 3. Process Execution - TOR Browser Launch

- **Timestamp:** `2025-05-06T22:54:57.097374Z`
- **Event:** User "svillan" opened the TOR browser. Subsequent processes associated with TOR browser, such as `firefox.exe` and `tor.exe`, were also created, indicating that the browser launched successfully.
- **Action:** Process creation of TOR browser-related executables detected.
- **File Path:** `C:\Users\svillan\Desktop\Tor Browser\Browser\TorBrowser\Tor\tor.exe`

### 4. Network Connection - TOR Network

- **Timestamp:** `2025-05-06T22:55:15.5353294Z`
- **Event:** A network connection to IP `107.189.8.181` on port `443` by user "svillan" was established using `tor.exe`, confirming TOR browser network activity.
- **Action:** Connection success.
- **Process:** `tor.exe`
- **File Path:** `C:\Users\svillan\Desktop\Tor Browser\Browser\TorBrowser\Tor\tor.exe`

### 5. Additional Network Connections - TOR Browser Activity

- **Timestamps:**
  - `2025-05-06T22:55:16Z` - Connected to `107.189.8.181` on port `443`.
  - `2025-05-06T22:55:29Z` - Local connection to `127.0.0.1` on port `9150`.
- **Event:** Additional TOR network connections were established, indicating ongoing activity by user "svillan" through the TOR browser.
- **Action:** Multiple successful connections detected.

### 6. File Creation - TOR Shopping List

- **Timestamp:** `2025-05-06T23:16:29.3232184Z`
- **Event:** The user "svillan" created a file named `tor-shopping-list.txt` on the desktop, potentially indicating a list or notes related to their TOR browser activities.
- **Action:** File creation detected.
- **File Path:** `C:\Users\svillan\Desktop\tor-shopping-list.txt`

---

## Summary

The user "svillan" on the "saul-mde" device initiated and completed the installation of the TOR browser. They proceeded to launch the browser, establish connections within the TOR network, and created various files related to TOR on their desktop, including a file named `tor-shopping-list.txt`. This sequence of activities indicates that the user actively installed, configured, and used the TOR browser, likely for anonymous browsing purposes, with possible documentation in the form of the "shopping list" file.

---

## Response Taken

TOR usage was confirmed on the endpoint `saul-mde` by the user `svillan`. The device was isolated, and the user's direct manager was notified.

- [Incident Report Email](https://github.com/saulsosa1/Threat-Hunting-Scenario-Tor/blob/main/incident-report-email.md)

---

## Conclusion and Recommendations

To prevent recurrence of unauthorized Tor usage and strengthen the organization's security posture, the following recommendations are proposed for management’s consideration:

- **Network Controls:** Configure firewalls to block known Tor ports (`9001`, `9030`, `9040`, `9050`, `9051`, `9150`) to prevent connections to Tor entry nodes. This reduces the risk of employees bypassing network security controls.

- **EDR Enhancements:** Implement Microsoft Defender for Endpoint rules to alert on Tor-related file and process activity (e.g., `tor.exe`, `firefox.exe` with Tor-specific command lines). This enables proactive detection of unauthorized software.

- **Data Loss Prevention (DLP):** Deploy DLP policies to monitor and block the creation or transfer of files with suspicious names (e.g., containing “tor” or “shopping-list”). This mitigates risks associated with sensitive data documentation.

- **User Awareness Training:** Conduct training to educate employees on the organization's acceptable use policy, emphasizing the risks of unauthorized software like Tor Browser, which could lead to data breaches or compliance violations.

- **Follow-Up Investigation:** Perform a forensic analysis of the `saul-mde` device to examine the contents of `tor-shopping-list.txt` and check for additional indicators of compromise, such as lateral movement or data exfiltration attempts. This could involve analyzing browser history or temporary files created by Tor.

- **Potential Escalations:** Depending on management’s review of the incident, consider escalating to HR for policy violation enforcement or to legal teams if the contents of `tor-shopping-list.txt` suggest illicit activities (e.g., accessing dark web marketplaces).

---


+++
title = 'Wireshark-TCPDUMP-Windows2003'
+++
- [Wireshark Download Link](#wireshark-download-link)
- [Step to Generate Report](#step-to-generate-report)
  - [Identify IP Adress \& Interface Name](#identify-ip-adress--interface-name)
  - [Install Wireshark](#install-wireshark)
  - [Open Wireshark](#open-wireshark)
  - [Start Capture Wireshark](#start-capture-wireshark)
  - [Stop Capturing After period of times](#stop-capturing-after-period-of-times)
  - [Open Statistics =\> Conversations](#open-statistics--conversations)
  - [View Captured Conversations](#view-captured-conversations)
- [Step to Export to Excel ( CSV File )](#step-to-export-to-excel--csv-file-)
  - [Open dump file in latest wireshark version](#open-dump-file-in-latest-wireshark-version)

# Wireshark Download Link

[:link: Wireshark Archived Download Link](https://2.na.dl.wireshark.org/win32/all-versions/)

# Step to Generate Report

## Identify IP Adress & Interface Name

Get IP Address & Interface Name from Windows Connections

| Type | Desc |
|---|---|
| Interface Name | Intel(R) PRO/1000 MT Network Connection |
| IP Address | 10.10.51.43 |

![MyBase64Image](/assets/images/tcpdump-wireshark-win2003/01_Get_Interface_IP.png?raw=true)

## Install Wireshark

Download & Install Wireshark

![MyBase64Image](/assets/images/tcpdump-wireshark-win2003/02_Install_Wireshark.png?raw=true)

:warning: **Note** : use the correct version for each windows version
```
Older versions of Windows which are outside Microsoft’s extended lifecycle support window are no longer supported. It is often difficult or impossible to support these systems due to circumstances beyond our control, such as third party libraries on which we depend or due to necessary features that are only present in newer versions of Windows such as hardened security or memory management.

Wireshark 4.0 was the last release branch to officially support Windows 8.1 and Windows Server 2012.
Wireshark 3.6 was the last release branch to officially support 32-bit Windows.
Wireshark 3.2 was the last release branch to officially support Windows 7 and Windows Server 2008 R2.
Wireshark 2.2 was the last release branch to support Windows Vista and Windows Server 2008 sans R2
Wireshark 1.12 was the last release branch to support Windows Server 2003.
Wireshark 1.10 was the last release branch to officially support Windows XP.
```

## Open Wireshark

Open wireshark from Windows Start Button


![MyBase64Image](/assets/images/tcpdump-wireshark-win2003/03_Open_wrieshark.png?raw=true)

## Start Capture Wireshark

Start Capture Package

![MyBase64Image](/assets/images/tcpdump-wireshark-win2003/04_Start_Capture.png?raw=true)

| # | Step | Desc |
|---|---|---|
| 1 | Open Capture Options | Click Capture Options |
| 2 | Select Network Interface for Capturing | Intel(R) PRO/1000 MT Network Connection, ${INTERFACE_NAME} |
| 3 | Limit Capturing Packet Length | 68 bytes |
| 4 | Filter Capturing Packet | host 10.10.51.43, host ${INTERFACE_IP} |
| 5 | Specify Output File | dump_10.10.51.43, ${FILENAME} |
| 6 | Start Capturing | Click Start |

## Stop Capturing After period of times

Stop Packet Capturing 


![MyBase64Image](/assets/images/tcpdump-wireshark-win2003/05_Stop_Capturing.png?raw=true)


| # | Step | Desc |
|---|---|---|
| 1 | Stop Packet Capturing | Click Stop |

## Open Statistics => Conversations

Open Statistics => Conversations

![MyBase64Image](/assets/images/tcpdump-wireshark-win2003/06_View_Statistics.png?raw=true)

| # | Step | Desc |
|---|---|---|
| 1 | Open Statistics | Click Menu Statistics |
| 2 | Open Conversations| Click Menu Conversations |

## View Captured Conversations

View Captured Conversations

![MyBase64Image](/assets/images/tcpdump-wireshark-win2003/07_TCP_port_Conversation.png?raw=true)

| # | Step | Desc |
|---|---|---|
| 1 | Open TCP Conversation | Click TCP |
| 2 | Source IP & Source Port | Source IP & Source Port is displayed |
| 3 | Dest IP & Dest Port | Dest IP & Source Port is displayed |


# Step to Export to Excel ( CSV File )

## Open dump file in latest wireshark version

Open dump file in latest wireshark version

![MyBase64Image](/assets/images/tcpdump-wireshark-win2003/08_ExportCSV.png.png?raw=true)

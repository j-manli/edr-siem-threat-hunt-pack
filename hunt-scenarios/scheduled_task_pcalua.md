# Scheduled Task Abuse Using pcalua.exe

## Summary

This scenario simulates a scheduled task being created to launch a payload via `pcalua.exe` (Program Compatibility Assistant Launcher), a legitimate Windows utility designed to help run older applications in compatibility mode. Although it serves a legitimate function, `pcalua.exe` can be abused to execute arbitrary programs, making it a commonly overlooked Living-off-the-Land Binary (LOLBIN).

This technique was brought to my attention through a Huntress threat report, which documented real-world abuse of scheduled tasks combined with Microsoft-signed binaries to evade detection. In the cases observed, attackers created tasks with plausible system-like names and used trusted utilities like `pcalua.exe` to launch their payloads.

This simulation mirrors that behavior by creating a scheduled task named `WinUpdateHelper_3541` that executes a benign payload (`calc.exe`) via `pcalua.exe`. The goal is to explore how stealthy scheduled task abuse appears in EDR telemetry, and evaluate how well Microsoft Defender for Endpoint detects and maps this behavior to MITRE ATT&CK.

**Source:**  
[The Hunt for RedCurl – Huntress](https://www.huntress.com/blog/the-hunt-for-redcurl-2)  

---

## Simulation Steps

The following steps were performed on a Windows 10 virtual machine with administrative privileges:

1. Opened an elevated Command Prompt (`Run as Administrator`).

2. Created a scheduled task named `WinUpdateHelper_3541` using `schtasks.exe`:
    
    ```
    schtasks /create /tn "WinUpdateHelper_3541" /tr "C:\Windows\System32\pcalua.exe -a C:\Windows\System32\calc.exe" /sc once /st 23:59
    ```

    >  This command uses the Windows `schtasks.exe` utility to create a scheduled task named `WinUpdateHelper_3541`, configured to run once at 11:59 PM. The `/tr` (task run) argument sets the program to be executed: `pcalua.exe`, which is a native Windows tool used for legacy compatibility. 
    > Here, `pcalua.exe` will (via the `-a` switch) launch `calc.exe` as our payload. In a real-world scenario, this could easily be replaced with a malicious binary or script.

    ![schtask_creation_winupdatehelper](https://github.com/user-attachments/assets/bde270db-0cf7-4f67-8864-d5c6dd40ade8)

3. Verified the task was created:
    
    ```
    schtasks /query /tn "WinUpdateHelper_3541"
    ```
   ![schtask_query_winupdatehelper](https://github.com/user-attachments/assets/b7aeca4f-6b33-4f66-a6b1-c1efcbca31c9)

4. Manually triggered the task to simulate execution without waiting for the scheduled time:
    
    ```
    schtasks /run /tn "WinUpdateHelper_3541"
    ```
    ![schtask_manual_trigger_winupdatehelper](https://github.com/user-attachments/assets/f295f658-da0a-4a30-a715-59a707e0c88b)

---

## Detection Analysis (Microsoft Defender for Endpoint)

The following KQL queries were run in Microsoft Defender for Endpoint's Advanced Hunting portal to identify traces of scheduled task creation and execution.

### 1. Detecting Scheduled Task Creation via schtasks.exe

    DeviceProcessEvents
    | where FileName =~ "schtasks.exe"
    | where ProcessCommandLine has "pcalua.exe"
    | project Timestamp, ActionType, InitiatingProcessFileName, FileName, ProcessCommandLine

This query revealed that the task was created using `schtasks.exe`, with a command line referencing `pcalua.exe`.  
The initiating process was `cmd.exe`, confirming that the action originated from the interactive terminal session.  

![mde_query_schtasks_creation_result](https://github.com/user-attachments/assets/d160c43f-0edc-49eb-b820-70d90d957ff2)


### 2. Detecting Execution of pcalua.exe

    DeviceProcessEvents
    | where FileName =~ "pcalua.exe"
    | project Timestamp, ActionType, InitiatingProcessFileName, FileName, ProcessCommandLine


![mde_query_pcalua_execution_result](https://github.com/user-attachments/assets/dbca8c11-ad73-4156-a7d9-bb99f3db40e2)  

> For those unfamiliar, Task Scheduler runs as a system service under `svchost.exe`, and when a scheduled task executes, it instructs the Task Scheduler service (running under `svchost.exe`), to execute the task, which is why `svchost.exe` appears as the initiating process.


### 3. Defender Alert and MITRE ATT&CK Mapping

Microsoft Defender for Endpoint generated a low-severity alert in response to the scheduled task creation, labeling it as **Masquerade Task or Service**. The alert was mapped to [MITRE ATT&CK technique T1036.004 (Masquerade: Masquerade Task or Service)](https://attack.mitre.org/techniques/T1036/004/), which covers adversary abuse of system-like task names to avoid detection.

*Screenshot: `mde_alert_task_masquerade_pcalua.png`*

This alert demonstrates that even without custom detection logic, MDE was able to surface the behavior—although at a low severity—highlighting it as suspicious based on the task name and use of a LOLBIN as the launcher.




   





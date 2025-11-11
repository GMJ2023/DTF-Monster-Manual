# 🧩 Data Transformation & PDF Processing System 
###   The Monster Manual

Author: Geoffrey Jones

... (previous content unchanged) ...

## Appendix E – Scheduled Tasks & Watchdogs

These Windows Scheduled Tasks act as the persistent background infrastructure for the Data Transformation Framework (DTF).  
They ensure that critical services remain active and responsive without user intervention, and provide recovery instructions if a fault or stall occurs.  

---

### 🕑 Scheduled Task – Monitor Browser & Services  

Purpose:  
Maintains operational awareness of both the Chrome automation browser and core service processes (`file_manager_service.ps1` and Zoho RPA Agent).  
The task ensures that essential DTF components continue running, sending alerts via Zoho Flow if anything stops or freezes.  

Trigger & Configuration:  
- Type: Scheduled Task (Windows)  
- Trigger: Every 1 minute, indefinitely  
- Startup behaviour: Begins automatically at system boot  
- Runs whether user is logged in or not  
- Executes with highest privileges  

Action & Target Script:  
```
powershell.exe -ExecutionPolicy Bypass -File "C:\binary-proxy\monitor_browser.ps1"
```

Behaviour:  
- Checks that Chrome is running with the correct `--remote-debugging-port=9222` flag  
- Confirms that `file_manager_service.ps1` is producing a live heartbeat  
- Verifies Zoho RPA Agent process state  
- Issues a webhook alert if any service stops or freezes  

Recovery Guidance:  
1. Open Task Scheduler  
2. Find FileMonitorService  
3. If it’s already running, select End  
4. Then select Run to restart  
5. If it still fails, check the recent change log for system modifications  

Notes:  
- Heartbeat window: 2 minutes  
- Grace period: 60 seconds at start-up  
- Self-healing logic prevents duplicate notifications  

---

### 🕑 Scheduled Task – Monitor Output Log  

Purpose:  
Reviews recent DTF transformation logs and checks for stalled or empty outputs.  
Designed to detect silent Catalyst failures where the system completes a run but produces a CSV with zero rows.  

Trigger & Configuration:  
- Type: Scheduled Task (Windows)  
- Trigger: Every 15 minutes  
- Startup behaviour: Enabled at system boot  
- Runs silently in background context  
- Highest privileges enabled  

Action & Target Script:  
```
powershell.exe -ExecutionPolicy Bypass -File "C:\binary-proxy\monitor_output_log.ps1"
```

Behaviour:  
- Locates the latest transformation log file  
- Extracts the RowCount metric or similar indicators  
- Sends a webhook if record count equals zero or log inactivity exceeds 30 minutes  

Recovery Guidance:  
- Check mapping templates and input files  
- Review Catalyst console logs  
- Re-run the affected transformation via `run_transform.bat`  

Notes:  
- Maintains cooldown to prevent duplicate alerts  
- Continues operating during RPA restarts  

---

## 🧾 Change Log  

- v10 – Added System Overview Diagram and updated dedication text (22 Oct 2025 – G.J.)  
- v11 – Initial structured draft of the Monster Manual (21 Oct 2025 – G.J.)  
- v12 – Editorial polish and structure refinement (22 Oct 2025 – G.J. & ChatGPT)  
- v15 - End of Master Version — DTF15 (October 2025 – Geoffrey Jones)  
- v16 – Added Appendix E (Scheduled Tasks & Watchdogs) including Monitor Browser & Services and Monitor Output Log definitions (Nov 2025 – G.J.)  

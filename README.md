## Overview

I built a hands-on Azure honeypot to learn the SIEM workflow in a real cloud setting. By spinning up a purposely “wide-open” Windows VM inside its own VNet and resource group, I generate real attack traffic that’s streamed into a Log Analytics workspace. From there, Azure Sentinel ingests the logs, lets me write KQL queries to detect unusual behavior, and enriches events with geo-IP data so I can visualize attacker locations on an interactive world map. This end-to-end setup mirrors how security teams capture, analyze, and respond to threats in production environments.  

![Architecture Diagram](screenshots/Architecture.PNG)  

## Part 1: Resource Group & Virtual Network

Before standing up any resources, I created a dedicated container and network to keep my honeypot isolated:

1. **Resource Group**  
   I made `rg-honeypot-lab-eastus-001` in the **East US** region so all related resources live together and can be torn down easily when I’m done.
   
   ![Create resource group](https://github.com/sandina-w/HoneyMap-Sentinel/blob/5e7a6a7f72dac317962b0bff16ea5271a79f30ab/screenshots/Screenshot%202025-06-21%20162403.png)

3. **Virtual Network**  
   Next I set up `vnet-honeypot-eastus-001` with a `/16` address space. This gives me plenty of room to add subnets later (for example, separate management or logging segments).
   
   ![VNet Basics tab](https://github.com/sandina-w/HoneyMap-Sentinel/blob/5e7a6a7f72dac317962b0bff16ea5271a79f30ab/screenshots/Screenshot%202025-06-21%20162623.png)
   
   ![VNet Basics tab2](https://github.com/sandina-w/HoneyMap-Sentinel/blob/5e7a6a7f72dac317962b0bff16ea5271a79f30ab/screenshots/Screenshot%202025-06-21%20162643.png)

## Part 2: Deploying the Honeypot VM

I chose a Windows 10 Pro image and named the machine **vm-corpfinance-db-eastus-01** to make it look like a high-value “corporate database” server in the **East US** region. This deceptive name, combined with the default size makes it a tempting target for attackers.

![VM Creation Basics](https://github.com/sandina-w/HoneyMap-Sentinel/blob/ee5bc4871f95b3c798891865e7db7bbf501c138b/screenshots/Screenshot%202025-06-21%20164131.png)

## Part 3: Exposing All Inbound Traffic

To turn this VM into a true honeypot, I deliberately opened every port and protocol. In the VM’s Network Security Group (`vm-corpfinance-db-eastus-01-nsg`), I added a catch-all inbound rule named **DANGER_AllowAnyCustomAnyInbound** with:

- **Source:** Any  
- **Destination:** Any  
- **Port ranges:** * (all)  
- **Protocol:** Any  
- **Action:** Allow  
- **Priority:** 100  

This ensures any attacker can hit the VM, generating the noisy traffic I need for my SIEM pipeline.  

![Allow all inbound traffic](https://github.com/sandina-w/HoneyMap-Sentinel/blob/c43ec565188704f87134879f82b07b4f22d587ab/screenshots/Screenshot%202025-06-21%20165126.png)  

## Part 4: Connect to the Honeypot & Disable Host Firewall

Before generating any logs, I verified I could reach and RDP into the VM, then turned off its Windows Defender firewall so all traffic is allowed:

1. **Ping Test**  
   From my local machine, I confirmed the VM’s public IP (20.119.76.112) was reachable:
   
   ![Ping the VM](https://github.com/sandina-w/HoneyMap-Sentinel/blob/64d5b662f94fc001614abfcfbc4e5ad7db4eab2f/screenshots/Screenshot%202025-06-21%20165918.png)

3. **RDP Login**  
   I opened Remote Desktop Connection and signed in with the credentials I set during VM creation:
   
   ![RDP into the VM](https://github.com/sandina-w/HoneyMap-Sentinel/blob/64d5b662f94fc001614abfcfbc4e5ad7db4eab2f/screenshots/Screenshot%202025-06-21%20170201.png)

4. **Disable Windows Firewall**  
   Inside the VM, I launched “Windows Defender Firewall with Advanced Security” and set **Firewall state: Off** under Domain, Private, and Public profiles. This ensures every inbound packet is logged in the next step.
    
   ![Firewall turned off](https://github.com/sandina-w/HoneyMap-Sentinel/blob/64d5b662f94fc001614abfcfbc4e5ad7db4eab2f/screenshots/Screenshot%202025-06-21%20170039.png)

## Part 5: Setting Up Log Analytics & Sentinel

With the honeypot VM logging every packet, it’s time to centralize those logs and hook them into Sentinel:

1. **Create Log Analytics Workspace**  
   I made **law-honeypot-lab-eastus-001** in the same resource group and region; this acts as the landing zone for all security events.
   
   ![Create Log Analytics workspace](https://github.com/sandina-w/HoneyMap-Sentinel/blob/4e15a3c4a9ce04c613532240daf3c3bdce936526/screenshots/Screenshot%202025-06-21%20170321.png)

3. **Onboard to Azure Sentinel**  
   In the Sentinel Content Hub I installed the **Windows Security Events** solution (via AMA) so Sentinel knows how to parse and enrich our VM’s security logs.
   
   ![Sentinel Content Hub – Windows Security Events](https://github.com/sandina-w/HoneyMap-Sentinel/blob/4e15a3c4a9ce04c613532240daf3c3bdce936526/screenshots/Screenshot%202025-06-21%20171003.png)

4. **Create Data Collection Rule**  
   Finally, I added a Data Collection Rule named **DCR-Windows** targeting my honeypot VM. This instructs the new Azure Monitor Agent to forward Event ID 4625 (failed logins) and other security events into our workspace.
   
   ![Create Data Collection Rule](https://github.com/sandina-w/HoneyMap-Sentinel/blob/4e15a3c4a9ce04c613532240daf3c3bdce936526/screenshots/Screenshot%202025-06-21%20171055.png)

## Part 6: Querying Failed-Login Events

With logs flowing in, I ran this KQL query to pull all Event ID 4625 (failed logins):

```kql
SecurityEvent
| where EventID == 4625
| project TimeGenerated, Account, Computer, Activity, IpAddress
```
![Failed-login results (1–6 of 282)](https://github.com/sandina-w/HoneyMap-Sentinel/blob/624d4d8cee7960f84de6da957fff5205098f578b/screenshots/Screenshot%202025-06-22%20010008.png)

You can already see a few failed-login attempts when we first ran the query, After leaving the honeypot exposed for a few hours, the volume of attack traffic spiked dramatically.

## Part 7: Geo-IP Enrichment with a Sentinel Watchlist

To give every failed-login event a location, I imported a public GeoIP CSV into Sentinel as a Watchlist:

1. On the **General** tab, I set:  
   - **Name:** `geoip`  
   - **Alias:** `geoip`
   
   ![Watchlist Wizard – General](https://github.com/sandina-w/HoneyMap-Sentinel/blob/624d4d8cee7960f84de6da957fff5205098f578b/screenshots/Screenshot%202025-06-22%20011544.png)

2. On the **Source** tab, I chose:  
   - **Source type:** Local file  
   - **File type:** CSV with header  
   - **Lines before header:** 0  
   - **Upload:** `geoip-summarized.csv`  
   - **Search key:** `network`
     
   ![Watchlist Wizard – Source](https://github.com/sandina-w/HoneyMap-Sentinel/blob/624d4d8cee7960f84de6da957fff5205098f578b/screenshots/Screenshot%202025-06-22%20011954.png)

3. I clicked **Review + create** → **Create**, and after it finished there were about **92 000** entries in my `geoip` watchlist—ready for KQL joins.
4. 
   ![Watchlist imported with 92K entries](https://github.com/sandina-w/HoneyMap-Sentinel/blob/624d4d8cee7960f84de6da957fff5205098f578b/screenshots/Screenshot%202025-06-22%20014239.png)

## Part 8: Building the Attack-Map Workbook

To bring everything together, I created a Sentinel workbook that plots failed-login attempts on a world map:

1. In Azure Sentinel, I navigated to **Workbooks** → **+ New workbook**.  
2. I deleted the default panels and added a **Query** element.  
3. In the **Advanced Editor**, I pasted my JSON definition, which:
   - Loads the `geoip` watchlist  
   - Queries `SecurityEvent` for EventID 4625  
   - Joins on the `network` field to enrich each IP with latitude/longitude  
   - Summarizes by country and aggregates failure counts  
   - Renders a `map` visualization with bubble sizes proportional to the number of failed logins  

Below shows the JSON in the Advanced Editor:

![Workbook JSON in Advanced Editor](https://github.com/sandina-w/HoneyMap-Sentinel/blob/58e51381ecff15f53d3063346788cc749f641b6f/screenshots/Screenshot%202025-06-22%20014610.png)

After saving and exiting edit mode, the map comes to life, each bubble shows where attacks are coming from, sized by volume:

![Attack Map Workbook](https://github.com/sandina-w/HoneyMap-Sentinel/blob/58e51381ecff15f53d3063346788cc749f641b6f/screenshots/Screenshot%202025-06-22%20134745.png)

Below the map is a summary row that lists the top regions:

| Location                        | Failures |
|---------------------------------|---------:|
| Ranchos (Argentina)             | 66.9 K   |
| Tilburg (Netherlands)           | 33.4 K   |
| Jordanow (Poland)               | 33.4 K   |
| Vadodara (India)                | 1.67 K   |
| Paulista (Brazil)               | 1.65 K   |
| Crateus (Brazil)                | 1.65 K   |
| Other                           | 160      |
| Los Angeles (United States)     | 144      |
| Polokwane (South Africa)        | 144      |

---

Finally, I re-ran the same KQL in Log Analytics to validate the volume over a longer window. You can see the total failed-login count surged to **68,759** entries:  

![Final failed-login results (1–13 of 68,759)](https://github.com/sandina-w/HoneyMap-Sentinel/blob/58e51381ecff15f53d3063346788cc749f641b6f/screenshots/Screenshot%202025-06-22%20134550.png)

## Conclusion

This lab gave me hands-on experience with the full Azure SIEM workflow: deploying a decoy VM, opening it to real-world scanners, centralizing logs in Log Analytics, enriching with GeoIP data via a Sentinel watchlist, and visualizing attacker activity on an interactive map. In a few hours it attracted over **68,759** failed-login attempts from around the globe, demonstrating how quickly exposed assets are probed and why end-to-end monitoring is critical.

##  References

- Lab walkthrough video: https://www.youtube.com/watch?v=g5JL2RIbThM&t=2322s
- Detailed lab notes: https://docs.google.com/document/d/143seB9PwT9GSsStc14vPQWgnCHQeVMVEC6XBRz67p_Q/edit?tab=t.0







   
   



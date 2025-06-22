## Overview

I built a hands-on Azure honeypot to learn the SIEM workflow in a real cloud setting. By spinning up a purposely “wide-open” Windows VM inside its own VNet and resource group, I generate real attack traffic that’s streamed into a Log Analytics workspace. From there, Azure Sentinel ingests the logs, lets me write KQL queries to detect unusual behavior, and enriches events with geo-IP data so I can visualize attacker locations on an interactive world map. This end-to-end setup mirrors how security teams capture, analyze, and respond to threats in production environments.  

![Architecture Diagram](screenshots/Architecture.PNG)  

## Resource Group & Virtual Network

Before standing up any resources, I created a dedicated container and network to keep my honeypot isolated:

1. **Resource Group**  
   I made `rg-honeypot-lab-eastus-001` in the **East US** region so all related resources live together and can be torn down easily when I’m done.  
   ![Create resource group](https://github.com/sandina-w/HoneyMap-Sentinel/blob/5e7a6a7f72dac317962b0bff16ea5271a79f30ab/screenshots/Screenshot%202025-06-21%20162403.png)

2. **Virtual Network**  
   Next I set up `vnet-honeypot-eastus-001` with a `/16` address space. This gives me plenty of room to add subnets later (for example, separate management or logging segments).
   ![VNet Basics tab](https://github.com/sandina-w/HoneyMap-Sentinel/blob/5e7a6a7f72dac317962b0bff16ea5271a79f30ab/screenshots/Screenshot%202025-06-21%20162623.png)
   ![VNet Basics tab2](https://github.com/sandina-w/HoneyMap-Sentinel/blob/5e7a6a7f72dac317962b0bff16ea5271a79f30ab/screenshots/Screenshot%202025-06-21%20162643.png)
   
   



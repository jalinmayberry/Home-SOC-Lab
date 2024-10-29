# Cribl-Splunk-Home-SOC

![cribl](https://github.com/user-attachments/assets/04984f1c-2060-4838-9b8c-bc255e90a1ee)

![splunk_rainbow](https://github.com/user-attachments/assets/5e0aca6b-8dcd-44db-a68d-41dfae774487)

## INTRO

It has been a year since I last posted a blog. I spent a lot of time delving deeper into Splunk and Cribl as I made my footing on our SIEM team at CDW. Allow me to set the stage for the tools we will be using in this write-up along with a brief intro for what they do!

Let’s start with the hardware at the focal point of my home SOC.

- **Ubiquiti UniFi Dream Machine Pro**
    - Every home lab needs a foundation, starting with the network to support it. However, you are free to use a firewall of your choice.
- **2 x Raspberry Pi 5’s**
    - I’ll note that I purchased 2 intending to run home automation daemons on one and additional test nodes/labs on the other…that was until I started playing with **Docker (more on that later)**.
- **Windows 10/11 Machine**
    - You can be a bit modular here too. However, the example drawn here will be utilizing a Windows 10 machine. I’d recommend it to get some experience with the procedure since you might often see a mixture of Windows & NIX environments in the real world.
    - I’m also utilizing my Windows 10 machine to run Splunk on Docker (there it is again!)

Now let’s overview the techstacks we will be using today to create our home SOC.

- **Docker**
    - What is Docker and why this instead of a VM?
        - Docker is a platform as a service product (PaaS) that uses OS-level virtualization to deliver software in packages called ‘**containers’**
        - They have a few advantages over using VMs (which are in no way a bad way to deploy aspects of this environment too)
            - Scalability: they are lightweight and and can be deployed very quickly
            - Efficient: they are less resource-intensive more resource-efficient
- **Splunk**
    - The SIEM of choice for the backbone of this lab but if you believe you will ingest more than 500 MB of data, you can elect to use an open-source alternative like Wazuh.
- **Cribl**
    - The star of this deployment. It has a lot of really useful capabilities:
        - Scalable with vertical and horizontal distribution in mind. Manage a plethora of nodes to handle data collection, optimization, and forwarding all from a single pane of glass.
        - Data agnostic which means you can collect and send your data to any suitable destination (AWS S3, Azure BLOB, Splunk, etc)
        - Data optimization utilizing unique pipelines thus reducing/filtering excess log content (overall reduction in ingest cost)
        - Data obfuscation by filtering the data before it ever touches destination systems (filtering early with an emphasis on early)
        - *Will replace the need for Splunk forwarder agents (more on that later)*

**Here is a Visual of the Environment**

![image](https://github.com/user-attachments/assets/48631073-b2d5-4760-b631-949ad95f24a3)

Since Cribl is the main focus of the lab, we will dive deeper into just what this diagram is speaking.

Cribl has a few main system separate functions but we will focus on the 2 relevant to the deployment:

- Cribl Stream
    - Cribl Stream can help collect, reduce, enrich, transform, and route data from Cribl Edge to any destination.
- Cribl Edge
    - Similar to Splunk Universal Forwarders, these agents are lightweight data collectors with a twist. Edge nodes have Stream functionality that allows you to search and capture data at rest, well…at the Edge.

We will be collecting syslog data from 3 sources and Windows Event logs from another. The syslog data will be collected from what are called worker nodes. You can think of worker nodes as frontline doers. The Windows machine requires a different collection method (as worker nodes are not supported on Windows) via an Edge node. We will then send this data using routes, which tell individual streams of data where to go (destinations) and whether or not they will be processed beforehand using packs, which are a bundle of pipelines that filter, enrich, and or normalize the data (i.e turning XML into JSON or correcting timestamps).

# INSTALLATIONS

## Initial Setup

1. Start by getting your Raspberry Pi’s situated. I’d recommend utilizing [Raspberry Pi Imager](https://www.raspberrypi.com/software/) 
    
    ![RPOS_Imaging](https://github.com/user-attachments/assets/4c6d194b-1855-406d-b4bd-91ae363c793e)

    1. I’d recommend giving them distinct hostnames for fewer headaches later
    2. Be sure to install OpenSSH so you can access the command line via tools like PuTTY or MobaXterm
2. Update and Install Packages 
    1. apt-get update
3. Install [Docker](https://docs.docker.com/engine/install/debian/)
4. Install [Portainer](https://docs.portainer.io/start/install-ce/server/docker/linux) 
    1. It is not required but I highly recommend utilizing Portainer for an added graphical layer to managing your containers
5. (Repeat for the other Raspberry Pi if utilizing a second)
6. Important Miscellaneous 
    1. We need a ‘cribl’ user with chown and sudo permissions
        1. NEVER MAKE INSTALL OR MAKE CHANGES TO CRIBL AS THE ROOT USER
    2. Create a Cribl user
        1. **`sudo adduser cribl`** 
    3. Give Cribl user sudo permissions
        1. **`usermod -aG sudo cribl`**
    4. Verify your architecture (Rasperry Pi’s are ARM based)
        1.   `uname -m`   
            
              ![image 1](https://github.com/user-attachments/assets/bab36f88-7a0d-4ce6-9d58-93d474333b0a)

        2. aarch64 = ARM
7. Navigate to **cd /opt (we will be installing our Cribl Leader instance here)**
8. Visit https://cribl.io/download/ (We are downloading the Edge and Stream suite)
    
   ![image 2](https://github.com/user-attachments/assets/15f40cff-123d-4903-b916-aa625c5886ad)

    1. curl -Lso - $(curl https://cdn.cribl.io/dl/latest-arm64) | tar zxv
9. Give our Cribl user chown privileges 
    1. `chown -R cribl:cribl /opt/cribl`
10. Configure systemd to manage the Cribl service
    1. `/opt/cribl/bin/cribl boot-start enable -u cribl`
12. Start and verify that the Cribl service is running 
    1.     systemctl start cribl   
    2.     systemctl status -l cribl    
13. Verify that Cribl is listening on port 9000
    1.     ss -tulpn    
14. Navigate to your Cribl Leader @ **http//:<leader_ip>:9000** 
    1. default user = admin
    2. default password = admin
    3. add an email to the license agreement and accept the terms

## Installing a Worker Node

We will use the worker node as the focal point of collecting our source logs. Ideally, the worker node would be on another host so we will use the other Raspberry Pi we setup earlier. 

1. Within Stream, navigate to the Workers tab
    
    ![image 3](https://github.com/user-attachments/assets/aecedd82-d3fa-4aac-b99c-c595ad7569c0)
    
2. Click on “Add/Update Worker Node”
    
    ![image 4](https://github.com/user-attachments/assets/10e5605e-64ec-4d60-8e61-f1b2b269899f)

3. I utilized Docker to deploy my worker node so feel free to follow along since we installed docker earlier. The bootstrap script will populate to the right of your configuration tabs.
    1. An aspect of this script will need some further editing. I found that by default, all ports are closed unless specified during the run script.
        1. -p 9000:9000 opens port 9000 for both the container and the host which is used for Cribl UI and bootstrapping worker nodes from an on-prem leader node.
        2. More context on ports can be found [here](https://docs.cribl.io/stream/ports/)
        
        ![image 5](https://github.com/user-attachments/assets/aef6b48f-14e2-4a10-9546-1c648e5dfd81)

    2. you can open the rest of the necessary ports via additional -p <port>:<port> args in your script, or you can use my Portainer method. 
4. After you’ve deployed the script on your Raspberry Pi, you should see your worker node populate the list 

![image 6](https://github.com/user-attachments/assets/464797ee-e024-4167-9b3f-827da95ea8c8)

1. Now, before we do anything further, hop over to Portainer on the machine you deployed your worker node via https://<host_ip>:9443 
    1. don’t worry about the certificate error (if you see one) for now.
    
   ![image 7](https://github.com/user-attachments/assets/075b9fe0-35db-4daa-aa90-96aca77bcbde)

2. Portainer is the graphical method of managing your docker containers and what we can do here is open additional ports pretty quickly and redeploy the worker image.
    1. navigate to “cribl-worker” or whatever you named your container in the original bootstrap script.
    2. click on “Duplicate/Edit” 
        1. Here we will see port mapping options. Syslog standardizes on 514 for conventional systems but I have elected to use non-standard/non-privileged ports. If your source supports it, I’d recommend opening a custom port to avoid interfering with other sources and reduce the need for pre-processing data blobs. 
    3. click on “Map additional port” and toggle TCP or UDP for each one. TCP is more reliable while UDP is faster.
        1. *14 = Syslog
        2. 8088 = Splunk HTTP Event Collector (HEC) 
        3. 9000 = Cribl UI & bootstrapping worker nodes
        4. 10300 = Cribl TCP which is toggled by default in the Cribl TCP destination configuration window but you can choose another port as well. 
    4. click on “Deploy the container” to redeploy the worker node.

![image 8](https://github.com/user-attachments/assets/a1b51ecd-38d1-492b-abfd-64836162922b)

Now you’re ready to install Cribl Edge!

## Installing Cribl Edge on a Windows 10 Machine

A prerequisite we can start with is setting up ICMP communication between your Raspberry Pi and the Windows Machine

1. On your Windows Machine, navigate to “Windows Defender Firewall with Advanced Security”
2. Inbound Rules > New Rule
    
    ![image 9](https://github.com/user-attachments/assets/24be5602-8e9a-4ad2-940b-b1d3b83a6ac7)

3. Select “Custom”
    
    ![image 10](https://github.com/user-attachments/assets/05dafbfe-fa72-4323-a441-310e95b87c01)
    
4. Select “All programs”
5. Select ICMPv4
6. on the Scope step
    1. you can leave the local IP address to “Any IP address”
    2. set remote IP address to “These addresses” and input the address of your Raspberry Pi
7. Allow the connection
8. Set Profile to “Private” only
    
    ![image 11](https://github.com/user-attachments/assets/ee070d6d-b61d-49a9-a821-cc1b9e767f58)
    
9. Give it a name and click “Finish”

This ensures your Windows machine can accept inbound pings from the Raspberry Pi.

1. Head back to your Cribl UI and navigate to Cribl Edge > Fleets 

![image 12](https://github.com/user-attachments/assets/453cddab-6aec-46f7-8ef5-b6e1045dcb11)

1. Add/Update Edge Node > Windows > Update 
    
![image 13](https://github.com/user-attachments/assets/5a4d274a-d4cf-40cb-85c3-4d3c907a4c0f)

2. Open Windows PowerShell as administrator and copy the bootstrap script into PowerShell
    1. You can also use the [install wizard](https://docs.cribl.io/edge/deploy-windows/)
3. You should see a “Cribl” folder populate in C:\Program Files

![image 14](https://github.com/user-attachments/assets/54c5892a-17b5-4d1a-8758-3c04269ed5ba)

1. You can then navigate back to the Cribl UI to view the node in your fleet

 ![image 15](https://github.com/user-attachments/assets/7411d374-93c6-4310-9165-441538bd5b55)

---

# CONFIGURATIONS

## Configuring the Raspberry Pi’s to Forward Syslog

Now that our worker node is prepped to capture syslog data, we must forward it from our Pi servers.

1. Navigate to your Leader Node machine’s command line
2. We will use rsyslog to forward our syslog data to the worker node machine.
    1. **Install `rsyslog` (if not already installed)**:
        
        ```bash
        
        sudo apt update
        sudo apt install rsyslog -y
        
        ```
        
    2. **Edit the `rsyslog` configuration file**:
    Open the main `rsyslog` configuration file with a text editor.
        
        ```bash
        
        sudo nano /etc/rsyslog.conf
        
        ```
        
    3. **Enable TCP or UDP Syslog Forwarding**:
    Uncomment (remove `#`) the following lines to enable TCP or UDP syslog reception. Use the appropriate protocol depending on your setup.
        - For UDP:
            
            ```
            
            module(load="imudp")
            input(type="imudp" port="<port for syslog")
            
            ```
            
        - For TCP:
            
            ```
            
            module(load="imtcp")
            input(type="imtcp" port="<port for syslog>")
            
            ```
            
    4. **Set Up the Forwarding Rule**:
    At the end of the configuration file, add a rule to forward all logs (`.*`) to your Cribl worker node IP address and port (change `<worker-node-ip>` and `<port>` to match your setup):
        
        ```
        
        *.* action(type="omfwd" target="<dest_ip>" port="10514" protocol="tcp") #For TCP
        *.* action(type="omfwd" target="<dest_ip>" port="10514" protocol="udp") #For UDP
        ```
        
    5. **Save and Exit**:
    Save your changes and exit the editor. In Nano, press `Ctrl+O` to save, and `Ctrl+X` to exit.
    6. **Restart `rsyslog` to Apply Changes**:
        
        ```bash
        
        sudo systemctl restart rsyslog
        ```
        
    7. **Verify Configuration**:
    Check the `rsyslog` status to ensure there are no errors:
        
        ```bash
        
        sudo systemctl status rsyslog
        ```
        
    8. **Testing the Setup** (Optional):
    Use the `logger` command to generate a test log message:
        
        ```bash
        
        logger "This is a test log from Raspberry Pi."
        ```
        
    9. Verify that the log is received on the Cribl worker node.
        
        ```bash
        
        sudo tail /var/log/syslog
        ```
        
    10. You should now see the message populate the log file!
        
   ![image 16](https://github.com/user-attachments/assets/7af56f8e-c72e-48b5-a26a-85847f83c71b)
     
    
## Configuring Cribl to Capture Syslog

Now that the Pi server is forwarding logs to our destination (where the Cribl worker is hosted), we will configure Cribl to capture these logs.

1. Log into your Cribl UI
2. Navigate to “Worker Groups > Data > Sources > Syslog

![image 17](https://github.com/user-attachments/assets/66a4aafc-23d7-47e4-b4cc-b8bb2830b0c4)

1. Click Add Source 
    1. give your source an “Input ID” like ‘NodeLab_Syslog’
    2. Leave “Address” set to ‘0.0.0.0’ 
    3. set your port in TCP or UDP to the port you configured in rsyslog.conf
    4. navigate to “Fields” 
        1. name = index
        2. value = ‘home_soc_syslog’ or a value of your choice
            1. we will create this index in Splunk later
        
      ![image 18](https://github.com/user-attachments/assets/78e44ba9-5761-4536-98ee-fb01c22c1f2b)

2. Navigate to “Connected Destinations” 
    1. I prefer “Send to Routes” so I will present this method for our routing examples.
3. Click Save 
4. We will also need to Commit & Deploy these changes 
    1. It is best practice to write a quick, but detailed description of any changes you commit
    2. You will need to do these each time you make source, destination, etc changes

![image 19](https://github.com/user-attachments/assets/6e638e95-105d-4190-92c7-58950dc63a82)

7. Click “Commit & Deploy”

![image 20](https://github.com/user-attachments/assets/ccf55f45-a6d5-4754-bfcd-29e497e91333)

1. Navigate to Routing > Data Routes
2. Click “Add Route” 
3. Configure the following 

![image 21](https://github.com/user-attachments/assets/15476cc2-f8a4-4b97-abcb-7ab985d9e287)

1.  Make sure the default route is moved underneath our new test route and save the changes
    1. Commit & Deploy
2. Click the “…” on our new route and select “Capture”

![image 22](https://github.com/user-attachments/assets/e9b1cd06-2589-4717-9e89-f0dc483063ef)

1. Make sure the following filter expression is present

![image 23](https://github.com/user-attachments/assets/8570d76e-3ea6-42c7-aa7a-97a1a743a14c)

1. Click “Capture” and use the following settings before clicking “Start”
    1. this is where we will test that our Syslog data is being captured by Cribl
        
![image 24](https://github.com/user-attachments/assets/c1616faa-4ecc-4ab5-bedf-082e3179472a)

2. Once you start the capture, go back to your Pi server command line and emit another logger query 
    1. logger "This is a test log from Raspberry Pi."
    2. this will populate the live capture with your test log!

![image 25](https://github.com/user-attachments/assets/37af438e-1b24-43f5-b95f-9fd1c4c14f3e)

Wonderful! We have one more data collection setup task to complete which is configuring Cribl Edge to collect Windows Event logs on our Windows machine. 

## Configuring Cribl Edge to Collect Windows Events

Cribl Edge, as we overviewed earlier, behaves much like a forwarder. We will be setting up a Windows Event Collector (WEC) to forward event logs from Edge to Stream. 

1. Navigate to Cribl Edge > Fleets > More > Sources
2. Select Windows Event Logs 
    
![image 26](https://github.com/user-attachments/assets/23314a5d-7b0f-429e-939c-d3edfe937ca4)

3. There will be an Input ID “in_win_event_logs” that we need to enable

![image 27](https://github.com/user-attachments/assets/bcd619c9-8955-4870-9aef-ad63c7e2209c)

1. Click on the input and match the configurations
    1. We will be monitoring System, Security, and Application logs

![image 28](https://github.com/user-attachments/assets/d5a56cd5-c938-4638-8758-233938ccde72)

1. Navigate to Advanced Settings and ensure “Use Windows Tools” is toggled

![image 29](https://github.com/user-attachments/assets/42181a24-5397-42e9-9f9e-b0872e37a38a)

1. Connected Destinations > Send to Routes 
2. Save/Commit & Deploy
3. Navigate back to the input
    1. Under the “Status” tab, you should see your Windows host and event logs populated this table

![image 30](https://github.com/user-attachments/assets/ba6dbe00-0bb6-44de-a51a-527a3c354b45)

Now we will set up Cribl TCP as a destination which will allow us to forward these logs to Cribl Stream

1. Navigate to More > Destinations
    
    ![image 31](https://github.com/user-attachments/assets/b2fd3a04-d145-431e-baf6-e2b9d0acbcea)

2. Select Cribl TCP
    
    ![image 32](https://github.com/user-attachments/assets/d4ac29c1-a7c9-41e7-a88b-c34515bf4618)

3. Copy these configurations 
    1. Persistent Queue is used to minimize data loss in case of receiver disturbance 
    2. We set up Docker to listen on port 10300 earlier so use it or another port if you configured otherwise

![image 33](https://github.com/user-attachments/assets/ae6eae58-6206-4574-a9d0-ae98842707e5)

![image 34](https://github.com/user-attachments/assets/c17a38fa-65f9-4bd7-8db2-a3215da2dd87)

1. Save/Commit & Deploy

Now we will navigate back to Stream to set up the Cribl TCP input.

1. Products > Stream > Worker Groups > Data > Sources
2. Scroll down to find “Cribl TCP”
    
    ![image 35](https://github.com/user-attachments/assets/8632e5ac-7b5c-4b74-a4c2-1221dfeb8172)

3. Add Source (if there isn’t already an Input ID of “in_cribl_tcp” present)
    1. copy the following configuration settings
    2. Leave address set to 0.0.0.0
    3. set port to 10300
    4. Connected Destinations > Send to Routes 
    
    ![image 36](https://github.com/user-attachments/assets/09af2060-0ae6-42a5-8119-0d17d84ade8d)

4. Save/Commit & Deploy

View the “Status” and “Live Data” tabs of the source and you should see event logs populating.

![image 37](https://github.com/user-attachments/assets/1e82aec0-b2b7-46e7-8280-143f66e59aa2)

You have officially deployed the start of your very own SOC homelab!

# CONCLUSION

## Summary of Cribl Setup

In this write-up, we've covered the essential steps to set up a robust data collection and processing pipeline using Cribl Stream and Edge. We've successfully:

- Configured Raspberry Pi's as Cribl Leader and Worker nodes
- Set up Cribl Edge on a Windows machine for collecting Windows Event logs
- Configured syslog forwarding from various sources to Cribl
- Established data routes and verified log capture in Cribl

## The Power of Cribl

Through this setup, we've demonstrated Cribl's capabilities in:

- Centralizing data collection from diverse sources
- Providing flexibility in data routing and processing
- Offering scalability through its distributed architecture

## Looking Ahead: Part 2 - Cribl Destinations

In the next part of this series, we'll explore setting up crucial destinations for our collected logs:

### 1. AWS S3 Bucket

We'll cover:

- Creating and configuring an S3 bucket for log storage
- Setting up IAM roles and policies for secure access
- Configuring Cribl to send data to S3 for long-term retention

### 2. Splunk HTTP Event Collector (HEC)

We'll explore:

- Setting up Splunk HEC for real-time log ingestion
- Configuring Cribl to forward processed logs to Splunk
- Optimizing data flow between Cribl and Splunk

By implementing these destinations, we'll complete our data pipeline, enabling both long-term storage in S3 and real-time analysis in Splunk. This setup will provide a comprehensive solution for log management, combining the flexibility of Cribl with the power of cloud storage and advanced analytics.

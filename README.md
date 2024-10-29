# Cribl-Splunk-Home-SOC

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

![image.png](https://prod-files-secure.s3.us-west-2.amazonaws.com/c50b480f-5501-4712-810c-1953ba81d366/9a296126-d907-465e-993c-639548dc45af/image.png)

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
    
    ![RPOS Imaging.jpg](https://prod-files-secure.s3.us-west-2.amazonaws.com/c50b480f-5501-4712-810c-1953ba81d366/d773c365-b3f8-4c2b-90b8-a3d89fc606c6/RPOS_Imaging.jpg)
    
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
            
            ![image.png](https://prod-files-secure.s3.us-west-2.amazonaws.com/c50b480f-5501-4712-810c-1953ba81d366/0e5c95a4-97b0-43f2-9775-6df948be948a/image.png)
            
        2. aarch64 = ARM
7. Navigate to **cd /opt (we will be installing our Cribl Leader instance here)**
8. Visit https://cribl.io/download/ (We are downloading the Edge and Stream suite)
    
    ![image.png](https://prod-files-secure.s3.us-west-2.amazonaws.com/c50b480f-5501-4712-810c-1953ba81d366/0f0b7026-0deb-42db-ba1f-d59207840641/image.png)
    
    1. curl -Lso - $(curl https://cdn.cribl.io/dl/latest-arm64) | tar zxv
9. Give our Cribl user chown privileges 
    1. **`chown -R cribl:cribl /opt/cribl`**
10. Configure systemd to manage the Cribl service
    1. **`/opt/cribl/bin/cribl boot-start enable -u cribl`**
11. Start and verify that the Cribl service is running 
    1.     systemctl start cribl   
    2.     systemctl status -l cribl    
12. Verify that Cribl is listening on port 9000
    1.     ss -tulpn    
13. Navigate to your Cribl Leader @ **http//:<leader_ip>:9000** 
    1. default user = admin
    2. default password = admin
    3. add an email to the license agreement and accept the terms

## Installing a Worker Node

We will use the worker node as the focal point of collecting our source logs. Ideally, the worker node would be on another host so we will use the other Raspberry Pi we setup earlier. 

1. Within Stream, navigate to the Workers tab
    
    ![image.png](https://prod-files-secure.s3.us-west-2.amazonaws.com/c50b480f-5501-4712-810c-1953ba81d366/a0f451ca-0e87-4b26-850c-87974ff46887/image.png)
    
2. Click on “Add/Update Worker Node”
    
    ![image.png](https://prod-files-secure.s3.us-west-2.amazonaws.com/c50b480f-5501-4712-810c-1953ba81d366/87b1910d-07be-464d-9573-a9f9c7d79f5a/image.png)
    
3. I utilized Docker to deploy my worker node so feel free to follow along since we installed docker earlier. The bootstrap script will populate to the right of your configuration tabs.
    1. An aspect of this script will need some further editing. I found that by default, all ports are closed unless specified during the run script.
        1. -p 9000:9000 opens port 9000 for both the container and the host which is used for Cribl UI and bootstrapping worker nodes from an on-prem leader node.
        2. More context on ports can be found [here](https://docs.cribl.io/stream/ports/)
        
        ![image.png](https://prod-files-secure.s3.us-west-2.amazonaws.com/c50b480f-5501-4712-810c-1953ba81d366/4f5218a3-5c90-48a2-8dab-b4517d1f9227/image.png)
        
    2. you can open the rest of the necessary ports via additional -p <port>:<port> args in your script, or you can use my Portainer method. 
4. After you’ve deployed the script on your Raspberry Pi, you should see your worker node populate the list 

![image.png](https://prod-files-secure.s3.us-west-2.amazonaws.com/c50b480f-5501-4712-810c-1953ba81d366/a78ee992-c982-4765-9406-09d07cc75733/image.png)

1. Now, before we do anything further, hop over to Portainer on the machine you deployed your worker node via https://<host_ip>:9443 
    1. don’t worry about the certificate error (if you see one) for now.
    
    ![image.png](https://prod-files-secure.s3.us-west-2.amazonaws.com/c50b480f-5501-4712-810c-1953ba81d366/c1952abd-7a21-407f-9628-abad6e28b530/image.png)
    
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

![image.png](https://prod-files-secure.s3.us-west-2.amazonaws.com/c50b480f-5501-4712-810c-1953ba81d366/5d7ff455-2263-454e-a9d5-157eedf06012/image.png)

Now you’re ready to install Cribl Edge!

## Installing Cribl Edge on a Windows 10 Machine

A prerequisite we can start with is setting up ICMP communication between your Raspberry Pi and the Windows Machine

1. On your Windows Machine, navigate to “Windows Defender Firewall with Advanced Security”
2. Inbound Rules > New Rule
    
    ![image.png](https://prod-files-secure.s3.us-west-2.amazonaws.com/c50b480f-5501-4712-810c-1953ba81d366/6f65ab23-db97-431f-ae4d-e55e8868c929/image.png)
    
3. Select “Custom”
    
    ![image.png](https://prod-files-secure.s3.us-west-2.amazonaws.com/c50b480f-5501-4712-810c-1953ba81d366/7b96987c-80bf-4bbc-aa36-2141ab23fc47/image.png)
    
4. Select “All programs”
5. Select ICMPv4
6. on the Scope step
    1. you can leave the local IP address to “Any IP address”
    2. set remote IP address to “These addresses” and input the address of your Raspberry Pi
7. Allow the connection
8. Set Profile to “Private” only
    
    ![image.png](https://prod-files-secure.s3.us-west-2.amazonaws.com/c50b480f-5501-4712-810c-1953ba81d366/29288128-ec2d-4e5f-9a65-e61e84132991/image.png)
    
9. Give it a name and click “Finish”

This ensures your Windows machine can accept inbound pings from the Raspberry Pi.

1. Head back to your Cribl UI and navigate to Cribl Edge > Fleets 

![image.png](https://prod-files-secure.s3.us-west-2.amazonaws.com/c50b480f-5501-4712-810c-1953ba81d366/58839fe6-6d82-400c-a9b6-67af76799c16/image.png)

1. Add/Update Edge Node > Windows > Update 
    
    ![image.png](https://prod-files-secure.s3.us-west-2.amazonaws.com/c50b480f-5501-4712-810c-1953ba81d366/8a440542-c8ae-4cef-ae61-eb1a2359eb4e/image.png)
    
2. Open Windows PowerShell as administrator and copy the bootstrap script into PowerShell
    1. You can also use the [install wizard](https://docs.cribl.io/edge/deploy-windows/)
3. You should see a “Cribl” folder populate in C:\Program Files

![image.png](https://prod-files-secure.s3.us-west-2.amazonaws.com/c50b480f-5501-4712-810c-1953ba81d366/f218e324-d574-4bdb-b3f6-83b0141bc967/image.png)

1. You can then navigate back to the Cribl UI to view the node in your fleet

![image.png](https://prod-files-secure.s3.us-west-2.amazonaws.com/c50b480f-5501-4712-810c-1953ba81d366/ccf70583-e47f-460a-85e9-0baa34c8d45c/image.png)

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
        
        ![image.png](https://prod-files-secure.s3.us-west-2.amazonaws.com/c50b480f-5501-4712-810c-1953ba81d366/d114ede4-2bd5-4c3b-88b1-e82722cc35f2/image.png)
        
    

## Configuring Cribl to Capture Syslog

Now that the Pi server is forwarding logs to our destination (where the Cribl worker is hosted), we will configure Cribl to capture these logs.

1. Log into your Cribl UI
2. Navigate to “Worker Groups > Data > Sources > Syslog

![image.png](https://prod-files-secure.s3.us-west-2.amazonaws.com/c50b480f-5501-4712-810c-1953ba81d366/65dab2f0-156a-4a30-8cc7-2574639d61a0/image.png)

1. Click Add Source 
    1. give your source an “Input ID” like ‘NodeLab_Syslog’
    2. Leave “Address” set to ‘0.0.0.0’ 
    3. set your port in TCP or UDP to the port you configured in rsyslog.conf
    4. navigate to “Fields” 
        1. name = index
        2. value = ‘home_soc_syslog’ or a value of your choice
            1. we will create this index in Splunk later
        
        ![image.png](https://prod-files-secure.s3.us-west-2.amazonaws.com/c50b480f-5501-4712-810c-1953ba81d366/88e2421e-5eaa-4007-8659-fd078b41e40a/image.png)
        
2. Navigate to “Connected Destinations” 
    1. I prefer “Send to Routes” so I will present this method for our routing examples.
3. Click Save 
4. We will also need to Commit & Deploy these changes 
    1. It is best practice to write a quick, but detailed description of any changes you commit
    2. You will need to do these each time you make source, destination, etc changes

![image.png](https://prod-files-secure.s3.us-west-2.amazonaws.com/c50b480f-5501-4712-810c-1953ba81d366/91a7c312-5616-4cf0-bfcc-4e54cd968f7a/image.png)

7. Click “Commit & Deploy”

![image.png](https://prod-files-secure.s3.us-west-2.amazonaws.com/c50b480f-5501-4712-810c-1953ba81d366/760f9000-8116-43a2-a7c6-1639df1cdeb4/image.png)

1. Navigate to Routing > Data Routes
2. Click “Add Route” 
3. Configure the following 

![image.png](https://prod-files-secure.s3.us-west-2.amazonaws.com/c50b480f-5501-4712-810c-1953ba81d366/789d0c94-c58a-4b3d-97a6-5776efa86d93/image.png)

1.  Make sure the default route is moved underneath our new test route and save the changes
    1. Commit & Deploy
2. Click the “…” on our new route and select “Capture”

![image.png](https://prod-files-secure.s3.us-west-2.amazonaws.com/c50b480f-5501-4712-810c-1953ba81d366/efb98d30-eeb9-45f0-a5ea-749f838e5c1a/image.png)

1. Make sure the following filter expression is present

![image.png](https://prod-files-secure.s3.us-west-2.amazonaws.com/c50b480f-5501-4712-810c-1953ba81d366/5c44b7fc-17f4-4dbf-8349-11ec28343fdd/image.png)

1. Click “Capture” and use the following settings before clicking “Start”
    1. this is where we will test that our Syslog data is being captured by Cribl
        
        ![image.png](https://prod-files-secure.s3.us-west-2.amazonaws.com/c50b480f-5501-4712-810c-1953ba81d366/e811c3fb-c31b-4631-96f0-18830783640f/image.png)
        
2. Once you start the capture, go back to your Pi server command line and emit another logger query 
    1. logger "This is a test log from Raspberry Pi."
    2. this will populate the live capture with your test log!

![image.png](https://prod-files-secure.s3.us-west-2.amazonaws.com/c50b480f-5501-4712-810c-1953ba81d366/1e07a8c6-750e-48f2-be04-4f8558a167ac/image.png)

Wonderful! We have one more data collection setup task to complete which is configuring Cribl Edge to collect Windows Event logs on our Windows machine. 

## Configuring Cribl Edge to Collect Windows Events

Cribl Edge, as we overviewed earlier, behaves much like a forwarder. We will be setting up a Windows Event Collector (WEC) to forward event logs from Edge to Stream. 

1. Navigate to Cribl Edge > Fleets > More > Sources
2. Select Windows Event Logs 
    
    ![image.png](https://prod-files-secure.s3.us-west-2.amazonaws.com/c50b480f-5501-4712-810c-1953ba81d366/940871fd-ecff-423f-a4c4-602e87bb2a01/image.png)
    
3. There will be an Input ID “in_win_event_logs” that we need to enable

![image.png](https://prod-files-secure.s3.us-west-2.amazonaws.com/c50b480f-5501-4712-810c-1953ba81d366/40a1b3ff-f1e6-4d8c-bcf2-32195213488c/image.png)

1. Click on the input and match the configurations
    1. We will be monitoring System, Security, and Application logs

![image.png](https://prod-files-secure.s3.us-west-2.amazonaws.com/c50b480f-5501-4712-810c-1953ba81d366/409d958d-5bed-438e-929c-110cfd66ed20/image.png)

1. Navigate to Advanced Settings and ensure “Use Windows Tools” is toggled

![image.png](https://prod-files-secure.s3.us-west-2.amazonaws.com/c50b480f-5501-4712-810c-1953ba81d366/e4f130dd-415c-4ed5-86e8-2de660a4627e/image.png)

1. Connected Destinations > Send to Routes 
2. Save/Commit & Deploy
3. Navigate back to the input
    1. Under the “Status” tab, you should see your Windows host and event logs populated this table

![image.png](https://prod-files-secure.s3.us-west-2.amazonaws.com/c50b480f-5501-4712-810c-1953ba81d366/f711b92f-fc42-415f-987e-dd9d6d86b56b/image.png)

Now we will set up Cribl TCP as a destination which will allow us to forward these logs to Cribl Stream

1. Navigate to More > Destinations
    
    ![image.png](https://prod-files-secure.s3.us-west-2.amazonaws.com/c50b480f-5501-4712-810c-1953ba81d366/65771442-73c2-4cbf-8365-2d4929a1c8e3/image.png)
    
2. Select Cribl TCP
    
    ![image.png](https://prod-files-secure.s3.us-west-2.amazonaws.com/c50b480f-5501-4712-810c-1953ba81d366/2a533336-ca27-4116-b818-17b085f5540d/image.png)
    
3. Copy these configurations 
    1. Persistent Queue is used to minimize data loss in case of receiver disturbance 
    2. We set up Docker to listen on port 10300 earlier so use it or another port if you configured otherwise

![image.png](https://prod-files-secure.s3.us-west-2.amazonaws.com/c50b480f-5501-4712-810c-1953ba81d366/63119191-7f1b-472c-acb6-6e98a0a31e17/image.png)

![image.png](https://prod-files-secure.s3.us-west-2.amazonaws.com/c50b480f-5501-4712-810c-1953ba81d366/c5f9d33b-fba0-425b-96ca-a49dc0b256ba/image.png)

1. Save/Commit & Deploy

Now we will navigate back to Stream to set up the Cribl TCP input.

1. Products > Stream > Worker Groups > Data > Sources
2. Scroll down to find “Cribl TCP”
    
    ![image.png](https://prod-files-secure.s3.us-west-2.amazonaws.com/c50b480f-5501-4712-810c-1953ba81d366/8f069a66-f027-4e45-8ea5-3a7ed9b335e8/image.png)
    
3. Add Source (if there isn’t already an Input ID of “in_cribl_tcp” present)
    1. copy the following configuration settings
    2. Leave address set to 0.0.0.0
    3. set port to 10300
    4. Connected Destinations > Send to Routes 
    
    ![image.png](https://prod-files-secure.s3.us-west-2.amazonaws.com/c50b480f-5501-4712-810c-1953ba81d366/462abeb1-cb72-4313-8d33-be5380982a7d/image.png)
    
4. Save/Commit & Deploy

View the “Status” and “Live Data” tabs of the source and you should see event logs populating.

![image.png](https://prod-files-secure.s3.us-west-2.amazonaws.com/c50b480f-5501-4712-810c-1953ba81d366/8242ca9f-f618-40e7-b4a2-307348e12ff4/image.png)

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

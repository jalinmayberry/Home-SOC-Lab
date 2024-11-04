# Cribl-Splunk-Home-SOC

![cribl](https://github.com/user-attachments/assets/04984f1c-2060-4838-9b8c-bc255e90a1ee)

![splunk_rainbow](https://github.com/user-attachments/assets/5e0aca6b-8dcd-44db-a68d-41dfae774487)

## INTRO

It has been a year since my last blog post. During this time, I focused on deepening my understanding of Splunk and Cribl as I established my role within the SIEM team at CDW. In this write-up, I will provide an overview of the tools we will be using, along with a brief introduction to their functions.

Let’s start with the hardware at the focal point of my home SOC.

- **Ubiquiti UniFi Dream Machine Pro**: Every home lab needs a solid foundation, starting with the network to support it. However, you are free to use a firewall of your choice.

- **2 x Raspberry Pi 5s**: I purchased two of these with the intention of running home automation daemons on one and additional test nodes/labs on the other. That was until I started experimenting with Docker *(more on that later)*.

- **Windows 10/11 Machine**: You can be a bit flexible here, but the example I’ll use will be based on a Windows 10 machine. I recommend this setup for gaining experience with procedures, as you'll often encounter a mix of Windows and UNIX environments in the real world. I am also using my Windows 10 machine to run Splunk on Docker.

Now let’s review the tech stack we will be using today to create our home SOC.

- **Docker**: What is Docker, and why choose it over a virtual machine (VM)?
  - Docker is a platform as a service (PaaS) that uses OS-level virtualization to deliver software packaged in units called **containers**.
  - Containers have several advantages over VMs (though VMs can also effectively serve parts of this environment):
    - **Scalability**: Containers are lightweight and can be deployed very quickly.
    - **Efficiency**: They are less resource-intensive and more efficient.
   
- **Splunk**: This is the SIEM of choice for the backbone of this lab. If you anticipate ingesting more than 500 MB of data, you might consider using an open-source alternative like Wazuh.

- **Cribl**: The star of this deployment, Cribl offers numerous useful capabilities:
  - It is designed for scalability, supporting both vertical and horizontal distribution. Manage multiple nodes to handle data collection, optimization, and forwarding—all from a single interface.
  - It is data agnostic, allowing you to collect and send data to various destinations (e.g., AWS S3, Azure BLOB, Splunk).
  - It provides data optimization through unique pipelines, which reduces and filters excess log content, leading to lower ingestion costs.
  - It facilitates data obfuscation by filtering data before it reaches destination systems (early filtering is emphasized).
  - **It will replace the need for Splunk forwarder agents** *(more on that later)*.


**Here is a Visual of the Environment**

![image](https://github.com/user-attachments/assets/565139f7-43d0-4f75-a0e8-1c2fa7a0b310)

Since Cribl is the main focus of the lab, we will take a closer look at what this diagram illustrates.

Cribl has several key system functions, but we will focus on two relevant to the deployment:

- **Cribl Stream:** This component helps collect, reduce, enrich, transform, and route data from Cribl Edge to any destination.
  
- **Cribl Edge:** Similar to Splunk Universal Forwarders, these agents are lightweight data collectors with an added twist. Edge nodes have Stream functionality that allows you to search and capture data at rest, effectively operating "at the Edge."

We will collect syslog data from three sources and Windows Event logs from another source. The syslog data will be gathered from what are known as Worker nodes, which can be thought of as the frontline doers. For the Windows machine, however, we will use a different collection method, as Worker nodes are not supported on Windows. Instead, we will utilize an Edge node.

After collecting the data, we will send it using routes, which direct individual streams of data to their respective destinations. Additionally, we can determine whether the data will be processed beforehand using packs, which are bundles of pipelines designed to filter, enrich, or normalize the data (for example, converting XML into JSON or correcting timestamps).

# INSTALLATIONS

## Initial Setup

1. Begin by setting up your Raspberry Pi devices. I recommend using the [Raspberry Pi Imager](https://www.raspberrypi.com/software/).

    ![RPOS_Imaging](https://github.com/user-attachments/assets/4c6d194b-1855-406d-b4bd-91ae363c793e)

   - Assign distinct hostnames to each device to avoid confusion later. For this demonstration, I have named one of the hostnames "NodeLab."
   - Make sure to install OpenSSH so you can access the command line with tools like PuTTY or MobaXterm.

2. Update and Install Packages:
   - Run the following command to update your package list:
     ```bash
     sudo apt-get update
     ```

3. Install [Docker](https://docs.docker.com/engine/install/debian/).

4. Install [Portainer](https://docs.portainer.io/start/install-ce/server/docker/linux):
   - While this step is not mandatory, I highly recommend using Portainer for an enhanced graphical interface to manage your containers.

5. (Repeat these steps for any additional Raspberry Pi devices you may be using.)

6. **Important Miscellaneous Steps:**

   1. Create a `cribl` user with `chown` and `sudo` permissions.
      - **NOTE: NEVER RUN CRIBL AS THE ROOT USER.**
   
   2. Create the Cribl user:
      ```bash
      sudo adduser cribl
      ```

   3. Grant the Cribl user sudo permissions:
      ```bash
      sudo usermod -aG sudo cribl
      ```

   4. Verify your architecture (Raspberry Pis are ARM-based):
      ```bash
      uname -m
      ```
      ![image 1](https://github.com/user-attachments/assets/50dfcdf1-730e-464d-a30f-009c3f2697da)

7. Navigate to the installation directory:
   ```bash
   cd /opt  # We will install our Cribl Leader instance here.
   ```

8. Visit https://cribl.io/download/ to download the Edge and Stream suite.

   ![image 2](https://github.com/user-attachments/assets/15f40cff-123d-4903-b916-aa625c5886ad)

   1. **Download and extract Cribl for ARM64 (as confirmed earlier):**
      ```bash
      curl -Lso - $(curl https://cdn.cribl.io/dl/latest-arm64) | tar zxv
      ```

9. **Assign ownership of the Cribl directory to the Cribl user:**
   ```bash
   sudo chown -R cribl:cribl /opt/cribl
   ```

10. **Configure systemd to manage the Cribl service:**
    ```bash
    /opt/cribl/bin/cribl boot-start enable -u cribl
    ```

11. **Start and verify that the Cribl service is running:**
    - To start the service:
      ```bash
      sudo systemctl start cribl
      ```
    - To check the service status:
      ```bash
      sudo systemctl status -l cribl
      ```

12. **Verify that Cribl is listening on port 9000:**
    ```bash
    ss -tulpn
    ```

13. Navigate to your Cribl Leader at **http://<leader_ip>:9000**:
    1. Default username: `admin`
    2. Default password: `admin`
    3. Add your email address to the license agreement and accept the terms.


## Installing a Worker Node

The Worker node will be the central point for collecting our source logs. Ideally, it should be on a separate host, so we will use another Raspberry Pi that we set up earlier.

1. In Stream, navigate to the **Workers** tab.
    
    ![image 3](https://github.com/user-attachments/assets/aecedd82-d3fa-4aac-b99c-c595ad7569c0)
    
2. Click on **Add/Update Worker Node**.
    
    ![image 4](https://github.com/user-attachments/assets/8681d395-3b11-4d9e-93a9-b0ed47b0e6e5)

3. I deployed my Worker node using Docker, so feel free to follow along since we installed Docker earlier. The bootstrap script will populate to the right of your configuration tabs.
   
    1. This script requires some editing. By default, all ports are closed unless specified during the run script.
        - Use `-p 9000:9000` to open port 9000 for both the container and the host. This port is used for the Cribl UI and for bootstrapping Worker nodes from an on-prem Leader node.
        - More context on ports can be found [here](https://docs.cribl.io/stream/ports/).
        
        ![image 5](https://github.com/user-attachments/assets/a28bdaec-aa68-40f9-9379-c4fa7b898bd1)

    2. You can open additional necessary ports by adding more `-p <port:port>` arguments in your script, or you can follow my Portainer method. 

4. After deploying the script on your Raspberry Pi, you should see your Worker node appear in the list.

    ![image 6](https://github.com/user-attachments/assets/96944189-c474-45cc-9334-04a2912f70e0)

5. Before proceeding further, go to Portainer on the machine where you deployed your Worker node by visiting https://<host_ip>:9443.
   
    1. Don't worry about the certificate error (if you see one) for now.
    
    ![image 7](https://github.com/user-attachments/assets/3a09179b-ab18-40f0-b325-da5dce2a1466)

6. Portainer is a graphical interface for managing your Docker containers, allowing you to quickly open additional ports and redeploy the Worker image.
   
    1. Navigate to **cribl-worker** (or whatever you named your container in the original bootstrap script).
    2. Click on **Duplicate/Edit**. 
        - This will show you the port mapping options. Syslog standardizes on port 514 for conventional systems, but I recommend using non-standard, non-privileged ports. If your source supports it, opt for a custom port to avoid interfering with other sources and reduce the need for pre-processing data blobs. 
    3. Click on **Map additional port** and toggle between TCP or UDP for each one. TCP is more reliable, while UDP is faster.
        - **14** = Syslog
        - **8088** = Splunk HTTP Event Collector (HEC) 
        - **9000** = Cribl UI & bootstrapping Worker nodes
        - **10300** = Cribl TCP (typically toggled by default in the Cribl TCP destination configuration window, but you can choose another port as well). 
    4. Click on **Deploy the container** to redeploy the Worker node.

    ![image 8](https://github.com/user-attachments/assets/a1b51ecd-38d1-492b-abfd-64836162922b)

You are now ready to install Cribl Edge!

## Installing Cribl Edge on a Windows 10 Machine

First, set up ICMP communication between your Raspberry Pi and the Windows machine:

1. On your Windows machine, open **Windows Defender Firewall with Advanced Security**.
2. Go to **Inbound Rules > New Rule**.
   
    ![image 9](https://github.com/user-attachments/assets/24be5602-8e9a-4ad2-940b-b1d3b83a6ac7)

3. Select **Custom**.
    
    ![image 10](https://github.com/user-attachments/assets/05dafbfe-fa72-4323-a441-310e95b87c01)
    
4. Choose **All programs**.
5. Select **ICMPv4**.
6. In the **Scope** step:
    1. Leave the local IP address as **Any IP address**.
    2. Set the remote IP address to **These addresses** and input the address of your Raspberry Pi.
7. Allow the connection.
8. Set the Profile to **Private** only.
    
    ![image 11](https://github.com/user-attachments/assets/ee070d6d-b61d-49a9-a821-cc1b9e767f58)
    
9. Give the rule a name and click **Finish**.

This ensures that your Windows machine can accept inbound pings from the Raspberry Pi.

1. Return to your Cribl UI and navigate to **Cribl Edge > Fleets**.

    ![image 12](https://github.com/user-attachments/assets/453cddab-6aec-46f7-8ef5-b6e1045dcb11)

2. Choose **Add/Update Edge Node > Windows > Add**.
    
    ![image 13](https://github.com/user-attachments/assets/9865e316-0044-4e12-b54c-59de1f6659f1)
    
3. Open **Windows PowerShell** as an administrator and paste the bootstrap script into PowerShell. You can also use the [install wizard](https://docs.cribl.io/edge/deploy-windows/).
   
4. You should see a **Cribl** folder populate in **C:\Program Files**.

    ![image 14](https://github.com/user-attachments/assets/54c5892a-17b5-4d1a-8758-3c04269ed5ba)

5. Finally, navigate back to the Cribl UI to view the node in your fleet.

    ![image 15](https://github.com/user-attachments/assets/19a63f2a-4a8c-4b57-a91c-ff6f23a786f8)

# CONFIGURATIONS

## Configuring Raspberry Pi to Forward Syslog

Now that our Worker node is set up to capture syslog data, we need to forward it from our Raspberry Pi servers.

1. **Access the Command Line on Your Leader Node Machine**
   
2. **Use `rsyslog` to Forward Syslog Data to the Worker Node**:
   
   a. **Install `rsyslog` (if not already installed)**:
   
   ```bash
   sudo apt update
   sudo apt install rsyslog -y
   ```
   
   b. **Edit the `rsyslog` Configuration File**:
   Open the main `rsyslog` configuration file using a text editor:
   
   ```bash
   sudo nano /etc/rsyslog.conf
   ```
   
   c. **Enable TCP or UDP Syslog Forwarding**:
   Depending on your setup, uncomment (remove `#`) the appropriate lines for TCP or UDP syslog reception:
   
   - For UDP:
   ```
   module(load="imudp")
   input(type="imudp" port="<udp_port_for_syslog>")
   ```
   
   - For TCP:
   ```
   module(load="imtcp")
   input(type="imtcp" port="<tcp_port_for_syslog>")
   ```
   
   d. **Set Up the Forwarding Rule**:
   At the end of the configuration file, add a rule to forward all logs (`.*`) to your Cribl Worker node's IP address and port (replace `<Worker-node-ip>` and `<port>` with appropriate values):
   
   ```
   *.* action(type="omfwd" target="<Worker-node-ip>" port="<port>" protocol="tcp") # For TCP
   *.* action(type="omfwd" target="<Worker-node-ip>" port="<port>" protocol="udp") # For UDP
   ```
   
   e. **Save and Exit**:
   Save your changes and exit the editor. In Nano, press `Ctrl+O` to save and `Ctrl+X` to exit.
   
   f. **Restart `rsyslog` to Apply Changes**:
   
   ```bash
   sudo systemctl restart rsyslog
   ```
   
   g. **Verify Configuration**:
   Check the status of `rsyslog` to ensure there are no errors:
   
   ```bash
   sudo systemctl status rsyslog
   ```
   
   h. **Testing the Setup (Optional)**:
   Use the `logger` command to generate a test log message:
   
   ```bash
   logger "This is a test log from Raspberry Pi."
   ```
   
   i. **Verify the Log is Received**:
   Check the log on the Cribl Worker node to ensure the log message has been received:
   
   ```bash
   sudo tail /var/log/syslog
   ```
   
   j. You should see the test message appear in the log file!

![image 16](https://github.com/user-attachments/assets/7af56f8e-c72e-48b5-a26a-85847f83c71b)
     
    
## Configuring Cribl to Capture Syslog

Now that the Pi server is forwarding logs to our destination (where the Cribl Worker is hosted), we will configure Cribl to capture these logs.

1. Log into your Cribl UI.
2. Navigate to **Worker Groups > Data > Sources > Syslog**.

   ![image 17](https://github.com/user-attachments/assets/66a4aafc-23d7-47e4-b4cc-b8bb2830b0c4)

3. Click **Add Source**.
   - Enter an “Input ID” such as ‘NodeLab_Syslog’.
   - Keep the “Address” set to ‘0.0.0.0’.
   - Set your port in TCP or UDP to the port specified in `rsyslog.conf`.
4. Navigate to **Fields**:
   - Name: `index`
   - Value: ‘home_soc_syslog’ or another value of your choice. This index will be created in Splunk later.

   ![image 18](https://github.com/user-attachments/assets/78e44ba9-5761-4536-98ee-fb01c22c1f2b)

5. Navigate to **Connected Destinations**:
   - I prefer using **Send to Routes**, so I will demonstrate this method for our routing examples.
6. Click **Save**.
7. We also need to **Commit & Deploy** these changes:
   - It’s best practice to write a brief yet detailed description of any changes you commit.
   - You will need to do this each time you make changes to sources, destinations, etc.

   ![image 19](https://github.com/user-attachments/assets/6e638e95-105d-4190-92c7-58950dc63a82)

8. Click **Commit & Deploy**.

   ![image 20](https://github.com/user-attachments/assets/ccf55f45-a6d5-4754-bfcd-29e497e91333)

9. Navigate to **Routing > Data Routes**.
10. Click **Add Route**.
11. Configure the following:

    ![image 21](https://github.com/user-attachments/assets/15476cc2-f8a4-4b97-abcb-7ab985d9e287)

12. Ensure the default route is moved underneath our new test route and save the changes:
    - Commit & Deploy.
13. Click the “…” on our new route and select **Capture**.

    ![image 22](https://github.com/user-attachments/assets/e9b1cd06-2589-4717-9e89-f0dc483063ef)

14. Ensure the following filter expression is present:

    ![image 23](https://github.com/user-attachments/assets/8570d76e-3ea6-42c7-aa7a-97a1a743a14c)

15. Click **Capture** and use the following settings before clicking **Start**:
    - This is where we will test that our Syslog data is being captured by Cribl.

    ![image 24](https://github.com/user-attachments/assets/c1616faa-4ecc-4ab5-bedf-082e3179472a)

16. Once the capture starts, return to your Pi server's command line and issue another logger command:
    - `logger "This is a test log from Raspberry Pi."`
    - This will populate the live capture with your test log!

    ![image 25](https://github.com/user-attachments/assets/37af438e-1b24-43f5-b95f-9fd1c4c14f3e)

Wonderful! We have one more data collection setup task to complete: configuring Cribl Edge to collect Windows Event logs on our Windows machine.

## Configuring Cribl Edge to Collect Windows Events

As we discussed earlier, Cribl Edge functions similarly to a forwarder. We will set up a Windows Event Collector (WEC) to forward event logs from Edge to Stream.

1. Navigate to **Cribl Edge > Fleets > More > Sources**.
2. Select **Windows Event Logs**.

   ![image 26](https://github.com/user-attachments/assets/70b3d151-d35b-4d43-9379-03b98433007f)

3. Enable the Input ID “in_win_event_logs”.

   ![image 27](https://github.com/user-attachments/assets/bcd619c9-8955-4870-9aef-ad63c7e2209c)

4. Click on the input and adjust the configurations:
   - We will monitor System, Security, and Application logs.

   ![image 28](https://github.com/user-attachments/assets/d5a56cd5-c938-4638-8758-233938ccde72)

5. In the **Fields** tab, set an index value of 'home_soc_winevents' 
6. Navigate to **Advanced Settings** and verify that “Use Windows Tools” is toggled on.
7. Under **Connected Destinations**, choose **Send to Routes**.
8. Save and Commit & Deploy.
9. Return to the input:
   - Under the **Status** tab, you should see your Windows host and event logs listed in this table.

   ![image 30](https://github.com/user-attachments/assets/ba6dbe00-0bb6-44de-a51a-527a3c354b45)

Now, we will set up Cribl TCP as a destination, allowing us to forward these logs to Cribl Stream.

1. Navigate to **More > Destinations**.

   ![image 31](https://github.com/user-attachments/assets/b2fd3a04-d145-431e-baf6-e2b9d0acbcea)

2. Select **Cribl TCP**.

   ![image 32](https://github.com/user-attachments/assets/e7eb829a-9e52-4abb-847a-49551f6fdde4)

3. Copy these configurations:
   - A **Persistent Queue** is used to minimize data loss in case of receiver disturbances.
   - We set up Docker to listen on port 10300 earlier, so use it or another port if configured otherwise.

   ![image 33](https://github.com/user-attachments/assets/ae6eae58-6206-4574-a9d0-ae98842707e5)

   ![image 34](https://github.com/user-attachments/assets/c17a38fa-65f9-4bd7-8db2-a3215da2dd87)

4. Save and Commit & Deploy.

Next, we'll navigate back to Stream to set up the Cribl TCP input.

1. Go to **Products > Stream > Worker Groups > Data > Sources**.
2. Scroll down to find **Cribl TCP**.

   ![image 35](https://github.com/user-attachments/assets/8632e5ac-7b5c-4b74-a4c2-1221dfeb8172)

3. **Add Source** (if there isn’t already an Input ID of “in_cribl_tcp” present)  
   1. Copy the following configuration settings.  
   2. Leave the address set to 0.0.0.0.  
   3. Set the port to 10300.  
   4. Under Connected Destinations, select "Send to Routes."  
   
   ![image 36](https://github.com/user-attachments/assets/09af2060-0ae6-42a5-8119-0d17d84ade8d)  

4. **Save/Commit & Deploy**  

Check the “Status” and “Live Data” tabs of the source; you should see event logs populating.  

![image 37](https://github.com/user-attachments/assets/177728a8-6ee6-4619-bb6b-bffb9cfdb8c9)  

Congratulations! You have officially deployed the beginning of your very own SOC homelab!

# CONCLUSION

## Summary of Cribl Setup

In this write-up, we've covered the essential steps to set up a robust data collection and processing pipeline using Cribl Stream and Edge. We have successfully:

- Configured Raspberry Pis as Cribl Leader and Worker nodes.
- Set up Cribl Edge on a Windows machine to collect Windows Event logs.
- Configured syslog forwarding from various sources to Cribl.
- Established data routes and verified log capture in Cribl.

## The Power of Cribl

Through this setup, we've demonstrated Cribl's capabilities in:

- Centralizing data collection from diverse sources.
- Providing flexibility in data routing and processing.
- Offering scalability through its distributed architecture.

## Looking Ahead: Part 2 - Cribl Destinations

In the next part of this series, we will explore setting up crucial destinations for our collected logs:

### 1. AWS S3 Bucket

We'll cover:

- Creating and configuring an S3 bucket for log storage.
- Setting up IAM roles and policies for secure access.
- Configuring Cribl to send data to S3 for long-term retention.

### 2. Splunk HTTP Event Collector (HEC)

We'll explore:

- Setting up Splunk HEC for real-time log ingestion.
- Configuring Cribl to forward processed logs to Splunk.
- Optimizing data flow between Cribl and Splunk.

By implementing these destinations, we will complete our data pipeline, enabling both long-term storage in S3 and real-time analysis in Splunk. This setup will provide a comprehensive solution for log management, combining the flexibility of Cribl with the power of cloud storage and advanced analytics.

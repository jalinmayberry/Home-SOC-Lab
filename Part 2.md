# Cribl-Splunk-Home-SOC

![image](https://github.com/user-attachments/assets/54c4659a-6754-43f9-9e36-e5b956db6685)
![image](https://github.com/user-attachments/assets/cdb44734-c055-4ed8-9d49-9d4979f2ea72)

Part 2 - Configuring Destinations

## INTRO

Welcome to Part 2 of the Home SOC Lab! In this section, we’re setting up critical destinations for our data pipeline. This will ensure that the logs collected in Part 1 are both stored long-term and available for real-time analysis.

We’ll be working with two powerful tools for our destinations:

- **AWS S3**: Provides long-term, secure storage.
- **Splunk HEC**: Enables real-time data analysis.

Let’s dive in and set these up!

---

## 1. Setting up AWS IAM Roles and S3 for Log Storage

### Setting Up an AWS Root User Account

The AWS root user account has full access to all resources and services within AWS. It's crucial to secure this account since it has elevated privileges. This guide will also highlight how to utilize AWS Free Tier to manage initial costs. Here’s a step-by-step guide to help set up and secure your AWS root user account:

### Step 1: Sign Up for an AWS Account (With Free Tier Access)

1. Go to [https://aws.amazon.com](https://aws.amazon.com/) and click **Create an AWS Account**.
2. Enter your email address, set a strong password, and choose a unique account name.
3. **Enable Free Tier Access**:
    - When signing up, AWS will automatically enroll your account in the Free Tier, which provides free usage for many popular AWS services, up to certain limits.
    - For more information on Free Tier limits, see [AWS Free Tier Details](https://aws.amazon.com/free/).
4. Complete the billing information and identity verification steps as required by AWS.

*Note*: The Free Tier includes 12 months of free access to select services, like 750 hours of Amazon EC2 (t2.micro) instances, 5GB of Amazon S3 storage, and more. Always monitor your usage to avoid unexpected charges.

### Step 2: Secure the Root User Account

1. **Enable Multi-Factor Authentication (MFA)**:
    - Go to **IAM Console** > **Dashboard** > **Activate MFA on your root account**.
    - Follow the steps to configure MFA using a virtual MFA device, hardware MFA device, or U2F security key.
    - MFA adds an extra layer of security, making it harder for unauthorized access.
2. **Set Up a Strong Password Policy** (Optional):
    - For additional security, enforce strong passwords for IAM users by configuring a password policy under **IAM Console** > **Account settings** > **Password Policy**.

![image](https://github.com/user-attachments/assets/6157cda2-43c5-4f1c-a5f4-bea314f53654)

### Step 3: Create an Admin IAM User for Day-to-Day Activities

Using the root user account for daily tasks is not recommended. Instead, create an IAM user with administrative privileges.

1. Go to **IAM Console** > **Users** > **Add User**.
2. Enter a username (e.g., "Admin") and select **AWS Management Console Access**.
3. Set permissions by attaching the **AdministratorAccess** policy.
    
    ![image 2](https://github.com/user-attachments/assets/8941d5b7-44b7-4e66-b43f-039164ba6e75)
    
4. Follow the steps to set a password

![image 2](https://github.com/user-attachments/assets/59ace59d-e1a8-497e-80e1-af93acbb2676)

![image 3](https://github.com/user-attachments/assets/9fe77387-539a-480e-98a1-092d0fd1b818)

1. And finally, set up MFA
    1. Verify that you are able to login properly as the Admin user we created via the console.
    
   ![e3e925c0-2cfc-47ae-b0ad-2317a62594ba](https://github.com/user-attachments/assets/9e7a8976-5663-40e7-87eb-87cf29572929)

    b. Navigate to IAM > Dashboard > Add MFA
    
    ![image 4](https://github.com/user-attachments/assets/1879cff0-ac28-4fb1-9ed9-94905c2dc1ee)

    c. Utilize authentication methods such as a YubiKey or an authenticator app
    
    ![image 5](https://github.com/user-attachments/assets/2284c0b4-7890-482f-ac47-13b6d30d769e)

    d. You should see no further security recommendations 
    
    ![image 6](https://github.com/user-attachments/assets/a285eee9-514a-4de0-9814-2ce11781ed6f)


### Step 4: Store Root User Credentials Securely

- Do not share the root user credentials.
- Consider using a password manager to securely store credentials if needed.
- Only use the root account for account and billing management tasks that require root-level permissions.

### Step 5: Monitor Free Tier Usage

- Use the **Billing and Cost Management Console** to track Free Tier usage and set up alerts if you're nearing the free limits.
- Go to **Billing Dashboard** > **Free Tier Usage Alerts** to set up notifications for usage beyond Free Tier limits, helping prevent accidental costs.

### Why AWS S3?

AWS S3 (Simple Storage Service) is a reliable and cost-effective way to store logs and other data long-term. By setting up S3, we’re creating a safe space where logs can be retained, analyzed later, or used in historical analyses.

## 1.1 Configure the S3 Bucket

Next, we’ll create an S3 bucket. Think of the bucket as a container for all your log data. Setting up access policies and security here is crucial since this is where all logs will reside.

- **Region**: Select a region close to your deployment to optimize performance.
    
    ![image 7](https://github.com/user-attachments/assets/cbf8a1a1-768a-4872-a97f-c4ecc7433612)

    
- Go to **S3 Console** > **Create bucket**.
    
    ![image 8](https://github.com/user-attachments/assets/41963c7c-074f-408b-8a8b-b1351df19f2c)


- **Bucket Name**: Choose a unique name

![image 9](https://github.com/user-attachments/assets/da814524-1024-48d1-b8db-5d644b560737)

### **1. Region**

- **Description**: AWS Region where your S3 bucket will be created.
- **Best Practice**: Choose a region that is geographically closest to you or your SOC deployment to reduce latency. If your primary operations are in the US, select **US East (N. Virginia)** or another nearby region. For compliance or data residency, ensure the data remains in the desired jurisdiction.

**Example**: `us-east-1`, `us-west-2`, `eu-west-1`.

### **2. Bucket Name**

- **Description**: The bucket name must be globally unique across all AWS accounts.
- **Best Practice**: Use a descriptive and meaningful name that reflects the purpose of the bucket, such as `home-soc-logs` or `syslog-monitoring-data`. Avoid special characters, uppercase letters, and keep it concise.

### 3. **Object Ownership**

- **Description**: Determines who owns the objects uploaded to the bucket. You can enforce that all objects uploaded are owned by the bucket owner.
- **Best Practice**: **Enable "Bucket owner enforced"**. This ensures that any data uploaded, regardless of who uploads it, is owned by the bucket owner, which can help with access control and security.

### 4. **Block Public Access Settings**

- **Description**: Controls whether public access to the bucket is allowed.
- **Best Practice**: **Block all public access**. Since this bucket is for your SOC and contains sensitive logs and monitoring data, it should not be accessible to the public. Ensure that the option to block all public access is enabled to prevent accidental exposure of data.

**Settings to Enable**:

- Block public access to ACLs
- Block public access to bucket policies
- Block public and cross-account access (recommended)

### 5. **Bucket Versioning**

- **Description**: Enables you to keep multiple versions of objects in the same bucket.
- **Best Practice**: **Enable versioning**. This will allow you to keep track of every version of your logs and monitoring data, ensuring that data is not accidentally overwritten or deleted. This can also help in disaster recovery scenarios.
- **Cost Consideration**: While versioning can increase storage costs, it’s a useful safety net for important log data.

### 6. **Tags**

- **Description**: Metadata that helps categorize and track the cost of resources.
- **Best Practice**: Add relevant tags to help organize and manage your resources. Tags could include `Project`, `Environment`, `Owner`, and `Purpose`.

**Example**:

- Key: `Project`, Value: `HomeSOC`
- Key: `Environment`, Value: `Production`
- Key: `Purpose`, Value: `LogStorage`

### 7. **Default Encryption**

- **Description**: Encrypts the data stored in the bucket to protect it from unauthorized access.
- **Best Practice**: **Enable server-side encryption (SSE)**. You can use either:
    - **SSE-S3 (AES-256)**: Managed by AWS, this is the simplest option and provides a good level of security.
    - **SSE-KMS**: Offers additional control over the encryption keys using AWS Key Management Service (KMS), but may introduce additional complexity.

**For Home SOC**: SSE-S3 (AES-256) should suffice for general log data, but you can opt for KMS if you want more control over encryption.

### 8. **Object Lock (Optional)**

- **Description**: Allows you to store objects using a write-once-read-many (WORM) model.
- **Best Practice**: If you require immutability for your log data, such as for audit compliance or to prevent tampering, enable Object Lock. It can be used to protect data against accidental deletion or modifications.

**Use Cases**: Critical logs or regulatory requirements may necessitate enabling Object Lock with either Governance or Compliance mode.

- Once complete, select ‘Create bucket’

![image 10](https://github.com/user-attachments/assets/6173862f-8352-43ac-85a1-8498ba969830)

## 1.2 Create an IAM Policy for the Cribl User

### Step 1: Create the IAM Policy in AWS

1. Go to **IAM** (Identity and Access Management).
2. In the left-hand sidebar, click on **Policies**.
3. Click **Create policy**.

### Step 2: Define the Policy Permissions

1. In the **Create Policy** window, go to the **JSON** tab to define the custom permissions.
2. Paste the following JSON code into the editor. Be sure to replace `"YOUR_S3_BUCKET_NAME"` with the actual name of the S3 bucket that Cribl will interact with.
    
    ```json
    
    {
      "Version": "2012-10-17",
      "Statement": [
        {
          "Effect": "Allow",
          "Action": [
            "s3:PutObject",
            "s3:GetObject",
            "s3:ListBucket"
          ],
          "Resource": [
            "arn:aws:s3:::YOUR_S3_BUCKET_NAME",
            "arn:aws:s3:::YOUR_S3_BUCKET_NAME/*"
          ]
        }
      ]
    }
    
    ```
    
    - **`s3:PutObject`**: Grants permission to write data to the S3 bucket.
    - **`s3:GetObject`**: Grants permission to read data from the S3 bucket.
    - **`s3:ListBucket`**: Allows listing the objects within the S3 bucket.
3. Click **Next: Tags** (adding tags is optional).
4. Click **Next: Review**.

### Step 3: Name and Create the Policy

1. Provide a **Name** for the policy, such as `CriblS3AccessPolicy`.
2. Review the policy summary to ensure it has the correct permissions.
3. Click **Create policy** to save it.

![image 11](https://github.com/user-attachments/assets/1c766950-8011-409e-8b06-50ece268ee11)

## 1.3 Create an AWS IAM User for Cribl

Let’s start by creating an IAM (Identity and Access Management) user. This user will have specific permissions to access the S3 bucket we’re about to create. Limiting access to only what’s necessary (known as the "Principle of Least Privilege") ensures better security for our setup.

- Go to **AWS IAM Console** > **Users** > **Create user**.

![image 12](https://github.com/user-attachments/assets/3bb3be93-e754-4dc0-8bd0-cf84a01d103e)

- **Username**: Enter a descriptive name, like `Cribl`.

![image 13](https://github.com/user-attachments/assets/fda3eeed-76d4-49f6-8c41-c7cf152ecb0e)

- **Permissions**: Attach the policy we created earlier directly

![image 14](https://github.com/user-attachments/assets/24aa2076-a07d-495b-8e36-8d445c297627)

![image 15](https://github.com/user-attachments/assets/55b5e968-c370-45bd-b361-340bdff875f2)

- Review and Create: Consider adding tags to assist with identifying and organizing resources, then select ‘Create user’

![image 16](https://github.com/user-attachments/assets/df7a5c84-4087-4851-a1a2-7fd8a84a98c8)

- Navigate to our new user

![image 17](https://github.com/user-attachments/assets/4827b90b-8239-45eb-9963-6ade0c48c3b2)

- Generate an access key; We will use this to give Cribl authentication access via our destination setup later.
    
    ![image 18](https://github.com/user-attachments/assets/c60b161d-e41f-4dfd-9566-fb78ea506cbe)


- Select ‘Application running outside AWS’

![image 19](https://github.com/user-attachments/assets/9f63e29c-d00e-4b80-adf9-c10b82655245)

- Set a description tag for organizational best practice. Then select ‘Create access key’
    - Be sure to keep the key components handy, as we will use them to set up our Cribl → S3 destination

![image 20](https://github.com/user-attachments/assets/04509645-a5b6-4d72-a94b-30f718e7127d)

## 1.4 Configure Cribl for S3 Destination

Now that we have an S3 bucket, let’s make sure Cribl can send logs directly to it. We’ll configure Cribl to authenticate with the IAM user credentials, enabling seamless log storage in S3.

- On our Cribl Leader instance (make sure you are on the Stream product) navigate to Worker Groups > default > **Data** > Destinations > **S3 > Add Destination**
- Only Configure the following:
    - Output ID: something to help identify the S3 destination (i.e S3_Syslog)
    - **S3 Bucket Name:** name of the bucket we created earlier
    - **Region:** match the region you used to create your bucket
    - **Key Prefix**: If you want to organize logs into folders by source or type, setting a key prefix can be helpful.
    - **File name prefix expression:** set this to define the output filename prefix (default is ‘CriblOut’)
    
    ![image 21](https://github.com/user-attachments/assets/12d52aeb-3d6c-4021-be69-30a2d7df3874)

- Set up **Authentication**:
    - Use the Access Key ID and Secret Access Key from your IAM user created in Step 1.1.

![image 22](https://github.com/user-attachments/assets/376d78b3-3549-4f36-b53e-28e8b522890f)

- Enable AssumeRole

![image 23](https://github.com/user-attachments/assets/40e62e4e-e7fa-4341-a1f9-9a90d201a38c)

- Save & Commit/Deploy
    - Test the destination to ensure connectivity and permissions. This is like giving Cribl the green light to store logs in your new S3 bucket!

![image 24](https://github.com/user-attachments/assets/3ab56dba-a49b-4adc-9830-75122d870b42)

NOTE: I’d recommend cloning the destination you just created and modifying the Output ID, key prefix, and file name prefix for additional log types (i.e Windows Events)

![image 25](https://github.com/user-attachments/assets/0fce9b2f-df27-4f49-be3f-d47fd5bbc65d)

![image 26](https://github.com/user-attachments/assets/f311bd7f-74d5-4daa-8afe-4542dab43bf4)

### Setup a Data Route for our Windows Events and Linux Syslog

- Navigate to Routing > Data Routes > Add Route
    
    ![image 27](https://github.com/user-attachments/assets/53da0d93-0aac-4591-92ef-d5f8480c405b)

    
**Raspberry Pi (Syslog)**

- Name your Route and set the filter to the following (this will filter for logs arriving from our Syslog input we set for the Raspberry Pi)
    - Set pipeline to passthru
    - Set the output to the S3 Destination you designated for your Raspberry Pi syslog data.
    - Enable **Final:**
        - **T**his toggle determines whether or not logs will fall to the next route step below it for further processing. It will be important to set non-Final routes when we add our Splunk destination as they will be placed above our subsequent S3 steps.
        - Further Explanation
            - Windows Events that are filtered through the designated route are forwarded to S3 and do not get processed further
            - What if Final is set to “No?”: events with the same filter (in the case of our future Splunk destination) would then fall to the next route for further processing. We will want our Splunk destination to allow a fallthrough but our S3 destination to mark the ending.

![image 28](https://github.com/user-attachments/assets/99d979e5-f5ab-47af-a1ee-057407f6fe53)

**Windows Event Logs**

- Name your Route and set the filter to the following (this will filter for logs arriving from our Windows Event collector)
    - Set pipeline to passthru
    - Set the output to the S3 Destination (if) you designated for your Windows Event data.
    - Enable **Final**

![image 29](https://github.com/user-attachments/assets/f51c6212-16eb-4a51-8d9c-ff44268dac97)

Align the routes in these chronological orders (the default must be placed last as it serves as a catch-all for events that do not meet the filtered criteria above it)

![image 30](https://github.com/user-attachments/assets/3c30ec09-8754-4d8d-961b-d46e96d02700)

- Save & Commit/Deploy

Navigate back to S3 to verify that you have data arriving to your bucket via the prefixes we setup earlier

![image 31](https://github.com/user-attachments/assets/9795fbb4-66f5-4ba7-81ff-eeaa5a3fb090)

You have successfully setup an S3 destination with Cribl! 

---

## 2. Setting up Splunk on Docker

### Why Splunk?

Splunk is a powerhouse when it comes to analyzing log data in real time. With Splunk, we can set up live monitoring, create dashboards, and even set alerts to stay ahead of potential security issues. Running Splunk in Docker allows us to easily deploy and manage it without complex installation steps.

### 2.1 Install Docker (if not already installed)

First, we’ll need Docker to run Splunk as a container. Splunk will not work on our Raspberry Pis as it does not support ARM architecture. I’ve personally spun up my container using of one my Windows host machines but you are free to use another spare machine of your choice here. If Docker isn’t already installed on your machine of choice, let’s set it up.

- For [Debian](https://docs.docker.com/engine/install/debian/)
- For [Ubuntu](https://docs.docker.com/engine/install/ubuntu/)
- For [RHEL](https://docs.docker.com/desktop/install/linux/rhel/)
- For [Windows](https://docs.docker.com/desktop/install/windows-install/) (uses Docker Desktop)

### 2.2 Run Splunk Container

Now, let’s pull the Splunk Docker image and start it up! Running Splunk in Docker simplifies things by bundling all dependencies within the container. 

Navigate to Command Prompt (Windows)

or

Navigate to Terminal (Linux)

- Pull the Splunk image:
    
    ```bash
    
    docker pull splunk/splunk:latest
    ```
    
- Start the Splunk container:
    
    ```bash
    
    docker run -d --name splunk -p 8000:8000 -p 8088:8088 -p 9997:9997 -e SPLUNK_START_ARGS="--accept-license" -e SPLUNK_PASSWORD=<YourSplunkPassword> splunk/splunk
    
    ```
    
    - **Port 8000**: Splunk Web UI
    - **Port 8088**: Splunk HEC (HTTP Event Collector)
    - **Port 9997**: Splunk Forwarding (if you later decide you want to use Universal Forwarders)

By using the `--accept-license` argument, we bypass manual license acceptance, and the `SPLUNK_PASSWORD` so be sure to add one in place of <YourSplunkPassword> to set our admin password.

![image 32](https://github.com/user-attachments/assets/b2558e2c-a7a8-4105-bdda-4c096621d9ce)

An example of a Splunk container running with Docker Desktop

### 2.3 Access the Splunk UI

Once Splunk is up and running, let’s access the web UI to finish configuration.

- Open a browser and go to `http://<host-ip>:8000`.
- Log in with the username `admin` and the password you set in the Docker run command.

![image 33](https://github.com/user-attachments/assets/9936775b-b1d1-41dd-8103-3cd762db5b8d)

See how easy that was?!

---

## 3. Creating Indexes in Splunk

- **Navigate to Indexes:**
    - Go to **Settings** > **Indexes**.
    - This will bring you to the Index Management page.
- **Create a New Index:**
    - Click **New Index** in the top right corner.
- **Configure the Index:**
    - **Index Name**: Enter the same name you used in your Cribl syslog source configuration. For example, if your Input ID in Cribl was `'home_soc_syslog'` set the index name here to `"home_soc_syslog"`
    - **Data Type**: Choose **Events** (typically for log data like syslog).
    - **App Context**: Set to the app context where this index will be accessible, often set to `search` if it needs to be universally accessible in searches.
    - Max Size of Hot/Warm/Cold Bucket: Set to `auto_high_volume`

![image 34](https://github.com/user-attachments/assets/0458318a-aec1-40fd-938a-ac93c12ddcec)

- **Save the Index:**
    - Once configured, click **Save** to create the index.

**NOTE**: Create another index for `home_soc_winevents` with the same parameters as this is where we will send our Windows Event logs. 

## 4. Configuring the Splunk HTTP Event Collector (HEC)

### Why HEC?

The HTTP Event Collector (HEC) in Splunk allows applications and services to send data directly to Splunk over HTTP. It’s ideal for ingesting logs in real-time, giving you instant visibility into your environment.

### 3.1 Enable HEC in Splunk

Let’s enable HEC so Cribl can send logs to Splunk in real-time. This enables seamless log ingestion and immediate analysis.

- In the Splunk UI, go to **Settings** > **Data Inputs** > **HTTP Event Collector** > **Global Settings**.
    - **Disable SSL**: SSL is recommended for secure data transmission, however leave it off for the purpose of the lab.
        - If you don’t turn this off, you will likely run into backpressure issues
    - **Port**: Ensure it matches the Docker-exposed port, typically **8088**.
    - Click **Save**.

![image 35](https://github.com/user-attachments/assets/2cca10de-be70-46d8-b500-7602d8cf0201)

### 3.2 Create a New HEC Token

Now, we’ll set up a token for secure communication between Cribl and Splunk. This token is like a unique passcode that tells Splunk to trust data sent from Cribl.

- In **HTTP Event Collector**, click **New Token**.
- Enter a name, such as `Cribl-Logs`.
- **Input Settings:**
    - Leave Source type set to “Automatic”
    - Select allowed indexes and choose a default index (i.e “main” or create a “test” index)

![image 36](https://github.com/user-attachments/assets/4877226c-b497-43a3-bad8-1cdcca3321a3)

- **Review & Submit**
- Take note of the **HEC Token**; you’ll need this for the Cribl configuration.

---

## 5. Configuring Cribl to Forward Logs to Splunk HEC

Finally, let’s set up Cribl to send logs directly to Splunk’s HEC. This integration lets us view logs in real-time within Splunk, providing a complete view of what’s happening in our environment.

- In Cribl, go to **Settings** > **Destinations** > **Add Destination** > **Splunk HEC**.
    - **Endpoint URL**: `http://<splunk-host-ip>:8088/services/collector/event`
        - You can toggle “**Load balancing”** off for the purpose of this lab as we
    - **Authentication Method**:
        - Set to **“Manual”**
        - Use the token from Step 3.2.

![image 37](https://github.com/user-attachments/assets/feeb1c27-d72b-4f1e-a0a8-5fcb6b1b7aee)

- **Optional Settings**:
    - **Backpressure Behavior**: This setting controls what Cribl does if the destination (Splunk) isn’t available or is slow to respond. Set this to **Persistent Queue** to avoid data loss during network issues.
        - You can utilize the following PQ settings for reference

![image 38](https://github.com/user-attachments/assets/c08fd69d-bbf3-4f35-bc1b-1e3e5c22ddae)

Save & Commit/Deploy and test the configuration to verify connectivity.

### Configuring the Data Routes for Splunk

Head on over to Routing > Data Routes > Add Route

**Raspberry Pi (Syslog)**

- Name your Route and set the filter to the following (this will filter for logs arriving from our Syslog input we set for the Raspberry Pi)
    - Set pipeline to passthru
    - Set the output to the Splunk Destination we setup in the previous step
    - Disable **Final:**
        - This is so that logs will first flow to Splunk, drop to the next processing route, and meet finally at its S3 destination.

![image 39](https://github.com/user-attachments/assets/5f0b6579-2b06-49d6-ab85-0c437f568a6e)

**Windows Event Logs**

- Name your Route and set the filter to the following (this will filter for logs arriving from our Windows Event collector)
    - Set pipeline to passthru
    - Set the output to the Splunk Destination we setup in the previous step
    - Disable **Final**

![image 40](https://github.com/user-attachments/assets/31cd4155-6d5c-4bf9-94ca-3e1d31bde3cd)

- Your route order should look a bit like this!

![image 41](https://github.com/user-attachments/assets/e5a050da-1a45-4d6f-92d3-300f0943014e)

- Save & Commit/Deploy

Make your way back over to Splunk

- Navigate to Search & Reporting
    - Enter index=”home_soc_winevents” into the search bar so we can verify that logs are arriving in Splunk

![image 42](https://github.com/user-attachments/assets/068ccdef-be55-4284-8e47-0bc5585fbd7f)

You might not see syslog data from the Raspberry Pi’s as they aren’t very chatty with minimal daemons or containers running, however you should start seeing some Windows Events at the least (my logs are modified via a pack so they will appear in raw JSON here). 

- You can initiate `logger test` queries via shell access to populate some test logs for viewing in Splunk out of your ‘home_soc_syslog’ index

You have now successfully set up your Splunk instance to receive data from Cribl via HEC!

---

### Summary

In Part 2, we configured AWS S3 and Splunk HEC as destinations in our Home SOC Lab. This setup provides a robust data pipeline with both long-term storage and real-time monitoring capabilities.

Next up, we’ll look into analyzing and visualizing data with Splunk to make the most out of our SOC setup. Stay tuned for further insights!

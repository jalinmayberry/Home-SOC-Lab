# Home SOC Lab

Welcome to the **Home SOC Lab**! This project is a comprehensive setup aimed at creating a Security Operations Center (SOC) environment for home use. It leverages tools like Cribl, Splunk, and AWS S3 for log management, monitoring, and data storage.

## Project Overview

The Home SOC Lab is designed to emulate a production SOC environment and includes detailed configurations for syslog forwarding, event monitoring, and data ingestion from network devices and endpoints.

### Project Structure

The lab is organized into two parts:

- **Part 1**: Initial setup, including Cribl configuration, syslog forwarding, and setting up Cribl Edge for collecting logs from various sources.
- **Part 2**: Configuring destinations for collected logs, covering setups for AWS S3 and Splunk HTTP Event Collector (HEC).

### Looking Ahead: Part 2 - Cribl Destinations

In Part 2, we will explore setting up crucial destinations for our collected logs:

1. **AWS S3 Bucket**  
   - Creating and configuring an S3 bucket for log storage.
   - Setting up IAM roles and policies for secure access.
   - Configuring Cribl to send data to S3 for long-term retention.

2. **Splunk HTTP Event Collector (HEC)**  
   - Setting up Splunk HEC for real-time log ingestion.
   - Configuring Cribl to forward processed logs to Splunk.
   - Optimizing data flow between Cribl and Splunk.

By implementing these destinations, we will complete our data pipeline, enabling both long-term storage and real-time analysis capabilities for our logs.

## Key Components

- **Cribl Stream and Edge** for data collection and routing.
- **Splunk** for data indexing and analysis.
- **AWS S3** for data storage.
- **Raspberry Pi and other devices** for log forwarding.

## Getting Started

1. Follow the instructions in `Part 1` for the initial setup.
2. Move to `Part 2` for configuring destinations and finalizing the data pipeline.

## Future Enhancements

- Integrating Alerts and Notifications
- Building Dashboards and Visualizations
- Advanced Data Processing and Enrichment

---

## Disclaimer

This project is intended for educational and informational purposes only. The documentation, scripts, and configurations provided here are meant to assist users in understanding and setting up environments involving open-source and proprietary tools such as Cribl, Splunk, Docker, and AWS. This repository is not endorsed by or affiliated with any of these companies.

Please note:
- This repository does not include proprietary software. Any usage of such software requires the appropriate licenses from the respective vendors.
- The provided code and configurations are intended to work in isolated or test environments. Use in production environments is at your own risk.
- The MIT License in this repository applies only to the documentation and code provided here and does not extend to any third-party software or services mentioned.

By using this project, you agree to do so at your own discretion and risk. Please ensure compliance with any applicable terms and conditions of the software vendors mentioned.


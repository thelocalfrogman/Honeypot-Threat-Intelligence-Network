# Global Honeypot Threat Intelligence Network

A globally distributed honeypot network deployed across multiple cloud regions, capturing real-world attacker TTPs and feeding enriched telemetry into a centralised SIEM for threat intelligence analysis and trend reporting.

**NOTE:** This is an ongoing project in development.

## Overview

This project deploys a network of honeypots across geographically diverse cloud regions to collect, normalise, and analyse real-world attack data at scale. The goal isn't just to catch scans and brute-force attempts, it's to build a pipeline that turns raw honeypot logs into actionable threat intelligence: tracking attacker behaviour patterns, identifying emerging TTPs, and producing trend reports backed by live data.

Every component is designed to be reproducible. If you've got a cloud account and a couple of hours, you can spin up your own node and start collecting.

## Architecture
*Not exact for obvious reasons.*
```
┌─────────────────────────────────────────────────────────────┐
│                      Cloud Regions                          │
│                                                             │
│  ┌──────────┐   ┌──────────┐   ┌──────────┐                 │
│  │ Sydney   │   │ Singapore│   │ US East  │   ...           │
│  │ (Node)   │   │ (Node)   │   │ (Node)   │                 │
│  └────┬─────┘   └────┬─────┘   └────┬─────┘                 │
│       │              │              │                       │
│       └──────────────┼──────────────┘                       │
│                      │                                      │
│              ┌───────▼────────┐                             │
│              │  Log Shipping  │                             │
│              │  (Encrypted)   │                             │
│              └───────┬────────┘                             │
│                      │                                      │
└──────────────────────┼──────────────────────────────────────┘
                       │
            ┌──────────▼──────────┐
            │   Centralised SIEM  │
            │   Ingestion Layer   │
            └──────────┬──────────┘
                       │
          ┌────────────┼────────────┐
          │            │            │
   ┌──────▼───┐  ┌─────▼─────┐  ┌──▼───────────┐
   │ Parsing  │  │ Enrichment│  │ Threat Intel │
   │ & Norm.  │  │ (GeoIP,   │  │ Correlation  │
   │          │  │  ASN, IoC)│  │              │
   └──────┬───┘  └─────┬─────┘  └───┬──────────┘
          │            │            │
          └────────────┼────────────┘
                       │
              ┌────────▼────────┐
              │   Dashboards &  │
              │  Trend Reports  │
              └─────────────────┘
```

## Honeypot Stack

Each node runs a combination of honeypot services designed to cover the most commonly targeted protocols and services:

| Service | Protocol/Port | Purpose |
|---|---|---|
| Cowrie | SSH (22), Telnet (23) | Captures shell interaction, credential stuffing attempts, and post-auth commands |
| Dionaea | SMB (445), FTP (21), HTTP (80), SIP, MSSQL | Catches malware samples and exploit attempts against common services |
| Conpot | Modbus (502), S7comm, BACnet, IPMI | ICS/SCADA-targeted attacks, surprisingly noisy even on general cloud IPs |
| Honeytrap | Catch-all listener | Picks up scanning and exploitation attempts on non-standard ports |
| Elasticpot | Elasticsearch (9200) | Targets the constant stream of bots hunting for exposed Elastic instances |

Nodes are deployed as lightweight VMs with each honeypot service isolated.

## Deployment

### Infrastructure as Code

Node provisioning is handled with Terraform, with per-region modules that spin up:

- A cloud VM (smallest viable instance as these don't need much compute)
- Security group/firewall rules exposing only honeypot ports
- A log shipper (Filebeat) configured to forward to the central SIEM
- Automated hardening via Ansible post-provision

```bash
# Deploy a new honeypot node to a region
cd terraform/
terraform init
terraform apply -var="region=ap-southeast-2" -var="node_name=syd-hp-01"
```

### Ansible Configuration

Post-provisioning configuration is handled by Ansible playbooks:

```bash
# Configure a freshly provisioned node
ansible-playbook -i inventory/cloud.yml playbooks/deploy-honeypot.yml --limit syd-hp-01
```

The playbook handles:
- OS hardening (SSH key-only, disable unnecessary services, firewall baselines)
- Honeypot service deployment and configuration
- Filebeat/log shipper setup with TLS certificates
- Health check and heartbeat configuration

### Node Health Monitoring

Each node reports a heartbeat to the central SIEM. If a node goes silent for more than 10 minutes, an alert fires. This matters if an attacker realises they're in a honeypot and tries to pivot to the host, I want to know immediately.

## Log Pipeline

### Collection and Shipping

Each honeypot writes structured logs locally. Filebeat picks these up and ships them to the central SIEM over a TLS-encrypted channel.

### Enrichment

Raw logs are enriched at ingest time with:

- **GeoIP data:** Map source IPs to countries, cities, and coordinates for geographic attack distribution
- **ASN lookup:** Identify the originating network (cloud provider, ISP, known bulletproof hosting)
- **Threat intel feeds:** Cross-reference source IPs against known IoC lists (AbuseIPDB, OTX, custom feeds)
- **MITRE ATT&CK mapping:** Tag observed behaviours with relevant technique IDs where possible

### SIEM Platform

The central Wazuh SIEM ingests, indexes, and correlates all honeypot telemetry. The platform provides:

- **Real-time dashboards:** Geographic attack heatmaps, top attacking IPs/ASNs, protocol distribution, credential analysis
- **Alerting:** Threshold-based and anomaly-based alerts for unusual activity spikes or novel attack patterns
- **Correlation rules:** Identify attackers hitting multiple regions, track campaigns across time
- **Long-term storage:** Retain data for trend analysis and historical comparison

## Analysis and Reporting

### What Gets Tracked

- **Credential analysis:** Most common username/password combinations, trends over time, regional variations
- **Malware collection:** Samples captured by Dionaea are hashed and submitted for analysis
- **Attack origin trends:** Geographic and ASN-level distribution, how it shifts over time
- **Protocol targeting:** Which services attract the most attention and how that changes
- **TTP cataloguing:** Observed attacker techniques mapped to MITRE ATT&CK where applicable
- **Campaign identification:** Correlating activity across nodes to identify coordinated scanning/exploitation campaigns

## Project Status

> **This project is actively under development.** The sections below outline what's been completed and what's in progress.

- [x] Architecture design and component selection
- [x] Terraform modules for node provisioning
- [x] Ansible playbooks for honeypot deployment and hardening
- [x] Initial node deployment (AP-Southeast-2)
- [ ] Multi-region expansion (Southeast Asia, US East, EU West)
- [ ] SIEM ingest pipeline and enrichment configuration
- [ ] Dashboard and visualisation buildout
- [ ] Automated reporting pipeline
- [ ] Public-facing threat intelligence summary page

## Lessons Learned

*This section will be updated as the project progresses with practical takeaways, unexpected findings, and things that broke along the way.*

## Tech Stack

| Component | Tool |
|---|---|
| Infrastructure | Terraform |
| Configuration | Ansible |
| Honeypots | Cowrie, Dionaea, Conpot, Honeytrap, Elasticpot |
| Log Shipping | Filebeat |
| SIEM | Elastic SIEM / Wazuh |
| Enrichment | GeoIP, AbuseIPDB, OTX, MaxMind |
| Dashboards | Kibana / Grafana |
| Alerting | ElastAlert / Wazuh Rules |

## Legal and Ethical Considerations

Honeypots are passive data collection systems. All captured data consists of unsolicited connections to infrastructure that serves no legitimate production purpose. No entrapment or active engagement with attackers is involved.

All nodes are deployed on cloud infrastructure owned or controlled by the operator. Malware samples are handled in isolated analysis environments and are not redistributed.

## Acknowledgements

Built on the shoulders of open-source honeypot projects including [Cowrie](https://github.com/cowrie/cowrie), [Dionaea](https://github.com/DinoTools/dionaea), and [Conpot](https://github.com/mushorg/conpot), as well as the broader threat intelligence community.
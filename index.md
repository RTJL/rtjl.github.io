# Reuben Tan
[reubentjl@gmail.com]

---

## **Professional Summary**
Results-driven software engineer with deep expertise in building scalable observability systems, infrastructure automation, and performance optimization. Experienced in designing and re-architecting large-scale systems, optimizing data pipelines, and implementing cost-effective cloud infrastructure. Proven ability to lead critical backend engineering efforts across distributed teams while consistently delivering measurable impact.

---

## **Technical Skills**
- **Languages:** Python, Java, SQL
- **Cloud Platform:** AWS
- **Data & Observability:** Grafana, ClickHouse, Datadog, OpenSearch (ELK)
- **DevOps & Tooling:** Terraform, Ansible

---

## **Experience**

### **Grab** • Jun 2021 – Present
- **Senior Software Engineer, Observability** • Apr 2023 – Present
- **Software Engineer, Observability** • Jun 2021 – Apr 2023

#### **Logging Platform – ClickHouse Feasibility Study**
*Proof of Concept & Cost Optimization*
- Led a successful POC demonstrating that ClickHouse could replace OpenSearch for log analytics, enabling a **~45% cost reduction** of OpenSearch components.
- Achieved **93% query compatibility** with OpenSearch and improved performance in **88% of tested queries** without requiring user-side changes.
- Built a query converter and benchmarking tool to evaluate performance and compatibility at scale.
- Deployed and tested log ingestion pipelines on STG and PRD ClickHouse clusters; analyzed results to validate production readiness.

#### **Logging Platform – BCDR Support**
*Disaster Recovery Logging Environment*
- Provisioned a functional logging cluster to support service observability in the **Business Continuity & Disaster Recovery (BCDR)** environment.
- Configured components including OpenSearch Dashboards, Logstash, and data nodes via Ansible automation.
- Enhanced the infrastructure initialization repository to support the BCDR use case, ensuring seamless environment setup and log flow continuity.

#### **SLA Monitoring Platform**
*Re-architecture, Performance Engineering, and System Optimization*
- Led architectural overhaul of platform to reduce downtime risk and improve reliability.
- Migrated time-series DB from InfluxDB to ClickHouse, reducing P99 query latency by **49%**.
- Designed optimized OLAP schemas for datasets exceeding **2 billion rows**.
- Refactored SLI calculation logic to API layer, reducing code complexity and improving maintainability.
- Developed query optimizer to reduce Datadog API usage by **55%**.
- Implemented configurable exclusion windows and finer time granularity (10secs) for SLI calculations.
- Reduced infrastructure costs by **$50+K/year** by rightsizing Kafka clusters and ClickHouse nodes.

#### **Infrastructure Modernization & Logging Platform Migration**
*Migrated Logging Pipelines Across AWS & Azure Environments*
- Executed large-scale migration from legacy ELK & ADX logging stacks to OpenSearch.
- Patched EC2 agents across **600+ instances** programmatically using Ansible & SSM.
- Standardized logging pipeline across AWS & Azure to reduce maintenance and duplication.
- Enabled cost savings of **$40+K/month** and eliminated redundant cloud deployments.
- Adapted automation pipelines to multiple deployment systems (Jenkins → AWX → SSM)

#### **Chargeback Platform**
*Cost Reporting System Design*
- Designed and implemented an internal chargeback system for metrics and logging infrastructure, handling **petabyte-scale log volumes**.
- Engineered pipelines to map raw platform usage to detailed cost attribution across services.
- Improved cost attribution accuracy by migrating log data tagging from inconsistent formats to unified service-level identifiers.
- Validated tagging changes across **petabyte-scale logging cluster**, ensuring invoice alignment with <3% variance.
- Added cost telemetry to Datadog dashboards to drive accountability within engineering orgs.
- Automated daily cost ingestion, processing, and report generation for OpenSearch and Datadog.

### **StrongArm Tech** • Jan 2020 – Aug 2020

*New York, United States of America*

**Software Engineer Intern**

- Led the design and development of RESTful APIs for a customer-facing web application using a test-driven development (TDD) approach.
- Improved API response times by over **30%**, enhancing user experience and contributing to cost efficiency.
- Implemented monitoring and alerting with Datadog for a serverless architecture using **AWS API Gateway** and **Lambda**, including dashboards and custom alerts.

---

## **Projects**

### **Automated Incident Investigation using LLMs**
*LLM-powered Observability Assistant*
- Developed an API-based observability tool that uses **large language models (LLMs)** to automate the initial investigation of software incidents.
- Consolidates signals from systems like **Datadog** and **OpenSearch** into a unified **Grafana dashboard**, reducing manual triage effort for on-call engineers.
- Designed modular "Analyzers" to handle specific troubleshooting flows such as infrastructure issues, Go panics, and Hystrix failures.
- Integrated with **Datadog alerts** and API triggers to automatically provide contextual health summaries at the start of an incident.
- Focused on generating a concise system state overview, enabling faster response.

---

## **Certifications**

- **ClickHouse Certified Developer** – ClickHouse ([Link](https://www.credly.com/badges/242ad1c2-b9cc-4c07-a896-d305cfc7a34f/public_url))

  *Issued: Feb 2025*

- **Python 3 Programming Specialization** – Coursera ([Link](https://www.coursera.org/account/accomplishments/specialization/X3JUDTYN5XQE))

  *Issued: Jul 2024*

- **AWS Certified Cloud Practitioner** – Amazon Web Services ([Link](https://www.credly.com/badges/8b3d865d-e97a-4fbd-a4ba-c9e528c17a92/public_url))

  *Issued: Dec 2020*

- **HashiCorp Certified: Terraform Associate** – HashiCorp ([Link](https://www.credly.com/badges/e9ef217d-dba0-4a8e-a477-af36b179418d))

  *Issued: Dec 2020*

---

## **Education**
**National University of Singapore**

Bachelor of Computing in Computer Science

2021

**Singapore Polytechnic**

Diploma in Infocomm Security Management with Merit

2015

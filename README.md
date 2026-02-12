# Rough Notes "Notification Worker Integration with Elasticsearch"

---

## Objective

Implement and validate an end-to-end notification system where employee records are indexed into Elasticsearch and consumed by a scheduled notification worker to send email alerts via SMTP.

This PoC demonstrates complete infrastructure provisioning, application integration, and successful email delivery validation.

---

# Architecture Overview

<img width="472" height="258" alt="Architecture Diagram" src="https://github.com/user-attachments/assets/93f4a32b-94cc-4e0f-9ad7-d36a878b0dba" />

---

# Infrastructure Setup

## Elasticsearch Deployment

* **Version:** 7.8.0
* **Installation Path:** `/home/ubuntu/elasticsearch-7.8.0`
* **Service Manager:** `systemd`
* **Port:** 9200
* **Access Scope:** Internal VPC

---

## systemd Service Configuration

**Service File Location:**

```
/etc/systemd/system/elasticsearch.service
```

### Key Configuration:

* Runs as `ubuntu` user
* Restart policy enabled
* `network.host: 0.0.0.0`
* `http.port: 9200`
* Custom `path.data` configured

---

## Service Management Commands

```bash
sudo systemctl daemon-reload
sudo systemctl enable elasticsearch
sudo systemctl start elasticsearch
sudo systemctl status elasticsearch
```

---

## Elasticsearch Validation

```bash
curl http://10.0.1.25:9200
curl http://10.0.1.25:9200/_cat/indices?v
```

Elasticsearch cluster reachable and index confirmed.

---

# Employee API Integration (Go Service)

### Modified File:

```
employee-api/api/api.go
```

### Updated Function:

```
CreateEmployeeData()
```

---

## Logic Implemented

Upon receiving an employee creation request:

* Initialized Elasticsearch client (`go-elasticsearch/v7`)
* Indexed employee document directly into:

```
employee-management
```

### Indexed Fields:

* `name`
* `email_id`
* `status`

---

## Dependency Installation

```bash
go get github.com/elastic/go-elasticsearch/v7
go mod tidy
```

Employee API is managed via:

```
employee-api.service (systemd)
```

---

# Manual Elasticsearch Validation (PoC Testing)

## Insert Test Employee Document

```bash
curl -X PUT "http://10.0.1.25:9200/employee-management/_doc/EMP001" \
  -H "Content-Type: application/json" \
  -d '{
    "email_id": "abhinavsikarwar011@gmail.com",
    "name": "John Doe"
  }'
```

This manually indexes a test employee record into Elasticsearch.

<img width="1600" height="199" alt="image" src="https://github.com/user-attachments/assets/f948ac8f-98d7-4391-af51-918a642ad033" />

<img width="1191" height="504" alt="Email Screenshot 1" src="https://github.com/user-attachments/assets/0121b8b2-62b4-4361-99f9-bc43081f8292" />

---

## Fetch All Documents

```bash
curl -X GET "http://10.0.1.25:9200/employee-management/_search?pretty" \
  -H "Content-Type: application/json" \
  -d '{
    "query": {
      "match_all": {}
    }
  }'
```

<img width="1600" height="746" alt="image" src="https://github.com/user-attachments/assets/fdad9a00-acb3-46fd-bcc2-ff2c1a0f718c" />

Confirms:

* Index exists
* Document stored correctly
* `email_id` available for worker consumption

---

# Notification Worker Setup (Python Service)

**Location:**

```
notification-worker/notification_api.py
```

---

## Execution Modes

* `external` → One-time execution
* `scheduled` → Periodic polling

Testing configuration:

```python
schedule.every(1).minutes.do(send_mail_to_all_users)
```

---

## Elasticsearch Query Used by Worker

```python
index = "employee-management"
query = match_all
```

The worker extracts:

```
data["_source"]["email_id"]
```

And triggers SMTP email delivery.

---

# SMTP Configuration

* **Provider:** Gmail
* **Port:** 587
* **TLS:** Enabled
* **Authentication:** Gmail App Password
* **Config Source:** `NOTIFICATION_CONFIG` environment variable

Connectivity validated using:

```bash
telnet smtp.gmail.com 587
```

---

# End-to-End Validation

## Step 1 – Create Employee

POST request to:

```
/api/v1/employee/create
```

Result:

* Document indexed into Elasticsearch

Validated via:

```bash
curl http://10.0.1.25:9200/employee-management/_search?pretty
```

---

## Step 2 – Execute Notification Worker

```bash
python3 notification_api.py -m external
```

Observed:

* Successful ES connection
* Successful query execution
* Successful SMTP authentication
* No runtime errors

---

## Step 3 – Email Delivery Confirmation

Email successfully received at the configured recipient address.

### Verified Details:

* **Subject:** Salary Slip
* **Body:** "Your salary slip is generated please check"
* **Sender:** Configured SMTP account
* **Timestamp:** Matches worker execution time

**Screenshot Attached as Proof of Delivery**

<img width="1241" height="587" alt="image" src="https://github.com/user-attachments/assets/58752d0c-d8ab-41b6-b0b6-b186a2670613" />

<img width="744" height="485" alt="Email Screenshot 2" src="https://github.com/user-attachments/assets/373d71d8-376d-467f-b218-b72673d9824b" />

---

## Validation Confirms

* Successful Elasticsearch data retrieval
* Proper SMTP transmission
* Email delivery to recipient inbox
* Fully operational application-to-inbox pipeline

---

# Observability & Logs

Logs verified using:

```bash
journalctl -u employee-api.service
journalctl -u elasticsearch
```

Notification worker logs validated via stdout.

---

# Network & Security Configuration

* Elasticsearch (9200) restricted to internal VPC access
* SSH access restricted via Security Groups
* SMTP outbound (587) allowed from worker EC2
* No public exposure of Elasticsearch

---

# PoC Limitations

* Poll-based model (not event-driven)
* `match_all` query (no filtering logic)
* No processed flag (duplicate email risk)
* No retry/backoff mechanism
* No monitoring/alerting integration

---

#  PoC Status

* Elasticsearch deployed and stable
* Employee API indexing integrated
* Manual ES validation successful
* Notification worker operational
* SMTP validated
* Email delivery confirmed (screenshots attached)

End-to-end notification flow successfully demonstrated.

---

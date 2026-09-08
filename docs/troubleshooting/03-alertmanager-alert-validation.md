# Troubleshooting 03 — Alertmanager Alert Validation

> **Area:** Monitoring / Grafana / Alertmanager
> **Severity:** Alert validation
> **Status:** Resolved

---

## 1. Problem

A monitoring alert named `High CPU Usage` was configured in Grafana and needed to be validated end-to-end.

The objective was not only to verify that Grafana generated the alert, but also to confirm that Alertmanager received it correctly and that the configured notification path worked.

The validation therefore covered the complete alerting pipeline:

    Prometheus
        |
        v
    Grafana
        |
        | High CPU Usage alert
        v
    Alertmanager
        |
        | Notification
        v
    Email

---

## 2. Symptoms

The main requirement was to determine whether the `High CPU Usage` alert was correctly generated and received by Alertmanager.

An alert being configured in Grafana does not by itself prove that the complete monitoring pipeline is working.

The investigation therefore focused on several questions:

- Is the alert rule correctly configured?
- Is the alert actually firing?
- Is Alertmanager receiving the alert?
- Is the expected receiver being selected?
- Does the notification reach the configured destination?
- Does the alert also generate a resolved notification?

---

## 3. Investigation

### 3.1 Check the Alertmanager service

The first step was to verify that Alertmanager was running:

    sudo systemctl status alertmanager --no-pager

The expected result was:

    Active: active (running)

This confirmed that the Alertmanager service itself was operational.

---

### 3.2 Check the Alertmanager API

The next step was to query the Alertmanager API:

    curl -s http://localhost:9093/api/v2/alerts

The response was inspected for the expected alert.

The relevant information included:

    alertname: High CPU Usage
    server: rhel-server-01
    receiver: email
    state: active

This confirmed that Alertmanager had received the alert.

The API was particularly useful because it allowed the investigation to distinguish between an alert-generation problem and a notification-delivery problem.

---

### 3.3 Confirm the alert state

The important state was:

    state: active

An `active` state indicated that the alert was currently firing.

The investigation therefore confirmed that the alert was not simply configured but was actually present in Alertmanager.

---

### 3.4 Confirm the expected receiver

The alert showed:

    receiver: email

This was important because Alertmanager uses routing rules to determine which receiver handles an alert.

The expected path was:

    Grafana
        |
        v
    High CPU Usage
        |
        v
    Alertmanager
        |
        v
    email receiver
        |
        v
    SMTP
        |
        v
    Recipient

This confirmed that the alert was being routed to the expected notification receiver.

---

### 3.5 Check Alertmanager logs

The Alertmanager logs were inspected to identify any errors:

    sudo journalctl -u alertmanager --since "10 minutes ago" --no-pager

The logs were used to determine whether Alertmanager had configuration, routing or notification errors.

This is an important troubleshooting step because the API confirms that an alert exists, while the logs provide information about what Alertmanager is doing with that alert.

---

### 3.6 Verify the Grafana alert rule

The Grafana alert rule responsible for the notification was also reviewed.

The alert was based on CPU utilisation and was named:

    High CPU Usage

The purpose of the rule was to generate an alert when CPU usage exceeded the configured threshold.

The important distinction was:

    Alert rule configured
            !=
    Alert actually firing

Therefore, the rule had to be tested rather than simply assumed to be working.

---

## 4. Root Cause / Finding

The investigation confirmed that the `High CPU Usage` alert was successfully reaching Alertmanager.

The Alertmanager API showed:

    alertname: High CPU Usage
    server: rhel-server-01
    receiver: email
    state: active

This demonstrated that the alert generation and Alertmanager reception stages were working.

The validation therefore established the following:

| Component | Status |
|-----------|--------|
| Prometheus metrics | Working |
| Grafana alert rule | Working |
| Alert generation | Working |
| Alertmanager service | Working |
| Alertmanager API | Working |
| Alertmanager alert reception | Working |
| Alertmanager routing | Working |
| Email receiver | Configured |

The previous SMTP authentication issue was a separate notification-delivery problem and was documented in:

    01-alertmanager-gmail-smtp.md

---

## 5. Resolution

The alert validation was completed by triggering the `High CPU Usage` condition and verifying that the alert appeared in Alertmanager.

The alert was checked with:

    curl -s http://localhost:9093/api/v2/alerts

The expected alert information was present:

    alertname: High CPU Usage
    server: rhel-server-01
    receiver: email
    state: active

This confirmed that the alert pipeline was functioning from Grafana through Alertmanager.

The notification path was then validated separately.

---

## 6. Validation

### 6.1 Validate Alertmanager service

Run:

    sudo systemctl is-active alertmanager

Expected result:

    active

---

### 6.2 Validate Alertmanager API

Run:

    curl -s http://localhost:9093/api/v2/alerts

Verify that the expected alert is present.

Expected information:

    alertname: High CPU Usage
    server: rhel-server-01
    receiver: email
    state: active

---

### 6.3 Validate alert firing

The CPU condition was triggered so that the Grafana alert rule became active.

The alert was then checked through Alertmanager.

The expected flow was:

    CPU usage increases
            |
            v
    Grafana alert rule
            |
            v
       Alert fires
            |
            v
      Alertmanager
            |
            v
       state: active

This confirmed that the alert was actually firing rather than merely being configured.

---

### 6.4 Validate email notification

After the SMTP authentication issue was resolved, the firing notification was successfully delivered by email.

The complete firing path was:

    CPU threshold exceeded
            |
            v
         Grafana
            |
            v
       Alertmanager
            |
            v
       Gmail SMTP
            |
            v
      Email received

---

### 6.5 Validate alert resolution

The CPU utilisation was allowed to return below the configured threshold.

The alert then changed from:

    state: active

to a resolved condition.

The resolved notification was also successfully received.

The complete lifecycle was therefore validated:

    ALERT FIRING
         |
         v
    Email received
         |
         v
    CPU returns below threshold
         |
         v
    ALERT RESOLVED
         |
         v
    Resolved email received

This was important because the Alertmanager configuration included:

    send_resolved: true

---

## 7. Troubleshooting Flow

    +----------------------+
    | Grafana Alert Rule   |
    |   High CPU Usage     |
    +----------+-----------+
               |
               v
          Alert firing?
               |
          +----+----+
          |         |
         NO        YES
          |         |
          v         v
    Check rule   Check
    and query    Alertmanager
                    |
                    v
             Alert in API?
                    |
               +----+----+
               |         |
              NO        YES
               |         |
               v         v
        Investigate    Check
        alert/rule     receiver
                           |
                           v
                    receiver: email
                           |
                           v
                     Check logs
                           |
                           v
                    SMTP working?
                           |
                      +----+----+
                      |         |
                     NO        YES
                      |         |
                      v         v
                 Troubleshoot  Email
                    SMTP       received
                                |
                                v
                         Resolve condition
                                |
                                v
                       Resolved email received

---

## 8. Useful Commands

### Check Alertmanager service

    sudo systemctl status alertmanager --no-pager

### Check whether Alertmanager is active

    sudo systemctl is-active alertmanager

### Check Alertmanager alerts

    curl -s http://localhost:9093/api/v2/alerts

### Check Alertmanager logs

    sudo journalctl -u alertmanager --since "10 minutes ago" --no-pager

### Check recent Alertmanager logs

    sudo journalctl -u alertmanager --since "2 minutes ago" --no-pager

### Check Alertmanager listening port

    sudo ss -lntp | grep 9093

### Check SMTP configuration

    sudo grep -E 'smtp_(from|auth_username|smarthost)' /etc/alertmanager/alertmanager.yml

---

## 9. What Each Check Proves

### `systemctl status alertmanager`

Proves that the Alertmanager service is running.

It does not prove that alerts are being received or emails are being delivered.

---

### `curl http://localhost:9093/api/v2/alerts`

Proves that Alertmanager currently has alerts available through its API.

This is one of the most useful commands for separating alert-generation problems from notification problems.

---

### `journalctl -u alertmanager`

Provides information about Alertmanager behaviour and errors.

It can reveal problems involving:

- Configuration
- Routing
- Receivers
- SMTP
- Authentication
- Notification delivery

---

### `ss -lntp | grep 9093`

Confirms whether something is listening on Alertmanager's HTTP port.

This can help identify service or connectivity problems.

---

## 10. Alert Lifecycle

An important concept in monitoring is that an alert has a lifecycle.

The simplified lifecycle used in this lab was:

    Normal
       |
       | CPU exceeds threshold
       v
    FIRING
       |
       | Alertmanager receives alert
       v
    Notification
       |
       | CPU returns below threshold
       v
    RESOLVED
       |
       v
    Resolved notification

This means that testing only the firing state is not sufficient for complete validation.

Both states should be tested whenever possible.

---

## 11. Lessons Learned

- An alert rule being configured does not prove that it is working.
- An alert should be deliberately triggered during validation.
- The Alertmanager API is useful for confirming alert reception.
- The `receiver` field helps verify Alertmanager routing.
- Alertmanager logs are essential when investigating notification problems.
- A running Alertmanager service does not automatically mean the complete alerting pipeline is working.
- Alert generation and notification delivery are separate troubleshooting stages.
- SMTP authentication problems can occur even when Alertmanager correctly receives an alert.
- Both firing and resolved notifications should be tested.
- End-to-end monitoring validation should verify the complete path from the metric to the final notification.

---

## 12. Final Status

**Status: RESOLVED**

The `High CPU Usage` alert was successfully validated end-to-end.

The investigation confirmed:

    CPU metric
        |
        v
    Grafana alert rule
        |
        v
    Alert generated
        |
        v
    Alertmanager
        |
        v
    Alert visible through API
        |
        v
    Email receiver
        |
        v
    Gmail SMTP
        |
        v
    Alert email received
        |
        v
    CPU returns below threshold
        |
        v
    Resolved notification received

The validation confirmed that the monitoring alerting pipeline was operating correctly and that both firing and resolved notification paths were functional.

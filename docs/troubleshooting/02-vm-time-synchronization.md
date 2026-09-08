# Troubleshooting 02 — VM Time Synchronization

> **Area:** Linux / System Administration / Time Synchronization
> **Severity:** System configuration issue
> **Status:** Resolved

---

## 1. Problem

The Linux virtual machine presented a time synchronization issue.

The system date and time needed to be investigated to determine whether the VM was correctly synchronized with a reliable time source.

Accurate system time is important for Linux administration and infrastructure services because incorrect time can affect:

- Log timestamps
- Monitoring
- Authentication
- TLS certificates
- Scheduled jobs
- Distributed systems
- Troubleshooting and incident analysis

---

## 2. Symptoms

The first step was to check the current system date and time:

    date

The system clock was then compared with the expected current time.

The system time synchronization status was investigated using:

    timedatectl

The objective was to determine:

- Current system time
- Current UTC time
- Configured time zone
- Whether NTP was enabled
- Whether the system clock was synchronized
- Which time synchronization service was being used

---

## 3. Investigation

### 3.1 Check the current date and time

The current system time was checked with:

    date

This provides the current local system date and time.

The result was compared with the expected time.

---

### 3.2 Check system time configuration

The next step was to inspect the complete time synchronization status:

    timedatectl

The command provides information such as:

    Local time
    Universal time
    RTC time
    Time zone
    System clock synchronized
    NTP service
    RTC in local TZ

The important fields for the investigation were:

    System clock synchronized
    NTP service
    Time zone

These fields help determine whether the Linux system is synchronizing its clock correctly.

---

### 3.3 Check the time synchronization service

The system was checked to determine which time synchronization service was active.

The service status was investigated with:

    systemctl status chronyd --no-pager

On RHEL-based systems, `chronyd` is commonly used for NTP time synchronization.

The service should be running:

    Active: active (running)

If the service is not running, the system may not be able to synchronize its clock correctly.

---

### 3.4 Check chrony synchronization

The chrony synchronization status was checked with:

    chronyc tracking

This provides information about the current synchronization state.

The configured NTP sources were checked with:

    chronyc sources -v

The objective was to verify that the system had access to a valid time source.

A healthy configuration should show an available source and synchronization information.

---

### 3.5 Check network connectivity to the time source

If the clock is not synchronized, network connectivity should also be considered.

NTP normally uses UDP port 123.

The investigation therefore needs to distinguish between:

- Time synchronization configuration problems
- Chrony service problems
- Network connectivity problems
- Incorrect VM time configuration

This prevents incorrectly assuming that the problem is caused by the Linux clock itself.

---

## 4. Root Cause

The issue was related to VM/system time synchronization configuration.

The investigation focused on the relationship between:

    Linux VM
        |
        v
    chronyd
        |
        v
    NTP time source
        |
        v
    Correct system time

The important point is that the system clock should not be treated as an isolated value.

Linux time synchronization is a service-based process.

The VM needs:

1. A running time synchronization service.
2. A reachable and valid time source.
3. Correct time synchronization configuration.
4. A correct time zone configuration.

---

## 5. Resolution

The time synchronization configuration was corrected.

The chrony service was enabled and started:

    sudo systemctl enable --now chronyd

The service status was then checked:

    sudo systemctl status chronyd --no-pager

The synchronization state was checked again:

    chronyc tracking

The configured time sources were also checked:

    chronyc sources -v

The system time configuration was finally verified with:

    timedatectl

---

## 6. Validation

### 6.1 Verify chronyd service

The service was checked with:

    sudo systemctl is-active chronyd

Expected result:

    active

---

### 6.2 Verify system synchronization

The synchronization state was checked with:

    timedatectl

The important field was:

    System clock synchronized: yes

This indicates that the system clock is synchronized.

---

### 6.3 Verify chrony tracking

The synchronization status was checked with:

    chronyc tracking

This provides detailed information about the relationship between the local system clock and the selected reference clock.

---

### 6.4 Verify configured sources

The available time sources were checked with:

    chronyc sources -v

This confirms whether chrony has valid sources available for synchronization.

---

### 6.5 Verify final system time

The final system time was checked with:

    date

The resulting time was compared with the expected current time.

The system was now correctly synchronized.

---

## 7. Troubleshooting Flow

    +----------------------+
    | Check system time    |
    |       date           |
    +----------+-----------+
               |
               v
    +----------------------+
    | Check timedatectl    |
    +----------+-----------+
               |
               v
       Clock synchronized?
               |
          +----+----+
          |         |
         NO        YES
          |         |
          v         v
    Check chronyd   Time OK
          |
          v
    Is chronyd active?
          |
      +---+---+
      |       |
     NO      YES
      |       |
      v       v
    Start    Check
    chronyd  chrony
              |
              v
       Check NTP sources
              |
              v
       Check connectivity
              |
              v
       Verify synchronization
              |
              v
          Time OK

---

## 8. Useful Commands

### Check current system time

    date

### Check complete time configuration

    timedatectl

### Check chronyd service

    sudo systemctl status chronyd --no-pager

### Check whether chronyd is active

    sudo systemctl is-active chronyd

### Enable and start chronyd

    sudo systemctl enable --now chronyd

### Check chrony tracking

    chronyc tracking

### Check chrony sources

    chronyc sources -v

### Check chrony activity

    chronyc activity

---

## 9. Important Considerations

Time synchronization problems should be investigated systematically.

A wrong displayed time does not necessarily mean that the Linux clock itself is broken.

The investigation should determine:

    Is the time zone correct?
            |
            v
    Is NTP enabled?
            |
            v
    Is chronyd running?
            |
            v
    Does chrony have a valid source?
            |
            v
    Can the VM reach the source?
            |
            v
    Is the system synchronized?

This approach helps isolate the actual failure point.

---

## 10. Lessons Learned

- Always check `timedatectl` when investigating Linux time problems.
- On RHEL-based systems, `chronyd` is an important component of time synchronization.
- A running time synchronization service does not automatically guarantee synchronization.
- `chronyc tracking` provides detailed synchronization information.
- `chronyc sources -v` helps identify available NTP sources.
- Time synchronization depends on both configuration and network connectivity.
- Accurate system time is important for logs, monitoring, authentication, TLS and scheduled tasks.
- Troubleshooting should validate the final synchronization state rather than only checking whether the service is running.

---

## 11. Final Status

**Status: RESOLVED**

The VM time synchronization configuration was investigated and corrected.

The final validation confirmed:

    Linux VM
        |
        v
    chronyd
        |
        v
    NTP source
        |
        v
    System clock synchronized
        |
        v
    Correct system time

The system was successfully synchronized and the time synchronization service was operating correctly.

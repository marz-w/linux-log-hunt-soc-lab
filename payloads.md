# 📋 Payloads: Quick-Reference Commands for Linux Log Investigation

Commands used and referenced in the **Linux Log Hunt** lab. All examples are for investigating logs on systems you own or are authorised to review. Use `sudo` where required.

---

## 1. Where to look

| Log type | Ubuntu / Debian | CentOS / RHEL |
|---|---|---|
| Auth / SSH / sudo | `/var/log/auth.log` | `/var/log/secure` |
| General system | `/var/log/syslog` | `/var/log/messages` |
| Cron | `/var/log/syslog` | `/var/log/cron` |
| Apache access | `/var/log/apache2/access.log` | `/var/log/httpd/access_log` |
| Nginx access | `/var/log/nginx/access.log` | `/var/log/nginx/access.log` |
| Audit | `/var/log/audit/audit.log` | `/var/log/audit/audit.log` |
| Login history | `/var/log/wtmp` | `/var/log/wtmp` |
| Failed logins | `/var/log/btmp` | `/var/log/btmp` |

---

## 2. Reading logs

```bash
# Last 50 lines, and follow live
sudo tail -n 50 /var/log/auth.log
sudo tail -f /var/log/auth.log

# Page through a file
sudo less /var/log/secure
```

---

## 3. SSH investigation

```bash
# Failed SSH logins (Ubuntu/Debian)
sudo grep "Failed password" /var/log/auth.log

# Failed SSH logins (CentOS/RHEL)
sudo grep "Failed password" /var/log/secure

# Successful SSH logins
sudo grep "Accepted" /var/log/auth.log

# Attempts against non-existent accounts
sudo grep "invalid user" /var/log/auth.log

# Count failed attempts per source IP (highest first)
sudo grep "Failed password" /var/log/auth.log | awk '{print $(NF-3)}' | sort | uniq -c | sort -nr

# Everything from one suspicious IP
sudo grep "203.0.113.45" /var/log/auth.log
```

---

## 4. systemd journal (`journalctl`)

```bash
# SSH service logs (service may be "ssh" on Ubuntu/Debian)
journalctl -u sshd
journalctl -u ssh

# Time-boxed search
journalctl -u sshd --since "2026-10-10 13:00" --until "2026-10-10 14:00"

# Errors and worse
journalctl -p err

# Follow live
journalctl -u sshd -f

# Entries from the sudo command
journalctl _COMM=sudo
```

---

## 5. Login history

```bash
# Successful logins, logouts and reboots (reads /var/log/wtmp)
last

# Read a specific file
last -f /var/log/wtmp

# Failed login attempts (reads /var/log/btmp, needs root)
sudo lastb

# Most recent login per user
lastlog
```

---

## 6. `sudo` and privilege escalation

```bash
# All sudo activity
sudo grep "sudo:" /var/log/auth.log

# Only the commands that were run
sudo grep "COMMAND=" /var/log/auth.log

# Sudo activity touching the shadow file
sudo grep "/etc/shadow" /var/log/auth.log
```

---

## 7. Cron and persistence

```bash
# Cron activity (RHEL/CentOS)
sudo cat /var/log/cron

# Cron activity (Ubuntu/Debian)
sudo grep CRON /var/log/syslog

# List current user's scheduled jobs
crontab -l

# List another user's scheduled jobs
sudo crontab -l -u username

# System-wide cron locations
ls -la /etc/cron.d /etc/cron.daily /etc/cron.hourly /etc/crontab
```

---

## 8. Web server logs (Apache / Nginx)

```bash
# Requests to the admin area
grep "/admin" /var/log/apache2/access.log

# Count responses by status code (status is field 9 in combined log format)
awk '{print $9}' /var/log/apache2/access.log | sort | uniq -c | sort -nr

# Top client IPs
awk '{print $1}' /var/log/apache2/access.log | sort | uniq -c | sort -nr | head

# Only successful requests / only errors
grep '" 200 ' /var/log/apache2/access.log
grep '" 404 ' /var/log/apache2/access.log
```

---

## 9. auditd

```bash
# Search the audit log for access to a file
sudo ausearch -f /etc/shadow

# Authentication summary report
sudo aureport -au

# Raw audit log
sudo less /var/log/audit/audit.log
```

---

## 10. Logging configuration

```bash
# rsyslog service and config
systemctl status rsyslog
cat /etc/rsyslog.conf
ls /etc/rsyslog.d/

# journald config
cat /etc/systemd/journald.conf

# Make the journal persistent across reboots
sudo mkdir -p /var/log/journal
sudo systemctl restart systemd-journald
```

---

## 11. Anatomy of key log entries

**SSH failure**
```
Oct 10 13:42:17 web01 sshd[8421]: Failed password for invalid user admin from 203.0.113.45 port 49822 ssh2
```
`timestamp` · `host` · `service[pid]` · `event` · `username` · `source IP` · `source port`

**sudo**
```
Oct 10 15:03:14 finance01 sudo: alex : TTY=pts/1 ; PWD=/home/alex ; USER=root ; COMMAND=/usr/bin/cat /etc/shadow
```
`who ran it` · `terminal` · `working dir` · `run as` · `command`

**Apache**
```
198.51.100.23 - - [10/Oct/2026:14:12:09 +0000] "GET /admin/login.php HTTP/1.1" 200 4281 "-" "Mozilla/5.0"
```
`client IP` · `timestamp` · `"method path protocol"` · `status` · `bytes` · `referrer` · `user agent`

---

## 12. HTTP status codes cheat sheet

| Code | Meaning |
|---|---|
| `200` | OK, request succeeded |
| `301` / `302` | Redirect |
| `401` | Unauthorised, authentication required |
| `403` | Forbidden |
| `404` | Not found |
| `500` | Server error |

---

## 13. Red flags to look for

- Many `Failed password` lines from one IP, followed by an `Accepted` line from the same IP
- Logins for `invalid user` names such as `admin`, `root` or `test`
- `sudo` commands that read `/etc/shadow` or `/etc/passwd`
- Cron jobs you do not recognise, especially at unusual hours such as 02:30
- Repeated requests to `/admin`, `/login` or other sensitive paths

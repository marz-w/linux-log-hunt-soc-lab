# 🕵️ Linux Log Hunt: The SOC Analyst's First Investigation

![Status](https://img.shields.io/badge/status-completed-brightgreen)
![Category](https://img.shields.io/badge/category-Log%20Analysis%20%7C%20SOC-blue)
![Tool](https://img.shields.io/badge/tool-journalctl%20%7C%20last%20%7C%20grep%20%7C%20auditd-orange)
![Platform](https://img.shields.io/badge/platform-Ubuntu%20%2F%20Debian%20%2F%20CentOS%20%2F%20RHEL-red)
![License](https://img.shields.io/badge/license-MIT-lightgrey)
![Made By](https://img.shields.io/badge/made%20by-Martha%20Wambui-purple)

A hands-on SOC analyst challenge covering how Linux generates, stores and manages logs across different distributions, and how to read those logs to triage suspicious SSH, web server, `sudo` and scheduled-task activity.

---

## 📖 Project Overview

This project documents a 17-question, scenario-based investigation. The scenario: you have just joined a Security Operations Centre (SOC) that runs Ubuntu, Debian, CentOS and RHEL servers. Alerts have come in for suspicious SSH logins, unusual web requests, privileged command execution and possible unauthorised scheduled tasks. Before investigating, you must show you can find and interpret the right logs.

**Skills practised:**
1. **Locating logs** — knowing where authentication, system, cron, web and audit logs live on Debian-family vs RHEL-family systems.
2. **Understanding Linux logging** — traditional `rsyslog` text files vs the `systemd-journald` journal.
3. **Reading log entries** — pulling out timestamps, usernames, source IPs, commands, HTTP requests and status codes.
4. **Correlating events** — spotting a successful login that follows repeated failures from the same IP.

> ⚠️ **Ethical note:** All log entries in this challenge are fictional sample data (documentation IP ranges such as `203.0.113.0/24` and `198.51.100.0/24`). No real systems were accessed.

---

## 🧰 Tech Stack

| Tool / Concept | Purpose |
|---|---|
| **rsyslog / syslog** | Traditional text-based logging to `/var/log/` |
| **systemd-journald + `journalctl`** | Modern binary journal and the tool used to query it |
| **`last` / `wtmp`** | History of successful interactive logins |
| **auditd** | Detailed audit trail of sensitive activity |
| **Apache / Nginx** | Web server access logs |
| **`sudo` log entries** | Evidence of privileged command execution |
| **cron logs** | Evidence of scheduled tasks (possible persistence) |

---

## ✨ Features

- ✅ Cheat sheet of log locations for Ubuntu/Debian vs CentOS/RHEL
- ✅ Worked explanation for every one of the 17 questions
- ✅ Breakdown of how to read SSH, `sudo` and Apache log lines field by field
- ✅ Correlation example: failed logins followed by a successful login from the same IP
- ✅ Reusable command reference in [`payloads.md`](payloads.md)
- ✅ Annotated answer sheet in Word format (see [Project Structure](#-project-structure))

---

## ✅ Prerequisites

- Basic familiarity with the Linux command line
- Awareness that Linux distributions name some log files differently
- A Linux VM is helpful for practising the commands, but is not required for the challenge itself

---

## 🚀 Usage Guide

### Part 1 — Know where the logs live

| Log type | Ubuntu / Debian | CentOS / RHEL |
|---|---|---|
| Authentication / SSH / sudo | `/var/log/auth.log` | `/var/log/secure` |
| General system events | `/var/log/syslog` | `/var/log/messages` |
| Cron (scheduled tasks) | `/var/log/syslog` | `/var/log/cron` |
| Apache access log | `/var/log/apache2/access.log` | `/var/log/httpd/access_log` |
| Nginx access log | `/var/log/nginx/access.log` | `/var/log/nginx/access.log` |
| Audit log (auditd) | `/var/log/audit/audit.log` | `/var/log/audit/audit.log` |
| Login history | `/var/log/wtmp` (via `last`) | `/var/log/wtmp` (via `last`) |

### Part 2 — Read an SSH log line

```
Oct 10 13:42:17 web01 sshd[8421]: Failed password for invalid user admin from 203.0.113.45 port 49822 ssh2
```

| Part | Meaning |
|---|---|
| `Oct 10 13:42:17` | Timestamp |
| `web01` | Hostname |
| `sshd[8421]` | Service and process ID |
| `Failed password` | What happened |
| `invalid user admin` | The username tried, which does not exist on the server |
| `203.0.113.45` | Source IP address |
| `port 49822` | Attacker's source port (not the IP) |

### Part 3 — Read a `sudo` log line

```
Oct 10 15:03:14 finance01 sudo: alex : TTY=pts/1 ; PWD=/home/alex ; USER=root ; COMMAND=/usr/bin/cat /etc/shadow
```

`alex` used `sudo` to run `cat` on `/etc/shadow` as root. That file holds password hashes, so reading it is a major red flag.

### Part 4 — Read an Apache log line

```
198.51.100.23 - - [10/Oct/2026:14:12:09 +0000] "GET /admin/login.php HTTP/1.1" 200 4281 "-" "Mozilla/5.0"
```

Client IP, timestamp, request (`GET /admin/login.php`), status code (`200` = OK), response size (`4281` bytes) and user agent.

### Part 5 — Correlate the events

| Time | Event | Source |
|---|---|---|
| 13:42:17 | `Failed password for invalid user admin` | `203.0.113.45` |
| 13:45:51 | `Accepted password for james` | `203.0.113.45` |

The same IP failed and then succeeded within minutes, which suggests a successful brute-force or password-guessing attack. The analyst should review what `james` did next, consider a password reset and block the IP.

---

## 📝 Answer Summary

| Q | Topic | Answer |
|---|---|---|
| 1 | HTTP status code in the Apache entry | `200` (OK) |
| 2 | Login history command and file | `last`, reads `/var/log/wtmp` |
| 3 | Cron log on RHEL/CentOS | `/var/log/cron` |
| 4 | SSH entries in the journal | `journalctl -u sshd` |
| 5 | Utility to query the systemd journal | `journalctl` |
| 6 | Default auditd log | `/var/log/audit/audit.log` |
| 7 | Sensitive file in the sudo event | `/etc/shadow` (password hashes) |
| 8 | Command run with elevated privileges | `cat` (logged as `/usr/bin/cat /etc/shadow`) * |
| 9 | Default Nginx access log | `/var/log/nginx/access.log` |
| 10 | Auth log on CentOS/RHEL | `/var/log/secure` |
| 11 | URL path requested | `/admin/login.php` |
| 12 | Apache access log on Ubuntu/Debian | `/var/log/apache2/access.log` |
| 13 | Success after failure | Same IP failed, then logged in as `james`: likely brute-force success |
| 14 | Username attempted | `admin` |
| 15 | Attacking IP | `203.0.113.45` |
| 16 | CentOS equivalent of `/var/log/syslog` | `/var/log/messages` |
| 17 | Ubuntu auth/SSH log | `/var/log/auth.log` |

\* The challenge platform did not accept the full path form on a first attempt, so the bare command name appears to be the expected format.

**Lab notes:** Question 7 refers to "the sudo event in Question 11" but that event is in Question 8, and Question 13 refers to Question 4 but the failed logins are in Questions 14 and 15. These look like numbering typos in the lab.

---

## 📁 Project Structure

```
linux-log-hunt-soc-lab/
│
├── README.md                                    # This file
├── payloads.md                                  # Quick-reference commands for log investigation
└── Linux_Log_Hunt_Answers_and_Explanations.docx # Lab brief with answers and explanations added
```

Add a `screenshots/` folder if you want to include your own evidence.

---

## 🔧 Troubleshooting

| Issue | Likely Cause | Fix |
|---|---|---|
| Answer rejected by the platform | Missing leading `/` in a file path, or wrong format | Enter the full absolute path, e.g. `/var/log/nginx/access.log` |
| `/var/log/auth.log` does not exist | You are on CentOS/RHEL, or the system uses only the journal | Check `/var/log/secure`, or use `journalctl -u sshd` |
| `journalctl` shows nothing after reboot | Journal is not persistent | Create `/var/log/journal` and restart `systemd-journald` |
| `lastb` or `ausearch` permission denied | These need root | Run with `sudo` |

---

## 🛠️ Defensive Takeaways

- **Centralise logs** (SIEM or remote syslog) so an attacker cannot erase evidence on one host
- **Disable SSH password logins** and use keys; add rate limiting such as `fail2ban`
- **Alert on success-after-failure** patterns from the same source IP
- **Monitor access to `/etc/shadow`** with auditd rules
- **Review cron jobs** regularly for unknown scheduled tasks

---

## 🤝 Contributing

This is a personal learning project, but suggestions are welcome. Fork the repo, create a branch, and open a Pull Request, or open an issue if you spot an error.

---

## 📄 License

This project is licensed under the **MIT License** — feel free to reference or adapt it for your own learning purposes, with attribution.

---

## 📬 Contact

**Martha Wambui**
📧 Email: [marthamichellewambui@gmail.com](mailto:marthamichellewambui@gmail.com)
🔗 LinkedIn: [linkedin.com/in/martha-wambui-4293873b5](https://www.linkedin.com/in/martha-wambui-4293873b5/)
🔗 GitHub: [github.com/marz-w](https://github.com/marz-w)

---

*This project was completed as a SOC log analysis challenge using fictional sample log data — no real systems were accessed.*

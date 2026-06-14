# Services & Task Automation

This lab covers managing Linux services with **systemd** (`systemctl`), reading logs with **journald** (`journalctl`), and automating tasks with **cron** and **systemd timers**.

Every section first explains **what it is about**, then gives a **Solution** subchapter with all the commands and file contents. This lab runs on a single VM (user `student`); use `sudo` for service/system changes.

---

## Lab Setup

### What this is about

Provision a VM (`scgc_lab<no>_<username>`, **SCGC Template**, flavor **g.medium**+, user `student`). This lab works directly on the VM — no nested topology.

### Solution

Connect to the VM as `student`; everything below runs there.

---

## systemd services

### What this is about

Most distributions use **systemd** as the init/service manager. `systemctl status` shows whether a unit is *enabled* (starts at boot), *active* (running now), its main process, and recent log lines.

### Solution

```bash
systemctl status ssh        # state of the SSH service: enabled?, active?, PID, recent logs
```

---

## Changing the state of services

### What this is about

Control services at runtime and at boot: `start`/`stop` affect *now*; `enable`/`disable` affect *whether it starts at boot*. The two are independent.

### Solution

General commands (replace `servicename`):

```bash
sudo systemctl start servicename     # start now
sudo systemctl stop servicename      # stop now
sudo systemctl enable servicename    # start automatically at boot
sudo systemctl disable servicename   # don't start at boot
```

**Exercise 1 — stop SSH and observe.** Stopping SSH drops your *next* connection attempt; an existing session stays. After a reboot SSH won't come back if you also disabled it.

```bash
sudo systemctl stop ssh
# try to open a NEW ssh session -> refused
sudo systemctl start ssh             # restore access
```

**Exercise 2 — install and toggle nginx** (don't disable critical services like SSH):

```bash
sudo apt update
sudo apt install -y nginx
systemctl status nginx
sudo systemctl disable nginx         # won't auto-start at boot
sudo systemctl enable nginx          # auto-start again
```

---

## Tuning service files

### What this is about

Unit files live in `/lib/systemd/system` (shipped defaults) and `/etc/systemd/system` (local overrides). `systemctl cat` shows the effective file. Instead of editing the shipped file, you create an **override** with `systemctl edit`, which writes a drop-in under `/etc/systemd/system/<unit>.d/override.conf` and survives package updates.

### Solution

View a unit's configuration:

```bash
systemctl cat nginx          # shows [Unit]/[Service]/[Install]: Description, After, ExecStart, KillMode...
```

**Exercise 3 — override a setting.** Limit nginx's open-file count to demonstrate an override:

```bash
sudo systemctl edit nginx
```

In the editor, add:

```ini
[Service]
LimitNOFILE=5
```

Then apply and observe (nginx likely fails — too few file descriptors), then check where the override landed:

```bash
sudo systemctl restart nginx
systemctl status nginx                       # observe the effect of the tiny limit
cat /etc/systemd/system/nginx.service.d/override.conf
```

---

## journald

### What this is about

systemd captures all unit output in the **journal**, queried with `journalctl`. By default it shows the current boot; you can filter by unit, follow live, or read previous boots. Logs are volatile unless you make storage persistent.

### Solution

Common usage and flags:

```bash
journalctl                   # all logs from the current boot
journalctl -b -1             # logs from the PREVIOUS boot
journalctl -e                # jump to the end
journalctl -x                # add explanatory help text to messages
journalctl -u nginx          # only the nginx unit
journalctl -t sometag        # filter by syslog tag
journalctl -n 50             # last 50 entries
journalctl -f                # follow live (like tail -f)
```

Make logs persist across reboots — edit `/etc/systemd/journald.conf`:

```ini
[Journal]
Storage=persistent
```

Then restart journald:

```bash
sudo systemctl restart systemd-journald
```

**Exercise 4 — watch nginx errors live.** In one terminal follow the unit; in another, break the config and restart to see the error logged:

```bash
# terminal 1:
journalctl -u nginx -f
```

```bash
# terminal 2: remove a ';' in /etc/nginx/nginx.conf, then:
sudo systemctl restart nginx   # error appears live in terminal 1
```

---

## Task automation using cron

### What this is about

**cron** runs commands on a schedule defined by five time fields (`minute hour day-of-month month day-of-week`) followed by the command. Edit your jobs with `crontab -e`. By default cron emails the job's output to you.

### Solution

Edit your crontab:

```bash
crontab -e
```

Example — back up a directory every minute (the `%` must be escaped as `\%` in crontab):

```cron
* * * * * umask 0077; tar zcvf "/home/student/backups/$(date +"\%F-\%H-\%M-\%S").tar.gz" /home/student/important
```

**Exercise 5 — count files every two minutes.** `*/2` in the minute field means "every 2 minutes":

```cron
*/2 * * * * find /usr/share -type f | wc -l
```

If you see `(CRON) info (No MTA installed, discarding output)`, cron has no mail system to deliver the output to — that's Exercise 6.

**Exercise 6 — capture cron output.** Install a local-only mail server, or redirect the output yourself:

```bash
sudo apt install -y postfix          # choose "Local only" during configuration
cat /var/mail/student                # cron output now lands here
```

Redirect output in the crontab instead of mailing it:

```cron
# append all output (stdout+stderr) to a file:
*/2 * * * * find /usr/share -type f | wc -l >>/home/student/count.log 2>&1
# or discard it entirely:
*/2 * * * * find /usr/share -type f | wc -l >/dev/null 2>&1
```

---

## Automating tasks using systemd timers

### What this is about

A **systemd timer** is the modern alternative to cron: a `name.timer` unit triggers the matching `name.service`. Timers integrate with journald logging and `systemctl`. Here you build a job that regenerates the login banner (`/etc/motd`).

### Solution — prerequisites

Stop the dynamic MOTD from overwriting yours — edit `/etc/pam.d/sshd` and comment out (or remove):

```
session    optional     pam_motd.so  motd=/run/motd.dynamic
```

Create the script `/usr/local/sbin/update-motd.sh`:

```bash
#!/bin/sh
set -eu -o pipefail

fetch_info() {
    echo "System information at $(date)"
    echo
    echo "Network interfaces:"
    ip -br a s | grep -E '^(eth|enp|virbr)'
    echo
    echo "Disk usage:"
    df -h /
    echo
}

echo "Fetching system information"
fetch_info > /etc/motd
echo "Finished successfully"
```

Make it executable:

```bash
sudo chmod +x /usr/local/sbin/update-motd.sh
```

### Solution — Exercise 7: service + timer

Create `/etc/systemd/system/motd-update.service`:

```ini
[Unit]
Description=Update the MOTD with system information

[Service]
Type=oneshot
ExecStart=/bin/bash /usr/local/sbin/update-motd.sh
```

Create `/etc/systemd/system/motd-update.timer` (runs every 2 minutes; no random delay):

```ini
[Unit]
Description=Run motd-update every two minutes

[Timer]
OnCalendar=*:0/2
Persistent=true

[Install]
WantedBy=timers.target
```

Reload systemd, enable **only the timer** (it pulls in the service), and verify:

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now motd-update.timer
systemctl list-timers                 # confirm motd-update.timer is scheduled
journalctl -u motd-update.service     # see "Fetching..."/"Finished successfully"
```

### Solution — Exercise 8: tag the logs

Give the service a clear log identifier with a full override:

```bash
sudo systemctl edit --full motd-update.service
```

Add to the `[Service]` section:

```ini
SyslogIdentifier=motd-update
```

Reload and re-check the journal — entries now show `update-motd[...]` (the tag) instead of `bash[...]`:

```bash
sudo systemctl daemon-reload
journalctl -u motd-update.service
```

# Practice Exam E — Certificates/GPG & Services Automation

**Labs:** `lab-cert` + services/automation · **Time:** 75 min · **Total:** 100 pts

## Scenario

Two parts on a single VM (user `student`, `sudo` for system changes). Part 1 is PKI/TLS/GPG with `openssl`/`gpg`; Part 2 is service management and task automation with systemd/cron.

```bash
cd work/ && wget https://repository.grid.pub.ro/cs/scgc/laboratoare/lab-cert.zip && unzip lab-cert.zip
```

---

## Tasks (prompts)

### Part 1 — Certificates & GPG

1. **(10 pts)** For `houdini.cs.pub.ro.crt-roedunet`, show its **issuer**, **subject**, **validity dates**, and **verify** it against its CA chain. *[inspecting certs]*
2. **(10 pts)** Fetch and decode the **live certificate** of `indico.upb.ro:443` and read its **Subject Alternative Names**. *[remote certs]*
3. **(15 pts)** Generate a **private key + CSR + CA-signed certificate** (CN `server.tld`), and prove the **key and certificate match**. *[generating certs]*
4. **(15 pts)** Start a **TLS server** with that key/cert and connect a client that **verifies the CA** (Verification: OK). Show that a capture reveals no plaintext. *[TLS client/server]*
5. **(10 pts)** With GPG: as Bob, **encrypt a file for Alice and sign it**; as Alice, **decrypt and verify** the signature. *[GPG encrypt/sign]*

### Part 2 — Services & automation

6. **(10 pts)** Install **nginx**, show its status, and override it so `LimitNOFILE=5`; restart and observe. Where did the override land? *[systemd tuning]*
7. **(10 pts)** Follow nginx logs **live** with journald; from another terminal break the config and restart to see the error logged. *[journald]*
8. **(10 pts)** Create a **cron** job that runs **every 2 minutes** and counts regular files in `/usr/share`, writing output to a file (not email). *[cron]*
9. **(10 pts)** Create a **systemd service + timer** `motd-update` that runs `/usr/local/sbin/update-motd.sh` every 2 minutes; enable only the timer and confirm it fired in the journal. *[systemd timers]*

---

## Answer key

<details><summary>Task 1 — inspect cert</summary>

```bash
cd cert-inspect
openssl x509 -in houdini.cs.pub.ro.crt-roedunet -noout -issuer
openssl x509 -in houdini.cs.pub.ro.crt-roedunet -noout -subject
openssl x509 -in houdini.cs.pub.ro.crt-roedunet -noout -dates
openssl verify -CAfile terena-ca-chain.pem houdini.cs.pub.ro.crt-roedunet
```
</details>

<details><summary>Task 2 — remote cert</summary>

```bash
echo | openssl s_client -connect indico.upb.ro:443 2>/dev/null \
  | sed -ne '/-BEGIN CERTIFICATE-/,/-END CERTIFICATE-/p' \
  | openssl x509 -noout -text     # read the Subject Alternative Name (DNS:) entries
```
</details>

<details><summary>Task 3 — generate cert</summary>

```bash
cd cert-gen
openssl genrsa -out server.key 2048
openssl req -new -key server.key -out server.csr     # CN = server.tld
openssl ca -config ca.cnf -policy signing_policy -extensions signing_req -in server.csr -out server.crt
# prove key <-> cert match (equal hashes):
openssl x509 -in server.crt -noout -modulus | md5sum
openssl rsa  -in server.key -noout -modulus | md5sum
openssl verify -CAfile ca/ca.crt server.crt
```
</details>

<details><summary>Task 4 — TLS server/client</summary>

Server:
```bash
openssl s_server -key server.key -cert server.crt -accept 12345
```
Client (verifies CA):
```bash
openssl s_client -CAfile ca/ca.crt -connect localhost:12345     # "Verification: OK"
```
Capture (third shell) shows only encrypted bytes:
```bash
sudo tcpdump -A -i lo port 12345
```
</details>

<details><summary>Task 5 — GPG encrypt + sign / decrypt + verify</summary>

Bob:
```bash
gpg --output doc.gpg --encrypt --recipient alice@stud.acs.upb.ro doc
gpg --armour --detach-sign doc.gpg          # -> doc.gpg.asc
```
Alice:
```bash
gpg --verify doc.gpg.asc doc.gpg            # Good signature from 'Bob ...'
gpg --output doc --decrypt doc.gpg
```
(Prereq: keys generated with `gpg --full-generate-key`, public keys exchanged/imported and signed.)
</details>

<details><summary>Task 6 — nginx + override</summary>

```bash
sudo apt update && sudo apt install -y nginx
systemctl status nginx
sudo systemctl edit nginx
```
In the editor:
```ini
[Service]
LimitNOFILE=5
```
```bash
sudo systemctl restart nginx
systemctl status nginx
cat /etc/systemd/system/nginx.service.d/override.conf     # where the override lands
```
</details>

<details><summary>Task 7 — journald live</summary>

Terminal 1:
```bash
journalctl -u nginx -f
```
Terminal 2: remove a `;` in `/etc/nginx/nginx.conf`, then:
```bash
sudo systemctl restart nginx     # error shows live in terminal 1
```
</details>

<details><summary>Task 8 — cron every 2 min</summary>

```bash
crontab -e
```
Add:
```cron
*/2 * * * * find /usr/share -type f | wc -l >>/home/student/count.log 2>&1
```
</details>

<details><summary>Task 9 — systemd timer</summary>

Script `/usr/local/sbin/update-motd.sh` (see [services_automation](../services_automation.md)), then:

`/etc/systemd/system/motd-update.service`:
```ini
[Unit]
Description=Update MOTD
[Service]
Type=oneshot
ExecStart=/bin/bash /usr/local/sbin/update-motd.sh
```
`/etc/systemd/system/motd-update.timer`:
```ini
[Unit]
Description=Run motd-update every two minutes
[Timer]
OnCalendar=*:0/2
Persistent=true
[Install]
WantedBy=timers.target
```
```bash
sudo systemctl daemon-reload
sudo systemctl enable --now motd-update.timer
systemctl list-timers
journalctl -u motd-update.service
```
</details>

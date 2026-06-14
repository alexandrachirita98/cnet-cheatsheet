# Certificates — PKI, X.509, TLS, GPG

This lab covers public-key security: inspecting and generating **X.509 certificates** with `openssl`, securing a client/server channel with **TLS**, and using **GPG** to encrypt and sign files.

Every section first explains **what it is about**, then gives a **Solution** subchapter with all the commands. Each cell is for a single role (you / server / client / Alice / Bob) — the heading says which.

---

## Lab Setup

### What this is about

Provision a VM (`scgc_lab<no>_<username>`, **SCGC Template**, flavor **g.medium**+, user `student`), download the lab. This lab runs on a single VM, so there is no `runvm.sh` topology.

### Solution

```bash
cd work/
wget https://repository.grid.pub.ro/cs/scgc/laboratoare/lab-cert.zip
unzip lab-cert.zip
```

---

## Creating and inspecting certificates

### What this is about

A certificate binds a **public key** to an **identity** (the Subject), signed by a **Certificate Authority (CA)**. TLS uses it to prove a server is who it claims to be. Here you read certificate fields, verify a cert against its CA chain, fetch a remote server's cert, and generate your own.

### Solution — inspecting local certificate files

```bash
cd cert-inspect

# Full human-readable dump of the whole certificate:
openssl x509 -in houdini.cs.pub.ro.crt-roedunet -noout -text

# Pull out individual fields:
openssl x509 -in houdini.cs.pub.ro.crt-roedunet -noout -pubkey      # the public key
openssl x509 -in houdini.cs.pub.ro.crt-roedunet -noout -startdate   # valid from
openssl x509 -in houdini.cs.pub.ro.crt-roedunet -noout -enddate     # valid until
openssl x509 -in houdini.cs.pub.ro.crt-roedunet -noout -dates       # both dates
openssl x509 -in houdini.cs.pub.ro.crt-roedunet -noout -issuer      # who signed it (the CA)
openssl x509 -in houdini.cs.pub.ro.crt-roedunet -noout -subject     # who it is for
openssl x509 -in houdini.cs.pub.ro.crt-roedunet -noout -modulus     # key modulus (for matching key<->cert)
```

Verify the certificate against its CA chain (trust is valid only if it chains to a trusted CA):

```bash
openssl verify -CAfile terena-ca-chain.pem houdini.cs.pub.ro.crt-roedunet   # expect: OK
```

Inspect a chain file (it contains several certificates: leaf → intermediate → root):

```bash
cat terena-ca-chain.pem
openssl crl2pkcs7 -nocrl -certfile terena-ca-chain.pem | openssl pkcs7 -print_certs -noout
```

**Exercise — verify the other two certs.** Check each cert's `-issuer`, then verify it against the chain whose root matches that issuer:

```bash
openssl x509 -in open-source.cs.pub.ro.crt-roedunet -noout -issuer
openssl verify -CAfile terena-ca-chain.pem open-source.cs.pub.ro.crt-roedunet

openssl x509 -in security.cs.pub.ro.crt-roedunet -noout -issuer
openssl verify -CAfile terena-ca-chain.pem security.cs.pub.ro.crt-roedunet
```

### Solution — inspecting remote certificates

Fetch a live server's certificate over TLS:

```bash
echo | openssl s_client -connect indico.upb.ro:443        # the empty echo closes stdin so it exits
```

Extract just the certificate from the handshake and decode it (note the SAN / DNS entries):

```bash
echo | openssl s_client -connect indico.upb.ro:443 2>/dev/null \
  | sed -ne '/-BEGIN CERTIFICATE-/,/-END CERTIFICATE-/p' \
  | openssl x509 -noout -text
```

### Solution — generating certificates

```bash
cd cert-gen

# 1) Private key (2048-bit RSA):
openssl genrsa -out server.key 2048

# 2) Certificate Signing Request — you'll be prompted for identity fields:
#      Organization Name      -> Cloud courses
#      Organizational Unit    -> Development
#      Common Name (CN)       -> server.tld
openssl req -new -key server.key -out server.csr

# 3) Sign the CSR with the lab CA to produce the certificate:
openssl ca -config ca.cnf -policy signing_policy -extensions signing_req -in server.csr -out server.crt
```

Confirm the key and certificate actually match (the two moduli must be identical):

```bash
openssl x509 -in server.crt -noout -modulus | md5sum
openssl rsa  -in server.key -noout -modulus | md5sum    # same hash as above
```

Verify the new certificate against the CA:

```bash
openssl verify -CAfile ca/ca.crt server.crt             # expect: OK
```

---

## Client/server communication

### What this is about

Show the difference between a plaintext channel (anyone sniffing sees the messages) and a TLS channel (encrypted, and the client can verify the server's certificate).

### Solution — unencrypted

On the **server** (your VM, one shell):

```bash
nc -l 12345                     # plain TCP listener on port 12345
```

On the **client** (second shell) — type messages:

```bash
nc localhost 12345
```

In a **third shell** — sniff the loopback and watch the messages appear in cleartext:

```bash
sudo tcpdump -A -i lo port 12345
```

### Solution — over SSL/TLS

On the **server** — start a TLS server with your generated key and cert:

```bash
openssl s_server -key server.key -cert server.crt -accept 12345
```

On the **client** — without the CA the cert can't be verified:

```bash
openssl s_client -connect localhost:12345                 # verification fails (unknown CA)
```

On the **client** — supply the CA so verification succeeds:

```bash
openssl s_client -CAfile ca/ca.crt -connect localhost:12345   # "Verification: OK"
```

In a **third shell** — the same tcpdump now shows only encrypted bytes, no readable text:

```bash
sudo tcpdump -A -i lo port 12345
```

---

## GPG

### What this is about

GPG provides public-key **encryption** (only the recipient can read a file) and **signatures** (prove who created a file and that it wasn't altered). Each person has a keypair; you exchange and sign each other's public keys to establish trust. The roles **Alice** and **Bob** below are two students (or two terminals).

> Tip: GPG often emits binary; always use `--output <file>` (or pipe to `cat -v`) so you don't garble the terminal.

### Solution — key setup

On **each side** (Alice and Bob) — generate a keypair:

```bash
gpg --full-generate-key
#   Key type : RSA and RSA (default)
#   Key size : 4096
#   Validity : 0 (never expires)
#   Name     : Student User
#   Email    : user@stud.acs.upb.ro
#   Comment  : SCGC Lab
gpg --list-key --with-subkey-fingerprint   # shows the primary (sign/certify) + encryption subkey
```

### Solution — share encrypted data

On **Alice** — export her public key and show its fingerprint:

```bash
gpg --armour --output alice.gpg --export <Alice's email>   # ASCII-armoured public key
gpg -v --fingerprint <Alice's email>                       # read the fingerprint aloud to verify
```

On **Bob** — import Alice's key, verify the fingerprint, then sign it (marks it trusted):

```bash
gpg --import alice.gpg
gpg --list-keys
gpg --edit-key alice
#   at the gpg> prompt:
#     fpr     -> show fingerprint (compare with what Alice told you)
#     sign    -> sign her key
#     save    -> save and exit
```

On **Bob** — encrypt a file for Alice (only she can decrypt it):

```bash
gpg --output doc.gpg --encrypt --recipient <Alice's email> doc
```

On **Alice** — decrypt the file Bob sent:

```bash
gpg --output doc --decrypt doc.gpg
cat doc
```

*Response exercise:* Alice repeats the same steps in reverse (encrypt for Bob, send back).

### Solution — verifying files with GPG (encrypt + sign)

On **Bob** — encrypt for Alice, then attach a detached signature:

```bash
gpg --output doc.gpg --encrypt --recipient alice@stud.acs.upb.ro doc
gpg --armour --detach-sign doc.gpg          # creates doc.gpg.asc (the signature)
```

On **Alice** — verify the signature matches the file:

```bash
gpg --verify doc.gpg.asc doc.gpg            # "Good signature from 'Bob ...' [full]"
```

### Solution — verify an ISO download

Verify a real Fedora release the way you'd verify any signed download:

```bash
# 1) Import Fedora's official GPG keys (from getfedora.org / the Security page):
gpg --import fedora.gpg

# 2) Verify the checksum file's signature with those keys:
gpg --verify Fedora-...-CHECKSUM            # confirms the checksum file is authentic

# 3) Only then trust the checksums to verify the ISO itself:
sha256sum -c Fedora-...-CHECKSUM            # confirms the ISO matches
```

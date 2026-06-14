# Containers — Docker

This lab covers **container-based virtualization** with Docker: running containers, building custom images with a `Dockerfile`, wiring multiple containers together (manually and with `docker compose`), and persisting data with volumes.

Every section first explains **what it is about**, then gives a **Solution** subchapter with all the commands and file contents. Commands run **inside the nested lab VM** unless noted (user `student`, password `student`).

---

## Lab Setup

### What this is about

Provision a VM (`scgc_lab<no>_<username>`, **SCGC Template**, flavor **g.medium**+, user `student`), download the lab, boot the nested VM and find its IP.

### Solution

On the **cloud VM**:

```bash
cd ~/work
wget https://repository.grid.pub.ro/cs/scgc/laboratoare/lab-docker.zip
unzip lab-docker.zip
bash runvm.sh                       # start the nested VM where Docker runs
virsh net-dhcp-leases labvms        # find the nested VM's IP, then ssh in (student/student)
```

---

## Core concepts

### What this is about

A **container** is an isolated environment for running an application, sharing the host **kernel** instead of booting its own. Compared to VMs they are lighter (no guest kernel/OS), start fast, and can be built in layers and shipped as images. Good for quick service installs, isolated test environments, and local replicas of production.

Trade-off: containers isolate *processes* but share the host kernel, so isolation is weaker than a full VM (a kernel/container escape affects the host).

---

## Starting containers

### What this is about

Run containers from an image in different modes: interactive shell, one-off command, and detached (background), plus how to inspect, exec into, and stop them.

### Solution

Interactive shell inside a container (`-it` = interactive + TTY):

```bash
sudo docker run -it gitlab.cs.pub.ro:5050/scgc/cloud-courses/ubuntu:18.04 bash
```

Run a single command and exit (container lives only for that command):

```bash
sudo docker run gitlab.cs.pub.ro:5050/scgc/cloud-courses/ubuntu:18.04 ps -ef
```

Run detached, then inspect / enter / stop it:

```bash
sudo docker run -d gitlab.cs.pub.ro:5050/scgc/cloud-courses/ubuntu:18.04 sleep 10000   # -d = background
sudo docker ps                                  # list running containers, note the ID
sudo docker exec -it <container-id> /bin/bash   # open a shell in the running container
sudo docker stop <container-id>                 # stop it
```

**Exercise — Rocky Linux 8 background container:**

```bash
sudo docker run -d gitlab.cs.pub.ro:5050/scgc/cloud-courses/rockylinux:8 sleep 10000
sudo docker ps
sudo docker exec -it <container-id> /bin/bash
# inside the container:
yum install -y bind-utils
exit                                            # disconnect (container keeps running)
```

---

## Building custom images

### What this is about

A **Dockerfile** is a recipe: start `FROM` a base image, then each `RUN` adds a layer. `docker build` turns it into a reusable image you can run or share.

### Solution

Create a file named `Dockerfile`:

```dockerfile
FROM gitlab.cs.pub.ro:5050/scgc/cloud-courses/ubuntu:18.04
ARG DEBIAN_FRONTEND=noninteractive          # don't prompt during apt installs
ARG DEBCONF_NONINTERACTIVE_SEEN=true
RUN apt-get update
RUN apt-get install -y software-properties-common
RUN apt-get install -y firefox
```

Build it and confirm the image exists:

```bash
docker build -t firefox-container .          # -t names the image; "." = build context (this dir)
docker image list
```

**Exercise — Rocky image with bind-utils.** Create `Dockerfile.rocky`:

```dockerfile
FROM gitlab.cs.pub.ro:5050/scgc/cloud-courses/rockylinux:8
RUN yum install -y bind-utils
```

Build (point `-f` at the non-default filename) and test it:

```bash
docker build -t rocky-dns -f Dockerfile.rocky .
docker run -it rocky-dns nslookup hub.docker.com    # bind-utils provides nslookup
```

---

## Multi-container applications

### What this is about

Real apps are several containers (e.g. a web app + its database) that talk over a shared **Docker network** by container name. Here you run WordPress + MySQL by hand.

### Solution — create the network

```bash
sudo docker network remove test-net          # ignore error if it doesn't exist yet
sudo docker network create test-net
```

Start the MySQL database (`--hostname db` so others reach it as `db`):

```bash
sudo docker run -d --hostname db --network test-net \
  -e "MYSQL_ROOT_PASSWORD=somewordpress" \
  -e "MYSQL_DATABASE=wordpress" \
  -e "MYSQL_USER=wordpress" \
  -e "MYSQL_PASSWORD=wordpress" \
  mysql:5.7
```

Start WordPress, pointing it at `db`, and publish its port 80 on host port 8000:

```bash
sudo docker run -d --hostname wordpress --network test-net \
  -p "8000:80" \
  -e "WORDPRESS_DB_HOST=db" \
  -e "WORDPRESS_DB_USER=wordpress" \
  -e "WORDPRESS_DB_PASSWORD=wordpress" \
  gitlab.cs.pub.ro:5050/scgc/cloud-courses/wordpress:latest
```

Browse to `http://<vm-ip>:8000` to reach WordPress.

**Exercise — NextCloud:** run a NextCloud container the same way, publishing its HTTP port, e.g.:

```bash
sudo docker run -d --hostname nextcloud --network test-net -p "8080:80" nextcloud:latest
```

---

## Docker Compose

### What this is about

`docker compose` declares the whole multi-container stack in one YAML file so you bring it up/down with a single command instead of many `docker run`s.

### Solution

Create `docker-compose.yml`:

```yaml
services:
  db:
    image: mysql:5.7
    networks:
      - wordpress-net
    environment:
      MYSQL_ROOT_PASSWORD: somewordpress
      MYSQL_DATABASE: wordpress
      MYSQL_USER: wordpress
      MYSQL_PASSWORD: wordpress
  wordpress:
    depends_on:
      - db
    image: wordpress:latest
    networks:
      - wordpress-net
    ports:
      - "8000:80"
    environment:
      WORDPRESS_DB_HOST: db
      WORDPRESS_DB_USER: wordpress
      WORDPRESS_DB_PASSWORD: wordpress
      WORDPRESS_DB_NAME: wordpress
networks:
  wordpress-net:
```

Bring the stack up, then tear it down:

```bash
sudo docker-compose up        # start everything (add -d to run in background)
sudo docker-compose down      # stop and remove the containers/network
```

**Exercise:** write an equivalent compose file that starts NextCloud (plus its database).

---

## Persistent storage

### What this is about

Container filesystems are **ephemeral** — data is lost when the container is removed. **Volumes** (managed by Docker) or **bind mounts** (a host directory) keep data across restarts.

### Solution — named volume

```bash
sudo docker run -d -v mysql-volume:/var/lib/mysql \
  -e "MYSQL_ROOT_PASSWORD=somewordpress" \
  -e "MYSQL_DATABASE=wordpress" \
  -e "MYSQL_USER=wordpress" \
  -e "MYSQL_PASSWORD=wordpress" \
  mysql:5.7                                  # DB data persists in the 'mysql-volume' volume
```

### Solution — bind mount a host directory

```bash
sudo docker run -d -v ~/mysql-vol/:/shared-dir \
  -e "MYSQL_ROOT_PASSWORD=somewordpress" \
  -e "MYSQL_DATABASE=wordpress" \
  -e "MYSQL_USER=wordpress" \
  -e "MYSQL_PASSWORD=wordpress" \
  mysql:5.7                                  # /shared-dir maps to ~/mysql-vol on the host
```

### Solution — volume in compose

```yaml
volumes:
  mysql-vol:/var/lib/mysql
```

**Exercise:** attach a volume to NextCloud at `/var/www/html`, restart the container, and confirm your data is still there.

---

## Security considerations

### What this is about

Containers limit what one container can see of another (process isolation), but they **share the host kernel**. A container escape can therefore reach the host. VMs isolate more strongly (separate kernel/OS) at the cost of more overhead — choose based on the threat model.

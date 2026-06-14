# Practice Exam D — Containers (Docker)

**Lab:** `lab-docker` · **Time:** 75 min · **Total:** 100 pts

## Scenario

Work inside the nested lab VM where Docker runs (user/pass `student`/`student`). Use `sudo` for docker commands. Prove each result with `docker ps` / `docker image list` / a browser or command output.

```bash
cd ~/work && wget https://repository.grid.pub.ro/cs/scgc/laboratoare/lab-docker.zip
unzip lab-docker.zip && bash runvm.sh
virsh net-dhcp-leases labvms     # find the nested VM's IP, then ssh in
```

---

## Tasks (prompts)

1. **(10 pts)** Start an **Ubuntu 18.04 container in the background** running `sleep`, list it, open a shell inside it, then stop it. *[running containers]*
2. **(15 pts)** Start a **Rocky Linux 8** background container, install `bind-utils` inside it, and run `nslookup hub.docker.com` from within. *[exec / package install]*
3. **(20 pts)** Write a **`Dockerfile.rocky`** based on Rocky 8 that installs `bind-utils`, build it as `rocky-dns`, and prove `nslookup` works from a container of that image. *[building images]*
4. **(20 pts)** Run **WordPress + MySQL** as two containers on a shared **Docker network**, with WordPress reachable on host port **8000**. Prove WordPress loads. *[multi-container]*
5. **(20 pts)** Reproduce the same WordPress+MySQL stack with a **`docker-compose.yml`**; bring it up and down with one command each. *[compose]*
6. **(15 pts)** Attach a **named volume** to the MySQL container's data dir so the database **survives container removal**. Prove persistence. *[volumes]*

---

## Answer key

<details><summary>Task 1 — background container</summary>

```bash
sudo docker run -d gitlab.cs.pub.ro:5050/scgc/cloud-courses/ubuntu:18.04 sleep 10000
sudo docker ps                                  # note the ID
sudo docker exec -it <id> /bin/bash
exit
sudo docker stop <id>
```
</details>

<details><summary>Task 2 — Rocky + bind-utils</summary>

```bash
sudo docker run -d gitlab.cs.pub.ro:5050/scgc/cloud-courses/rockylinux:8 sleep 10000
sudo docker ps
sudo docker exec -it <id> /bin/bash
# inside:
yum install -y bind-utils
nslookup hub.docker.com
exit
```
</details>

<details><summary>Task 3 — Dockerfile.rocky</summary>

`Dockerfile.rocky`:
```dockerfile
FROM gitlab.cs.pub.ro:5050/scgc/cloud-courses/rockylinux:8
RUN yum install -y bind-utils
```
Build & test:
```bash
docker build -t rocky-dns -f Dockerfile.rocky .
docker image list
docker run -it rocky-dns nslookup hub.docker.com
```
</details>

<details><summary>Task 4 — WordPress + MySQL (manual)</summary>

```bash
sudo docker network create test-net

sudo docker run -d --hostname db --network test-net \
  -e "MYSQL_ROOT_PASSWORD=somewordpress" -e "MYSQL_DATABASE=wordpress" \
  -e "MYSQL_USER=wordpress" -e "MYSQL_PASSWORD=wordpress" \
  mysql:5.7

sudo docker run -d --hostname wordpress --network test-net -p "8000:80" \
  -e "WORDPRESS_DB_HOST=db" -e "WORDPRESS_DB_USER=wordpress" \
  -e "WORDPRESS_DB_PASSWORD=wordpress" \
  gitlab.cs.pub.ro:5050/scgc/cloud-courses/wordpress:latest
```
Browse `http://<vm-ip>:8000`.
</details>

<details><summary>Task 5 — docker compose</summary>

`docker-compose.yml`:
```yaml
services:
  db:
    image: mysql:5.7
    networks: [wordpress-net]
    environment:
      MYSQL_ROOT_PASSWORD: somewordpress
      MYSQL_DATABASE: wordpress
      MYSQL_USER: wordpress
      MYSQL_PASSWORD: wordpress
  wordpress:
    depends_on: [db]
    image: wordpress:latest
    networks: [wordpress-net]
    ports: ["8000:80"]
    environment:
      WORDPRESS_DB_HOST: db
      WORDPRESS_DB_USER: wordpress
      WORDPRESS_DB_PASSWORD: wordpress
      WORDPRESS_DB_NAME: wordpress
networks:
  wordpress-net:
```
```bash
sudo docker-compose up -d
sudo docker-compose down
```
</details>

<details><summary>Task 6 — persistent volume</summary>

```bash
sudo docker run -d -v mysql-volume:/var/lib/mysql \
  -e "MYSQL_ROOT_PASSWORD=somewordpress" -e "MYSQL_DATABASE=wordpress" \
  -e "MYSQL_USER=wordpress" -e "MYSQL_PASSWORD=wordpress" \
  mysql:5.7
```
Prove: create data, `docker rm -f` the container, start a new one with the **same** `-v mysql-volume:/var/lib/mysql`, and the data is still there.
</details>

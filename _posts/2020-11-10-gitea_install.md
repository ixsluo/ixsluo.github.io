---

title: Gitea Installation Guide
layout: post
date: 2020-11-10 +0800

# Gitea
---

## Install with Docker-Compose

Relay on `docker`, `docker-compose`

Add user "git" to group "docker" to run docker-compose in user "git"
```sh
gpassed -a git docker
```

Docker-compose
```sh
su git
mkdir gitea ; cd gitea
cat > docker-compose.yml << EOF
version: "3"

networks:
  gitea:
    external: false

services:
  server:
    image: gitea/gitea:1
    container_name: gitea
    environment:
      - USER_UID=1002  # UID of git
      - USER_GID=1002  # GID of git
    restart: always
    networks:
      - gitea
    volumes:
      - /home/git/data:/data  # mount, HOST:docker
      - /etc/timezone:/etc/timezone:ro  # ro: read only
      - /etc/localtime:/etc/localtime:ro
    ports:  # port mapping, HOST:docker
      - "10080:3000"  # web for gitea
      - "10022:22"  # ssh for gitea
EOF
docker-compose up -d  # run in background
```

Visit `ip:10080/install`

(You may make a port mapping on router \<outer ip\>, 10022->10022, 10080->10080)
- simplily choose SQLite3 for database
- fill SSH domain and port with **External Domain and Port**
  - `<outer ip>:10022`
- fill HTTP port with **Internal Port**
  - default `3000`
- fill basic URL with **External URL**
  - `http://<outer ip>:10080`

Done!

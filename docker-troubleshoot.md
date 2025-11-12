# Docker troubleshoot

Remove Docker.

```bash
sudo apt remove docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

Check the available Docker version

```bash
apt-cache madison docker-ce
```

Install a specific version

```bash
sudo apt install docker-ce=5:28.5.2-1~ubuntu.24.04~noble \
                 docker-ce-cli=5:28.5.2-1~ubuntu.24.04~noble \
                 containerd.io=1.7.28-1~ubuntu.24.04~noble
```

then start and enable the docker container.

```bash
sudo systemctl start docker.service
sudo systemctl enable docker.service
```


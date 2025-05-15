# Gitlab-Docker-CI-CD


# 🧰 Deployment Guide: GitLab + Runner + Registry (HTTPS + auth)

## Prerequisites
### Required open firewall tcp ports
```
8929 - Gitlab
5050 - Registry
2222 - ssh
```
Adjust for your setup.

---
### /etc/hosts (on local PC)
```bash
192.168.1.3 gitlab.local registry.local
```

### Configure ```~/.ssh/config``` (on local PC)
```bash
nano ~/.ssh/config
```

```bash
Host gitlab
  HostName 192.168.1.3
  Port 2222
  User git
  IdentityFile ~/.ssh/id_rsa # use your generated key
```

## 🗂️ Step 1: Create required directory structure

```bash
sudo mkdir -p /srv/gitlab/{config,logs,data,ssl}
sudo mkdir -p /srv/gitlab-runner/certs
sudo mkdir -p /srv/gitlab/config/registry
```

---

## 🔐 Step 2: Generate TLS certificate using `mkcert`

### Install mkcert (if not already installed)

```bash
sudo apt install libnss3-tools
wget https://dl.filippo.io/mkcert/latest?for=linux/amd64 -O mkcert
chmod +x mkcert
sudo mv mkcert /usr/local/bin/
```
### Install local CA
```
mkcert -install
```

### Generate certificate

```bash
mkcert -cert-file /srv/registry/certs/domain.crt \ 
-key-file /srv/registry/certs/domain.key \
localhost 192.168.1.3 Laptop-Krystian
```
You can add other ``/etc/hosts`` aliases.

---

## 🔐 Step 3: Add certificate to the system trust store

```bash
sudo cp /srv/registry/certs/domain.crt /usr/local/share/ca-certificates/registry.crt
sudo update-ca-certificates
```
Repeat the same on all clients (e.g., runner computer or CI).

### Copy certificates to gitlab and gitlab-runner catalogs

```bash
cp /srv/registry/certs/domain.crt /srv/gitlab-runner/certs/192.168.1.3.crt

cp /srv/registry/certs/domain.crt /srv/gitlab/ssl/192.168.1.3.crt
cp /srv/registry/certs/domain.key /srv/gitlab/ssl/192.168.1.3.key
```

---

## 🔐 Step 4: Setup registry authentication (basic auth)

```bash
docker run --rm --entrypoint htpasswd httpd:2 -Bbn user B@j0JAJO | sudo tee /srv/gitlab/config/registry/htpasswd > /dev/null
```

---

## 🐳 Step 5: Prepare required files

Create the following:

- `.env`
- `docker-compose.yml`
- `.gitlab-ci.yml`

Example `.env`:

```ini
HOSTNAME=192.168.1.3
PORT=8929
REGISTRY_PORT=5050
REGISTRY_USER=user
REGISTRY_PASS=B@j0JAJO
TOKEN=r3g1str4t10n
GITLAB_ROOT_PASSWORD=Abcd@0123456789
GITLAB_ROOT_EMAIL=admin@buildwithlal.com
```

---

## 🧱 Step 6: Start the GitLab stack

```bash
docker compose --env-file .env up -d gitlab gitlab-runner
```

---

## 🔁 Step 7: Register GitLab runner

```bash
docker compose run --rm register-gitlab-runner
```

---

## 🧪 Step 8: Test registry access

```bash
curl -k -u user:B@j0JAJO https://192.168.1.3:5050/v2/
# Expected output: {}
```

---

## 🔧 Step 9: Add CI/CD variables to GitLab project

In **Settings → CI/CD → Variables**, add:

| Key           | Value               | Masked | Protected |
|---------------|---------------------|--------|-----------|
| REGISTRY_URL  | 192.168.1.3:5050    | No     | No        |
| REGISTRY_USER | user                | Yes    | No        |
| REGISTRY_PASS | B@j0JAJO            | Yes    | No        |

---

Add ssh key to Gitlab from ```~/.ssh/id_rsa.pub```
```bash
cat ~/.ssh/id_rsa.pub
```
---

## 🚧 Step 10: Common issues and solutions

| Problem                                  | Solution                                                                          |
|------------------------------------------|-----------------------------------------------------------------------------------|
| `x509: unknown authority`                | Make sure the `.crt` file is trusted and available in `/etc/docker/certs.d/`     |
| `Runner has never contacted this instance` | Ensure runner has the certificate in `/etc/gitlab-runner/certs/`                 |
| `job pending`                            | Add `tags: [docker]` in `.gitlab-ci.yml`, set `privileged = true` in `config.toml` |
| `docker login` fails                     | Regenerate `htpasswd` file and ensure credentials are correct                    |

---

## 🧪 Step 11: Test image push/pull

```bash
docker login 192.168.1.3:5050
docker build -t 192.168.1.3:5050/myapp:test .
docker push 192.168.1.3:5050/myapp:test
docker pull 192.168.1.3:5050/myapp:test
```

---

✅ **Done! Your GitLab + CI/CD + Registry stack is fully operational with HTTPS and authentication.**



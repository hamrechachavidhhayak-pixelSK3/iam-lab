# DevOps Lab: Docker on AWS EC2

**Course:** Identity Access and Management (22CY73)
**Platform:** AWS EC2, Amazon Linux 2023 (Region: ap-south-1, Mumbai)
**Client:** Windows 10/11, PowerShell with the built-in OpenSSH client

This repository documents two experiments:

- **Experiment 2:** Installing Docker on an Amazon Linux EC2 instance
- **Experiment 5:** Building a custom Docker image (Python Flask app) and running it as a container on EC2

---

## Prerequisites (common to both experiments)

1. An active AWS account.
2. A running **EC2 instance** using the **Amazon Linux 2023** AMI (t2.micro / t3.micro is enough).
3. A **key pair** (`.pem` file) downloaded when the instance was launched.
4. The instance's **Security Group** must allow these inbound rules:

   | Type       | Protocol | Port | Source                  | Needed for   |
   |------------|----------|------|-------------------------|--------------|
   | SSH        | TCP      | 22   | My IP                   | Both         |
   | Custom TCP | TCP      | 5000 | 0.0.0.0/0 (or My IP)    | Experiment 5 |

5. The instance's **Public IPv4 address**, from EC2 Console → Instances → select instance.

> **Note:** The public IP changes every time the instance is stopped and started, unless an Elastic IP is attached. Always check the current IP in the console before connecting.

---

# Experiment 2: Install Docker on an AWS Linux EC2 Instance

## Aim
To install, configure, and verify Docker Engine and Docker Compose on an Amazon Linux EC2 instance.

## Step 1: Fix private key file permissions (Windows)

OpenSSH refuses to use a key file that other users can read. On Windows this produces:

```
WARNING: UNPROTECTED PRIVATE KEY FILE!
Permissions for 'C:\Users\<you>\Downloads\myserverkey.pem' are too open.
Load key "...myserverkey.pem": bad permissions
Permission denied (publickey,gssapi-keyex,gssapi-with-mic).
```

To fix it, remove inherited permissions and give read access only to your user. Run in PowerShell:

```powershell
# Remove inherited permissions
icacls "C:\Users\<you>\Downloads\myserverkey.pem" /inheritance:r

# Grant read-only access to the current user only
icacls "C:\Users\<you>\Downloads\myserverkey.pem" /grant:r "$($env:USERNAME):(R)"

# Verify: only your user should be listed
icacls "C:\Users\<you>\Downloads\myserverkey.pem"
```

Expected output:
```
C:\Users\<you>\Downloads\myserverkey.pem LAPTOP-XXXX\<you>:(R)
Successfully processed 1 files; Failed processing 0 files
```

> **Linux/macOS equivalent:** `chmod 400 myserverkey.pem`

## Step 2: Connect to the EC2 instance via SSH

```powershell
ssh -o IdentitiesOnly=yes -i "C:\Users\<you>\Downloads\myserverkey.pem" ec2-user@<EC2-PUBLIC-IP>
```

- `-i` selects the private key file.
- `-o IdentitiesOnly=yes` makes SSH use only that key instead of trying others first.
- `ec2-user` is the default username for Amazon Linux.

> **Troubleshooting:** `Permission denied (publickey...)` with no permission warning means you are using the **wrong key** for this instance. Check the key pair name in EC2 Console → Instance → Details → "Key pair assigned at launch".

On success, the Amazon Linux 2023 banner appears and the prompt becomes:
```
[ec2-user@ip-172-31-xx-xx ~]$
```

## Step 3: Update the system packages

```bash
sudo yum update -y
```

This refreshes the repository metadata and installs any pending updates. On a fresh instance it may simply say `Nothing to do. Complete!`.

> On Amazon Linux 2023, `yum` is an alias for `dnf`. Both work.

## Step 4: Install Docker

```bash
sudo yum install -y docker
```

This installs Docker Engine and its dependencies:

| Package                 | Purpose                                        |
|-------------------------|------------------------------------------------|
| `docker`                | Docker Engine and CLI                          |
| `containerd`            | Container runtime used by Docker               |
| `runc`                  | Low-level runtime that actually starts containers |
| `iptables-nft`, `libnftnl` etc. | Container networking / port forwarding |
| `container-selinux`     | SELinux policies for containers                |
| `pigz`                  | Parallel gzip for faster image layer extraction |

Expected ending:
```
Installed:
  docker-25.x  containerd-2.x  runc-1.x  ...
Complete!
```

## Step 5: Start the Docker service

```bash
sudo systemctl start docker
```
(Equivalent older syntax: `sudo service docker start`)

## Step 6: Enable Docker to start on boot

```bash
sudo systemctl enable docker
```

Expected output:
```
Created symlink /etc/systemd/system/multi-user.target.wants/docker.service → /usr/lib/systemd/system/docker.service.
```

## Step 7: Verify the Docker service is running

```bash
sudo systemctl status docker
```

Look for:
```
Active: active (running)
```
Press `q` to exit the status view.

## Step 8: Add ec2-user to the docker group (run Docker without sudo)

```bash
sudo usermod -aG docker ec2-user
```

Group changes only apply to new login sessions, so log out and reconnect:

```bash
exit
```
```powershell
ssh -o IdentitiesOnly=yes -i "C:\Users\<you>\Downloads\myserverkey.pem" ec2-user@<EC2-PUBLIC-IP>
```

Confirm membership:
```bash
groups
# output should include: docker
```

## Step 9: Verify the Docker installation

```bash
docker --version
```
Expected: `Docker version 25.x.x, build xxxxxxx`

Run the test container:
```bash
docker run hello-world
```
Expected (abridged):
```
Hello from Docker!
This message shows that your installation appears to be working correctly.
```

> If you get `permission denied while trying to connect to the Docker daemon socket`, you did not log out and back in after Step 8. Either reconnect or prefix the command with `sudo`.

## Step 10 (Optional): Install Docker Compose

```bash
sudo curl -L "https://github.com/docker/compose/releases/download/1.29.2/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
sudo chmod +x /usr/local/bin/docker-compose
docker-compose --version
```
Expected: `docker-compose version 1.29.2, build ...`

> **Note:** 1.29.2 is the legacy Compose v1 used in the lab manual. Modern Docker uses the Compose v2 plugin (`docker compose`, with a space), but v1 is fine for this lab.

## Step 11 (Optional): Log in to AWS ECR

Only needed if you will pull or push images from Amazon Elastic Container Registry:
```bash
aws ecr get-login-password --region <region> | sudo docker login --username AWS --password-stdin <aws-account-id>.dkr.ecr.<region>.amazonaws.com
```

## Result
Docker Engine was installed, started, enabled at boot, and verified on an Amazon Linux 2023 EC2 instance by running the `hello-world` container.

## Screenshots for record
1. Successful SSH login showing the Amazon Linux 2023 banner.
2. Output of `docker --version` and `docker run hello-world`.

---

# Experiment 5: Build a Custom Docker Image and Run a Container

## Aim
To containerize a simple Python Flask web application using a Dockerfile, build a custom image, and run it as a container on AWS EC2.

## Prerequisite
Docker must be installed and **running** on the instance (complete Experiment 2 first).

## Step 1: Connect to the EC2 instance

```powershell
ssh -o IdentitiesOnly=yes -i "C:\Users\<you>\Downloads\myserverkey.pem" ec2-user@<EC2-PUBLIC-IP>
```

## Step 2: Make sure the Docker daemon is running

This step is important. If Docker is installed but not started, every Docker command fails with:

```
ERROR: Cannot connect to the Docker daemon at unix:///var/run/docker.sock. Is the docker daemon running?
```

Start and enable it, then check the status:
```bash
sudo systemctl start docker
sudo systemctl enable docker
sudo systemctl status docker      # must show: Active: active (running)
```

## Step 3: Create a project directory

```bash
mkdir -p ~/python-flask-app
cd ~/python-flask-app
```

## Step 4: Create the Flask application (app.py)

```bash
cat > app.py <<'EOF'
from flask import Flask

app = Flask(__name__)

@app.route('/')
def hello():
    return "Hello, Docker on AWS!"

if __name__ == "__main__":
    app.run(host='0.0.0.0', port=5000)
EOF
```

How the code works:
- `Flask(__name__)` creates the web application.
- `@app.route('/')` maps the root URL to the `hello()` function.
- `host='0.0.0.0'` makes Flask listen on all network interfaces. This is required inside a container; the default `127.0.0.1` would be unreachable from outside the container.
- `port=5000` is the port the app listens on.

## Step 5: Create the dependencies file (requirements.txt)

```bash
echo "Flask" > requirements.txt
```

## Step 6: Create the Dockerfile

```bash
cat > Dockerfile <<'EOF'
FROM python:3.8-slim
WORKDIR /app
COPY . /app
RUN pip install -r requirements.txt
EXPOSE 5000
CMD ["python", "app.py"]
EOF
```

| Instruction                          | Purpose                                                    |
|--------------------------------------|------------------------------------------------------------|
| `FROM python:3.8-slim`               | Base image: a lightweight Debian image with Python 3.8     |
| `WORKDIR /app`                       | Sets `/app` as the working directory inside the image      |
| `COPY . /app`                        | Copies `app.py` and `requirements.txt` into the image      |
| `RUN pip install -r requirements.txt`| Installs Flask at build time                               |
| `EXPOSE 5000`                        | Documents that the container listens on port 5000          |
| `CMD ["python", "app.py"]`           | Default command run when the container starts              |

Verify all three files exist:
```bash
ls
# app.py  Dockerfile  requirements.txt
```

## Step 7: Build the Docker image

```bash
sudo docker build -t python-flask-app .
```
- `-t python-flask-app` tags (names) the image.
- `.` means the build context is the current directory.

Expected ending:
```
[+] Building 9.5s (9/9) FINISHED
 => [1/4] FROM docker.io/library/python:3.8-slim
 => [2/4] WORKDIR /app
 => [3/4] COPY . /app
 => [4/4] RUN pip install -r requirements.txt
 => exporting to image
 => => naming to docker.io/library/python-flask-app
```

Confirm the image exists:
```bash
sudo docker images
```

## Step 8: Run the container

```bash
sudo docker run -d -p 5000:5000 --name flask-container python-flask-app
```
- `-d` runs the container in the background (detached).
- `-p 5000:5000` maps port 5000 of the EC2 host to port 5000 of the container.
- `--name flask-container` gives the container a readable name.

Docker prints the full container ID, for example:
```
0f9d1ccc818b06b021d55a524a1dbe882d3a195865c25f59b97b3bb5d04dd4dd
```

## Step 9: Verify the container is running

```bash
sudo docker ps
```
Expected:
```
CONTAINER ID   IMAGE              COMMAND           STATUS         PORTS                    NAMES
0f9d1ccc818b   python-flask-app   "python app.py"   Up 12 seconds  0.0.0.0:5000->5000/tcp   flask-container
```

## Step 10: Test the application from inside the EC2 instance

```bash
curl http://localhost:5000
```
Expected:
```
Hello, Docker on AWS!
```

## Step 11: Access the application from a browser

1. Make sure the Security Group has an inbound rule for **TCP port 5000** (see Prerequisites).
2. Open a browser on your laptop and go to:
```
   http://<EC2-PUBLIC-IP>:5000
```
3. The page should display **Hello, Docker on AWS!**

> If `curl localhost:5000` works but the browser times out, the cause is almost always the missing port 5000 rule in the Security Group. Also make sure you typed `http://`, not `https://`.

## Step 12: Useful management commands

```bash
sudo docker logs flask-container      # view Flask logs
sudo docker stop flask-container      # stop the container
sudo docker start flask-container     # start it again
sudo docker rm -f flask-container     # remove the container
sudo docker rmi python-flask-app      # remove the image
```

> **Note:** `docker run --name flask-container ...` fails if a container with that name already exists. Remove the old one first with `sudo docker rm -f flask-container`.

## Result
A custom Docker image `python-flask-app` was built from a Dockerfile and run as the container `flask-container` on AWS EC2. The application returned "Hello, Docker on AWS!" both via `curl` on the instance and through a browser at the instance's public IP on port 5000.

## Screenshots for record
1. Output of `sudo docker build` showing `FINISHED`, and `sudo docker ps` showing the running container.
2. Browser at `http://<EC2-PUBLIC-IP>:5000` showing "Hello, Docker on AWS!" (or the `curl` output).

---

## Cleanup (to avoid AWS charges)

When finished, stop or terminate the instance from EC2 Console → Instances → Instance state → Stop / Terminate.

## Troubleshooting summary

| Error | Cause | Fix |
|-------|-------|-----|
| `UNPROTECTED PRIVATE KEY FILE` / `bad permissions` | `.pem` file readable by other users | `icacls ... /inheritance:r` then `/grant:r "$($env:USERNAME):(R)"` |
| `Permission denied (publickey)` | Wrong key for this instance, or wrong username | Use the key pair assigned at launch; user is `ec2-user` |
| `Cannot connect to the Docker daemon` | Docker service not started | `sudo systemctl start docker` |
| `permission denied ... docker.sock` | User not in docker group yet | Re-login after `usermod`, or use `sudo` |
| Browser can't reach port 5000 | Security Group blocks the port | Add inbound rule: TCP 5000 |
| `Conflict. The container name is already in use` | Old container with same name | `sudo docker rm -f flask-container` |

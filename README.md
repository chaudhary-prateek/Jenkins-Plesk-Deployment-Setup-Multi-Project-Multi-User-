
* User setup per project
* SSH key sharing from Jenkins
* Permissions management
* Root access
* Directory handling

---

# 🚀 Jenkins Deployment to Plesk — Multi-User Setup

## 🧠 Overview

You have:

* A **Plesk server** with main user: `ubuntu`
* **Multiple projects**, each should have:

  * Its **own Linux user**
  * A **dedicated Jenkins pipeline**
  * Jenkins server should deploy code to that user's project directory
* The `ubuntu` user (main user) should have full access to all project directories for supervision.

---

## 🛠️ Step-by-Step Setup

### 🔐 1. Create Project-Specific User

Each project will have its own user.

```bash
sudo adduser prateek
```

Explanation:

* `adduser`: Creates a new user `prateek`
* It will prompt for password and user details

### 🔑 2. Generate SSH Key on Jenkins Server (if not already present)

On Jenkins server:

```bash
ssh-keygen -t rsa -b 4096 -C "jenkins@plesk-project"
```

Explanation:

* `-t rsa`: Key type
* `-b 4096`: Key size
* Saves key in default path `~/.ssh/id_rsa`

### 📤 3. Copy Jenkins Public Key to Project User on Plesk

Use `ssh-copy-id` or manually copy:

```bash
ssh-copy-id prateek@your_plesk_ip
```

Or:

```bash
cat ~/.ssh/id_rsa.pub  # Jenkins server public key
```

On Plesk (logged in as `ubuntu`):

```bash
sudo mkdir -p /home/prateek/.ssh
sudo nano /home/prateek/.ssh/authorized_keys
# Paste Jenkins public key
```

```bash
sudo chown -R prateek:prateek /home/prateek/.ssh
sudo chmod 700 /home/prateek/.ssh
sudo chmod 600 /home/prateek/.ssh/authorized_keys
```

### 📁 4. Set Project Directory Ownership

Create or use existing project directory (e.g., `/var/www/html/prateek`):

```bash
sudo mkdir -p /var/www/html/prateek
sudo chown -R prateek:prateek /var/www/html/prateek
```

### 👁️ 5. Let Jenkins Access the Project Directory via SSH

In your Jenkins pipeline:

```groovy
pipeline {
  agent any
  stages {
    stage('Connect to Plesk') {
      steps {
        sshagent(['YOUR_SSH_CREDENTIAL_ID']) {
          sh '''
            ssh -o StrictHostKeyChecking=no prateek@plesk_server_ip 'cd /var/www/html/prateek && ls'
          '''
        }
      }
    }
  }
}
```

Make sure Jenkins has the SSH private key added in **Credentials → SSH Username with private key**.

---

## 🛡️ 6. Root (Ubuntu) Access to All Project Directories

Project directories are usually owned like:

```bash
drwx--x--- prateek psaserv ...
```

To allow the `ubuntu` user to access:

### Option 1: Add Ubuntu to Each User's Group

```bash
sudo usermod -aG psaserv ubuntu
```

OR

### Option 2: Add Ubuntu to Each Project User’s Group

```bash
sudo usermod -aG prateek ubuntu
```

Then reload:

```bash
newgrp prateek
```

### Option 3: Give Full Permission to Ubuntu User (Root Level)

```bash
sudo chmod -R 775 /var/www/html/prateek
sudo chown -R prateek:ubuntu /var/www/html/prateek
```

---

## 🔓 7. Give Root Access to Ubuntu User (If Not Already)

Ubuntu usually has `sudo` rights. Verify:

```bash
getent group sudo
```

Add if not listed:

```bash
sudo usermod -aG sudo ubuntu
```

Now test root access:

```bash
sudo su
```

---

## ✅ Final Notes

* **Each project should have a dedicated user** for isolation and permission control.
* Jenkins connects to that user's account via SSH and executes deployment commands.
* `ubuntu` user has visibility over all directories via group memberships or adjusted permissions.
* This setup ensures **secure**, **modular**, and **scalable** deployments via Jenkins.

---

## 📦 Summary Table

| Purpose                    | Command / Config                              |
| -------------------------- | --------------------------------------------- |
| Create user                | `sudo adduser prateek`                        |
| Add Jenkins SSH key        | Paste to `/home/prateek/.ssh/authorized_keys` |
| Give directory ownership   | `chown -R prateek:prateek /project/path`      |
| Give root access to ubuntu | `usermod -aG sudo ubuntu`                     |
| Add ubuntu to group        | `usermod -aG <group> ubuntu`                  |
| Jenkins SSH usage          | Use `sshagent` in pipeline                    |

---


# Jenkins Multi-Project SSH Deployment Setup On Same Server

A comprehensive guide for setting up secure, isolated Jenkins deployment on Linux servers using SSH key authentication and user-based directory isolation.

## 📋 Overview

This repository provides a complete solution for deploying multiple projects from different Jenkins servers to a single Linux server while maintaining strict security isolation. Each project gets its own dedicated user account and directory, accessible only via SSH keys.

### 🎯 Key Features

- **🔒 Complete Isolation**: Each project has its own user and directory
- **🔑 SSH Key Authentication**: Password-less, secure access
- **🌐 Multi-Jenkins Support**: Different Jenkins servers for different projects
- **📁 Organized Structure**: Clean directory hierarchy
- **⚡ Production Ready**: Follows Linux security best practices


### 🏗️ Architecture

```
Linux Server
├── project1 user → /var/www/html/project1/ ← Jenkins1 (SSH Key Auth)
├── project2 user → /var/www/html/project2/ ← Jenkins2 (SSH Key Auth)
└── Admin users (ubuntu, root) → Full Access
```


## 🚀 Quick Start

### Prerequisites

- Linux server (Ubuntu/Debian/CentOS)
- Sudo access
- Web server (Apache/Nginx) - optional for web deployment
- Jenkins servers with SSH key pairs


### Installation

Run these commands one by one on your Linux server:

```bash
# 1. Create Users
sudo useradd -m -s /bin/bash project1
sudo passwd -l project1
sudo useradd -m -s /bin/bash project2
sudo passwd -l project2

# 2. Create Directories
sudo mkdir -p /var/www/html/project1
sudo mkdir -p /var/www/html/project2

# 3. Set Ownership
sudo chown -R project1:project1 /var/www/html/project1
sudo chown -R project2:project2 /var/www/html/project2

# 4. Configure Group Access
sudo usermod -aG project1 www-data
sudo usermod -aG project2 www-data
sudo usermod -aG project1 root
sudo usermod -aG project2 root
sudo usermod -aG project1 ubuntu
sudo usermod -aG project2 ubuntu

# 5. Set Permissions
sudo chmod -R 755 /var/www/html/project1
sudo chmod -R 755 /var/www/html/project2

# 6. Setup SSH
sudo -u project1 mkdir -p /home/project1/.ssh
sudo -u project1 chmod 700 /home/project1/.ssh
sudo -u project1 touch /home/project1/.ssh/authorized_keys
sudo -u project1 chmod 600 /home/project1/.ssh/authorized_keys

sudo -u project2 mkdir -p /home/project2/.ssh
sudo -u project2 chmod 700 /home/project2/.ssh
sudo -u project2 touch /home/project2/.ssh/authorized_keys
sudo -u project2 chmod 600 /home/project2/.ssh/authorized_keys

# 7. Create Test Files
sudo -u project1 bash -c "echo '<h1>Project1 Ready for Jenkins</h1>' > /var/www/html/project1/index.html"
sudo -u project2 bash -c "echo '<h1>Project2 Ready for Jenkins</h1>' > /var/www/html/project2/index.html"
```


### Add Jenkins SSH Keys

Replace `YOUR_JENKINS_PUBLIC_KEY` with your actual Jenkins server SSH public keys:

```bash
# For Jenkins Server 1 → project1
sudo -u project1 bash -c "echo 'ssh-rsa YOUR_JENKINS1_PUBLIC_KEY jenkins1' >> /home/project1/.ssh/authorized_keys"

# For Jenkins Server 2 → project2
sudo -u project2 bash -c "echo 'ssh-rsa YOUR_JENKINS2_PUBLIC_KEY jenkins2' >> /home/project2/.ssh/authorized_keys"
```


## 🔐 Security Model

### User Isolation Matrix

| User | project1 Directory | project2 Directory | System Access |
| :-- | :-- | :-- | :-- |
| **project1** | ✅ Full Access | ❌ No Access | 🔒 Limited |
| **project2** | ❌ No Access | ✅ Full Access | 🔒 Limited |
| **ubuntu/root** | ✅ Admin Access | ✅ Admin Access | 👑 Full |
| **www-data** | ✅ Read Only | ✅ Read Only | 🌐 Web Server |

### Permission Structure

```
/var/www/html/project1/    (Owner: project1, Group: project1, Mode: 755)
├── index.html             (project1 can read/write, others can read)
├── assets/                (project1 can manage, www-data can serve)
└── logs/                  (project1 deployment logs)

/var/www/html/project2/    (Owner: project2, Group: project2, Mode: 755)
├── index.html             (project2 can read/write, others can read)
├── assets/                (project2 can manage, www-data can serve)
└── logs/                  (project2 deployment logs)
```


## 🔧 Jenkins Configuration

### SSH Connection Setup

In your Jenkins servers, configure SSH connections:

**Jenkins Server 1 (for project1):**

```yaml
Host: YOUR_SERVER_IP
Username: project1
Private Key: /path/to/jenkins1_private_key
Deploy Directory: /var/www/html/project1/
```

**Jenkins Server 2 (for project2):**

```yaml
Host: YOUR_SERVER_IP
Username: project2
Private Key: /path/to/jenkins2_private_key
Deploy Directory: /var/www/html/project2/
```


### Jenkins Pipeline Example

```groovy
pipeline {
    agent any
    
    stages {
        stage('Deploy to Server') {
            steps {
                sshagent(['project1-ssh-key']) {
                    sh '''
                        # Create backup
                        ssh project1@${SERVER_IP} "tar -czf /var/www/html/project1/backup_$(date +%Y%m%d_%H%M%S).tar.gz -C /var/www/html/project1 . || true"
                        
                        # Deploy files
                        scp -r ./build/* project1@${SERVER_IP}:/var/www/html/project1/
                        
                        # Set permissions
                        ssh project1@${SERVER_IP} "find /var/www/html/project1 -type f -name '*.html' -exec chmod 644 {} \\;"
                        
                        # Verify deployment
                        ssh project1@${SERVER_IP} "ls -la /var/www/html/project1/"
                    '''
                }
            }
        }
    }
}
```


## ✅ Verification \& Testing

### Quick Verification Commands

```bash
# Check users exist
id project1
id project2

# Check directories
ls -la /var/www/html/

# Check SSH setup
ls -la /home/project1/.ssh/
ls -la /home/project2/.ssh/

# Test access permissions
sudo -u project1 test -w /var/www/html/project1 && echo "✅ project1 access OK"
sudo -u project2 test -w /var/www/html/project2 && echo "✅ project2 access OK"

# Test web access (if web server installed)
curl http://localhost/project1/
curl http://localhost/project2/
```


### SSH Connection Test

From your Jenkins servers, test SSH connectivity:

```bash
# Test from Jenkins Server 1
ssh project1@YOUR_SERVER_IP "ls -la /var/www/html/project1/"

# Test from Jenkins Server 2
ssh project2@YOUR_SERVER_IP "ls -la /var/www/html/project2/"
```


## 🔍 Troubleshooting

### Common Issues

**1. Permission Denied on SSH**

```bash
# Check SSH key permissions
ls -la /home/project1/.ssh/
# Should be: drwx------ (700) for .ssh directory
# Should be: -rw------- (600) for authorized_keys file

# Fix if needed
sudo -u project1 chmod 700 /home/project1/.ssh
sudo -u project1 chmod 600 /home/project1/.ssh/authorized_keys
```

**2. Cannot Write to Deployment Directory**

```bash
# Check directory ownership
ls -ld /var/www/html/project1/
# Should be: drwxr-xr-x project1 project1

# Fix if needed
sudo chown -R project1:project1 /var/www/html/project1/
sudo chmod -R 755 /var/www/html/project1/
```

**3. Web Server Cannot Access Files**

```bash
# Check if www-data is in project group
groups www-data

# Add if missing
sudo usermod -aG project1 www-data
sudo usermod -aG project2 www-data
```


### Debug Commands

```bash
# Check user details
id project1
getent passwd project1

# Check group memberships
groups project1
groups www-data

# Check directory permissions
stat /var/www/html/project1/
stat /home/project1/.ssh/

# Check SSH key format
sudo -u project1 ssh-keygen -l -f /home/project1/.ssh/authorized_keys
```


## 📈 Scaling to More Projects

To add project3, project4, etc.:

```bash
# Create new user
sudo useradd -m -s /bin/bash project3
sudo passwd -l project3

# Create directory
sudo mkdir -p /var/www/html/project3
sudo chown -R project3:project3 /var/www/html/project3
sudo chmod -R 755 /var/www/html/project3

# Configure groups
sudo usermod -aG project3 www-data
sudo usermod -aG project3 root
sudo usermod -aG project3 ubuntu

# Setup SSH
sudo -u project3 mkdir -p /home/project3/.ssh
sudo -u project3 chmod 700 /home/project3/.ssh
sudo -u project3 touch /home/project3/.ssh/authorized_keys
sudo -u project3 chmod 600 /home/project3/.ssh/authorized_keys

# Add Jenkins SSH key
sudo -u project3 bash -c "echo 'ssh-rsa JENKINS3_PUBLIC_KEY jenkins3' >> /home/project3/.ssh/authorized_keys"
```


## 🧹 Cleanup

To remove a project setup:

```bash
# Backup before removal
sudo tar -czf /tmp/project1_backup_$(date +%Y%m%d).tar.gz -C /var/www/html project1

# Remove user and directory
sudo userdel -r project1
sudo rm -rf /var/www/html/project1

# Clean group memberships (automatically handled by userdel)
```


## 🔒 Security Best Practices

### ✅ Do's

- ✅ Use SSH key authentication only
- ✅ Lock user passwords with `passwd -l`
- ✅ Set restrictive permissions (700 for .ssh, 600 for keys)
- ✅ Regular backup before deployments
- ✅ Monitor SSH access logs
- ✅ Use different SSH keys for different projects
- ✅ Regularly rotate SSH keys


### ❌ Don'ts

- ❌ Never set 777 permissions
- ❌ Don't run Jenkins as root
- ❌ Don't share SSH keys between projects
- ❌ Don't allow password authentication
- ❌ Don't give cross-project access


## 📝 Advanced Configuration

### Web Server Virtual Hosts

**Apache Configuration:**

```apache
<VirtualHost *:80>
    ServerName project1.example.com
    DocumentRoot /var/www/html/project1
    ErrorLog ${APACHE_LOG_DIR}/project1_error.log
    CustomLog ${APACHE_LOG_DIR}/project1_access.log combined
</VirtualHost>
```

**Nginx Configuration:**

```nginx
server {
    listen 80;
    server_name project1.example.com;
    root /var/www/html/project1;
    index index.html index.php;
    
    location / {
        try_files $uri $uri/ =404;
    }
}
```


### Automated Backup Script

```bash
#!/bin/bash
# backup-projects.sh
for project in project1 project2; do
    backup_name="/backup/${project}_$(date +%Y%m%d_%H%M%S).tar.gz"
    tar -czf "$backup_name" -C /var/www/html "$project"
    echo "Backup created: $backup_name"
done
```


## 📚 Related Resources

- [SSH Key Management Best Practices](https://www.ssh.com/academy/ssh/keygen)
- [Jenkins SSH Agent Plugin](https://plugins.jenkins.io/ssh-agent/)
- [Linux File Permissions Guide](https://www.linux.com/training-tutorials/understanding-linux-file-permissions/)
- [Nginx Virtual Hosts](https://nginx.org/en/docs/beginners_guide.html)
- [Apache Virtual Hosts](https://httpd.apache.org/docs/2.4/vhosts/)


## 🤝 Contributing

1. Fork the repository
2. Create your feature branch (`git checkout -b feature/improvement`)
3. Test your changes thoroughly
4. Commit your changes (`git commit -am 'Add some improvement'`)
5. Push to the branch (`git push origin feature/improvement`)
6. Create a Pull Request

**⭐ Star this repository if it helped you!**

Made with ❤️ for the DevOps community

# 🔑 EC2 SSH Key Recovery & Management Guide

> A comprehensive guide to recover access to AWS EC2 instances when you've lost your SSH key

## 📋 Table of Contents

- [Overview](#overview)
- [Prerequisites](#prerequisites)
- [Generating a New SSH Key](#generating-a-new-ssh-key)
- [Recovery Methods](#recovery-methods)
- [Connecting to Your Instances](#connecting-to-your-instances)
- [Best Practices](#best-practices)
- [Troubleshooting](#troubleshooting)

## 🎯 Overview

This guide helps you regain access to your EC2 instances when you've lost your SSH private key. It covers multiple recovery methods and best practices for future key management.

## ⚙️ Prerequisites

- AWS Console access
- Basic understanding of SSH and AWS EC2
- Appropriate IAM permissions for your chosen recovery method

## 🔐 Generating a New SSH Key

### On Linux/Mac

```bash
# Generate a new 4096-bit RSA key pair
ssh-keygen -t rsa -b 4096 -f ~/.ssh/my-new-key

# Set appropriate permissions
chmod 600 ~/.ssh/my-new-key
chmod 644 ~/.ssh/my-new-key.pub
```

### On Windows (PowerShell)

```powershell
# Generate a new key pair
ssh-keygen -t rsa -b 4096 -f C:\Users\YourUsername\.ssh\my-new-key
```

### View Your Public Key

```bash
# Linux/Mac
cat ~/.ssh/my-new-key.pub

# Windows (PowerShell)
Get-Content C:\Users\YourUsername\.ssh\my-new-key.pub
```

Copy this public key content - you'll need it for the recovery methods below.

## 🚀 Recovery Methods

### Method 1: EC2 Instance Connect (Easiest) ⭐

**Best for:** Amazon Linux 2, Ubuntu 16.04+, and instances with EC2 Instance Connect preinstalled

**Steps:**

1. Navigate to **AWS Console** → **EC2** → **Instances**
2. Select your instance
3. Click **Connect** button
4. Choose **EC2 Instance Connect** tab
5. Click **Connect**

Once connected, run:

```bash
# Add your new public key
echo "ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAA... your-email@example.com" >> ~/.ssh/authorized_keys

# Set correct permissions
chmod 600 ~/.ssh/authorized_keys
chmod 700 ~/.ssh
```

### Method 2: AWS Systems Manager (Session Manager) 🔧

**Best for:** Instances with SSM Agent installed and proper IAM role

**Requirements:**
- SSM Agent installed on the instance
- IAM role attached with `AmazonSSMManagedInstanceCore` policy

**Steps:**

1. Go to **AWS Console** → **EC2** → **Instances**
2. Select your instance
3. Click **Connect** → **Session Manager** tab
4. Click **Connect**

Once connected:

```bash
# Switch to your user (adjust based on your AMI)
sudo su - ec2-user  # Amazon Linux
# sudo su - ubuntu  # Ubuntu
# sudo su - centos  # CentOS

# Add your new public key
echo "ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAA... your-email@example.com" >> ~/.ssh/authorized_keys

# Set correct permissions
chmod 600 ~/.ssh/authorized_keys
chmod 700 ~/.ssh
```

### Method 3: User Data Script (For Stopped Instances) 📝

**Best for:** Instances that can be temporarily stopped

**Steps:**

1. **Stop** the instance (not terminate!)
2. Select instance → **Actions** → **Instance Settings** → **Edit User Data**
3. Add this script:

```bash
#!/bin/bash
# Replace ec2-user with your actual username (ubuntu, centos, admin, etc.)
echo "ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAA... your-email@example.com" >> /home/ec2-user/.ssh/authorized_keys
chmod 600 /home/ec2-user/.ssh/authorized_keys
chown ec2-user:ec2-user /home/ec2-user/.ssh/authorized_keys
```

4. **Start** the instance
5. The script runs once at boot and adds your key

⚠️ **Important:** Remove the user data after successful connection for security!

### Method 4: Volume Attachment (Last Resort) 💾

**Best for:** When all other methods fail

**Steps:**

1. **Stop** the target instance
2. Note the instance's Availability Zone
3. Go to **Volumes** in EC2 console
4. Find and **detach** the root volume from your instance
5. Launch a temporary instance in the **same Availability Zone**
6. **Attach** the detached volume to the temporary instance as `/dev/sdf`

On the temporary instance:

```bash
# Create mount point
sudo mkdir /mnt/recovery

# Mount the volume (adjust device name if needed)
sudo mount /dev/xvdf1 /mnt/recovery

# Add your public key
sudo bash -c 'echo "ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAA... your-email@example.com" >> /mnt/recovery/home/ec2-user/.ssh/authorized_keys'

# Set proper permissions
sudo chmod 600 /mnt/recovery/home/ec2-user/.ssh/authorized_keys
sudo chown 1000:1000 /mnt/recovery/home/ec2-user/.ssh/authorized_keys

# Unmount
sudo umount /mnt/recovery
```

7. **Detach** the volume from temporary instance
8. **Reattach** to original instance as `/dev/xvda` (or original device name)
9. **Start** the original instance
10. **Terminate** the temporary instance

## 🔌 Connecting to Your Instances

After adding your new key, connect using:

```bash
# Basic connection
ssh -i ~/.ssh/my-new-key ec2-user@your-instance-public-ip

# With specific username
ssh -i ~/.ssh/my-new-key ubuntu@your-instance-public-ip

# Using SSH config for convenience
```

### Create SSH Config for Easy Access

Edit `~/.ssh/config`:

```
Host my-ec2-instance
    HostName 54.123.45.67
    User ec2-user
    IdentityFile ~/.ssh/my-new-key
    
Host my-ubuntu-instance
    HostName 54.123.45.68
    User ubuntu
    IdentityFile ~/.ssh/my-new-key
```

Now you can connect simply with:

```bash
ssh my-ec2-instance
```

### Common Usernames by AMI

| AMI Type | Default Username |
|----------|-----------------|
| Amazon Linux 2 | `ec2-user` |
| Ubuntu | `ubuntu` |
| Debian | `admin` |
| CentOS | `centos` |
| RHEL | `ec2-user` |
| SUSE | `ec2-user` |
| Fedora | `fedora` |

## 🛡️ Best Practices

### Key Management

1. **Use a Password Manager**
   - Store SSH keys in AWS Secrets Manager or a password manager
   - Use 1Password, LastPass, or Bitwarden for personal keys

2. **Multiple Key Pairs**
   - Use different keys for different environments (dev, staging, prod)
   - Rotate keys periodically (every 90-180 days)

3. **Backup Your Keys**
   ```bash
   # Create an encrypted backup
   tar -czf ssh-keys-backup.tar.gz ~/.ssh/
   gpg -c ssh-keys-backup.tar.gz
   ```

### Security Hardening

1. **Disable Password Authentication**
   
   Edit `/etc/ssh/sshd_config`:
   ```
   PasswordAuthentication no
   PubkeyAuthentication yes
   ```

2. **Use SSH Agent**
   ```bash
   # Start SSH agent
   eval "$(ssh-agent -s)"
   
   # Add your key
   ssh-add ~/.ssh/my-new-key
   ```

3. **Enable AWS Systems Manager**
   - Set up Session Manager for keyless access
   - Reduces attack surface by eliminating open SSH ports

4. **Use Bastion Hosts**
   - Place instances in private subnets
   - Access through a hardened bastion host

5. **Regular Audits**
   ```bash
   # Check authorized keys on your instance
   cat ~/.ssh/authorized_keys
   
   # Remove unused keys
   ```

## 🔍 Troubleshooting

### Connection Refused

```bash
# Check security group allows SSH (port 22)
# Verify instance is running
# Confirm you're using correct IP/DNS
```

### Permission Denied (publickey)

```bash
# Check key permissions
chmod 600 ~/.ssh/my-new-key

# Verify you're using correct username
ssh -v -i ~/.ssh/my-new-key ec2-user@instance-ip
```

### Instance Not Accessible via Session Manager

```bash
# Check SSM Agent status
sudo systemctl status amazon-ssm-agent

# Restart if needed
sudo systemctl restart amazon-ssm-agent
```

### "Unprotected Private Key" Error

```bash
# Fix permissions
chmod 600 ~/.ssh/my-new-key
```

## 📚 Additional Resources

- [AWS EC2 Key Pairs Documentation](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-key-pairs.html)
- [AWS Systems Manager Session Manager](https://docs.aws.amazon.com/systems-manager/latest/userguide/session-manager.html)
- [SSH Best Practices](https://www.ssh.com/academy/ssh/best-practices)

## 📝 License

This documentation is provided as-is for educational purposes.

---

**⚠️ Security Reminder:** Never commit private keys to Git! Always add `*.pem` and private key files to your `.gitignore`.

**🔒 Pro Tip:** Consider using AWS Session Manager with port forwarding to completely eliminate the need for SSH keys while maintaining secure access to your instances.

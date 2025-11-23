# **How to SSH into a Public EC2 Instance Using a Key Pair**

This guide explains step-by-step how to establish a secure SSH connection from your local machine to a **public EC2 instance** using a **key pair (.pem file)**. It includes instructions for Linux, macOS, Windows, SSH config, security requirements, and troubleshooting.

---

# ✅ **1. Prerequisites**

Before connecting, ensure you have:

* The **private key (.pem)** file downloaded during EC2 creation
* The **public IP** or **public DNS** of your EC2 instance
* The correct **SSH username** (depends on AMI)
* Security Group allows:

  ```
  Inbound → SSH → Port 22 → Your IP (x.x.x.x/32)
  ```
* Route table has **0.0.0.0/0 → Internet Gateway**

### **Common EC2 usernames:**

| OS           | Username   |
| ------------ | ---------- |
| Amazon Linux | `ec2-user` |
| Ubuntu       | `ubuntu`   |
| CentOS       | `centos`   |
| RHEL         | `ec2-user` |
| Debian       | `admin`    |

---

# 🖥️ **2. SSH from Linux & macOS**

### **Step 1: Fix key permissions**

```bash
chmod 400 my-key.pem
```

### **Step 2: Connect to EC2**

```bash
ssh -i my-key.pem ec2-user@<EC2_PUBLIC_IP>
```

### **Examples:**

```bash
ssh -i my-key.pem ec2-user@44.201.120.15
ssh -i my-key.pem ubuntu@44.201.120.15
```

---

# 🪟 **3. SSH from Windows (PowerShell / CMD)**

Windows supports SSH natively.

```powershell
ssh -i C:\Users\User\Downloads\my-key.pem ec2-user@44.201.120.15
```

### ✔ If using PuTTY:

* Convert `.pem` → `.ppk` using **PuTTYgen**.
* Connect using PuTTY with SSH key authentication.

---

# 🌐 **4. Connect using Public DNS instead of IP**

```bash
ssh -i my-key.pem ec2-user@ec2-44-201-120-15.compute-1.amazonaws.com
```

---

# ⚙️ **5. Using SSH Config File (Shortcut Method)**

Create/Edit:

```
~/.ssh/config
```

Add:

```
Host my-ec2
    HostName 44.201.120.15
    User ec2-user
    IdentityFile ~/.ssh/my-key.pem
```

Now connect with:

```bash
ssh my-ec2
```

---

# 🔐 **6. Security Group Requirements**

### **Inbound rule required:**

| Type | Protocol | Port | Source               |
| ---- | -------- | ---- | -------------------- |
| SSH  | TCP      | 22   | Your IP (x.x.x.x/32) |

❌ **Avoid:**

```
SSH open to world: 0.0.0.0/0
```

(Use only for temporary testing.)

---

# 🚫 **7. Troubleshooting SSH Issues**

### ❌ *Permission denied (publickey)*

Fix:

* Wrong username → use correct AMI username
* Wrong key permissions →

  ```bash
  chmod 400 my-key.pem
  ```

### ❌ *Connection timed out*

Possible causes:

* SG rule missing for port 22
* NACL blocking port 22
* No public IP assigned
* Route table missing IGW entry
* Internet Gateway not attached

### ❌ *Unprotected Private Key File*

Fix:

```bash
chmod 400 my-key.pem
```

### ❌ *No such file or directory*

* Incorrect path to PEM
* Incorrect remote directory

---

# 🧪 **8. Test Connectivity Before SSH**

Check port 22 reachability:

```bash
nc -zv <EC2_PUBLIC_IP> 22
```

OR

```bash
telnet <EC2_PUBLIC_IP> 22
```

---

# 🎯 **Summary**

To SSH into a public EC2 instance using a key pair:

1. Fix permissions:

   ```bash
   chmod 400 key.pem
   ```
2. Connect:

   ```bash
   ssh -i key.pem ec2-user@public-ip
   ```
3. Ensure:

   * Correct username
   * SG allows port 22
   * Instance has public IP + IGW

---

If you want, I can also add:

* **MD for SSH to Private EC2 via Bastion**
* **MD for SSM Session Manager (SSH without key)**
* **MD for EC2 File Transfer 
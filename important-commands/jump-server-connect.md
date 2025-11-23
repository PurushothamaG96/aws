# **Copy Local Files to a Jump Server (Bastion Host) – In-Depth Guide**

A **Jump Server (Bastion Host)** is an intermediate secure server used to connect to private instances in a VPC. Copying files to a jump server is a common task for deployments, debugging, transferring logs, or setting up tools.

This guide explains all methods to securely copy files from your **local system → jump server**, with examples for Linux, macOS, Windows, SCP, SFTP, RSYNC, SSH tunneling, and AWS SSM.

---

# 📌 **1. Prerequisites**

Before you transfer files, ensure:

* You have the **jump server's public IP** or DNS
* You have the **SSH key pair** (PEM or PPK)
* Your IP is whitelisted in the **jump server Security Group** (SSH port 22)
* The file exists locally
* Correct username (depending on OS)

  * Amazon Linux → `ec2-user`
  * Ubuntu → `ubuntu`
  * CentOS → `centos`
  * RHEL → `ec2-user`

---

# 🚀 **2. Copy File Using SCP (Most Common Method)**

SCP = Secure Copy Protocol (built on SSH)

## **Basic Syntax**

```bash
scp -i /path/to/key.pem /local/file/path username@jump-server-ip:/remote/path
```

## **Example**

```bash
scp -i ~/Downloads/jump-key.pem ./app.zip ec2-user@18.120.45.10:/home/ec2-user/
```

## **Copy a Directory Recursively**

```bash
scp -i ~/.ssh/jump-key.pem -r ./project-folder ec2-user@18.120.45.10:/home/ec2-user/
```

## **Copy to a Specific Port (non-standard SSH port)**

```bash
scp -P 2222 -i ~/.ssh/key.pem ./file.txt ec2-user@54.12.33.22:/tmp/
```

---

# 🔁 **3. Copy Files Using RSYNC (Faster & Resume-Support)**

RSYNC is better for large files or repeated syncs.

## **Example**

```bash
rsync -avz -e "ssh -i ~/.ssh/jump-key.pem" ./local-folder/ ec2-user@18.120.45.10:/home/ec2-user/remote-folder/
```

### Why Use RSYNC?

* Faster
* Resume broken transfers
* Sync only changed files
* Compression supported

---

# 📥 **4. Copy Files Using SFTP (Secure FTP Over SSH)**

Interactive session for file transfer.

## **Start SFTP Session**

```bash
sftp -i ~/.ssh/jump-key.pem ec2-user@18.120.45.10
```

## **Put a File**

```bash
put localfile.txt
```

## **Put a Directory**

```bash
put -r myfolder
```

## **Get a File (Download)**

```bash
get remotefile.txt
```

---

# 🖥️ **5. Copy Files From Windows (PuTTY / WinSCP)**

## **WinSCP (Graphical Tool)**

* Convert `.pem` to `.ppk` using PuTTYgen
* Open WinSCP
* Host: Jump server IP
* Username: ubuntu / ec2-user
* Key file: `.ppk`
* Drag and drop files

## **pscp (command-line for Windows)**

```powershell
pscp -i key.ppk localfile.txt ec2-user@18.120.45.10:/home/ec2-user/
```

---

# 🔐 **6. Copy File Through Jump Server to Private EC2 (ProxyJump)**

If the final instance is in a **private subnet**, use jump server as middle hop.

## **Using SSH ProxyJump**

```bash
scp -i ~/.ssh/key.pem -o ProxyJump=ec2-user@18.120.45.10 \
  ./localfile.sh ubuntu@10.0.2.25:/home/ubuntu/
```

## **Using SSH ProxyCommand** (older method)

```bash
scp -o "ProxyCommand ssh -W %h:%p -i ~/.ssh/key.pem ec2-user@18.120.45.10" \
  -i ~/.ssh/private-key.pem ./file.zip ubuntu@10.0.2.25:/home/ubuntu/
```

---

# ⚡ **7. AWS SSM – Copy Files without SSH or Bastion (Alternative)**

If SSM Agent is enabled → no SSH required.

## **Upload File to S3 First**

```bash
aws s3 cp ./localfile.zip s3://my-bucket/uploads/
```

## **Then download inside EC2**

```bash
aws s3 cp s3://my-bucket/uploads/localfile.zip /home/ec2-user/
```

This avoids exposing port 22 publicly.

---

# 🛡️ **8. Security Best Practices**

* Never expose SSH (22) to `0.0.0.0/0` permanently
* Use **IAM SSM Session Manager** instead of SSH whenever possible
* Enable **MFA** for SSH via Identity Center if required
* Rotate SSH keys regularly
* Restrict jump server security groups to **your IP only**
* Use **logs + CloudTrail** to track access

---

# ❗ **9. Common Troubleshooting**

### 🔸 Permission denied (publickey)

* Wrong username
* Wrong key
* Key file permissions (fix with below):

```bash
chmod 400 ~/.ssh/key.pem
```

### 🔸 Connection timeout

* SG inbound rule for SSH missing
* NACL blocking port 22
* Your IP not whitelisted

### 🔸 "No such file or directory"

* Incorrect remote path
* Missing write permissions

---

# 🎯 **10. Summary**

You can copy a local file to a jump server using multiple secure methods:

* **SCP** – simplest and most common
* **RSYNC** – efficient for large and repeated syncs
* **SFTP** – interactive transfers
* **Windows tools** – WinSCP, pscp
* **ProxyJump** – reach private EC2 through bastion
* **SSM + S3** – no SSH needed

Let me know if you want a dedicated **diagram**, **cheat sheet**, or **SSH config file automation** for jump server setups!

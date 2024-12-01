# ACIT-2420-Assignment-3-Part 2

This project builds upon Part 1 by adding new features such as setting up two servers, adding a load balancer, and creating a file server to host and serve documents. The servers use an updated `generate_index` script, and the load balancer distributes traffic between the two servers. A firewall is configured using `ufw` for security.

---

## Updated Workflow

This updated workflow reflects the enhancements made in Part 2.

```plaintext
+---------------------+
| Create System Users |
| (webgen, home dir)  |
+----------+----------+
           |
           v
+----------------------------+
| Generate Updated Index     |
| (Updated generate_index)   |
+----------+-----------------+
           |
           v
+----------------------------+
| Automate with Systemd      |
| - Service: Runs script     |
| - Timer: Runs daily        |
+----------+-----------------+
           |
           v
+----------------------------+
| Serve with Nginx           |
| - Host index.html and files|
| - Configures HTTP server   |
+----------+-----------------+
           |
           v
+----------------------------+
| Add File Server            |
| - Host /documents          |
| - Serve downloadable files |
+----------+-----------------+
           |
           v
+----------------------------+
| Add Load Balancer          |
| - Distribute traffic       |
| - Ensure high availability |
+----------------------------+
           |
           v
+----------------------------+
| Secure with UFW            |
| - SSH & HTTP allowed       |
| - SSH rate-limited         |
+----------------------------+
           |
           v
+----------------------------+
| Verify                     |
| - Test index.html and      |
|   /documents in browser    |
| - Test load balancer       |
+----------------------------+
```

## Setup Instructions (Updated for Part 2)

### Create Two Servers and Configure the Load Balancer
**1. Create two Arch Linux droplets in `DigitalOcean`:**
- Assign them the tag `web`.
- Ensure they are in the same `region (SFO3)` for load balancing.

**2. Configure a load balancer in DigitalOcean:**
- Region: `SFO3`
- Default `VPC`
- External(`public`)
- Use the tag `web` to balance traffic between the servers.

**3. SSH into both servers:**
```bash
ssh i ~ /.ssh/do_key arch@137.184.224.55 # Replace the key path and IP with your actual local key and server-1 IP
ssh i ~ /.ssh/do_key arch@64.23.251.8 # Replace the key path and IP with your actual local key and server-2 IP
``` 
**4. Set up the `webgen` user, directories, and permissions on both servers:** -*[1]*
```bash
sudo useradd -r -d /var/lib/webgen -s /usr/sbin/nologin webgen
sudo mkdir -p /var/lib/webgen/bin /var/lib/webgen/HTML /var/lib/webgen/documents
sudo chown -R webgen:webgen /var/lib/webgen
```
**5. Add test files to the `/documents` directory:** -*[8]*
```bash
sudo bash -c 'cat <<EOF > /var/lib/webgen/documents/file-one
This is file-one content.
EOF'
```
```bash
sudo bash -c 'cat <<EOF > /var/lib/webgen/documents/file-one
This is file-two content.
EOF'
```
6. Change the permissions of the files:
```bash
sudo chown webgen:webgen /var/lib/webgen/documents/*
```

### Automate Index Generation
**1. Create the `generate_index` script**

Create `/var/lib/webgen/bin/generate_index`:
```bash
#!/bin/bash

set -euo pipefail

# this is the generate_index script
# you shouldn't have to edit this script

# Variables
KERNEL_RELEASE=$(uname -r)
OS_NAME=$(grep '^PRETTY_NAME' /etc/os-release | cut -d '=' -f2 | tr -d '"')
DATE=$(date +"%d/%m/%Y")
PACKAGE_COUNT=$(pacman -Q | wc -l)
PUBLIC_IP_ADRESS=$(ip -4 a show dev eth0 | grep inet | awk '{print $2}' | cut -d/ -f1)
OUTPUT_DIR="/var/lib/webgen/HTML"
OUTPUT_FILE="$OUTPUT_DIR/index.html"

# Ensure the target directory exists
if [[ ! -d "$OUTPUT_DIR" ]]; then
    echo "Error: Failed to create or access directory $OUTPUT_DIR." >&2
    exit 1
fi

# Create the index.html file
cat <<EOF > "$OUTPUT_FILE"
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>System Information</title>
</head>
<body>
    <h1>System Information</h1>
    <p><strong>Kernel Release:</strong> $KERNEL_RELEASE</p>
    <p><strong>Operating System:</strong> $OS_NAME</p>
    <p><strong>Date:</strong> $DATE</p>
    <p><strong>Number of Installed Packages:</strong> $PACKAGE_COUNT</p>
    <p><strong>Public IP address of server:</strong> $PUBLIC_IP_ADRESS</p>
</body>
</html>
EOF

# Check if the file was created successfully
if [ $? -eq 0 ]; then
    echo "Success: File created at $OUTPUT_FILE."
else
    echo "Error: Failed to create the file at $OUTPUT_FILE." >&2
    exit 1
fi
```
**2. Configure and test the script**
   ```bash
   sudo chmod +x /var/lib/webgen/bin/generate_index
   /var/lib/webgen/bin/generate_index
   nvim /var/lib/webgen/HTML/index.html
   ```
**3. Create the `systemd service` and `timer`** -*[5]*
```bash
sudo nvim /etc/systemd/system/generate-index.service
```
Add the following content:
```plaintext
[Unit]
Description=Generate updated system information HTML file
After=network-online.target
Wants=network-online.target

[Service]
User=webgen
Group=webgen
ExecStart=/var/lib/webgen/bin/generate_index
```
Create the `timer`:
```bash
sudo nvim /etc/systemd/system/generate-index.timer
```

Add the following content:
```plaintext
[Unit]
Description=Run generate-index.service daily

[Timer]
OnCalendar=*-*-* 05:00:00
Persistent=true

[Install]
WantedBy=timers.target
```
**4. Enable and start the timer**
```bash
sudo systemctl daemon-reload
sudo systemctl enable --now generate-index.timer
```
### Configure Nginx for File Server


**1. Install nginx**
   ```bash
   sudo pacman -S nginx
   ```

**2. Modify nginx.conf** -*[6]*
   
   Open `/etc/nginx/nginx.conf` and set:
   ```bash
   user webgen;
   ```

**3. Create required directories**
   ```bash
   sudo mkdir /etc/nginx/sites-available 
   sudo mkdir /etc/nginx/sites-enabled
   ```

**4. Edit the Main nginx.conf** -*[7]*
   ```bash
   sudo nvim /etc/nginx/nginx.conf
   ```
   Add to the end of the http block:
   ```bash
   include /etc/nginx/sites-enabled/*;
   ```
   Example http block:
   ```bash
   http {
       ...
       include /etc/nginx/sites-enabled/*;
   }
   ```

**5. Create a Server Block File**
   ```bash
   sudo nvim /etc/nginx/sites-available/system-page
   ```
   Add the following content:
   ```bash
   server {
       listen 80;
       listen [::]:80;
       server_name 164.92.126.16; # Your droplet's IP address

       root /var/lib/webgen/HTML;
       index index.html;

       location / {
           try_files $uri $uri/ =404;
       }
       
       # This is the updated section
       location /documents {
        root /var/lib/webgen;
        autoindex on;
       }
   }
   ```

6. **Enable the site**
   ```bash
   sudo ln -s /etc/nginx/sites-available/system-page /etc/nginx/sites-enabled/
   ```

7. **Test and restart nginx**
   ```bash
   sudo nginx -t
   sudo systemctl restart nginx
   ```

### Verify the Configuration
**1. Visit**
- `http://<server-ip>` to check `index.html`.
- `http://<server-ip>/documents` to verify downloadable files.

**2. Access `http://<load balancer-ip>` and confirm it balances traffic between servers.**


## References

1. [Create System User](https://unix.stackexchange.com/questions/233064/how-to-add-system-local-user-like-mysql-or-tomcat)
2. [The `nologin` Shell](https://www.baeldung.com/linux/create-non-login-user)
3. [Check System User Creation](https://unix.stackexchange.com/questions/233064/how-to-add-system-local-user-like-mysql-or-tomcat)
4. [Find Out to What Group a File Belongs](https://www.oreilly.com/library/view/wyntk-unix-system/1565921046/ch04s07.html#:~:text=of%20O'Reilly.-,Use%20ls%20%2Dlg%20to%20find%20out%20to%20what%20group%20a,ls%20%2Dlg%20on%20the%20file.)
5. [Unit File User/Group Identity](https://www.freedesktop.org/software/systemd/man/latest/systemd.exec.html)
6. [`Nginx` - Running Under Different User](https://wiki.archlinux.org/title/Nginx#Running)
7. [`Nginx` - Managing Server Entries](https://wiki.archlinux.org/title/Nginx)
8. [What does `bash -c` do?](https://stackoverflow.com/questions/20858381/what-does-bash-c-do)

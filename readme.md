### Comprehensive Guide: Automatic Captive Portal Authentication Setup

#### System Requirements
- Ubuntu 20.04/22.04 LTS
- Active internet connection for initial setup
- sudo privileges

---

### Step 1: Create the Authentication Script

1. Create script file:
```bash
sudo nano /usr/local/bin/captive-portal-auth.sh
```

2. Paste the following content:
```bash
#!/bin/bash

# =====================================================================
# Ultra-Optimized Captive Portal Authentication Script
# Version: 1.2
# =====================================================================

# Configuration - EDIT THESE VALUES
USERNAME="your_company_username"     # Replace with your username
PASSWORD="your_company_password"     # Replace with your password
TEST_URL="http://www.captiveportal.com"  # Portal detection URL
SESSION_DURATION=43200               # 12-hour session validity (in seconds)

# =====================================================================
# DO NOT MODIFY BELOW THIS LINE UNLESS YOU KNOW WHAT YOU'RE DOING
# =====================================================================

# System paths and files
STATE_DIR="/var/lib/captive_auth"
STATE_FILE="$STATE_DIR/state"
LOG_FILE="/var/log/captive_auth.log"
LAST_AUTH_FILE="$STATE_DIR/last_auth"
LOCK_FILE="/tmp/captive_auth.lock"

# Initialize system
init_system() {
    # Create state directory
    sudo mkdir -p "$STATE_DIR"
    sudo touch "$LOG_FILE" "$STATE_FILE" "$LAST_AUTH_FILE"
    sudo chown $USER:$USER "$STATE_DIR" "$LOG_FILE" "$STATE_FILE" "$LAST_AUTH_FILE"
    sudo chmod 644 "$LOG_FILE" "$STATE_FILE" "$LAST_AUTH_FILE"
    
    # Initialize state if missing
    [ ! -s "$STATE_FILE" ] && echo "unknown" > "$STATE_FILE"
    [ ! -s "$LAST_AUTH_FILE" ] && date +%s > "$LAST_AUTH_FILE"
}

# Logging function
log() {
    echo "[$(date +'%Y-%m-%d %H:%M:%S')] $1" >> "$LOG_FILE"
}

# Minimal internet check
has_internet() {
    # Use DNS query instead of full HTTP request
    if timeout 2 getent hosts www.google.com >/dev/null 2>&1; then
        return 0
    fi
    return 1
}

# Check session validity
check_session_valid() {
    last_auth=$(cat "$LAST_AUTH_FILE" 2>/dev/null)
    current_time=$(date +%s)
    if [ -z "$last_auth" ] || (( current_time - last_auth >= SESSION_DURATION )); then
        return 1  # Session expired
    fi
    return 0  # Session valid
}

# Core authentication function
authenticate() {
    # Get redirect URL
    html_content=$(curl -k -sL -m 5 "$TEST_URL")
    redirect_url=$(grep -oP 'window.location="\K[^"]+' <<< "$html_content")
    
    # Extract token
    token=$(grep -oP 'fgtauth\?\K[0-9a-f]+' <<< "$redirect_url")
    
    # Get login page with cookie preservation
    cookie_header=$(curl -k -sI -m 5 "$redirect_url" | grep -i 'set-cookie' | tr -d '\r\n')
    login_page=$(curl -k -sL -m 5 -H "$cookie_header" "$redirect_url")
    
    # Extract hidden fields
    magic_value=$(grep -oP 'name="magic"\s+value="\K[^"]+' <<< "$login_page")
    redir_value=$(grep -oP 'name="4Tredir"\s+value="\K[^"]+' <<< "$login_page")
    
    # Submit credentials
    response=$(curl -k -sL -m 10 "$redirect_url" \
        -H "$cookie_header" \
        -d "username=$USERNAME" \
        -d "password=$PASSWORD" \
        -d "magic=$magic_value" \
        -d "4Tredir=$redir_value" \
        -d "submit=Login")
    
    # Verify success
    if echo "$response" | grep -iq "success\|welcome\|logout"; then
        log "✅ Authentication successful"
        date +%s > "$LAST_AUTH_FILE"
        echo "authenticated" > "$STATE_FILE"
        return 0
    else
        log "❌ Authentication failed"
        echo "$response" > "$STATE_DIR/last_failed_response.html"
        echo "failed" > "$STATE_FILE"
        return 1
    fi
}

# Main execution flow
main() {
    # Initialize system
    init_system
    
    # Create lock file
    exec 9>"$LOCK_FILE"
    flock -n 9 || { log "⚠️ Another instance is running. Exiting."; exit 1; }
    
    # Get current state
    current_state=$(cat "$STATE_FILE")
    log "=== Started execution (State: $current_state) ==="
    
    # State machine
    case $current_state in
        "authenticated")
            if check_session_valid; then
                if has_internet; then
                    log "ℹ️ Session valid and internet available"
                else
                    log "🌐 Internet lost - reauthenticating"
                    authenticate
                fi
            else
                log "🕒 Session expired - reauthenticating"
                authenticate
            fi
            ;;
        *)
            if ! has_internet; then
                log "🔑 Initial authentication attempt"
                if authenticate; then
                    log "💫 First authentication successful"
                else
                    log "🔥 Initial authentication failed"
                fi
            else
                log "🌐 Internet available - marking authenticated"
                echo "authenticated" > "$STATE_FILE"
                date +%s > "$LAST_AUTH_FILE"
            fi
            ;;
    esac
    
    log "=== Finished execution ==="
    flock -u 9
}

# Start main function
main
```

3. Make the script executable:
```bash
sudo chmod +x /usr/local/bin/captive-portal-auth.sh
```

---

### Step 2: Create Systemd Service Files

1. Create the service file:
```bash
sudo nano /etc/systemd/system/captive-auth@.service
```
```ini
[Unit]
Description=Captive Portal Authentication Service for %i
After=network.target
Wants=network-online.target
ConditionPathExists=/sys/class/net/%i

[Service]
Type=oneshot
User=root
ExecStart=/usr/local/bin/captive-portal-auth.sh
Environment="INTERFACE=%i"
StandardOutput=journal
StandardError=journal

# Restart on failure but with backoff
Restart=on-failure
RestartSec=30s
StartLimitInterval=5min
StartLimitBurst=3

[Install]
WantedBy=multi-user.target
```

2. Create the path unit (interface watcher):
```bash
sudo nano /etc/systemd/system/captive-auth.path
```
```ini
[Unit]
Description=Watch for Network Interface Changes
Requires=network.target
After=network.target

[Path]
# Trigger when interface operational state changes
PathChanged=/sys/class/net/wlan0/operstate
Unit=captive-auth@wlan0.service

[Install]
WantedBy=multi-user.target
```

3. Create the timer unit (session validation):
```bash
sudo nano /etc/systemd/system/captive-auth.timer
```
```ini
[Unit]
Description=Hourly Captive Portal Session Check

[Timer]
# Run 5 minutes after boot
OnBootSec=5min
# Check hourly with 15-minute random delay
OnUnitActiveSec=1h
RandomizedDelaySec=15m
# Allow 5-minute window for execution
AccuracySec=5m

[Install]
WantedBy=timers.target
```

---

### Step 3: Set Up State Directory and Logs

1. Create state directory:
```bash
sudo mkdir -p /var/lib/captive_auth
```

2. Create log file:
```bash
sudo touch /var/log/captive_auth.log
```

3. Set permissions:
```bash
sudo chown $USER:$USER /var/lib/captive_auth /var/log/captive_auth.log
sudo chmod 755 /var/lib/captive_auth
sudo chmod 644 /var/log/captive_auth.log
```

4. Initialize state files:
```bash
echo "unknown" | sudo tee /var/lib/captive_auth/state
date +%s | sudo tee /var/lib/captive_auth/last_auth
```

---

### Step 4: Enable and Start Services

1. Reload systemd configuration:
```bash
sudo systemctl daemon-reload
```

2. Enable and start the path unit:
```bash
sudo systemctl enable captive-auth.path
sudo systemctl start captive-auth.path
```

3. Enable and start the timer:
```bash
sudo systemctl enable captive-auth.timer
sudo systemctl start captive-auth.timer
```

4. Verify installation:
```bash
# Check path unit status
systemctl status captive-auth.path

# Check timer status
systemctl status captive-auth.timer

# Check timer schedule
systemctl list-timers captive-auth.timer
```

---

### Step 5: Test the System

1. Trigger a manual authentication test:
```bash
sudo /usr/local/bin/captive-portal-auth.sh
```

2. Check the log file:
```bash
tail -f /var/log/captive_auth.log
```

3. Simulate a connection event:
```bash
# Bring interface down
sudo ip link set wlan0 down

# Bring interface up
sudo ip link set wlan0 up

# Check logs for interface change detection
journalctl -f -u captive-auth@wlan0.service
```

---

### Step 6: Verify Automatic Operation

1. Check systemd units after reboot:
```bash
sudo reboot

# After reboot:
systemctl status captive-auth.path
systemctl status captive-auth.timer
```

2. Monitor session validation:
```bash
# Check last authentication time
cat /var/lib/captive_auth/last_auth
date -d @$(cat /var/lib/captive_auth/last_auth)

# Check current state
cat /var/lib/captive_auth/state
```

3. Force session expiration test:
```bash
# Set last auth time to 13 hours ago
echo $(($(date +%s) - 46800)) | sudo tee /var/lib/captive_auth/last_auth

# Trigger validation
sudo systemctl start captive-auth@wlan0.service

# Check logs
tail -f /var/log/captive_auth.log
```

---

### Troubleshooting Guide

1. **Authentication Fails**:
   ```bash
   # Check last failed response
   less /var/lib/captive_auth/last_failed_response.html
   
   # Test captive portal detection
   curl -v http://www.captiveportal.com
   ```

2. **Service Not Triggering**:
   ```bash
   # Check interface state
   cat /sys/class/net/wlan0/operstate
   
   # Test path unit
   systemctl status captive-auth.path
   
   # Manually trigger service
   sudo systemctl start captive-auth@wlan0.service
   ```

3. **Session Not Persisting**:
   ```bash
   # Verify file permissions
   ls -l /var/lib/captive_auth/
   
   # Check state transitions
   grep 'State:' /var/log/captive_auth.log
   ```

4. **View Detailed Logs**:
   ```bash
   # Follow live logs
   tail -f /var/log/captive_auth.log
   
   # View systemd logs
   journalctl -u captive-auth@wlan0.service -f
   ```

---

### Maintenance and Updates

1. **Update Credentials**:
   ```bash
   sudo nano /usr/local/bin/captive-portal-auth.sh
   # Update USERNAME and PASSWORD
   ```

2. **Rotate Logs**:
   ```bash
   # Install logrotate
   sudo apt install logrotate
   
   # Create logrotate config
   sudo nano /etc/logrotate.d/captive-auth
   ```
   ```conf
   /var/log/captive_auth.log {
       weekly
       missingok
       rotate 4
       compress
       delaycompress
       notifempty
       create 644 root root
   }
   ```

3. **Uninstall**:
   ```bash
   # Stop and disable services
   sudo systemctl stop captive-auth.path captive-auth.timer
   sudo systemctl disable captive-auth.path captive-auth.timer
   
   # Remove files
   sudo rm /usr/local/bin/captive-portal-auth.sh
   sudo rm -r /etc/systemd/system/captive-auth.*
   sudo rm -r /var/lib/captive_auth
   sudo rm /var/log/captive_auth.log
   
   # Reload systemd
   sudo systemctl daemon-reload
   ```

This comprehensive setup provides a robust, efficient, and maintainable solution for automatic captive portal authentication. The system is optimized for minimal resource usage while ensuring reliable connectivity through state management and smart triggering.
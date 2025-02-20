
# File Monitor Script for Ubuntu in a Proxmox Container

## Installation

1. Save the script below as `file_monitor.sh`
2. Make it executable: `chmod +x file_monitor.sh`
3. Configure the variables at the top of the script

## Main Script

```bash
#!/bin/bash

# Configuration variables
SOURCE_DIR="/path/to/source/directory"
SMB_MOUNT="/mnt/windows_share"
SMB_SERVER="//server/share"
SMB_USER="username"
SMB_PASS="password"
LOG_FILE="/var/log/file_monitor.log"

# Function to log messages
log_message() {
    echo "$(date '+%Y-%m-%d %H:%M:%S') - $1" >> "$LOG_FILE"
}

# Function to verify file copy
verify_copy() {
    local source="$1"
    local dest="$2"
    
    # Check if destination file exists
    if [ ! -f "$dest" ]; then
        return 1
    fi
    
    # Compare file sizes
    source_size=$(stat -c%s "$source")
    dest_size=$(stat -c%s "$dest")
    
    if [ "$source_size" -eq "$dest_size" ]; then
        return 0
    else
        return 1
    fi
}

# Install required packages
apt-get update
apt-get install -y inotify-tools cifs-utils

# Create mount point
mkdir -p "$SMB_MOUNT"

# Create credentials file
SMB_CREDS="/root/.smb_credentials"
echo "username=$SMB_USER" > "$SMB_CREDS"
echo "password=$SMB_PASS" >> "$SMB_CREDS"
chmod 600 "$SMB_CREDS"

# Mount SMB share
mount -t cifs "$SMB_SERVER" "$SMB_MOUNT" -o credentials="$SMB_CREDS",vers=3.0

# Create source directory if it doesn't exist
mkdir -p "$SOURCE_DIR"

# Create log file
touch "$LOG_FILE"
chmod 644 "$LOG_FILE"

log_message "File monitoring service started"

# Monitor directory for new files
inotifywait -m "$SOURCE_DIR" -e create -e moved_to --format '%w%f' | while read NEWFILE
do
    filename=$(basename "$NEWFILE")
    log_message "New file detected: $filename"
    
    # Wait a moment to ensure file is completely written
    sleep 1
    
    # Check if file still exists (wasn't deleted)
    if [ -f "$NEWFILE" ]; then
        # Copy file to SMB share
        cp "$NEWFILE" "$SMB_MOUNT/"
        
        # Verify copy
        if verify_copy "$NEWFILE" "$SMB_MOUNT/$filename"; then
            log_message "File $filename copied successfully"
            
            # Delete original file
            rm "$NEWFILE"
            log_message "Original file deleted"
        else
            log_message "Error: File verification failed for $filename"
        fi
    else
        log_message "Error: File $filename no longer exists"
    fi
done

# Error handling for script termination
trap 'log_message "Service stopped"; umount "$SMB_MOUNT"' EXIT
```

## Systemd Service Configuration

To run this as a service, create a systemd service file:

```bash
sudo nano /etc/systemd/system/file-monitor.service
```

Then add the following content:

```ini
[Unit]
Description=File Monitor Service
After=network.target

[Service]
Type=simple
ExecStart=/path/to/file_monitor.sh
Restart=always
User=****

[Install]
WantedBy=multi-user.target
```

## Service Installation

Enable and start the service:

```bash
systemctl daemon-reload
systemctl enable file-monitor
systemctl start file-monitor
```

## Monitoring

Monitor the logs with:

```bash
tail -f /var/log/file_monitor.log
```

## Features

- Monitors source directory for new files
- Securely mounts SMB share
- Verifies successful file copies
- Logs all actions
- Runs as a system service
- Automatically restarts on failure

---

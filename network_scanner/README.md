# Network Scanner with Web UI

A Python-based network scanner with web interface using Flask and nmap.

## Code

```python
from flask import Flask, render_template
import nmap
import threading
import time
import socket
import netifaces
from datetime import datetime

app = Flask(__name__)

# Store scan results globally
scan_results = []
last_scan_time = None

def get_device_name(ip):
    try:
        return socket.gethostbyaddr(ip)[0]
    except:
        return "Unknown"

def scan_network():
    global scan_results, last_scan_time
    nm = nmap.PortScanner()
    
    while True:
        current_results = []
        
        # Scan the network
        nm.scan(hosts='10.1.100.0/24', arguments='-sn')
        
        for host in nm.all_hosts():
            try:
                hostname = get_device_name(host)
                mac = nm[host]['addresses'].get('mac', 'Unknown')
                vendor = nm[host].get('vendor', {}).get(mac, 'Unknown')
                
                device_info = {
                    'ip': host,
                    'hostname': hostname,
                    'mac': mac,
                    'vendor': vendor,
                    'status': nm[host].state()
                }
                current_results.append(device_info)
            except:
                continue
        
        scan_results = current_results
        last_scan_time = datetime.now()
        time.sleep(300)  # Scan every 5 minutes

@app.route('/')
def index():
    return render_template('index.html', 
                         devices=scan_results, 
                         last_scan=last_scan_time)

# Create templates/index.html
index_html = """
<!DOCTYPE html>
<html>
<head>
    <title>Network Device Scanner</title>
    <style>
        body {
            font-family: Arial, sans-serif;
            margin: 20px;
        }
        table {
            width: 100%;
            border-collapse: collapse;
            margin-top: 20px;
        }
        th, td {
            padding: 10px;
            border: 1px solid #ddd;
            text-align: left;
        }
        th {
            background-color: #f5f5f5;
        }
        tr:nth-child(even) {
            background-color: #f9f9f9;
        }
        .refresh-time {
            color: #666;
            margin-bottom: 20px;
        }
    </style>
    <meta http-equiv="refresh" content="300">
</head>
<body>
    <h1>Network Device Scanner</h1>
    {% if last_scan %}
    <div class="refresh-time">Last scan: {{ last_scan.strftime('%Y-%m-%d %H:%M:%S') }}</div>
    {% endif %}
    <table>
        <tr>
            <th>IP Address</th>
            <th>Hostname</th>
            <th>MAC Address</th>
            <th>Vendor</th>
            <th>Status</th>
        </tr>
        {% for device in devices %}
        <tr>
            <td>{{ device.ip }}</td>
            <td>{{ device.hostname }}</td>
            <td>{{ device.mac }}</td>
            <td>{{ device.vendor }}</td>
            <td>{{ device.status }}</td>
        </tr>
        {% endfor %}
    </table>
</body>
</html>
"""

def setup_app():
    # Create templates directory if it doesn't exist
    import os
    if not os.path.exists('templates'):
        os.makedirs('templates')
    
    # Create index.html template
    with open('templates/index.html', 'w') as f:
        f.write(index_html)

if __name__ == '__main__':
    # Setup the application
    setup_app()
    
    # Start the scanning thread
    scan_thread = threading.Thread(target=scan_network)
    scan_thread.daemon = True
    scan_thread.start()
    
    # Run the Flask application
    app.run(host='0.0.0.0', port=8080)
```

## Setup Instructions

1. Install required packages:
```bash
apt update
apt install -y python3-pip python3-venv nmap
```

2. Create project directory:
```bash
mkdir network_scanner
cd network_scanner
```

3. Create and activate virtual environment:
```bash
python3 -m venv venv
source venv/bin/activate
```

4. Install Python packages:
```bash
pip install flask python-nmap netifaces
```

5. Create `scanner.py` with the above code.

6. Run the application:
```bash
python scanner.py
```

## SystemD Service Setup

Create service file:
```bash
cat << EOF > /etc/systemd/system/network-scanner.service
[Unit]
Description=Network Scanner Web UI
After=network.target

[Service]
Type=simple
User=****
WorkingDirectory=/path/to/network_scanner
Environment=PATH=/path/to/network_scanner/venv/bin
ExecStart=/path/to/network_scanner/venv/bin/python scanner.py
Restart=always

[Install]
WantedBy=multi-user.target
EOF
```

Enable and start service:
```bash
systemctl enable network-scanner
systemctl start network-scanner
```

## Features
- Scans 10.1.100.0/24 subnet every 5 minutes
- Displays IP addresses, hostnames, MAC addresses, vendors, and status
- Auto-refreshing web interface
- Runs as a system service
- Responsive web design
- Threaded scanning

**Note:** Requires root privileges for network scanning. Secure container appropriately.

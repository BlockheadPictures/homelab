# Domain Availability Monitor for macOS

A Python program that monitors domain availability and sends notifications via SMS (Twilio) and email when specified domains become available.

## Main Program (domain_monitor.py)

```python
import whois
import time
import smtplib
from email.mime.text import MIMEText
from twilio.rest import Client
import yaml
from datetime import datetime
import socket
import logging
from pathlib import Path

class DomainMonitor:
    def __init__(self, config_file='config.yml'):
        self.load_config(config_file)
        self.setup_logging()
        self.setup_twilio()
        self.setup_email()

    def load_config(self, config_file):
        config_path = Path.home() / 'domain_monitor' / config_file
        try:
            with open(config_path, 'r') as file:
                self.config = yaml.safe_load(file)
        except Exception as e:
            raise Exception(f"Error loading config file: {e}")

    def setup_logging(self):
        log_path = Path.home() / 'domain_monitor' / 'domain_monitor.log'
        log_path.parent.mkdir(exist_ok=True)
        
        logging.basicConfig(
            level=logging.INFO,
            format='%(asctime)s - %(levelname)s - %(message)s',
            handlers=[
                logging.FileHandler(log_path),
                logging.StreamHandler()
            ]
        )
        self.logger = logging.getLogger(__name__)

    def setup_twilio(self):
        self.twilio_client = Client(
            self.config['twilio']['account_sid'],
            self.config['twilio']['auth_token']
        )

    def setup_email(self):
        self.email_server = self.config['email']['smtp_server']
        self.email_port = self.config['email']['smtp_port']
        self.email_username = self.config['email']['username']
        self.email_password = self.config['email']['password']

    def check_domain_availability(self, domain):
        try:
            w = whois.whois(domain)
            
            # If no expiration date is found, domain might be available
            if w.expiration_date is None:
                return True
                
            # If domain_name is None, domain might be available
            if w.domain_name is None:
                return True
                
            return False
        except Exception as e:
            if "No match for domain" in str(e):
                return True
            self.logger.error(f"Error checking domain {domain}: {e}")
            return None

    def send_sms(self, message):
        try:
            self.twilio_client.messages.create(
                body=message,
                from_=self.config['twilio']['from_number'],
                to=self.config['twilio']['to_number']
            )
            self.logger.info("SMS notification sent successfully")
        except Exception as e:
            self.logger.error(f"Error sending SMS: {e}")

    def send_email(self, subject, message):
        try:
            msg = MIMEText(message)
            msg['Subject'] = subject
            msg['From'] = self.email_username
            msg['To'] = self.config['email']['to_address']

            with smtplib.SMTP(self.email_server, self.email_port) as server:
                server.starttls()
                server.login(self.email_username, self.email_password)
                server.send_message(msg)
            
            self.logger.info("Email notification sent successfully")
        except Exception as e:
            self.logger.error(f"Error sending email: {e}")

    def notify(self, domain):
        message = f"Domain {domain} is now available! Check it at https://www.namecheap.com/domains/registration/results/?domain={domain}"
        subject = f"Domain Available: {domain}"
        
        self.send_sms(message)
        self.send_email(subject, message)

    def monitor_domains(self):
        while True:
            for domain in self.config['domains']:
                self.logger.info(f"Checking domain: {domain}")
                
                is_available = self.check_domain_availability(domain)
                
                if is_available:
                    self.logger.info(f"Domain {domain} is available!")
                    self.notify(domain)
                else:
                    self.logger.info(f"Domain {domain} is not available")
                
                # Wait between checks to avoid rate limiting
                time.sleep(5)
            
            # Wait for the configured check interval
            time.sleep(self.config['check_interval_hours'] * 3600)

def create_config_template():
    config = {
        'domains': [
            'example.com',
            'example.net'
        ],
        'check_interval_hours': 24,
        'twilio': {
            'account_sid': 'your_account_sid',
            'auth_token': 'your_auth_token',
            'from_number': '+1234567890',
            'to_number': '+1234567890'
        },
        'email': {
            'smtp_server': 'smtp.gmail.com',
            'smtp_port': 587,
            'username': 'your_email@gmail.com',
            'password': 'your_app_specific_password',
            'to_address': 'recipient@email.com'
        }
    }
    
    config_dir = Path.home() / 'domain_monitor'
    config_dir.mkdir(exist_ok=True)
    
    with open(config_dir / 'config.yml', 'w') as file:
        yaml.dump(config, file, default_flow_style=False)

if __name__ == "__main__":
    # Create config template if it doesn't exist
    config_path = Path.home() / 'domain_monitor' / 'config.yml'
    if not config_path.exists():
        create_config_template()
        print("Config template created at ~/domain_monitor/config.yml")
        print("Please edit the config file with your settings before running again.")
        exit(0)

    monitor = DomainMonitor()
    monitor.monitor_domains()
```

## Configuration File (config.yml)

```yaml
domains:
  - example.com
  - example.net
check_interval_hours: 24
twilio:
  account_sid: 'your_account_sid'
  auth_token: 'your_auth_token'
  from_number: '+1234567890'
  to_number: '+1234567890'
email:
  smtp_server: 'smtp.gmail.com'
  smtp_port: 587
  username: 'your_email@gmail.com'
  password: 'your_app_specific_password'
  to_address: 'recipient@email.com'
```

## LaunchAgent File (com.user.domainmonitor.plist)

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>Label</key>
    <string>com.user.domainmonitor</string>
    <key>ProgramArguments</key>
    <array>
        <string>/usr/local/bin/python3</string>
        <string>/path/to/domain_monitor.py</string>
    </array>
    <key>RunAtLoad</key>
    <true/>
    <key>KeepAlive</key>
    <true/>
    <key>StandardErrorPath</key>
    <string>/Users/yourusername/domain_monitor/error.log</string>
    <key>StandardOutPath</key>
    <string>/Users/yourusername/domain_monitor/output.log</string>
</dict>
</plist>
```

## Setup Instructions

1. Install required packages:
```bash
pip3 install python-whois twilio pyyaml
```

2. Create Twilio account:
- Sign up at https://www.twilio.com
- Get Account SID and Auth Token
- Get a Twilio phone number

3. Setup Gmail:
- Enable 2-factor authentication
- Create App-specific password

4. Create directories and files:
```bash
mkdir -p ~/domain_monitor
mkdir -p ~/Library/LaunchAgents
```

5. Save the main program as `~/domain_monitor/domain_monitor.py`

6. Run once to create config template:
```bash
python3 domain_monitor.py
```

7. Edit the configuration at `~/domain_monitor/config.yml`

8. Create and load the LaunchAgent:
```bash
# Edit the plist file with correct paths
nano ~/Library/LaunchAgents/com.user.domainmonitor.plist

# Load the LaunchAgent
launchctl load ~/Library/LaunchAgents/com.user.domainmonitor.plist
```

## Features

- Multiple domain monitoring
- SMS notifications via Twilio
- Email notifications
- Configurable check interval
- Automatic logging
- Error handling
- Rate limiting
- YAML configuration
- macOS LaunchAgent integration

## Logs

- Main log: `~/domain_monitor/domain_monitor.log`
- LaunchAgent logs:
  - `~/domain_monitor/output.log`
  - `~/domain_monitor/error.log`

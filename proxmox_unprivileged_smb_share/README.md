Here's how to format that for GitHub with proper markdown:

# SMB Share Access from Unprivileged LXC Container to Host

### Mount Command
```bash
sudo mount -t cifs //HOST_IP/SHARE_NAME /mount/point -o username=YOUR_USERNAME,password=YOUR_PASSWORD,uid=100000,gid=100000,file_mode=0770,dir_mode=0770,noperm
```

### Important Configuration Points
1. Replace `HOST_IP` with your Proxmox host IP
2. Replace `SHARE_NAME` with your shared folder name
3. Replace `YOUR_USERNAME` and `YOUR_PASSWORD` with your SMB credentials
4. The `uid=100000,gid=100000` is crucial for unprivileged containers as they use subuid/subgid mapping
5. `noperm` option bypasses permission issues common in unprivileged containers
6. `file_mode` and `dir_mode` set appropriate permissions

### Permanent Mount Configuration
Add this line to `/etc/fstab`:
```bash
//HOST_IP/SHARE_NAME /mount/point cifs username=YOUR_USERNAME,password=YOUR_PASSWORD,uid=100000,gid=100000,file_mode=0770,dir_mode=0770,noperm 0 0
```

### Create Mount Point
Before mounting, create the mount point directory:
```bash
sudo mkdir -p /mount/point
```

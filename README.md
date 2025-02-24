# Super6C CM4 Cluster Setup Guide

## 📌 Parts Needed
- **Super6C board**
- **Minimum of 3 CM4 boards** (I am using all 6 CM4 boards; 2 are 4GB models with onboard storage)
- **Storage:** SD cards (using SD cards for all CM4 boards, not using onboard eMMC)
- **NVMe drives (optional)** for persistent storage  
- **Ubuntu Server OS image** (from Raspberry Pi Imager)  
- **Notepad++ (or similar)** to create text files  
- **Optional:** PC for remote SSH access  

---

# 🛠 Phase 1: Flash Ubuntu Server to CM4 & Set Up Static IPs

### **1️⃣ Flash Ubuntu Server on CM4 using Raspberry Pi Imager**
- Select **Ubuntu Server (22.04.5 LTS 64-bit)**
- **General settings:**
  - Set hostname (`node1-6.local`)
  - Set username & password (`geek/geek`)
  - Configure **Wi-Fi settings** (if applicable)
- **Enable SSH** (use password authentication)
- Apply settings and flash SD cards  

### **2️⃣ Find the IP Address of Each Node**
Plug in the SD card, power on the CM4 board, and find its IP via:
```bash
arp -a
```
Or check your router UI for connected devices.

### **3️⃣ SSH into Each CM4 Node and Set Static IPs**
```bash
ssh geek@<assigned_ip>
sudo nano /etc/netplan/50-cloud-init.yaml
```
Modify the file:
```yaml
network:
  ethernets:
    eth0:
      dhcp4: no
      addresses:
        - 192.168.70.50/24  # Set unique IP per node
      routes:
        - to: default
          via: 192.168.70.1  # Your gateway
      nameservers:
        addresses:
          - 8.8.8.8
          - 1.1.1.1
  version: 2
```

### **4️⃣ Apply New IP Settings**
```bash
sudo netplan generate
sudo netplan apply
```
Reconnect using the **new static IP**.

### **5️⃣ Disable Automatic Updates**
```bash
sudo systemctl stop unattended-upgrades
sudo systemctl disable unattended-upgrades
sudo systemctl mask unattended-upgrades
sudo nano /etc/apt/apt.conf.d/20auto-upgrades
```
Modify the file:
```plaintext
APT::Periodic::Update-Package-Lists "0";
APT::Periodic::Unattended-Upgrade "0";
```

### **6️⃣ Update System & Install Dependencies**
```bash
sudo apt-get update && sudo apt-get upgrade -y
sudo apt install curl -y
sudo apt autoremove -y && sudo apt clean
sudo reboot
```

---

# 🛠 Phase 2: Setup NVMe Drives for Persistent Storage

### **1️⃣ Identify NVMe Storage**
```bash
lsblk
```
Look for `/dev/nvme0n1`.

### **2️⃣ Partition the NVMe Drive**
```bash
sudo parted /dev/nvme0n1
```
Inside `parted`, type:
```plaintext
mklabel gpt
mkpart primary ext4 0% 100%
quit
```

### **3️⃣ Format the Partition**
```bash
sudo mkfs.ext4 /dev/nvme0n1p1
```

### **4️⃣ Mount the NVMe Drive**
```bash
sudo mkdir -p /mnt/nvme
sudo mount /dev/nvme0n1p1 /mnt/nvme
df -h | grep nvme
```

### **5️⃣ Make the Mount Persistent**
```bash
sudo blkid /dev/nvme0n1p1
sudo nano /etc/fstab
```
Add this line:
```plaintext
UUID=<your-nvme-uuid> /mnt/nvme ext4 defaults,nofail 0 2
```

### **6️⃣ Verify Auto-Mount**
```bash
sudo mount -a
sudo reboot
df -h | grep nvme
```

### **7️⃣ Repeat for All Nodes**
After completion, verify all NVMe drives are mounted.

---

# 🛠 Phase 3: Install Docker & Assign NVMe Storage

### **1️⃣ Install Docker**
```bash
curl -fsSL https://get.docker.com | sudo sh
```

### **2️⃣ Verify Docker Installation**
```bash
docker --version
```

### **3️⃣ Move Docker Storage to NVMe**
```bash
sudo systemctl stop docker
sudo mkdir -p /mnt/nvme/docker
sudo mv /var/lib/docker /mnt/nvme/docker
sudo ln -s /mnt/nvme/docker /var/lib/docker
sudo systemctl start docker
```

### **4️⃣ Verify Storage Location**
```bash
docker info | grep "Docker Root Dir"
```

### **5️⃣ Test Docker with a Container**
```bash
docker run --rm hello-world
```

### **6️⃣ Repeat on All Nodes & Verify Storage**
```bash
for i in {50..55}; do ssh geek@192.168.70.$i "docker info | grep 'Docker Root Dir'"; done
```

---

# 🛠 Phase 4: Setup Docker Swarm + Load Balancing

### **1️⃣ Initialize Swarm on `node1`**
```bash
ssh geek@192.168.70.50
sudo docker swarm init --advertise-addr 192.168.70.50
```
Copy the **join token** from the output.

### **2️⃣ Join Worker Nodes**
```bash
ssh geek@192.168.70.51
sudo docker swarm join --token <SWARM_TOKEN> 192.168.70.50:2377
```
Repeat for **nodes 52-55**.

### **3️⃣ Verify Swarm Setup**
```bash
docker node ls
```

### **4️⃣ Test Load Balancing with Nginx**
```bash
docker service create --name test-nginx --publish 8080:80 --replicas 6 nginx
```

### **5️⃣ Verify the Service is Running**
```bash
docker service ls
docker service ps test-nginx
```

### **6️⃣ Test Load Balancing from WSL**
```bash
for i in {1..10}; do curl -s http://192.168.70.50:8080 | grep "<h1>"; done
```

### **7️⃣ Clean Up Test Service**
```bash
docker service rm test-nginx
```

---

# ✅ Final Checklist
✅ **All CM4 nodes configured with static IPs**  
✅ **NVMe drives partitioned, formatted, and auto-mounted**  
✅ **Docker installed and using NVMe storage**  
✅ **Docker Swarm initialized and all nodes joined**  
✅ **Swarm Load Balancing tested and verified**  

---

# 🚀 Next Steps (For Future Videos/Guides)
- **Phase 5:** Persistent storage for Swarm containers  
- **Phase 6:** Monitoring (Grafana, Prometheus, Portainer)  
- **Phase 7:** Deploying production workloads  
- **Phase 8:** Reverse proxy with Traefik  

---

### If you found this helpful, please subscribe to my youtube channel at www.youtube.com/@GeekofallTrades ###

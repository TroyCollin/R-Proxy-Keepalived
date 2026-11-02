### High-Availability Nginx Reverse Proxy with Keepalived on Docker

This repository contains the configuration files to deploy a highly available, load-balanced Nginx Reverse Proxy infrastructure across two independent Docker nodes. High availability is achieved using Virtual Router Redundancy Protocol (VRRP) via Keepalived, allowing a shared Virtual IP (VIP) to failover instantly if the primary node goes offline. 

### Architecture Overview

* **Virtual IP (VIP):** 192.168.5.62 (Point your DNS/Domains here)
* **Node 1 (Primary MASTER):** 192.168.1.11 (Priority: 101)
* **Node 2 (Secondary BACKUP):** 192.168.1.12 (Priority: 100)
* **Host Network Adapter:** ens18 (Adjust based on your environment)

### Prerequisites

Before starting the containers, you must run the following commands on **BOTH Node 1 and Node 2**. 

By default, Linux prevents services from binding to an IP address that does not explicitly belong to the local system. Because only one node holds the Virtual IP at any given time, the backup node's Nginx service will fail to start without enabling non-local binding. 

bash

# Temporarily enable non-local IP binding
sudo sysctl -w net.ipv4.ip_nonlocal_bind=1

# Make the setting permanent across system reboots
echo "net.ipv4.ip_nonlocal_bind=1" | sudo tee -a /etc/sysctl.conf

Use code with caution.

### Repository Structure

text

├── node-1/
│   ├── docker-compose.yml
│   ├── keepalived.conf
│   └── nginx.conf
└── node-2/
    ├── docker-compose.yml
    ├── keepalived.conf
    └── nginx.conf

Use code with caution.

### Configuration Files

### 📂 Node 1 (MASTER)

#### 1. keepalived.conf

ini

global_defs {
    script_user root
}

vrrp_script check_nginx {
    script "/usr/bin/killall -0 nginx"
    interval 2
    weight 2
}

vrrp_instance VI_1 {
    state MASTER
    interface ens18
    virtual_router_id 51
    priority 101
    advert_int 1

    unicast_src_ip 192.168.1.11
    unicast_peer {
        192.168.1.12
    }

    authentication {
        auth_type PASS
        auth_pass SecretN8
    }

    virtual_ipaddress {
        192.168.5.62/24 dev ens18
    }

    track_script {
        check_nginx
    }
}

Use code with caution.

#### 2. docker-compose.yml

yaml

services:
  nginx-proxy:
    image: nginx:alpine
    container_name: nginx-proxy
    restart: unless-stopped
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf:ro

  keepalived:
    image: osixia/keepalived:2.0.20
    container_name: keepalived
    restart: unless-stopped
    network_mode: host
    cap_add:
      - NET_ADMIN
      - NET_RAW
    volumes:
      - ./keepalived.conf:/usr/local/etc/keepalived/keepalived.conf:ro

Use code with caution.

### 📂 Node 2 (BACKUP)

#### 1. keepalived.conf

ini

global_defs {
    script_user root
}

vrrp_script check_nginx {
    script "/usr/bin/killall -0 nginx"
    interval 2
    weight 2
}

vrrp_instance VI_1 {
    state BACKUP
    interface ens18
    virtual_router_id 51
    priority 100
    advert_int 1

    unicast_src_ip 192.168.1.12
    unicast_peer {
        192.168.1.11
    }

    authentication {
        auth_type PASS
        auth_pass SecretN8
    }

    virtual_ipaddress {
        192.168.5.62/24 dev ens18
    }

    track_script {
        check_nginx
    }
}

Use code with caution.

*Note: The docker-compose.yml and nginx.conf files for Node 2 are structurally identical to Node 1.* 

### Deployment Steps

1. Clone or copy the respective files to Node 1 and Node 2.
2. Navigate to your project folder on **Node 1** and start the stack: 

bash

docker compose up -d

Use code with caution.
3. Navigate to your project folder on **Node 2** and start the stack: 

bash

docker compose up -d

Use code with caution.

### Verification & Testing Failover

### 1. Verify VIP Allocation

Run the following command on **Node 1**: 

bash

ip addr show ens18

Use code with caution.

You should see the Virtual IP 192.168.5.62 attached to the interface. Running the same command on Node 2 should show that Node 2 does not yet hold the IP. 

### 2. Run Failover Test

1. From a separate machine on the network, ping the Virtual IP continuously: 

bash

ping 192.168.5.62

Use code with caution.
2. Intentionally stop the Docker stack or simulate a crash on **Node 1**: 

bash

docker compose down

Use code with caution.
3. Observe the ping stream. You will see near-zero packet loss. Keepalived on Node 2 will immediately recognize the Master's absence, claim the VIP, and route incoming web traffic seamlessly.

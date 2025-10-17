# HashiCorp Vault PoC — Google OAuth + Dynamic MySQL Credentials

## Objective

Demonstrate how to:

* Authenticate users into **HashiCorp Vault UI** using **Google OAuth (OIDC)**.
* Dynamically generate **MySQL credentials** valid for **1 hour**, managed by Vault.
* Securely connect to a **MySQL database** hosted on a **private AWS EC2 instance**.

---

## Architecture Overview

| Component                      | Description                                                       |
| ------------------------------ | ----------------------------------------------------------------- |
| **Vault EC2 (Public Subnet)**  | Hosts Vault server and UI for users to authenticate using Google. |
| **MySQL EC2 (Private Subnet)** | Runs the database Vault connects to and rotates credentials for.  |
| **Google OAuth**               | Provides user authentication via Google account.                  |
| **Vault Secrets Engine**       | Issues dynamic MySQL credentials valid for 1 hour.                |

### Network Layout

```
+-------------------------------+
|        Public Subnet          |
|-------------------------------|
| Vault EC2 (port 8200)         |
| Auth via Google (OIDC)        |
| Dynamic Secrets Engine        |
+-------------------------------+
             |
             | (port 3306)
             v
+-------------------------------+
|        Private Subnet         |
| MySQL EC2 (eta_demo DB)       |
+-------------------------------+
```

---

## Phase 1: Infrastructure Setup

### Step 1.1 — Network Design

* **Public Subnet:** Vault Server
  Allow inbound `8200` (Vault UI) from your IP only.
* **Private Subnet:** MySQL Server
  Allow inbound `3306` only from Vault SG.

### Step 1.2 — Launch EC2 Instances

```bash
# Vault Server (Public)
AMI: Ubuntu 22.04
Type: t3.small
Inbound: 8200, 22

# MySQL Server (Private)
AMI: Ubuntu 22.04
Type: t3.micro
Inbound: 3306 (from vault-sg)
```

---

## Phase 2: MySQL Database Setup

### Step 2.1 — Install MySQL

```bash
sudo apt update
sudo apt install mysql-server -y
sudo mysql_secure_installation
sudo systemctl enable --now mysql
```

### Step 2.2 — Create Demo Database

```sql
-- Connect to MySQL
sudo mysql

-- Create database
CREATE DATABASE eta_demo;

-- Use the database
USE eta_demo;

-- Create ETA table
CREATE TABLE eta_table (
    id INT AUTO_INCREMENT PRIMARY KEY,
    vehicle_id VARCHAR(50) NOT NULL,
    route_name VARCHAR(100) NOT NULL,
    estimated_arrival DATETIME NOT NULL,
    last_updated TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Insert sample data
INSERT INTO eta_table (vehicle_id, route_name, estimated_arrival) VALUES
('BUS_001', 'Downtown Express', '2024-01-15 08:30:00'),
('BUS_002', 'Airport Shuttle', '2024-01-15 09:15:00'),
('TRAIN_101', 'Red Line', '2024-01-15 10:00:00');

-- Create Vault user for management
CREATE USER 'vault-admin'@'%' IDENTIFIED BY 'secure-password-123';
GRANT ALL PRIVILEGES ON eta_demo.* TO 'vault-admin'@'%';
FLUSH PRIVILEGES;
```
<img width="1099" height="453" alt="Screenshot 2025-10-17 at 2 16 18 PM" src="https://github.com/user-attachments/assets/54d8708b-14ee-4d1e-8273-8ec17d16df6e" />
<img width="1115" height="540" alt="Screenshot 2025-10-17 at 2 16 49 PM" src="https://github.com/user-attachments/assets/d9805956-1562-4ad2-bbb4-54e84d1740b1" />

---

## Phase 3: Vault Installation & Configuration

### Step 3.1 — Install Vault

```bash
sudo apt update
sudo apt install gpg curl -y
curl -fsSL https://apt.releases.hashicorp.com/gpg | sudo gpg --dearmor -o /usr/share/keyrings/hashicorp-archive-keyring.gpg
echo "deb [signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] https://apt.releases.hashicorp.com $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/hashicorp.list
sudo apt update && sudo apt install vault -y
vault --version
```

### Step 3.2 — Vault Config (`/etc/vault.d/vault.hcl`)

```bash
sudo mkdir -p /etc/vault.d
sudo mkdir -p /opt/vault/data
```


```hcl
sudo tee /etc/vault.d/vault.hcl <<EOF
ui = true
disable_mlock = true

storage "file" {
  path = "/opt/vault/data"
}

listener "tcp" {
  address     = "0.0.0.0:8200"
  tls_disable = 1
}

api_addr = "http://demo-alb-362976722.us-west-2.elb.amazonaws.com"
EOF
```
```
# Set permissions
sudo chown -R vault:vault /etc/vault.d /opt/vault
```
```
# Create systemd service
sudo tee /etc/systemd/system/vault.service <<EOF
[Unit]
Description=Vault
Documentation=https://www.vaultproject.io/docs/
After=network.target

[Service]
Type=notify
ExecStart=/usr/bin/vault server -config=/etc/vault.d/vault.hcl
ExecReload=/bin/kill -HUP $MAINPID
LimitNOFILE=65536
User=vault
Group=vault
RuntimeDirectory=vault
RuntimeDirectoryMode=0755

[Install]
WantedBy=multi-user.target
EOF
```

```
# Start Vault service
sudo systemctl daemon-reload
sudo systemctl enable vault
sudo systemctl start vault

# Check status
sudo systemctl status vault
```

<img width="1602" height="448" alt="Screenshot 2025-10-17 at 2 20 56 PM" src="https://github.com/user-attachments/assets/5aabf92b-cfb8-47f1-b85e-fd7076644ae3" />


### Step 3.3 — Initialize & Unseal

```bash
export VAULT_ADDR='http://localhost:8200'
vault operator init
vault operator unseal [UNSEAL_KEY]
vault login [ROOT_TOKEN]
```

---

## Phase 4: Configure Google OAuth (OIDC)

### Step 4.1 — Google Cloud Console

1. Create or select project
2. Enable “Google+ API”
3. Create OAuth 2.0 Credentials → Web App
4. Add Redirect URI:

<img width="1405" height="937" alt="Screenshot 2025-10-17 at 2 22 44 PM" src="https://github.com/user-attachments/assets/c3401aae-fe67-4973-bc25-463b463cca60" />

   ```
   http://demo-alb-362976722.us-west-2.elb.amazonaws.com/ui/vault/auth/oidc/oidc/callback
   ```
   
<img width="1710" height="1112" alt="Screenshot 2025-10-17 at 2 23 25 PM" src="https://github.com/user-attachments/assets/62166f16-57ca-4538-929d-8300aefe86ce" />
   

### Step 4.2 — Vault OIDC Setup

```bash
vault auth enable oidc

vault write auth/oidc/config \
    oidc_discovery_url="https://accounts.google.com" \
    oidc_client_id="your-google-client-id" \
    oidc_client_secret="your-google-client-secret" \
    default_role="demo-role"

vault write auth/oidc/role/demo-role \
    user_claim="sub" \
    allowed_redirect_uris="http://demo-alb-362976722.us-west-2.elb.amazonaws.com/ui/vault/auth/oidc/oidc/callback" \
    bound_audiences="your-google-client-id" \
    policies="mysql-access" \
    ttl=1h
```
<img width="1030" height="309" alt="Screenshot 2025-10-17 at 2 25 21 PM" src="https://github.com/user-attachments/assets/8ca0128e-9f3a-4106-9f3a-ac247292d6fb" />


### Step 4.3 — Ensure AWS CLI setup

```bash
# Download the AWS CLI v2 installer
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"

# Unzip the installer
sudo apt-get install unzip -y
unzip awscliv2.zip

# Run the installer
sudo ./aws/install

# Verify installation
aws --version

# configure aws
aws configure
```

---

## Phase 5: MySQL Secrets Engine Setup

```bash
vault secrets enable database

# Step 1: Get the private IP of the MySQL EC2 instance
MYSQL_PRIVATE_IP=$(aws ec2 describe-instances \
  --instance-ids <instance-id> \
  --query 'Reservations[0].Instances[0].PrivateIpAddress' \
  --output text)

echo "MySQL Private IP: $MYSQL_PRIVATE_IP"

# Step 2: Write the database config to Vault
vault write database/config/eta-mysql \
  plugin_name="mysql-database-plugin" \
  connection_url="{{username}}:{{password}}@tcp(${MYSQL_PRIVATE_IP}:3306)/" \
  allowed_roles="mysql-dynamic-role" \
  username="vault-admin" \
  password="secure-password-123"

vault write database/roles/mysql-dynamic-role \
    db_name=eta-mysql \
    creation_statements="CREATE USER '{{name}}'@'%' IDENTIFIED BY '{{password}}'; GRANT SELECT, INSERT, UPDATE, DELETE ON eta_demo.* TO '{{name}}'@'%';" \
    default_ttl="1h" \
    max_ttl="24h"
```
<img width="890" height="135" alt="Screenshot 2025-10-17 at 2 30 07 PM" src="https://github.com/user-attachments/assets/9d5c576a-3d2e-43ed-b1c9-8ec65415661f" />
<img width="1444" height="416" alt="Screenshot 2025-10-17 at 2 30 41 PM" src="https://github.com/user-attachments/assets/bdeeedf4-0144-4944-b3d3-c9550adc98f2" />

---

## Phase 6: Vault Policy

```bash
vault policy write mysql-access - <<EOF
path "database/creds/mysql-dynamic-role" {
  capabilities = ["read"]
}
path "auth/token/lookup-self" {
  capabilities = ["read"]
}
EOF
```

### Step 6.1 — Ensure mysql client setup

```bash
# Install MySQL client on the Vault server
sudo apt update
sudo apt install mysql-client-core-8.0 -y

# Verify installation
mysql --version
```

---

## Phase 7: Testing the Flow

### From Vault UI:

1. Visit:
   `http://demo-alb-362976722.us-west-2.elb.amazonaws.com/ui`
2. Login → Google OIDC
3. Navigate: **Secrets → database → creds → mysql-dynamic-role**
4. Copy temporary credentials (valid for 1 hour)

<img width="1686" height="933" alt="Screenshot 2025-10-17 at 2 34 18 PM" src="https://github.com/user-attachments/assets/204dfb8e-bfb5-40fd-be29-cd640181af7a" />
<img width="1710" height="1112" alt="Screenshot 2025-10-17 at 12 53 09 PM" src="https://github.com/user-attachments/assets/d776accf-56aa-4cfd-83c4-0950ad8eedb4" />
<img width="1710" height="1112" alt="Screenshot 2025-10-17 at 12 53 50 PM" src="https://github.com/user-attachments/assets/6e47e195-591b-4d34-9dec-ab3d6408e021" />
<img width="1710" height="1112" alt="Screenshot 2025-10-17 at 12 54 15 PM" src="https://github.com/user-attachments/assets/ee2bec32-f475-46b0-8bcc-417cb8cb9596" />


### From CLI:

```bash
vault read database/creds/mysql-dynamic-role
```

<img width="1161" height="156" alt="Screenshot 2025-10-17 at 2 41 07 PM" src="https://github.com/user-attachments/assets/cf473179-348b-46fa-aff2-fd33a3bbebbd" />


---

## Phase 8: Validate MySQL Access

```bash
sudo apt install mysql-client -y
mysql -h <MYSQL_PRIVATE_IP> -u <DYNAMIC_USER> -p<DYNAMIC_PASSWORD> -D eta_demo
```

<img width="1710" height="1112" alt="Screenshot 2025-10-17 at 2 37 15 PM" src="https://github.com/user-attachments/assets/4897bc15-2422-47bf-b447-74e195b925ba" />

Run:

```sql
SHOW DATABASES;
USE eta_demo;
SELECT * FROM eta_table;
INSERT INTO eta_table (vehicle_id, route_name, estimated_arrival)
VALUES ('DEMO_BUS_001', 'Test Route', NOW());
```

After 1 hour, the credentials will expire automatically.

---

## ETA (Estimated Time Allocation)

| Phase     | Description                 | Duration    |
| --------- | --------------------------- | ----------- |
| 1–2       | Infra + MySQL Setup         | 1–2 hrs     |
| 3         | Vault Installation          | 1 hr        |
| 4         | OIDC Config                 | 30 mins     |
| 5–6       | Secrets + Policy Setup      | 1 hr        |
| 7–8       | Validation + Demo           | 1 hr        |
| **Total** | **End-to-End PoC Duration** | **4–5 hrs** |

---


## Contact Information

| Name            | Email Address                                                                           |
| --------------- | --------------------------------------------------------------------------------------- |
| Aditya Tripathi | aditya.tripathi.snaatak@mygurukulam.co                                                |


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
sudo mysql

CREATE DATABASE eta_demo;
USE eta_demo;

CREATE TABLE eta_table (
  id INT AUTO_INCREMENT PRIMARY KEY,
  vehicle_id VARCHAR(50),
  route_name VARCHAR(100),
  estimated_arrival DATETIME,
  last_updated TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

INSERT INTO eta_table (vehicle_id, route_name, estimated_arrival)
VALUES ('BUS_001','Downtown Express','2024-01-15 08:30:00');

CREATE USER 'vault-admin'@'%' IDENTIFIED BY 'secure-password-123';
GRANT ALL PRIVILEGES ON eta_demo.* TO 'vault-admin'@'%';
FLUSH PRIVILEGES;
```

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

```hcl
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
```

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

   ```
   http://demo-alb-362976722.us-west-2.elb.amazonaws.com/ui/vault/auth/oidc/oidc/callback
   ```

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
aws -configure
```

---

## Phase 5: MySQL Secrets Engine Setup

```bash
vault secrets enable database

vault write database/config/eta-mysql \
    plugin_name=mysql-database-plugin \
    connection_url="{{username}}:{{password}}@tcp($(aws ec2 describe-instances --instance-ids <MYSQL_INSTANCE_ID> --query 'Reservations[0].Instances[0].PrivateIpAddress' --output text):3306)/" \    
    allowed_roles="mysql-dynamic-role" \
    username="vault-admin" \
    password="secure-password-123"

vault write database/roles/mysql-dynamic-role \
    db_name=eta-mysql \
    creation_statements="CREATE USER '{{name}}'@'%' IDENTIFIED BY '{{password}}'; GRANT SELECT, INSERT, UPDATE, DELETE ON eta_demo.* TO '{{name}}'@'%';" \
    default_ttl="1h" \
    max_ttl="24h"
```

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

### From CLI:

```bash
vault read database/creds/mysql-dynamic-role
```

---

## Phase 8: Validate MySQL Access

```bash
sudo apt install mysql-client -y
mysql -h <MYSQL_PRIVATE_IP> -u <DYNAMIC_USER> -p<DYNAMIC_PASSWORD> -D eta_demo
```

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


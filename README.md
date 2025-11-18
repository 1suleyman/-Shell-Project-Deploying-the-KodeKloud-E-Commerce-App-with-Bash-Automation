# 🛒 Shell Project – Deploying the KodeKloud E-Commerce App with Bash Automation

In this project, I wrote a Bash script `deploy-e-commerce-application.sh` to **fully automate** the deployment of the **KodeKloud E-Commerce learning app** on a CentOS server.

The script installs and configures:

* **FirewallD**
* **MariaDB** (database)
* **Apache (httpd) + PHP**
* The **sample e-commerce web app** from [GitHub](https://github.com/kodekloudhub/learning-app-ecommerce)

It also **sets up the database schema, loads product inventory, opens firewall ports, and verifies that the app is serving product data correctly**.

---

## 📋 Project Overview

**Goal**

* Automate the deployment of the *KodeKloud E-Commerce Application* using a single Bash script:

  * Configure **firewall rules** (ports `3306` and `80`)
  * Install and configure **MariaDB** + load initial inventory
  * Install and configure **Apache + PHP**
  * Deploy the **`learning-app-ecommerce`** codebase [GitHub](https://github.com/kodekloudhub/learning-app-ecommerce)
  * Verify that the e-commerce web page **loads and lists products**

**Initial State**

* Opening the **“KodeKloud E-Commerce Application”** tab showed:
  `502 Bad Gateway – nginx/1.27.4` (app not deployed / web tier failing).

**End State**

* Hitting the app URL shows the **actual e-commerce storefront**, with products like **Laptop, Drone, VR, Tablet, Watch, Phone** populated from the DB. [GitHub](https://github.com/kodekloudhub/learning-app-ecommerce)

---

## 🧱 Architecture & Components

**Single CentOS host running:**

* **FirewallD** – controls access to ports `80` (web) and `3306` (DB).
* **MariaDB Server** – runs the `ecomdb` database, `products` table and data. [GitHub](https://github.com/kodekloudhub/learning-app-ecommerce)
* **Apache (httpd) + PHP** – serves `index.php` and connects to MariaDB.
* **Git + `learning-app-ecommerce` repo** – static assets + PHP front-end. [GitHub](https://github.com/kodekloudhub/learning-app-ecommerce)

Database layout (as per official solution & repo):

* DB name: `ecomdb`
* User: `ecomuser` / password `ecompassword`
* Table: `products (id, Name, Price, ImageUrl)` seeded with sample items. 

---

## 🛠 Script Structure & Functions

Script: `/home/bob/deploy-e-commerce-application.sh`

### 1️⃣ Shebang & Strict Mode

At the top of the script:

```bash
#!/bin/bash
set -euo pipefail
```

* `set -e` – exit on first error
* `set -u` – error on undefined variables
* `set -o pipefail` – catch errors in piped commands

This turns the script into a **fail-fast** deployment tool (very “production-grade”).

---

### 2️⃣ Helper: Pretty Step Logging

I added a small helper to print nice section headers:

```bash
print_step() {
  echo -e "\n=============================="
  echo "➡️  $1"
  echo "=============================="
}
```

Used throughout to make the script output readable (e.g. “Installing FirewallD…”, “Configuring MariaDB firewall…”).

---

### 3️⃣ `install_firewalld` Function

```bash
install_firewalld() {
  print_step "Installing and Enabling FirewallD"

  if ! sudo yum install -y firewalld; then
    echo "❌ Failed to install firewalld"; exit 1; fi

  if ! sudo systemctl start firewalld; then
    echo "❌ Failed to start firewalld"; exit 2; fi

  if ! sudo systemctl enable firewalld; then
    echo "❌ Failed to enable firewalld"; exit 3; fi

  if sudo systemctl is-active --quiet firewalld; then
    echo "✅ FirewallD is running and enabled."
  else
    echo "❌ FirewallD is NOT active:" 
    sudo systemctl status firewalld || true
    exit 10
  fi
}

install_firewalld
```

**Key idea:** encapsulate install+start+enable+health-check with clear exit codes so any problem stops the automation.

---

### 4️⃣ `install_mariadb` Function

```bash
install_mariadb() {
  print_step "Installing and Configuring MariaDB"

  sudo yum install -y mariadb-server || { echo "❌ Failed to install MariaDB"; exit 20; }

  # Ensure basic config exists (non-interactive, no vi)
  if ! grep -q "bind-address" /etc/my.cnf; then
    echo "[mysqld]"           | sudo tee -a /etc/my.cnf >/dev/null
    echo "bind-address=0.0.0.0" | sudo tee -a /etc/my.cnf >/dev/null
  fi

  sudo systemctl start mariadb   || { echo "❌ Failed to start MariaDB";  exit 23; }
  sudo systemctl enable mariadb  || { echo "❌ Failed to enable MariaDB"; exit 24; }

  if sudo systemctl is-active --quiet mariadb; then
    echo "✅ MariaDB is installed, running, and enabled."
  else
    echo "❌ MariaDB is NOT active:" 
    sudo systemctl status mariadb || true
    exit 25
  fi
}

install_mariadb
```

This keeps everything **scriptable** (no manual `vi /etc/my.cnf`) while matching the intent in the official docs. 

---

### 5️⃣ `configure_firewalld_for_mariadb` Function

```bash
configure_firewalld_for_mariadb() {
  print_step "Configuring FirewallD for MariaDB (Port 3306)"

  sudo firewall-cmd --permanent --zone=public --add-port=3306/tcp || exit 30
  sudo firewall-cmd --reload || exit 31

  if sudo firewall-cmd --zone=public --list-ports | grep -q "3306/tcp"; then
    echo "✅ FirewallD successfully configured for MariaDB (3306/tcp)."
  else
    echo "❌ 3306/tcp not found after reload:"
    sudo firewall-cmd --zone=public --list-all || true
    exit 32
  fi
}

configure_firewalld_for_mariadb
```

Later, the example script uses a similar pattern for port `80` (HTTP).

---

### 6️⃣ Importing the Course’s Example Script

After building my own “strict mode + helper” functions, I integrated the **official solution script** from the KodeKloud notes, which handles the **full end-to-end deployment**. 

That script adds:

#### `print_color`, `check_service_status`, `is_firewalld_rule_configured`, `check_item`

```bash
function print_color() { ... }              # colored logging
function check_service_status() { ... }     # systemctl is-active check
function is_firewalld_rule_configured() { ... }  # validate firewall rules
function check_item() { ... }              # verify product names in HTML
```

These are then used during:

* FirewallD + MariaDB setup
* HTTPD + PHP setup
* Final web-app validation (`curl` output check)

---

## ⚙️ Database Setup & Inventory Load

The script configures the DB like this: 

```bash
cat > setup-db.sql <<-EOF
  CREATE DATABASE ecomdb;
  CREATE USER 'ecomuser'@'localhost' IDENTIFIED BY 'ecompassword';
  GRANT ALL PRIVILEGES ON *.* TO 'ecomuser'@'localhost';
  FLUSH PRIVILEGES;
EOF

sudo mysql < setup-db.sql
```

Then it creates and seeds the `products` table:

```bash
cat > db-load-script.sql <<-EOF
USE ecomdb;
CREATE TABLE products (
  id mediumint(8) unsigned NOT NULL auto_increment,
  Name varchar(255) default NULL,
  Price varchar(255) default NULL,
  ImageUrl varchar(255) default NULL,
  PRIMARY KEY (id)
) AUTO_INCREMENT=1;

INSERT INTO products (Name,Price,ImageUrl) VALUES
  ("Laptop","100","c-1.png"),
  ("Drone","200","c-2.png"),
  ("VR","300","c-3.png"),
  ("Tablet","50","c-5.png"),
  ("Watch","90","c-6.png"),
  ("Phone Covers","20","c-7.png"),
  ("Phone","80","c-8.png"),
  ("Laptop","150","c-4.png");
EOF

sudo mysql < db-load-script.sql
```

A quick verification query:

```bash
mysql_db_results=$(sudo mysql -e "use ecomdb; select * from products;")

if [[ $mysql_db_results == Laptop ]]; then
  print_color "green" "Inventory data loaded into MySQL"
else
  print_color "red" "Inventory data not loaded into MySQL"
  exit 1
fi
```

---

## 🌐 Web Server & App Deployment

The web tier section of the script:

1. **Install web stack**

   ```bash
   sudo yum install -y httpd php php-mysqlnd
   ```

2. **Open HTTP (port 80) in firewalld**

   ```bash
   sudo firewall-cmd --permanent --zone=public --add-port=80/tcp
   sudo firewall-cmd --reload
   is_firewalld_rule_configured 80
   ```

3. **Make Apache use `index.php`**

   ```bash
   sudo sed -i 's/index.html/index.php/g' /etc/httpd/conf/httpd.conf
   ```

4. **Start & enable httpd**

   ```bash
   sudo systemctl start httpd
   sudo systemctl enable httpd
   check_service_status httpd
   ```

5. **Clone the e-commerce repo & configure DB connection**

   ```bash
   sudo yum install -y git
   sudo git clone https://github.com/kodekloudhub/learning-app-ecommerce.git /var/www/html/

   # Uncomment and fix DB connection in index.php
   sudo sed -i 's#// \(.*mysqli_connect.*\)#\1#' /var/www/html/index.php
   sudo sed -i 's/172.20.1.101/localhost/g' /var/www/html/index.php
   ```

   This matches the official `learning-app-ecommerce` setup: web + DB on the same node use `localhost` as DB host. [GitHub](https://github.com/kodekloudhub/learning-app-ecommerce)

---

## ✅ End-to-End Verification

At the end, the script validates that the web app is **actually serving real data**:

```bash
web_page=$(curl http://localhost)

for item in Laptop Drone VR Watch Phone
do
  check_item "$web_page" $item
done
```

For each product name:

* If found → prints a **green “Item X is present on the web page”**
* If not found → prints a **red warning**

Final manual check:

* Open the **“KodeKloud E-Commerce Application”** tab again
* Confirm that the 502 error is gone and the **product grid is visible** with all items listed.

---

## 🧠 Key Concepts Reinforced

| Concept                               | Description                                                                                                                              |
| ------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------- |
| **Strict mode (`set -euo pipefail`)** | Makes the script fail fast and exposes hidden errors early.                                                                              |
| **Functions for reuse**               | `install_firewalld`, `install_mariadb`, `configure_firewalld_for_mariadb`, `print_color`, `check_item` keep the script DRY and readable. |
| **FirewallD management**              | Adding ports (`3306`, `80`), reloading, and verifying with `--list-ports`.                                                               |
| **DB bootstrap via SQL files**        | Using heredocs + `mysql` to create DB, users, tables, and seed data.                                                                     |
| **Config editing without editors**    | Using `sed` and `tee` instead of `vi` for full automation.                                                                               |
| **Deployment validation**             | Using `curl` + simple string checks to confirm app health and data visibility.                                                           |

---

## 🗂️ Key Commands Summary

| Task                     | Command                                               |
| ------------------------ | ----------------------------------------------------- |
| Run deployment script    | `bash deploy-e-commerce-application.sh`               |
| Check FirewallD status   | `sudo systemctl status firewalld`                     |
| List open firewall ports | `sudo firewall-cmd --zone=public --list-ports`        |
| Check DB tables          | `sudo mysql -e "USE ecomdb; SHOW TABLES;"`            |
| Inspect products         | `sudo mysql -e "USE ecomdb; SELECT * FROM products;"` |
| Check web app HTML       | `curl http://localhost`                               |

---

## 🏁 End of Project

By the end of this project I had:

* Turned a **broken 502 Bad Gateway** page into a **working e-commerce storefront**
* Built a **modular, function-driven Bash deployment script**
* Combined **course solution logic** with my own **strict error-handling patterns**
* Practiced real-world DevOps skills: *idempotent provisioning, config management, and automated verification*

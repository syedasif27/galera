# galera


# Setting Up MariaDB Galera Cluster for Nextcloud

## Objective:
To set up a high-availability MariaDB Galera Cluster on two database servers to support a Nextcloud environment.

---

## Prerequisites:
- Two Ubuntu-based database servers (Server 1 and Server 2).
- Basic knowledge of MariaDB and Nextcloud installation.
- Root privileges on both database servers.

---

## Step 1: Install MariaDB and Galera Cluster on Both Servers

### On Both Database Servers (Server 1 and Server 2):
1. Update the package list and install the required packages:
   ```bash
   sudo apt update
   sudo apt install mariadb-server mariadb-client galera-4
   ```

2. Verify the installation:
   ```bash
   mariadb --version
   ```

---

## Step 2: Configure MariaDB for Galera Cluster

### On Server 1 (First Node):
1. Open the MariaDB configuration file for editing:
   ```bash
   sudo nano /etc/mysql/mariadb.conf.d/50-server.cnf
   ```

2. Add the following configuration to enable Galera clustering:
   ```ini
   [mysqld]
   wsrep_on=ON
   wsrep_cluster_address=gcomm://  # Empty address for the first node
   wsrep_cluster_name="nextcloud_cluster"
   wsrep_node_address="Server1_IP"  # Replace with Server 1's IP
   wsrep_node_name="node1"  # Unique node name for this server
   wsrep_sst_method=rsync
   wsrep_sst_auth="sstuser:password"  # Replace with your SST user credentials
   ```

### On Server 2 (Second Node):
1. Open the MariaDB configuration file for editing:
   ```bash
   sudo nano /etc/mysql/mariadb.conf.d/50-server.cnf
   ```

2. Modify the configuration to connect to the first node (Server 1):
   ```ini
   [mysqld]
   wsrep_on=ON
   wsrep_cluster_address=gcomm://Server1_IP  # Replace with Server 1's IP
   wsrep_cluster_name="nextcloud_cluster"
   wsrep_node_address="Server2_IP"  # Replace with Server 2's IP
   wsrep_node_name="node2"  # Unique node name for this server
   wsrep_sst_method=rsync
   wsrep_sst_auth="sstuser:password"  # Replace with your SST user credentials
   ```

---

## Step 3: Create the Galera SST (State Snapshot Transfer) User

1. Log into MariaDB on Server 1:
   ```bash
   sudo mysql -u root
   ```

2. Create the SST user:
   ```sql
   CREATE USER 'sstuser'@'%' IDENTIFIED BY 'password';
   GRANT ALL PRIVILEGES ON *.* TO 'sstuser'@'%' WITH GRANT OPTION;
   FLUSH PRIVILEGES;
   ```

---

## Step 4: Start the First Node (Server 1)

1. Stop the MariaDB service on Server 1:
   ```bash
   sudo systemctl stop mariadb
   ```

2. Bootstrap the cluster (this starts the first node):
   ```bash
   sudo galera_new_cluster
   ```

3. Check the status of MariaDB:
   ```bash
   sudo systemctl status mariadb
   ```

   The first node should now be running and acting as the primary node.

---

## Step 5: Start the Second Node (Server 2)

1. Start the MariaDB service on Server 2:
   ```bash
   sudo systemctl start mariadb
   ```

2. Check if the second node successfully joins the cluster:
   ```bash
   sudo mysql -u root -e "SHOW STATUS LIKE 'wsrep_cluster_size';"
   ```

   The output should show `2`, indicating that both nodes are part of the cluster.

---

## Step 6: Create the Nextcloud Database and User

1. Log into MariaDB on either Server 1 or Server 2:
   ```bash
   sudo mysql -u root
   ```

2. Create the Nextcloud database and user:
   ```sql
   CREATE DATABASE nextcloud;
   CREATE USER 'nextcloud_user'@'%' IDENTIFIED BY 'your_secure_password';
   GRANT ALL PRIVILEGES ON nextcloud.* TO 'nextcloud_user'@'%';
   FLUSH PRIVILEGES;
   ```

---

## Step 7: Configure Nextcloud to Use the Galera Cluster

1. On the Web Server, navigate to the `config.php` file in your Nextcloud installation directory (typically found in `/var/www/nextcloud/config/config.php`).

2. Edit the configuration file to include the Galera cluster details:
   ```php
   'dbtype' => 'mysql',
   'dbname' => 'nextcloud',
   'dbuser' => 'nextcloud_user',
   'dbpassword' => 'your_secure_password',
   'dbhost' => 'Server1_IP,Server2_IP',  # IPs of both database nodes
   'dbport' => '3306',
   'dbtableprefix' => 'oc_',
   ```

---

## Step 8: Test the Setup

1. Access the Nextcloud web interface through your browser.
2. Verify that Nextcloud successfully connects to the database and functions correctly.

   - You can also check the database connection by logging into the MariaDB server and verifying the cluster size:
     ```bash
     sudo mysql -u root -e "SHOW STATUS LIKE 'wsrep_cluster_size';"
     ```

3. Test the failover by stopping MariaDB on one node and ensuring that the other node still handles requests.

---

## Step 9: Secure the Cluster (Optional)

1. Consider enabling SSL/TLS encryption for communication between the nodes, especially if they are not in the same data center.
   
   - To enable SSL, configure the MariaDB server settings to use `--ssl-ca`, `--ssl-cert`, and `--ssl-key` options.

2. Disable remote root login for security:
   ```sql
   sudo mysql -u root
   UPDATE mysql.user SET plugin = 'mysql_native_password' WHERE user = 'root' AND host != 'localhost';
   FLUSH PRIVILEGES;
   ```

3. Ensure strong passwords for all users and use firewalls to restrict access to the database servers.

---

## Step 10: Monitor the Cluster

1. Install monitoring tools like Prometheus, Percona Monitoring and Management (PMM), or MariaDB’s built-in tools to track performance, replication status, and failover events.

2. Regularly check the status of the cluster and the database:
   ```bash
   sudo mysql -u root -e "SHOW STATUS LIKE 'wsrep_%';"
   ```

---

## Conclusion:
You now have a MariaDB Galera Cluster set up for your Nextcloud environment with high availability and fault tolerance. Your cluster will automatically handle node failover, ensuring that your Nextcloud instance remains operational even if one of the database nodes goes down.

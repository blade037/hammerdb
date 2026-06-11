**Here’s a complete, practical step-by-step guide** to benchmark **MySQL on Ubuntu** using **HammerDB** on your two VMware vSphere 9.0 hosts.

### Recommended Architecture (Best Practice)
- **Host 1** → MySQL Database VM (System Under Test)
- **Host 2** → HammerDB Load Generator VM

This gives you proper isolation and realistic network conditions. You can also run both VMs on one host for quick testing.

### 1. Create the Virtual Machines in vSphere

**MySQL VM (recommended specs)**
- vCPU: **8–16** (start with 8)
- Memory: **32–64 GB**
- Hard disk: **200 GB+** (use a fast datastore — NVMe/SSD preferred)
- SCSI Controller: **VMware Paravirtual** (PVSCSI)
- Network Adapter: **VMXNET3**
- Hardware version: Latest supported by vSphere 9.0
- Guest OS: **Linux → Ubuntu 64-bit**

**HammerDB VM (recommended specs)**
- vCPU: **4–8**
- Memory: **8–16 GB**
- Hard disk: **50–100 GB**
- Same SCSI + Network settings as above

**After creating the VMs:**
1. Mount the **Ubuntu 24.04 LTS Server ISO** and install Ubuntu (minimal/server installation is fine).
2. After installation, install VMware tools:
   ```bash
   sudo apt update && sudo apt upgrade -y
   sudo apt install -y open-vm-tools
   ```
   (Reboot after installation)

### 2. Basic Configuration on Both VMs

Run these commands on **both** VMs:

```bash
sudo apt update && sudo apt upgrade -y

# Set CPU governor to performance (important for benchmarking)
sudo apt install -y cpufrequtils
sudo cpufreq-set -g performance

# (Optional but recommended) Install useful tools
sudo apt install -y htop iotop sysstat vim net-tools
```

Configure static IPs (or ensure both VMs can reach each other reliably).

### 3. MySQL Server Setup (on MySQL VM)

```bash
sudo apt install -y mysql-server
sudo mysql_secure_installation
```

#### Edit MySQL Configuration

```bash
sudo nano /etc/mysql/mysql.conf.d/mysqld.cnf
```

Add/replace with these recommended settings for benchmarking (adjust buffer pool size according to your RAM):

```ini
[mysqld]
# Basic
bind-address = 0.0.0.0
port = 3306
default_authentication_plugin = mysql_native_password

# InnoDB Tuning (HammerDB best practices)
innodb_buffer_pool_size = 24G          # ← Change according to your RAM (use ~70-80%)
innodb_buffer_pool_instances = 8
innodb_log_file_size = 2G
innodb_log_files_in_group = 4
innodb_flush_log_at_trx_commit = 2     # 0 or 2 for higher performance (less durability)
innodb_flush_method = O_DIRECT
innodb_doublewrite = 0
innodb_file_per_table = 1
innodb_io_capacity = 20000
innodb_io_capacity_max = 40000
innodb_read_io_threads = 16
innodb_write_io_threads = 16
innodb_thread_concurrency = 0
innodb_adaptive_hash_index = 0
performance_schema = OFF

max_connections = 2000
table_open_cache = 8000
```

Save and restart MySQL:

```bash
sudo systemctl restart mysql
```

#### Create HammerDB User

```bash
sudo mysql -u root -p
```

```sql
CREATE USER 'hammerdb'@'%' IDENTIFIED BY 'YourStrongPassword';
GRANT ALL PRIVILEGES ON *.* TO 'hammerdb'@'%' WITH GRANT OPTION;
FLUSH PRIVILEGES;
EXIT;
```

### 4. Install HammerDB on the Load Generator VM

1. Go to the official site and download the latest version:  
   **https://www.hammerdb.com/download.html** → HammerDB 5.0 (`.deb` for Ubuntu 24.04)

2. Install it:

```bash
cd ~/Downloads
sudo apt install -f ./hammerdb_5.0-1.ubu24.amd64.deb
```

3. Run HammerDB (GUI):

```bash
cd /opt/hammerdb
./hammerdb
```

(For CLI use: `hammerdbcli`)

### 5. Run the Benchmark (Recommended: Start with GUI)

#### Step A: Build the Schema

In HammerDB GUI:

1. Go to **Benchmark → TPROC-C**
2. Double-click **Options** under **Schema Build**
3. Configure:
   - **MySQL Host** → IP of your MySQL VM
   - **MySQL Port** → `3306`
   - **MySQL User** → `hammerdb`
   - **MySQL User Password** → your password
   - **MySQL Database** → `tpcc`
   - **Number of Warehouses** → Start with **100–500** (scale later)
   - **Virtual Users to Build Schema** → `4` or `8` (match vCPU)
   - **Partition Tables** → **True** (recommended)
   - **Storage Engine** → `innodb`

4. Click **OK**, then double-click **Build** to create the schema.

#### Step B: Run the Workload

1. Double-click **Options** under **Driver Script**
2. Set similar connection details as above.
3. Configure test parameters:
   - **Rampup Time** → `2` minutes
   - **Test Duration** → `5–10` minutes (start small)
   - **Use All Warehouses** → Checked

4. Go to **Virtual Users** → Create the number of virtual users (start with 10–50 and increase gradually).
5. Click **Run** (Timed Test).

HammerDB will show **NOPM** (New Orders Per Minute) and **TPM**. **NOPM** is the most important metric for comparison.

### 6. Useful Tips & Best Practices

| Area                    | Recommendation                                      | Why |
|-------------------------|-----------------------------------------------------|-----|
| **Schema Size**         | Start with 100–500 warehouses                       | Faster testing |
| **Virtual Users**       | Increase gradually until performance plateaus       | Find saturation point |
| **MySQL Tuning**        | `innodb_flush_log_at_trx_commit = 2` or `0`         | Higher throughput |
| **Partitioning**        | Enable in schema build                              | Better performance on larger schemas |
| **Monitoring**          | Use `htop`, `iostat -x 1`, MySQL `SHOW ENGINE INNODB STATUS` | Identify bottlenecks |
| **Repeatability**       | Use **CLI** with scripts after you’re comfortable   | Consistent results |
| **Power Management**    | Keep CPU governor on `performance`                  | Avoid throttling |

### CLI Example (for automation)

You can create a script like this:

```tcl
dbset db mysql
diset connection mysql_host 192.168.10.10
diset connection mysql_port 3306
diset connection mysql_user hammerdb
diset connection mysql_pass YourStrongPassword
diset tpcc mysql_count_ware 200
diset tpcc mysql_num_vu 8
diset tpcc mysql_partition true
buildschema
```

Then run the timed test similarly.

### Next Steps / Scaling

- Once comfortable with 200–500 warehouses, increase to 1000+.
- Run multiple tests and take the average.
- Monitor host-level metrics in vSphere (CPU Ready, Disk latency, etc.).

Would you like me to also provide:
- A ready-to-use CLI script?
- Recommended my.cnf for different RAM sizes?
- How to monitor during the test?

Just let me know how you want to proceed!
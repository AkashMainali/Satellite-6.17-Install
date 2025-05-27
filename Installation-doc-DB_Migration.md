# Red Hat Satellite 6.17 Installation and Database Migration Guide

This document outlines the steps to install Red Hat Satellite 6.17 on a RHEL 9 server using an external PostgreSQL database on a separate RHEL 9 server, and the process to migrate data from an existing Satellite 6.16.x instance.

## Table of Contents

1.  [Prerequisites and Planning](#1-prerequisites-and-planning)
2.  [Setting Up the External PostgreSQL Server (RHEL 9)](#2-setting-up-the-external-postgresql-server-rhel-9)
3.  [Installing Satellite 6.17 on RHEL 9](#3-installing-satellite-617-on-rhel-9)
4.  [Database Migration from Satellite 6.16.x](#4-database-migration-from-satellite-616x)

---

## 1. Prerequisites and Planning

Before starting the installation or migration, ensure the following:

* **Documentation Review:** Familiarize yourself with the official Red Hat Satellite 6.17 documentation (Installation Guide, Administration Guide, Upgrading Guide).
* **System Requirements:**
    * **New Satellite Server:** RHEL 9, meeting Satellite 6.17 hardware/software specs (CPU, RAM, disk space for `/var/lib/pulp`, etc.).
    * **New PostgreSQL Server:** RHEL 9, meeting PostgreSQL hardware/software specs (CPU, RAM, disk space for `/var/lib/pgsql`).
* **Backups:**
    * **Critical:** Perform a **full, verified backup** of your existing Satellite 6.16.x server and its database. This is your primary rollback mechanism.
* **Network Configuration:**
    * Static IP addresses for both new RHEL 9 servers.
    * Correct DNS resolution (forward and reverse) for both new servers.
    * Firewall ports opened:
        * Between Satellite Server and PostgreSQL Server (e.g., TCP 5432).
        * Between Satellite Server and client hosts (RHEL 8 & RHEL 9) for registration, patching, remote execution (e.g., TCP 443, 22).
        * Refer to Satellite documentation for a complete list of required ports.
* **Subscription Manifest:**
    * Download an up-to-date Red Hat Subscription Manifest from the Red Hat Customer Portal (for the new Satellite 6.17 server).
* **Existing Satellite 6.16 Configuration:**
    * Ensure your existing Satellite 6.16 server has the necessary RHEL 9 repositories (BaseOS, AppStream) synced and published in a Content View.
    * Create an Activation Key on Satellite 6.16 for registering the new RHEL 9 PostgreSQL server, linked to this Content View.
* **Disconnected Environment (If Applicable):**
    * Download all necessary ISO images (RHEL 9, Satellite 6.17) and repository content if your new Satellite 6.17 will be in a disconnected network.

---

## 2. Setting Up the External PostgreSQL Server (RHEL 9)

This server will host the database for your new Satellite 6.17 instance. It will obtain its OS packages, including PostgreSQL, from your existing Satellite 6.16 server.

**Hostname:** `<your_postgres_server_fqdn>`
**IP Address:** `<your_postgres_server_ip>`
**Existing Satellite 6.16 FQDN:** `<your_satellite_6.16_fqdn>`

1.  **Install RHEL 9:**
    * Perform a minimal installation of Red Hat Enterprise Linux 9 on the server designated for PostgreSQL.

2.  **Prepare for Registration to Satellite 6.16:**
    * Download the consumer CA certificate from your existing Satellite 6.16 server. This allows the new server to trust your Satellite 6.16.
    ```bash
    sudo curl -o /etc/pki/rpm-gpg/RPM-GPG-KEY-katello-server-ca.pem http://<your_satellite_6.16_fqdn>/pub/katello-server-ca.pem
    sudo rpm --import /etc/pki/rpm-gpg/RPM-GPG-KEY-katello-server-ca.pem
    sudo curl -o /etc/yum.repos.d/katello-rhsm-consumer.repo http://<your_satellite_6.16_fqdn>/pub/katello-rhsm-consumer.repo
    sudo dnf install -y katello-ca-consumer-latest.noarch
    ```
    *Alternatively, you can download `katello-ca-consumer-latest.noarch.rpm` and install it:*
    ```bash
    # sudo curl -o katello-ca-consumer-latest.noarch.rpm http://<your_satellite_6.16_fqdn>/pub/katello-ca-consumer-latest.noarch.rpm
    # sudo rpm -Uvh katello-ca-consumer-latest.noarch.rpm
    ```

3.  **Register to Existing Satellite 6.16 Server:**
    * Register the new RHEL 9 PostgreSQL server to your existing Satellite 6.16 server using an Activation Key. This key should associate the server with a Content View on Satellite 6.16 that provides RHEL 9 repositories (including BaseOS and AppStream, which contain PostgreSQL).
    ```bash
    sudo subscription-manager register --org="<Your_Sat6.16_Organization_Label>" --activationkey="<Your_RHEL9_PostgreSQL_Activation_Key_from_Sat6.16>"
    ```
    * Verify registration:
    ```bash
    sudo subscription-manager status
    sudo subscription-manager list --available --all # Should show RHEL 9 repos from Sat 6.16
    ```

4.  **Verify Repository Access from Satellite 6.16:**
    * Ensure the RHEL 9 repositories are enabled and point to your Satellite 6.16 server.
    ```bash
    sudo dnf repolist
    ```
    *You should see repositories sourced from your Satellite 6.16 server.*

5.  **Install PostgreSQL (from Satellite 6.16):**
    * Install the PostgreSQL server packages. `dnf` will now use the repositories provided by your Satellite 6.16 server.
    ```bash
    sudo dnf install postgresql-server postgresql-contrib
    ```

6.  **Initialize Database:**
    ```bash
    sudo postgresql-setup --initdb
    ```

7.  **Configure PostgreSQL for Satellite 6.17:**
    * **Edit `postgresql.conf`** (usually located at `/var/lib/pgsql/data/postgresql.conf`):
        * Set `listen_addresses` to allow connections from your new Satellite 6.17 server's IP and localhost. Example: `listen_addresses = 'localhost, <your_new_satellite_6.17_server_ip>'`
        * Adjust `max_connections` if needed (Satellite documentation might provide recommendations).
    * **Edit `pg_hba.conf`** (usually located at `/var/lib/pgsql/data/pg_hba.conf`):
        * Add lines to allow the new Satellite 6.17 server to connect to the `foreman` and `pulpcore` databases using password authentication (md5 or scram-sha-256).
        ```
        # TYPE  DATABASE        USER            ADDRESS                 METHOD
        host    foreman         foreman_user    <your_new_satellite_6.17_server_ip>/32  md5  # or scram-sha-256
        host    pulpcore        pulp_user       <your_new_satellite_6.17_server_ip>/32  md5  # or scram-sha-256
        ```
        *Replace `<your_new_satellite_6.17_server_ip>` with the actual IP of your new Satellite 6.17 server.*
        *The users `foreman_user` and `pulp_user` will be specified during the Satellite 6.17 installation.*

8.  **Start and Enable PostgreSQL Service:**
    ```bash
    sudo systemctl start postgresql
    sudo systemctl enable postgresql
    sudo systemctl status postgresql
    ```

9.  **Firewall Configuration (on PostgreSQL Server):**
    * Allow traffic on the PostgreSQL port (default 5432) from the new Satellite 6.17 server.
    ```bash
    sudo firewall-cmd --permanent --add-rich-rule='rule family="ipv4" source address="<your_new_satellite_6.17_server_ip>/32" port port="5432" protocol="tcp" accept'
    sudo firewall-cmd --reload
    ```

---

## 3. Installing Satellite 6.17 on RHEL 9

This server will host the Satellite 6.17 application. It will get its *Satellite* packages from Red Hat CDN (or your disconnected setup if applicable).

**Hostname:** `<your_new_satellite_6.17_server_fqdn>`
**IP Address:** `<your_new_satellite_6.17_server_ip>`

1.  **Install RHEL 9:**
    * Perform a minimal installation of RHEL 9.
2.  **Register and Subscribe (to Red Hat CDN):**
    * Register the server to Red Hat Subscription Management for RHEL and Satellite 6.17 packages.
    ```bash
    sudo subscription-manager register --username=<your_rh_username> --password=<your_rh_password> --autosubscribe
    # Or use --activationkey if you have one for RHEL & Satellite from RHN
    ```
3.  **Enable Required Repositories (from Red Hat CDN):**
    ```bash
    sudo subscription-manager repos \
        --enable=rhel-9-for-x86_64-baseos-rpms \
        --enable=rhel-9-for-x86_64-appstream-rpms \
        --enable=satellite-6.17-for-rhel-9-x86_64-rpms \
        --enable=satellite-maintenance-6.17-for-rhel-9-x86_64-rpms
    ```
4.  **Install Satellite Packages:**
    ```bash
    sudo dnf install satellite
    ```
5.  **Run the Satellite Installer:**
    * Execute the `satellite-installer` command, providing details for the external PostgreSQL server (which you set up in Section 2).
    * **Important:**
        * The `--foreman-db-username` and `--foreman-proxy-content-pulp-db-username` should match the users you configured in `pg_hba.conf` on the PostgreSQL server.
        * Choose strong passwords for `--foreman-db-password` and `--foreman-proxy-content-pulp-db-password`.
    ```bash
    sudo satellite-installer --scenario satellite \
      --foreman-initial-organization "<Your_Initial_Organization>" \
      --foreman-initial-location "<Your_Initial_Location>" \
      --foreman-admin-username <admin_username> \
      --foreman-admin-password <admin_password> \
      --foreman-db-host "<your_postgres_server_fqdn>" \
      --foreman-db-port "5432" \
      --foreman-db-name "foreman" \
      --foreman-db-username "foreman_user" \
      --foreman-db-password "<foreman_db_secure_password>" \
      --foreman-proxy-content-pulp-db-host "<your_postgres_server_fqdn>" \
      --foreman-proxy-content-pulp-db-port "5432" \
      --foreman-proxy-content-pulp-db-name "pulpcore" \
      --foreman-proxy-content-pulp-db-username "pulp_user" \
      --foreman-proxy-content-pulp-db-password "<pulp_db_secure_password>"
      # Add other necessary parameters as per your requirements.
    ```
    * The installer will connect to the external PostgreSQL server and set up the databases (`foreman`, `pulpcore`) and schemas.
6.  **Verify Installation:**
    * After the installer completes, check the installation log for errors (`/var/log/foreman-installer/satellite.log`).
    * Access the Satellite web UI at `https://<your_new_satellite_6.17_server_fqdn>`.
    * Run a health check:
        ```bash
        sudo satellite-maintain health check
        ```
7.  **Firewall Configuration (on New Satellite 6.17 Server):**
    * The `satellite-installer` usually configures the local firewall. Verify that necessary ports (e.g., 443, 80, 5647, 9090) are open.
    ```bash
    sudo firewall-cmd --list-all
    ```

---

## 4. Database Migration from Satellite 6.16.x

**CRITICAL WARNING:** Directly migrating a Satellite 6.16.x database (from RHEL 8) to a fresh Satellite 6.17 installation (on RHEL 9 with an external DB) is **NOT a straightforward or officially recommended path by Red Hat for preserving all data and ensuring stability.** The database schemas change between versions.

**Recommended Approach (In-Place Upgrade First, then Migrate DB):**

The most reliable method involves upgrading your *existing* Satellite 6.16.x on RHEL 8 to Satellite 6.17 *first*. This ensures the database schema is correctly updated by the official upgrade process. Then, you migrate this upgraded 6.17 database to your new external PostgreSQL server.

**Steps (Assuming you've upgraded your old Satellite to 6.17 on RHEL 8):**

1.  **Prepare the New RHEL 9 PostgreSQL Server:**
    * Ensure the PostgreSQL server is set up as described in [Section 2](#2-setting-up-the-external-postgresql-server-rhel-9), but **do not run the Satellite installer against it yet if you intend to restore a migrated DB.** The databases (`foreman`, `pulpcore`) should ideally be empty or non-existent before restoring.

2.  **Backup the Upgraded Satellite 6.17 Database (on the old RHEL 8 server):**
    * Stop Satellite services on your *old* (now upgraded to 6.17) Satellite server:
        ```bash
        sudo satellite-maintain service stop
        ```
    * Perform a database dump for the `foreman` and `pulpcore` databases using `pg_dump`.
        * **For the `foreman` database (Candlepin, Foreman):**
            ```bash
            sudo -u postgres pg_dump -Fc foreman > /backup/foreman_db_617.dump
            ```
        * **For the `pulpcore` database (Pulp 3):**
            ```bash
            sudo -u postgres pg_dump -Fc pulpcore > /backup/pulpcore_db_617.dump
            ```
        *(Adjust paths and ensure the `postgres` user has write permissions to `/backup` or your chosen backup location.)*
    * Restart services on the old Satellite if you need it running temporarily:
        ```bash
        sudo satellite-maintain service start
        ```

3.  **Transfer Backup Files:**
    * Securely transfer `foreman_db_617.dump` and `pulpcore_db_617.dump` to your **new RHEL 9 PostgreSQL server**.

4.  **Restore Databases on the New RHEL 9 PostgreSQL Server:**
    * **Important:** Ensure no Satellite 6.17 installation has been run against these databases yet, or drop and recreate them if necessary.
    * Create the database users (`foreman_user`, `pulp_user`) if they don't exist. These must match what you intend to use in the `satellite-installer` command later.
        ```sql
        -- Connect to psql as postgres user
        CREATE USER foreman_user WITH PASSWORD '<foreman_db_secure_password>';
        CREATE USER pulp_user WITH PASSWORD '<pulp_db_secure_password>';
        ```
    * Create the databases and set ownership:
        ```sql
        CREATE DATABASE foreman OWNER foreman_user;
        CREATE DATABASE pulpcore OWNER pulp_user;
        ```
    * Restore the databases:
        ```bash
        sudo -u postgres pg_restore -d foreman -U foreman_user --no-owner --no-privileges --clean --if-exists /path/to/foreman_db_617.dump
        sudo -u postgres pg_restore -d pulpcore -U pulp_user --no-owner --no-privileges --clean --if-exists /path/to/pulpcore_db_617.dump
        ```
        * `--no-owner --no-privileges` are often used to avoid permission issues when restoring to a different environment/user setup. The `satellite-installer` should later fix permissions.
        * `--clean --if-exists` helps ensure a clean restore.

5.  **Run Satellite 6.17 Installer on the New RHEL 9 Satellite Server:**
    * Now, run the `satellite-installer` command as detailed in [Section 3, Step 5](#3-installing-satellite-617-on-rhel-9).
    * Use the **same database names, usernames, and passwords** that you used when creating/restoring the databases on the PostgreSQL server.
    * The installer should detect the existing (migrated) databases and adapt/update them as necessary for version 6.17.

6.  **Post-Migration Checks:**
    * Thoroughly check Satellite functionality: content sync, host registration, errata, etc.
    * Run `sudo satellite-maintain health check`.
    * Review logs for any errors: `/var/log/foreman-installer/satellite.log`, `/var/log/foreman/production.log`, Pulp logs in `/var/log/pulpcore/`.

**Alternative (Risky - Fresh Install & Data Re-Sync/Re-Configure):**

If an in-place upgrade of the old Satellite is not feasible, the alternative is:
1.  Install Satellite 6.17 fresh on RHEL 9 with the external DB (as per Sections 2 & 3).
2.  Manually re-create configurations (organizations, locations, activation keys, content views, sync plans, etc.) on the new Satellite.
3.  Re-sync all Red Hat repository content from scratch.
4.  Re-register all client hosts to the new Satellite server.
This method does not preserve historical data, job history, or trend data.

**Always consult official Red Hat Satellite documentation and consider Red Hat support for complex migration scenarios.**
*** https://docs.redhat.com/en/documentation/red_hat_satellite/6.17/html/installing_satellite_server_in_a_connected_network_environment/installing_server_connected_satellite ***

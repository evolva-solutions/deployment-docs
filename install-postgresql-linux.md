# Install PostgreSQL on Ubuntu

Steps to install PostgreSQL on Ubuntu, start the service, and create an application database and user. Commands assume you are logged in as a user who can run `sudo`.

## Before you start

Confirm the OS:

```bash
cat /etc/os-release
```

The commands below use the Ubuntu packages. For a specific major version from the PostgreSQL project repository, see [Install a specific major version](#install-a-specific-major-version).

## Install PostgreSQL

```bash
sudo apt update
sudo apt install -y postgresql postgresql-contrib
```

The installer creates a `postgres` system user and starts the service.

Check the version and service:

```bash
psql --version
sudo systemctl status postgresql
sudo systemctl enable postgresql
```

## Create a database and user

Switch to the `postgres` system user and open the SQL shell:

```bash
sudo -u postgres psql
```

Run the following in `psql`. Replace the password and names before you run them.

```sql
CREATE USER app_user WITH PASSWORD 'replace-with-a-strong-password';
CREATE DATABASE app_db OWNER app_user;
GRANT ALL PRIVILEGES ON DATABASE app_db TO app_user;
\q
```

Test the login:

```bash
psql -h 127.0.0.1 -U app_user -d app_db
```

If the password prompt fails on a local connection, the server is still using peer or ident authentication for local sockets. Connect as the `postgres` user with `sudo -u postgres psql`.

## Remote access

### Step 1: Enable public connections

```bash
sudo nano /etc/postgresql/16/main/postgresql.conf
```

Find:

```text
#listen_addresses = 'localhost'
```

Change to:

```text
listen_addresses = '*'
```

### Step 2: Allow remote connections

```bash
sudo nano /etc/postgresql/16/main/pg_hba.conf
```

Add this at the bottom if you want to allow a specific IP:

```text
host    evolva_billing    postgres    YOUR_PUBLIC_IP/32    scram-sha-256
```

If you want all IP access, add this instead:

```text
host    all    all    0.0.0.0/0    scram-sha-256
```

This allows password-authenticated connections for all databases and users from any IP address.

If you also want IPv6 connections, add:

```text
host    all    all    ::/0    scram-sha-256
```

### Step 3: Open port 5432

Run:

```bash
sudo ufw allow 5432/tcp
```

Check:

```bash
sudo ufw status
```

Also allow TCP port 5432 in your hosting provider's firewall/security rules.

### Step 4: Restart PostgreSQL

```bash
sudo systemctl restart postgresql
```

Check:

```bash
sudo systemctl status postgresql
```

### Step 5: Verify PostgreSQL is listening publicly

```bash
sudo ss -lntp | grep 5432
```

Expected output should contain:

```text
0.0.0.0:5432
```

## Install a specific major version

Distribution packages may not be the current major release. To install a chosen version from the PostgreSQL project (example: PostgreSQL 16):

```bash
sudo apt install -y postgresql-common
sudo /usr/share/postgresql-common/pgdg/apt.postgresql.org.sh
sudo apt install -y postgresql-16 postgresql-contrib-16
```

Change `16` to the major version you need. The service name stays `postgresql`.

## Useful checks

| Task | Command |
| --- | --- |
| Service status | `sudo systemctl status postgresql` |
| Installed version | `sudo -u postgres psql -c "SELECT version();"` |
| Listening port | `sudo ss -lntp \| grep 5432` |
| List databases | `sudo -u postgres psql -c "\l"` |
| Recent logs | `sudo journalctl -u postgresql -n 50 --no-pager` |

## Notes

- Port `5432` is open to any address. Use a strong password, and allow TCP port `5432` in the hosting provider firewall as well.
- Store the application password in the application secret store, not in shell history or this document.
- Take a backup before upgrades. A logical dump of one database:

  ```bash
  sudo -u postgres pg_dump -Fc app_db > app_db.dump
  ```

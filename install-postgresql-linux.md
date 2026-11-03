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

If the password prompt fails on a local connection, the server is still using peer or ident authentication for local sockets. Connect as the `postgres` user with `sudo -u postgres psql`, or follow the authentication section below.

## Allow password login from localhost

Edit the host-based authentication file. The path includes the major version, for example `16`:

```bash
sudo -u postgres psql -c "SHOW hba_file;"
```

Open the file that command prints. For application connections over TCP, use a `scram-sha-256` line **above** any broader `ident` or `peer` lines:

```text
# TYPE  DATABASE  USER      ADDRESS         METHOD
host    app_db    app_user  127.0.0.1/32    scram-sha-256
host    app_db    app_user  ::1/128         scram-sha-256
```

Set the password encryption method and reload:

```bash
sudo -u postgres psql -c "ALTER SYSTEM SET password_encryption = 'scram-sha-256';"
sudo systemctl reload postgresql
```

Set the user password again so it is stored with SCRAM:

```bash
sudo -u postgres psql -c "ALTER USER app_user WITH PASSWORD 'replace-with-a-strong-password';"
```

## Remote access

Skip this section if the application runs on the same server. PostgreSQL listens only on localhost until you change it.

1. Allow the server to listen on the network interface:

   ```bash
   sudo -u postgres psql -c "ALTER SYSTEM SET listen_addresses = '*';"
   ```

   Use a specific address, such as `'10.0.0.10'`, when the server has more than one interface.

2. Add a client rule in `pg_hba.conf` for the application network only. Example for a single application host:

   ```text
   host    app_db    app_user    10.0.0.20/32    scram-sha-256
   ```

   Do not use `0.0.0.0/0` on a server that is reachable from the public internet.

3. Reload PostgreSQL:

   ```bash
   sudo systemctl reload postgresql
   ```

4. Open port `5432` for that client address with `ufw`:

   ```bash
   sudo ufw allow from 10.0.0.20 to any port 5432 proto tcp
   sudo ufw status
   ```

5. From the application host:

   ```bash
   psql "host=SERVER_IP port=5432 dbname=app_db user=app_user"
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

- Keep the database on a private network. Expose port `5432` only to the application hosts that need it.
- Store the application password in the application secret store, not in shell history or this document.
- Take a backup before upgrades. A logical dump of one database:

  ```bash
  sudo -u postgres pg_dump -Fc app_db > app_db.dump
  ```

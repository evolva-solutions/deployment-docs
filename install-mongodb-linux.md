# Install MongoDB on Ubuntu

Steps to install MongoDB 9.0 Community Edition from the official MongoDB repository, start the service, and create an application user. Commands assume you are logged in as a user who can run `sudo`.

The `mongodb` package shipped by Ubuntu is not the MongoDB Inc. package and conflicts with `mongodb-org`. Remove it before installing if it is present:

```bash
sudo apt remove -y mongodb mongodb-server mongodb-clients
```

## Before you start

Confirm the Ubuntu release:

```bash
cat /etc/os-release
```

MongoDB 9.0 Community supports Ubuntu 22.04 (jammy), 24.04 (noble), and 26.04 (resolute).

## Install MongoDB

Install `gnupg` and `curl`, then add the MongoDB 9.0 signing key:

```bash
sudo apt-get install -y gnupg curl
curl -fsSL https://pgp.mongodb.com/server-9.asc | sudo gpg -o /usr/share/keyrings/mongodb-server-9.gpg --dearmor
```

Add the repository that matches the Ubuntu release. Run only one of these:

Ubuntu 26.04:

```bash
echo "deb [ arch=amd64,arm64 signed-by=/usr/share/keyrings/mongodb-server-9.gpg ] https://repo.mongodb.org/apt/ubuntu resolute/mongodb-org/9.0 multiverse" | sudo tee /etc/apt/sources.list.d/mongodb-org-9.0.list
```

Ubuntu 24.04:

```bash
echo "deb [ arch=amd64,arm64 signed-by=/usr/share/keyrings/mongodb-server-9.gpg ] https://repo.mongodb.org/apt/ubuntu noble/mongodb-org/9.0 multiverse" | sudo tee /etc/apt/sources.list.d/mongodb-org-9.0.list
```

Ubuntu 22.04:

```bash
echo "deb [ arch=amd64,arm64 signed-by=/usr/share/keyrings/mongodb-server-9.gpg ] https://repo.mongodb.org/apt/ubuntu jammy/mongodb-org/9.0 multiverse" | sudo tee /etc/apt/sources.list.d/mongodb-org-9.0.list
```

Install and start MongoDB:

```bash
sudo apt-get update
sudo apt-get install -y mongodb-org
sudo systemctl enable --now mongod
sudo systemctl status mongod
```

If `systemctl start` reports that `mongod.service` is not found, reload units and start again:

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now mongod
```

## Create an administrator and an application user

Connect on the server before authentication is turned on:

```bash
mongosh
```

In `mongosh`, replace both passwords, then exit:

```javascript
use admin
db.createUser({
  user: "admin",
  pwd: "replace-with-a-strong-password",
  roles: [ { role: "root", db: "admin" } ]
})

use app_db
db.createUser({
  user: "app_user",
  pwd: "replace-with-a-strong-password",
  roles: [ { role: "readWrite", db: "app_db" } ]
})

exit
```

Turn on access control. Edit `/etc/mongod.conf` and set:

```yaml
security:
  authorization: enabled
```

Restart and sign in as the application user:

```bash
sudo systemctl restart mongod
mongosh "mongodb://app_user@127.0.0.1:27017/app_db" --authenticationDatabase app_db
```

`mongosh` asks for the password. A successful prompt then shows `app_db`.

## Remote access

Skip this section if the application runs on the same server. MongoDB listens only on `127.0.0.1` until you change it. Enable authorization in the previous section before opening any remote address.

1. Edit `/etc/mongod.conf` and set `bindIp` to localhost plus the server address that clients should use:

   ```yaml
   net:
     port: 27017
     bindIp: 127.0.0.1,10.0.0.10
   ```

   Do not set `bindIp` to `0.0.0.0` on a server that is reachable from the public internet.

2. Restart MongoDB:

   ```bash
   sudo systemctl restart mongod
   ```

3. Open port `27017` for the application host only with `ufw`:

   ```bash
   sudo ufw allow from 10.0.0.20 to any port 27017 proto tcp
   sudo ufw status
   ```

4. From the application host:

   ```bash
   mongosh "mongodb://app_user@10.0.0.10:27017/app_db" --authenticationDatabase app_db
   ```

## Useful checks

| Task | Command |
| --- | --- |
| Service status | `sudo systemctl status mongod` |
| Installed version | `mongod --version` |
| Listening port | `sudo ss -lntp \| grep 27017` |
| Recent logs | `sudo tail -n 50 /var/log/mongodb/mongod.log` |
| List databases as admin | `mongosh --username admin --authenticationDatabase admin --eval "db.adminCommand({ listDatabases: 1 })"` |

## Notes

- Keep MongoDB on a private network. Expose port `27017` only to the application hosts that need it.
- Store passwords in the application secret store, not in shell history or this document.
- Take a dump before upgrades:

  ```bash
  mongodump --username admin --authenticationDatabase admin --db app_db --out /var/backups/app_db
  ```

- To stay on the installed 9.0 packages until you choose to upgrade, hold them:

  ```bash
  echo "mongodb-org hold" | sudo dpkg --set-selections
  echo "mongodb-org-database hold" | sudo dpkg --set-selections
  echo "mongodb-org-server hold" | sudo dpkg --set-selections
  echo "mongodb-mongosh hold" | sudo dpkg --set-selections
  echo "mongodb-org-mongos hold" | sudo dpkg --set-selections
  echo "mongodb-org-tools hold" | sudo dpkg --set-selections
  ```

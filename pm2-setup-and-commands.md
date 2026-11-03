# PM2 Setup and Commands on Ubuntu

Steps to install PM2 on Ubuntu, run a Node.js app under it, and keep that app running after a reboot. Commands assume you are logged in as the user that should own the app process.

## Before you start

PM2 needs Node.js and npm. Check both:

```bash
node -v
npm -v
```

If either command is missing, install a current Node.js LTS release with nvm before continuing. See [Install Node.js with nvm](install-node-nvm.md). The Ubuntu `nodejs` package is often too old for current apps. With nvm, install PM2 as that same user, without `sudo`.

Run PM2 as the application user, not as root. Create that user if the server does not already have one:

```bash
sudo adduser --disabled-password --gecos "" app
sudo mkdir -p /var/www/app
sudo chown -R app:app /var/www/app
```

## Install PM2

```bash
npm install -g pm2
pm2 -v
```

If `pm2` is not found afterward, the shell does not have nvm loaded. Open a login shell and check `node -v` again. See [Install Node.js with nvm](install-node-nvm.md). The global bin directory is `$(npm prefix -g)/bin`, inside `~/.nvm`, and nvm adds it to `PATH` when it loads.

## Start an application

From the application directory:

```bash
cd /var/www/app
pm2 start server.js --name app
```

Other common entry points:

```bash
pm2 start npm --name app -- start
pm2 start yarn --name app -- start
pm2 start dist/main.js --name api
```

Give the process a fixed name with `--name`. Later commands use that name.

Check that it stayed up:

```bash
pm2 status
pm2 logs app --lines 50
```

## Save the process list and start on boot

`pm2 start` lasts only until the server reboots unless you save the list and install a startup script.

```bash
pm2 save
pm2 startup
```

`pm2 startup` prints one `sudo` command. Run that command as printed. It registers a systemd service for the current user. Then save again so the service restores the same apps:

```bash
pm2 save
```

Confirm the unit exists:

```bash
systemctl status pm2-$USER
```

After a reboot, `pm2 status` should show the saved apps without starting them by hand.

## Everyday commands

| Task | Command |
| --- | --- |
| List processes | `pm2 status` |
| Start by name | `pm2 start app` |
| Stop | `pm2 stop app` |
| Restart | `pm2 restart app` |
| Reload with zero downtime (cluster mode) | `pm2 reload app` |
| Delete from PM2 | `pm2 delete app` |
| Follow logs | `pm2 logs app` |
| Last 100 log lines | `pm2 logs app --lines 100` |
| Process details | `pm2 show app` |
| Live CPU and memory | `pm2 monit` |
| Save the current list | `pm2 save` |
| Restore the saved list | `pm2 resurrect` |

`pm2 delete` removes the process from PM2. It does not delete application files. Run `pm2 save` after start, stop, or delete if that change should survive a reboot.

## Restart after a new deploy

After the new build is on disk:

```bash
cd /var/www/app
pm2 restart app
pm2 save
```

Use `pm2 reload app` when the app was started in cluster mode and should keep serving traffic during the restart.

## Cluster mode

Run one process per CPU when the app is stateless:

```bash
pm2 start server.js --name app -i max
```

`-i 2` starts two instances. `-i max` starts one per CPU. Cluster mode applies to Node.js apps. For an `npm` script, prefer an ecosystem file so PM2 clusters the script correctly.

## Ecosystem file

Use an ecosystem file when the app needs a fixed script, working directory, environment, or instance count. Create `ecosystem.config.cjs` in the app directory:

```javascript
module.exports = {
  apps: [
    {
      name: "app",
      cwd: "/var/www/app",
      script: "server.js",
      instances: 1,
      exec_mode: "fork",
      env: {
        NODE_ENV: "production",
        PORT: 3000,
      },
    },
  ],
};
```

For cluster mode, set `instances` to `"max"` (or a number) and `exec_mode` to `"cluster"`.

Apply it:

```bash
pm2 start ecosystem.config.cjs
pm2 save
```

Update a running app after editing the file:

```bash
pm2 reload ecosystem.config.cjs
pm2 save
```

Start only one app from a file that defines several:

```bash
pm2 start ecosystem.config.cjs --only app
```

## Logs

PM2 writes stdout and stderr under `~/.pm2/logs/` by default.

```bash
pm2 logs app
pm2 flush app
```

`pm2 flush` clears stored logs. It does not stop the process.

Install log rotation so logs do not fill the disk:

```bash
pm2 install pm2-logrotate
pm2 set pm2-logrotate:max_size 50M
pm2 set pm2-logrotate:retain 14
pm2 set pm2-logrotate:compress true
```

## Environment and port

Pass variables for one start:

```bash
pm2 start server.js --name app --update-env -- PORT=3000
```

Prefer `env` in the ecosystem file for values that must stay set across restarts. Do not put secrets in the ecosystem file if that file is committed. Load secrets from the server environment or a file that is not in git.

Confirm the port is listening:

```bash
ss -lntp | grep 3000
```

## Useful checks

| Task | Command |
| --- | --- |
| PM2 version | `pm2 -v` |
| Process list | `pm2 status` |
| Why a process crashed | `pm2 logs app --err --lines 100` |
| Startup system | `pm2 startup` |
| Saved process dump | `pm2 prettylist` or `cat ~/.pm2/dump.pm2` |
| Startup service | `systemctl status pm2-$USER` |

## Update or remove PM2

```bash
npm install -g pm2@latest
pm2 update
```

To remove PM2 from boot, then uninstall it:

```bash
pm2 unstartup
npm uninstall -g pm2
```

`pm2 unstartup` prints a `sudo` command. Run that command so systemd no longer starts PM2.

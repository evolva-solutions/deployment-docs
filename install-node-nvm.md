# Install Node.js with nvm on Ubuntu

Steps to install nvm and a Node.js LTS release for the application user. Commands that use `sudo` can be run by an administrator. nvm and Node.js are installed as the user that will own the app and run PM2.

## Before you start

Confirm the OS:

```bash
cat /etc/os-release
```

nvm installs into that user's home directory. It is not shared with other accounts. Use the application user from the [PM2 setup](pm2-setup-and-commands.md). Create it if the server does not already have one:

```bash
sudo adduser --disabled-password --gecos "" app
sudo mkdir -p /var/www/app
sudo chown -R app:app /var/www/app
```

Log in as that user for the nvm steps (`sudo su - app`). Do not install nvm as root. A root install lands in `/root/.nvm`, and the app user cannot see it.

Install Git first. The nvm installer clones its own repository. See [Install Git](install-git.md).

The nvm installer needs `curl`. Install it once:

```bash
sudo apt update
sudo apt install -y curl ca-certificates
```

## Install nvm

As the `app` user:

```bash
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.40.8/install.sh | bash
```

The installer clones nvm into `~/.nvm` and adds the load lines to `~/.bashrc`. Load it in the current shell:

```bash
export NVM_DIR="$HOME/.nvm"
[ -s "$NVM_DIR/nvm.sh" ] && . "$NVM_DIR/nvm.sh"
```

Check that the function is available. `which nvm` stays empty, because nvm is a shell function, not a binary:

```bash
command -v nvm
nvm --version
```

Open a new login shell, or run `bash -lc 'command -v nvm'`, and confirm nvm is still found.

## Install Node.js

Install the current LTS line and make it the default for new shells:

```bash
nvm install --lts
nvm alias default lts/*
node -v
npm -v
```

To pin a major line instead, pass the number. This installs the latest Node.js 22 release and selects it:

```bash
nvm install 22
nvm alias default 22
```

The default alias is what a new shell runs. `nvm use 22` changes only the current shell.

## New shells and non-interactive commands

Ubuntu's `~/.bashrc` returns immediately when the shell is not interactive, so the nvm lines at the bottom never run for `bash -c` or most deploy scripts. Login shells still work, because Ubuntu's `~/.profile` sources `~/.bashrc` and a login shell is interactive.

Confirm a login shell sees Node:

```bash
bash -lc 'node -v && npm -v'
```

For a command that does not start a login shell, load nvm in that command:

```bash
bash -c 'export NVM_DIR="$HOME/.nvm"; . "$NVM_DIR/nvm.sh"; node -v'
```

Run `pm2 startup` from a shell where `node -v` already works. PM2 records the `PATH` of that shell into the systemd unit. If `node` is missing there, the app will not come back after a reboot. Install PM2 without `sudo` so it lands in the nvm Node prefix. See [PM2 setup](pm2-setup-and-commands.md).

## Native npm modules

Some packages compile C++ addons during `npm ci`. Install the compilers once, as an administrator:

```bash
sudo apt install -y build-essential python3
```

Skip this if `npm ci` finishes without a `node-gyp` error.

## Everyday commands

Run these as the `app` user, after nvm is loaded.

| Task | Command |
| --- | --- |
| List installed Node versions | `nvm ls` |
| Install current LTS | `nvm install --lts` |
| Install a major line | `nvm install 22` |
| Use a version in this shell | `nvm use 22` |
| Default for new shells | `nvm alias default 22` |
| Path to the current node binary | `nvm which current` |
| Update nvm itself | see [Update nvm](#update-nvm) |

## Update nvm

```bash
cd "$HOME/.nvm"
git fetch --tags origin
git checkout v0.40.8
. "$HOME/.nvm/nvm.sh"
```

`git` must already be installed. See [Install Git](install-git.md). Replace `v0.40.8` with the release you want. Then open a new shell and check `nvm --version`.

## Notes

- Do not run `sudo npm` or `sudo nvm` for this user. `sudo` switches to root, which has a different home directory and will not see this Node install.
- Each Linux user who needs Node needs their own nvm install.
- `nvm alias default` must be set before you rely on Node in a new shell. The first version you install becomes the default until you change it.

# Install Git on Ubuntu

Steps to install Git, add a deploy key, and clone the application repository. Commands that use `sudo` can be run by an administrator. The key and the clone belong to the user that will own the app.

## Before you start

Confirm the OS:

```bash
cat /etc/os-release
```

Use the application user from the [PM2 setup](pm2-setup-and-commands.md). Create it if the server does not already have one:

```bash
sudo adduser --disabled-password --gecos "" app
sudo mkdir -p /var/www/app
sudo chown -R app:app /var/www/app
```

Log in as that user for the key and clone steps (`sudo su - app`).

## Install Git

```bash
sudo apt update
sudo apt install -y git ca-certificates
git --version
```

`ca-certificates` lets Git verify HTTPS remotes.

## Clone with a deploy key

A private repository needs a key that belongs to the `app` user. Create a deploy key with an empty passphrase so a later pull does not wait for input. Grant that key read access to this repository only.

```bash
ssh-keygen -t ed25519 -C "app@example.com" -f ~/.ssh/id_ed25519 -N ""
cat ~/.ssh/id_ed25519.pub
```

Add the printed public key on the Git host, then confirm and clone. `/var/www/app` must be empty:

```bash
ssh -T git@github.com
git clone git@github.com:org/repo.git /var/www/app
```

Change the host and the repository path to match the project. GitHub answers `ssh -T` with a success message and a non-zero status; that still means the key was accepted.

A public repository can be cloned over HTTPS instead, with no key:

```bash
git clone https://github.com/org/repo.git /var/www/app
```

## Name and email

Set a name and email only if this user will create commits:

```bash
git config --global user.name "App"
git config --global user.email "app@example.com"
git config --global init.defaultBranch main
```

Pulls and clones do not need this.

## Everyday commands

Run these as the `app` user.

| Task | Command |
| --- | --- |
| Git version | `git --version` |
| Current branch and status | `git -C /var/www/app status` |
| Pull the app | `git -C /var/www/app pull` |
| Show the remote | `git -C /var/www/app remote -v` |
| Recent commits | `git -C /var/www/app log -5 --oneline` |

## Notes

- Keep the deploy private key on the server. Copy only the `.pub` file to the Git host.
- One deploy key should open only the repository it deploys. Do not reuse a personal key that can push to other repositories.
- Node.js for this app is installed separately. See [Install Node.js with nvm](install-node-nvm.md).

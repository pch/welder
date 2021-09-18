# 👨‍🏭 Welder

Welder allows you to set up a Linux server with plain shell scripts.

I wrote it out of frustration with Ansible. Ansible is an amazing and powerful
tool, but for my needs it's just too much. 90% of the time all I need is to
be able to run a shell script on the server, without extra dependencies.

In most basic terms, that's what welder does.

But there's some more.

**⚠️ NOTE**: if you're looking for the previous version of welder, you'll
[find it here](https://github.com/pch/welder/tree/classic).

## Features

- set up your server with a single command (`welder run <playbook> <server>`)
- run a set of organized reusable shell scripts
- use simple template syntax (`{{ VAR_NAME }}`) to substitute config variables

### Directory structure

An example directory structure:

```sh
├── playbook.conf
├── config.conf
├── firewall
│   ├── files
│   │   └── rules.v4
│   └── firewall.sh
├── nginx
│   └── nginx.sh
├── system
│   ├── files
│   │   ├── 10periodic
│   │   ├── 50unattended-upgrades
│   │   └── ssh_key
│   └── system.sh
└── website
    ├── files
    │   └── site.conf.template
    └── website.sh
```

### Playbook

Playbook is just a list of modules to execute. Example:

```sh
# playbook.conf

system
firewall
nginx
website
```

### Config

Config file:

```sh
SITE_DOMAIN = "example.com"
SITE_DIR = "/var/www"
```

You can reference config variables in your scripts like this:

```sh
#!/bin/sh
set -xeu

. ./config.conf

echo $SITE_DOMAIN
```

### Templates

Welder offers simple `sed`-based templates that interpolate variables in double brackets
with values defined in config.

```lua
# website/files/nginx-site.conf.template

server {
    listen 80;

    server_name {{ SITE_DOMAIN }};
    root {{ SITE_DIR }}/current/public;
}
```

## Usage

Run the playbook with the following command:

```sh
welder run playbook.conf user@example.com
```

### How it works

Welder goes through the modules defined in `playbook.conf`, copies them to a
cache directory, compiles config files and templates, `rsync`s the directory to
the server. Then it runs the `setup` script that invokes all `*.sh` files
within the playbook (all scripts will be called with `sudo`).

### Example setup script

```sh
# nginx/nginx.sh

# NOTE: sudo isn't necessary because the whole script will be
#       invoked as `sudo nginx/nginx.sh`

set -xeu # 'u' will give you warnings on unbound config variables

add-apt-repository -y ppa:nginx/stable
sapt-get update && apt-get install -y nginx

service nginx start

cp files/nginx.conf /etc/nginx/nginx.conf

# Disable default site
if [ -f /etc/nginx/sites-enabled/default ]; then
  rm /etc/nginx/sites-enabled/default
fi

service nginx restart
```

## Installation

The only dependency required by welder is `rsync` (which should be
pre-installed on your system in most cases).

1. Check out welder into `~/.welder` (or whatever location you prefer):

   ```sh
   $ git clone https://github.com/pch/welder.git ~/.welder
   ```

2. Add `~/.welder/bin` to your `$PATH` for access to the `welder`
   command-line utility.

   ```sh
   $ echo 'export PATH="$PATH:$HOME/.welder/bin"' >> ~/.bash_profile
   ```

   **Ubuntu Desktop note**: Modify your `~/.bashrc` instead of `~/.bash_profile`.

   **Zsh note**: Modify your `~/.zshrc` file instead of `~/.bash_profile`.

3. Restart your shell so that PATH changes take effect. (Opening a new
   terminal tab will usually do it.) Now check if welder was set up:

   ```sh
   $  which welder
   /Users/my-user/Code/welder/bin/welder
   ```

## Caveats

Since welder allows you to run **anything** on the server, you should use it
with caution. It won't protect you from screw-ups, like
`rm -rf "/$undefined_variable"`.

Use at your own risk.

## Alternatives

There's an [alternative version](https://gitlab.com/welder-cm/welder) of welder
(the classic version), re-implemented in Python by
[@thomas-mc-work](https://github.com/thomas-mc-work).

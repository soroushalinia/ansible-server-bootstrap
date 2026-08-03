# ansible-server-bootstrap

One Ansible playbook that sets up a brand new Ubuntu or Debian server the way
a careful engineer would on day one:

- a dedicated admin user with key based SSH login (passwords disabled,
  modern crypto only: key exchange, ciphers and MACs are pinned to
  OpenSSH 8.9+ algorithms)
- a firewall (ufw) that blocks all incoming traffic except what you open
- Docker from the official repository
- automatic security updates (unattended-upgrades)
- fail2ban, which blocks IPs that try to guess your SSH password
- node_exporter, a small monitoring agent for Prometheus on port 9100
- optional support for an apt mirror and a Docker registry mirror

There are no roles, no plugins, no galaxy glue. One playbook file. It is safe
to run once, twice, or a hundred times: Ansible only changes what needs
changing, so a second run does nothing. That property is called idempotency.

## Before and after

**Before. A fresh box.** You can only get in with the default user and a
password, the firewall is off, and none of the tools are installed.

```
ubuntu@server1:~$ sudo ufw status
Status: inactive
ubuntu@server1:~$ docker --version
bash: docker: command not found
```

**After. One run of the playbook.** The box is locked down, and everything
is running and verified.

```
admin@server1:~$ sudo ufw status verbose
Status: active
Default: deny (incoming), allow (outgoing)
22/tcp   ALLOW IN   Anywhere
9100/tcp ALLOW IN   10.0.0.0/8   # only once node_exporter_allow_cidr is set

admin@server1:~$ sudo /usr/sbin/sshd -T | grep -E "PasswordAuthentication|PermitRootLogin|AuthenticationMethods"
passwordauthentication no
permitrootlogin no
authenticationmethods publickey

admin@server1:~$ docker --version
Docker version 27.4.0

admin@server1:~$ curl -s localhost:9100/metrics | head -1
# HELP node_boot_time_seconds Node boot time, in unixtime.
```

The third play of the playbook checks all of this itself and prints a summary
at the end, so you do not need to log in to verify. Rerunning the playbook is
a no op: every step reports `ok`.

## Do I need Ansible experience?

No. This page walks you through everything, including installing Ansible.
What you need:

- your own computer with Linux or macOS (this is the "controller", the
  machine Ansible runs from)
- a fresh Ubuntu 22.04 or 24.04, or Debian 12 or 13, server that you can
  reach over SSH, with a user that can use `sudo` (cloud providers call this
  user `ubuntu` or `debian`, and it is created when the server is made)

## What is Ansible, in 30 seconds

Ansible is a tool that connects to machines over SSH and makes them match a
description you wrote. The description is called a playbook: a text file with
a list of steps. You run `ansible-playbook playbook.yml` on your computer,
Ansible connects to the server, and each step either changes something or
reports `ok` because nothing needed to change.

That is the whole idea. One file, one command, repeatable.

## Step 1. Install Ansible on your computer

Linux (Debian or Ubuntu based):

```bash
sudo apt install ansible
```

macOS (with Homebrew):

```bash
brew install ansible
```

Check that it works:

```bash
ansible --version
```

Now install the extra Ansible "collections" (small libraries the playbook
uses). Run this inside the repository folder:

```bash
ansible-galaxy collection install -r requirements.yml
```

## Step 2. Copy the repository to your computer

```bash
git clone <this-repo-url>
cd ansible-server-bootstrap
```

If you downloaded the files instead of using git, just unpack them and open a
terminal in the folder.

## Step 3. Tell Ansible where your server is

The file `inventory/hosts.yml` lists your servers. Open it and uncomment the
example, then fill in the IP address and the user you can log in as:

```yaml
my-server:
  ansible_host: 192.168.1.50      # your server's IP
  ansible_user: ubuntu            # the user you use to log in today
```

You can name the entry anything you like. `my-server` is just a label.

## Step 4. Decide how you will log in as the admin user

The playbook creates a user named `admin` on your server. To log in as
`admin`, you need an SSH key pair. You have two options:

**Option A: use a key you already have (recommended).** If you already use
SSH keys, find your public key and copy its contents:

```bash
cat ~/.ssh/id_ed25519.pub
```

Paste the whole line into `inventory/group_vars/all.yml`:

```yaml
admin_ssh_public_key: "ssh-ed25519 AAAA... your-email@example.com"
```

**Option B: let the playbook create a key for you.** Leave
`admin_ssh_public_key` empty (it is empty by default). During the first run
the playbook creates a key pair on your computer in the folder
`keys/<your-server-name>/` and prints the public key and its fingerprint.

Do not have a key pair at all? Create one first:

```bash
ssh-keygen -t ed25519
```

Then use option A with the file it created.

## Step 5. Run the playbook

```bash
ansible-playbook playbook.yml -i inventory/hosts.yml -K
```

What the parts mean:

- `ansible-playbook` is the Ansible program that runs playbooks
- `playbook.yml` is the playbook file
- `-i inventory/hosts.yml` tells Ansible where your servers are
- `-K` asks you for the `sudo` password of the current user, because some
  steps need root rights (this is the password of the `ubuntu` or `debian`
  user, not a new one)

The first run takes several minutes. It installs packages, downloads Docker,
and applies the hardening. You will see every step print `ok` or `changed`.
`changed` means "this step did something on this run". A second run prints
`ok` everywhere.

## What happens during the run

The playbook has three parts, called plays:

1. **Bootstrap.** It logs in as the user you put in the inventory, creates
   the `admin` user, gives it passwordless `sudo`, and installs the SSH key.
   If you chose option B, the key is generated now.
2. **Harden and provision.** It reconnects as `admin` using the key. This is
   deliberate: it proves key login works before anything is locked down.
   Then it sets the timezone, updates packages, enables the firewall, applies
   the SSH hardening, installs fail2ban, enables automatic security updates,
   installs Docker, and installs node_exporter.
3. **Verify.** It checks that the firewall is active, SSH login is key based
   only, Docker is running, fail2ban is running, and node_exporter answers on
   port 9100. If anything is wrong, the run fails loudly.

## After the first run

The old user (`ubuntu` or `debian`) can no longer log in with a password.
That is the point. Log in with the admin user instead:

```bash
ssh -i keys/my-server/id_ed25519 admin@192.168.1.50
```

(If you used your own key, drop the `-i` part and use your normal setup.)

The file `keys/my-server/id_ed25519` is your private key. Treat it like a
password. It is ignored by git, so it will never end up in the repository.

## Rerunning the playbook

The playbook is safe to run any time; it brings the server back to the
described state. One thing changed though: the old user is locked out, so
Ansible must use `admin` from now on. In `inventory/hosts.yml`:

```yaml
my-server:
  ansible_host: 192.168.1.50
  ansible_user: admin                        # changed from ubuntu
  ansible_ssh_private_key_file: keys/my-server/id_ed25519
```

If you use your own key with the SSH agent, you can skip the last line.
Then run the same command as before. Everything reports `ok`.

## Running only part of the playbook

Every step has a tag, so you can run subsets. This is handy after you change
a setting, for example the firewall rules.

| Tag | What it runs |
|---|---|
| `bootstrap` | Play 1: admin user, sudo, SSH key |
| `user` | admin user, sudoers entry, SSH key |
| `ssh` | SSH daemon hardening |
| `ufw` | firewall rules |
| `sysctl` | kernel and network hardening |
| `fail2ban` | login attempt blocker |
| `unattended` | automatic security updates |
| `docker` | Docker install and configuration |
| `node_exporter` | monitoring agent and firewall rule |
| `upgrade` | package upgrade and reboot handling |
| `verify` | Play 3, the health checks |
| `baseline` | timezone and clock sync |
| `apt-mirror` | apt mirror rewrite |

Examples:

```bash
# only touch the firewall
ansible-playbook playbook.yml -i inventory/hosts.yml -t ufw

# only reapply SSH and firewall hardening
ansible-playbook playbook.yml -i inventory/hosts.yml -t ssh,ufw

# only run the health checks
ansible-playbook playbook.yml -i inventory/hosts.yml -t verify
```

## Configuration

Everything you might want to change lives in one file:
`inventory/group_vars/all.yml`. Open it and read the comments; each setting
is explained right where it is. The most common ones:

| Variable | Default | What it does |
|---|---|---|
| `admin_user` | `admin` | the user created on the server |
| `admin_ssh_public_key` | empty | your public key, or leave empty to generate one |
| `ssh_port` | `22` | SSH port; set in the sshd config (`Port`) and opened in the firewall |
| `apt_mirror` | empty | an apt mirror URL, for example `https://mirror.example.org/ubuntu` |
| `docker_registry_mirrors` | empty list | registry mirrors for Docker image pulls |
| `node_exporter_allow_cidr` | empty | who may read the metrics on port 9100. Empty keeps the port closed by the firewall. Set it to your monitoring server, for example `10.0.0.0/8`, to expose it. `0.0.0.0/0` opens it to the whole internet |
| `ufw_extra_allow` | empty | extra open ports, for example `{"8080": tcp}` |
| `unattended_reboot` | `true` | reboot automatically after security updates |
| `timezone` | `UTC` | server timezone |

### Using mirrors

Set the apt mirror when your server should not talk to the public Ubuntu or
Debian archives:

```yaml
apt_mirror: "https://mirror.example.org/ubuntu"
```

The playbook rewrites the package source files so that the main archive, the
security archive, and (on arm64) the ports archive all point at your mirror.
The mirror has to carry all of those suites. Leave the value empty to use the
official archives.

Set Docker registry mirrors to speed up image pulls in your network:

```yaml
docker_registry_mirrors:
  - "https://registry-mirror.example.org"
```

Both the apt mirror and the registry mirrors also accept a plain string when
you pass them on the command line, for example
`-e 'docker_registry_mirrors=https://mirror-a.example.org,https://mirror-b.example.org'`.
The same goes for `ufw_extra_allow` in JSON form:
`-e 'ufw_extra_allow={"8080":"tcp"}'`.

### A note on the docker group

The playbook adds `admin` to the `docker` group so you can run containers
without `sudo`. Keep in mind that membership in the `docker` group is
effectively root: anyone in it can start a container that mounts the host
filesystem and gain full access to the box. Only add users you trust.

## Troubleshooting

- **`Permission denied` during Play 1.** Either the user or password in the
  inventory is wrong, or the server does not allow SSH logins with that user.
  Check `ansible_user` in `inventory/hosts.yml` and the IP address.
- **`Permission denied` during Play 2.** The admin key is not where Ansible
  looks for it. Either add it to your SSH agent with
  `ssh-add keys/my-server/id_ed25519` or set `ansible_ssh_private_key_file`
  in `inventory/hosts.yml` (see Step 2).
- **`Permission denied` during Play 1 on a rerun.** The old user is locked
  out after the first run. Switch `ansible_user` in the inventory to `admin`.
- **`apt update` fails after you set a mirror.** The mirror must carry the
  main, security, and ports suites. Check with the mirror owner, or empty
  `apt_mirror` to go back to the official archives. If the playbook warned
  that the mirror rewrite matched nothing, the source file format probably
  changed; check `/etc/apt/sources.list` and `/etc/apt/sources.list.d/`.
- **Port 9100 not reachable from your monitoring server.** The default
  `node_exporter_allow_cidr` is empty, so ufw keeps the port closed (the
  verify play still checks metrics on localhost). Set the variable to your
  Prometheus CIDR and rerun with `-t ufw,node_exporter`.
- **Port 9100 reachable from the whole internet.** You set
  `node_exporter_allow_cidr` to `0.0.0.0/0`. Restrict it to the IP range of
  your monitoring server instead.
- **Host key warnings / `HOST KEY VERIFICATION FAILED`.** This project
  disables SSH host key checking in `ansible.cfg`, because a fresh server
  always has a new host key. If you prefer strict checking, set
  `host_key_checking = True` and pass
  `--ssh-common-args="-o StrictHostKeyChecking=accept-new"` on first runs.
- **You are locked out of the server.** Go to your cloud provider's web
  console and run `rm /etc/ssh/sshd_config.d/99-hardening.conf`, then
  `systemctl restart ssh`. That restores password login, so you can get back
  in and fix whatever went wrong.
- **Docker published ports are not filtered by the firewall.** This is
  normal Docker behavior: Docker manages its own traffic rules, which bypass
  ufw. Publish only the ports you need and put a reverse proxy in front of
  your containers.

## Files

```
playbook.yml                  the whole playbook, in three plays
inventory/hosts.yml           where your servers are listed
inventory/group_vars/all.yml  all settings and comments explaining them
templates/                    sshd settings, Docker config, fail2ban, systemd units, sudoers
files/                        sysctl hardening, apt update schedule
requirements.yml              the Ansible collections this playbook needs
ansible.cfg                   Ansible defaults for this project
keys/                         generated SSH keys, gitignored, created on first run
```

## License

MIT

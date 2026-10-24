# 🔐 Split-SSH on Qubes OS 4.3

> *The key never leaves the vault. Every use goes through dom0.*

A hands-on guide to keeping an SSH private key in an **offline qube** while using it from a **networked dev qube**, with an explicit approval prompt in dom0 for every use. Tested on **Qubes OS 4.3.2** with a Debian template, used here to authenticate to GitHub.

Includes the real troubleshooting log from my own setup: every error I hit, how I diagnosed it, and the fix.

---

## Why

On a classic workstation, `~/.ssh/id_ed25519` sits on the same machine that browses the web, clones untrusted code and installs packages. If that machine is compromised, the key can be copied and reused anywhere, silently.

Split-SSH applies two principles:

- **Compartmentalization**: the private key lives in a qube with **no network at all**.
- **Least privilege + explicit consent**: the dev qube can only *ask* the vault to sign an authentication challenge, and dom0 prompts me each time.

If `dev` is compromised, an attacker can at most trigger a prompt I can refuse. They can never exfiltrate the key.

## Architecture

```
┌──────────────┐   qrexec: qubes.SshAgent   ┌──────────────┐
│     dev      │ ─────────────────────────▶ │    vault     │
│  (network)   │                            │ (no network) │
│              │        ▲ approval          │              │
│ ssh / git    │        │                   │ ssh-agent    │
│     │        │   ┌────┴────┐              │ + private key│
│ socat socket │   │  dom0   │              │ socat → agent│
└──────────────┘   │ policy  │              └──────────────┘
                   └─────────┘
```

1. `ssh` in **dev** talks to a local UNIX socket, as if it were a normal agent.
2. `socat` forwards that socket over **qrexec** to the `qubes.SshAgent` service.
3. **dom0** checks the policy and asks for confirmation.
4. In **vault**, the service connects the stream to the real `ssh-agent`, which signs and replies. The private key itself never crosses the boundary.

## Setup

Replace `debian-12-xfce` with your template name (`qvm-ls --class TemplateVM`). In this guide the qubes are named `dev` and `vault`, with the default user `user`.

> ⚠️ Hardcoded paths are used on purpose. Several of my bugs came from a lost `$` in a variable (see troubleshooting). Absolute paths are boring and reliable.

### 1. dom0: create the qubes

```bash
qvm-create --class AppVM --template debian-12-xfce --label blue  dev
qvm-create --class AppVM --template debian-12-xfce --label black vault
qvm-prefs vault netvm ''
qvm-prefs vault netvm      # must print nothing
```

AppVMs only persist `/home`, `/rw` and `/usr/local`: everything else comes read-only from the template and is reset on reboot.

### 2. Template: install socat

```bash
sudo apt update && sudo apt install -y socat
```

Then shut the template down from dom0: `qvm-shutdown --wait debian-12-xfce`.

### 3. vault: key, agent, qrexec service

Generate the key (with a passphrase):

```bash
ssh-keygen -t ed25519 -C "my-key-comment"
```

Start an agent on a **fixed** socket, at the end of `~/.bashrc`:

```bash
export SSH_AUTH_SOCK="/home/user/.ssh/agent.sock"
ssh-add -l >/dev/null 2>&1
if [ $? -eq 2 ]; then                     # 2 = no agent reachable
  rm -f "$SSH_AUTH_SOCK"
  eval "$(ssh-agent -a "$SSH_AUTH_SOCK")" >/dev/null
fi
```

Create the qrexec service (persistent in `/usr/local`):

```bash
sudo mkdir -p /usr/local/etc/qubes-rpc
sudo tee /usr/local/etc/qubes-rpc/qubes.SshAgent >/dev/null <<'EOF'
#!/bin/sh
notify-send "[$(qubesdb-read /name)] SSH agent access from $QREXEC_REMOTE_DOMAIN"
exec socat - UNIX-CONNECT:/home/user/.ssh/agent.sock
EOF
sudo chmod +x /usr/local/etc/qubes-rpc/qubes.SshAgent
```

### 4. dom0: the policy

```bash
echo 'qubes.SshAgent  *  dev  vault  ask default_target=vault' \
  | sudo tee /etc/qubes/policy.d/30-user.policy
```

Only `dev` may call this service, only towards `vault`, and **always with a prompt**.

### 5. dev: the local relay

`/rw/config/rc.local` runs as root at every boot:

```bash
sudo tee /rw/config/rc.local >/dev/null <<'EOF'
#!/bin/sh
rm -f /home/user/.SSH_AGENT_vault
sudo -u user /bin/sh -c 'umask 177 && exec socat UNIX-LISTEN:/home/user/.SSH_AGENT_vault,fork "EXEC:qrexec-client-vm vault qubes.SshAgent"' &
EOF
sudo chmod +x /rw/config/rc.local
```

At the end of `~/.bashrc`:

```bash
export SSH_AUTH_SOCK="/home/user/.SSH_AGENT_vault"
```

### 6. Test

1. Restart both qubes (`qvm-shutdown --wait vault dev`).
2. In **vault**, open a terminal and load the key: `ssh-add`.
3. In **dev**: `ssh-add -L` → dom0 prompt → vault notification → the public key is printed.
4. Add that public key to GitHub (*Settings → SSH and GPG keys*).
5. `ssh -T git@github.com` → *"Hi <user>! You've successfully authenticated, but GitHub does not provide shell access."* ✅

---

## Troubleshooting log

The useful part. Each error tells you **which link of the chain** is broken, so read them from left to right: dev socket → dom0 policy → vault service → vault agent → GitHub.

| # | Symptom (in dev) | Broken link | Root cause | Fix |
|---|---|---|---|---|
| 1 | `ssh-add -L`: *Error connecting to agent: No such file or directory* | dev relay socket | `rc.local` contained `UNIX-LISTEN:SSH_SOCK` instead of `$SSH_SOCK`: socat tried to create a socket named `SSH_SOCK` in `/` → *permission denied* (seen in `journalctl -b -u qubes-misc-post`) | Rewrite `rc.local` with an absolute path |
| 2 | Same error, journal now clean | dev environment | Same lost `$` in `~/.bashrc`: `SSH_AUTH_SOCK` pointed to a non-existent path | Hardcode the path, open a new terminal |
| 3 | dom0 popup *"qubes.SshAgent denied"*, **no** confirmation asked | dom0 policy | Typo in the service name inside `rc.local`: the request matched no rule, so dom0 denied by default | Same service name, character for character, in policy, vault service file and `qrexec-client-vm` call |
| 4 | Prompt accepted, then *communication with agent failed*, **no** vault notification | vault service | The RPC script was not executed (name/permissions) | Recreate it, `chmod +x`, check `ls -l /usr/local/etc/qubes-rpc/` |
| 5 | Same error, notification **does** appear | vault agent | Agent socket not where the service expects it (lost `$` again, in vault's `~/.bashrc`) | Hardcode the path, restart the agent, `ssh-add` |
| 6 | Worked, then *communication with agent failed* again later | vault agent | The agent was no longer reachable after vault changed state | New vault terminal, `ssh-add -l`, reload key |
| 7 | `ssh -T git@github.com`: *Permission denied (publickey)* | GitHub | The key loaded in the agent no longer matched the one registered on GitHub | Compare `SHA256` fingerprints (`ssh-keygen -lf`, `ssh-add -l`, GitHub), `ssh-add -D && ssh-add`, re-upload the public key |

### Useful diagnostic commands

```bash
# dev
echo $SSH_AUTH_SOCK
ls -la /home/user/.SSH_AGENT_vault       # must start with "s" (socket)
pgrep -a socat
sudo journalctl -b -u qubes-misc-post --no-pager | tail -20

# dom0
cat /etc/qubes/policy.d/30-user.policy
sudo journalctl -b -u qubes-qrexec-policy-daemon --no-pager | tail -20

# vault
ls -l /usr/local/etc/qubes-rpc/
SSH_AUTH_SOCK=/home/user/.ssh/agent.sock ssh-add -l
ssh-keygen -lf ~/.ssh/id_ed25519.pub
sudo journalctl -b --no-pager | grep -i qrexec | tail -20

# end to end
ssh -vT git@github.com 2>&1 | grep -iE "offering|agent|authenticat"
```

### Lessons learned

- **A missing `$` fails silently**: the shell happily uses the literal string. Three of my seven errors came from that. Prefer absolute paths in boot scripts.
- **Each error message points to one link of the chain.** "No such file" = local socket, "denied" without prompt = policy mismatch, "communication failed" = vault side. The vault notification tells you whether the service ran at all.
- **qrexec denies by default**: a request that matches no rule is refused without a prompt. That is a feature.
- **Never re-run `ssh-keygen` on an existing key** without meaning to: answer `n` to *Overwrite?*.

## Limits and next steps

- While the key is loaded, the agent will sign anything I approve. A compromised `dev` cannot steal the key, but it can trigger a prompt at a convenient moment: **only approve prompts you expect**.
- The agent must be reloaded (`ssh-add`) after each vault restart. Next step: start it automatically with a systemd user service.
- Possible hardening: `ssh-add -c` for a second confirmation inside vault, `ssh-add -t 3600` for a key lifetime, and one key per purpose (GitHub, servers…).

## References

- [Qubes OS documentation: qrexec](https://www.qubes-os.org/doc/qrexec/)
- [Qubes OS documentation: RPC policy](https://www.qubes-os.org/doc/rpc-policy/)
- Qubes community guide on split-SSH

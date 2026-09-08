# ufw-reloaded agent

The privileged half of **UFW Reloaded**. A single Python file, standard library only, that
validates a JSON request, runs `ufw` and handles rules.

It is reached through a **dedicated `sshd` on port 2026** with its own config, host key and systemd
unit. It never reads `/etc/ssh/sshd_config`, so nothing it does can affect your SSH on port 22. The
`authorized_keys` entry forces the agent as the command, so an enrolled key gets no shell, no PTY,
no forwarding and no SFTP — only one JSON object on stdin.

The boundary it holds is **firewall control, not arbitrary root**: no client-supplied paths, no
client-supplied file content, no shell anywhere, and no exec, self-update or key-enrolment verb.

## Installing it

Through the app, which generates the keypair and hands you the command. **The app is not released
yet**, so there is nothing to install from here for now.

`install.sh` is self-contained: it carries the agent as base64 and writes the sshd config, systemd
unit and rollback helper itself, so it fetches nothing at install time. It runs on Debian or Ubuntu
with systemd, `ufw`, `openssh-server` and `python3`; `--help` lists the key management, status and
uninstall flags. It is meant to be read before it is run as root.

To read the embedded agent rather than take it on trust:

```sh
sed -n "/base64 -d <<'UFW_AGENT_B64'/,/^UFW_AGENT_B64/p" install.sh |
  sed '1d;$d' | base64 -d | less
```

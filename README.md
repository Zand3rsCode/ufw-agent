# ufw-reloaded agent

The privileged half of [UFW Reloaded]. A single Python file, standard library only,
that validates a JSON request and runs `ufw` and handles rules..

It is reached through a **dedicated `sshd` on port 2026** with its own config, host key and systemd
unit. The `authorized_keys` entry forces the agent as the command, so an enrolled key gets no shell, no PTY,
no forwarding and no SFTP — only one JSON object on stdin.

The boundary it holds is **firewall control, not arbitrary root**: no client-supplied paths, no
client-supplied file content, no shell anywhere, and no exec, self-update or key-enrolment verb.

## Installing it

Through the app, which generates the keypair and hands you the command. **The app is not released
yet**, so there is nothing to install from here for now.

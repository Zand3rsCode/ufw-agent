# UFW Reloaded

A desktop app for managing **UFW** firewalls on remote Linux servers — with a real on/off switch
for rules, a preview of every change before it runs, and hard guards against locking yourself out.

Windows, macOS and Linux. Built with Electron.

![The rule table](docs/images/rules.png)

---

## Why

`ufw status numbered` gives you a flat list of text, and three things about it are quietly awkward:

- **Order is meaning.** Rules match first-to-last, so position is part of the configuration — but
  nothing in the output tells you that.
- **Every rule appears twice.** One `ufw allow` creates an IPv4 entry and an IPv6 entry with
  unrelated numbers, and `ufw delete <n>` removes only one of them.
- **There is no off switch.** A rule either exists or it doesn't. Turning one off temporarily means
  deleting it and remembering how to put it back — in the right place.

This app collapses the v4/v6 pair into one row, gives that row a toggle that genuinely works, and
never touches the firewall until you've seen exactly what will happen.

## What it does

**Staged changes.** Edits queue into a pending drawer instead of firing on click. Before you
apply, you get the plan, the *effective policy diff* — which traffic actually changes verdict —
and the literal `ufw` commands that will run.

**Lockout prevention.** Every change is simulated against your own connection first. If it would
cut your access, it's blocked rather than warned about.

**An honest view.** Docker publishes container ports straight past UFW, so the app detects Docker
and says so instead of showing you a port as "denied" while the service behind it is reachable
from anywhere.

<p align="center">
  <img src="docs/images/lockout.png" width="49%" alt="A change that would cut SSH access is blocked">
  <img src="docs/images/add-server.png" width="49%" alt="Adding a server">
</p>

## The agent

Rather than logging in as yourself and running `sudo ufw`, the app manages a host through a small
agent you install there.

The agent listens on **its own hardened `sshd` on port 2026** — separate config, separate host key,
separate systemd unit. It never reads `/etc/ssh/sshd_config`, so nothing it does can affect your
real SSH on port 22, and stopping or removing it cannot lock you out.

That choice means **no networking, cryptography or authentication code was written for this
project.** OpenSSH does that part. Because the instance is ours, it's locked far tighter than a
general-purpose sshd: key-only, one forced command, no shell, no PTY, no forwarding, no SFTP. The
agent itself is a validator and an executor, nothing more.

### Installing it

The app generates a keypair, shows you the command, and the command downloads the installer and
**stops** so you can read it before running anything as root:

```sh
curl -fsSL --proto '=https' --tlsv1.2 \
  https://raw.githubusercontent.com/gencaps/UFW-Reloaded/main/agent/dist/install.sh -o install.sh

# read it, then:
sudo sh install.sh --pubkey 'ssh-ed25519 AAAA... you@laptop' --label laptop \
  --from 203.0.113.0/24
```

`--from` scopes the firewall rule for the agent's own port to the network you connect from. It is
optional and the installer warns when you leave it out: authentication is key-only so nobody gets
*in* either way, but an unrestricted port can be tied up by anyone who can reach it, and losing the
management port is the failure this whole project exists to prevent.

The app also shows the installer's SHA-256 so you can verify the copy you downloaded, and both the
app and the installer print the key fingerprint so you can check nothing was swapped in between.

Removing it is one command, and it leaves your firewall rules exactly as it found them:

```sh
sudo sh install.sh --uninstall
```

Requires Debian or Ubuntu with systemd, `ufw`, `openssh-server` and `python3` — all of which a
stock install already has. The agent is a **single Python file with no dependencies**, so you can
read the whole thing before you trust it.

## Safety

Three behaviours of UFW shape the whole design, and are worth knowing whether or not you use this:

**`ufw --dry-run` cannot preview a batch.** Each dry-run is evaluated against the firewall's
*current* state, not the state left by the previous staged command. Any changeset whose steps
interact will be misreported. This app simulates the ruleset client-side instead.

**A successful apply is not proof you're safe.** UFW permits established connections, so removing
the rule that allows your SSH does not drop the session you're on. It looks like nothing broke —
you find out on your next connection. Verification has to open a *fresh* one.

**`ufw insert N` doesn't mean what the numbering suggests.** It indexes the per-family list, not
the combined numbering `status numbered` prints, and a position one past the end wraps to the
front rather than appending.

There's an opt-in rollback timer, too: before applying, the agent can arm a detached job that
restores the previous ruleset unless the app reconnects successfully. A lockout becomes a
two-minute wait instead of a trip to the console.

[`docs/SECURITY.md`](docs/SECURITY.md) has the full model, including a plain account of what
someone who steals your client key can still do.

## Status

Early, and honest about it. Working today:

- Reading, toggling, adding and deleting rules — inbound, outbound and forwarded
- Staged changes with preview, effective policy diff, and apply
- Lockout evaluator, and the rollback timer
- Agent install, key enrolment and rotation, uninstall
- Multiple saved servers, with host-key pinning

Not there yet:

- Editing a rule in place, and drag reordering
- Firewall-level controls in the UI (master switch, default policies, logging level)
- Only tested against Ubuntu 24.04 and Debian-family hosts

## Building

See [`DEVELOPMENT.md`](DEVELOPMENT.md).

```sh
npm install
npm run dev:mock    # runs against a built-in fake ufw — no server needed
npm test
```

The mock is a full emulator, so the entire interface can be developed and demonstrated without
touching a real firewall.

## License

MIT

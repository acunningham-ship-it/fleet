# fleet

Durable terminal sessions for AI coding agents. Run a pack of Claude Code (or any CLI) sessions that survive disconnects, reboots, and the `/tmp` sweep, with an optional AI "manager" session that watches them.

The problem it fixes: you ssh into a box, start a few `claude` sessions, your connection drops, and they all die with it (SIGHUP). Or they're in a `/tmp`-socketed tmux and systemd's 30-day temp sweep orphans the socket. fleet runs every session in one dedicated tmux server on a private socket that lives outside `/tmp` and outside your ssh ptys, so neither a disconnect nor a sweep can touch them. ssh in once, `fleet attach`, and everything's still running days later.

## Install

```bash
git clone https://github.com/acunningham-ship-it/fleet
install -m755 fleet/fleet ~/.local/bin/fleet        # put it on your PATH
mkdir -p ~/.tmux && cp fleet/fleet.conf ~/.tmux/fleet.conf
```

For boot durability (sessions survive logout/reboot), install the systemd user unit:

```bash
mkdir -p ~/.config/systemd/user
cp fleet/fleet.service ~/.config/systemd/user/fleet.service
systemctl --user enable --now fleet
loginctl enable-linger "$USER"   # so it runs without an active login session
```

## Commands

```
fleet new <name> [cmd...]   spawn a window (default cmd: claude)
fleet ls                    list sessions + live state (idle / working / waiting)
fleet status [--json]       machine-readable state of every session (for the AI manager)
fleet peek <name> [lines]   show what a session is currently doing
fleet send <name> <text>    type a line into a session and submit it
fleet attach [name]         attach to the fleet (optionally jump to one window)
fleet kill <name>           close a session
fleet manager               launch the AI manager (see below)
fleet detect                inventory every agent session on the box (in-fleet vs orphan)
fleet adopt-live <pid>      pull a running raw-pty session into the fleet (experimental, see note)
```

## The AI manager

`fleet manager` launches an interactive Claude Code session preloaded with `MANAGER.md`: a coordinator that watches the fleet through `fleet status --json` / `peek`, unblocks `waiting` sessions with `send`, and rolls up status. It's a real interactive session on purpose (it bills to a subscription), not a headless `claude -p` (which bills the API).

## Notes

- **State lives outside `/tmp`.** The tmux socket is `~/.tmux/fleet.sock`, so the systemd temp-file sweep won't orphan it.
- **`adopt-live` is experimental.** It uses `reptyr` to pull an already-running raw-pty session into the fleet. It works, but `reptyr` is flaky on multithreaded Node processes and can occasionally crash the target. Don't adopt a session you can't afford to lose; safer to let it finish and start the next one inside fleet.
- Requires `tmux`. `reptyr` only for `adopt-live`.

## License

MIT

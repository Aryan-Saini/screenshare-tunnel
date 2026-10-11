# screenshare-tunnel

A worker has no screen. Some steps still need a human in front of a browser: a login, a 2FA
prompt, a captcha, an OAuth consent page. `screenshare` puts the worker's browser in a window on
the controller in one command. You click through it, close the window, and the agent carries on
with a browser that is now logged in.

```bash
screenshare                       # bring it all up, open the viewer
screenshare https://site/login    # same, and open that URL on the worker
screenshare shot                  # PNG of the remote screen, no human needed
```

**With [worker-chrome](https://github.com/Aryan-Saini/worker-chrome)** running on the worker,
screenshare shows that Chrome: the same screen and logged-in profile your agents drive, so a
login you finish in the viewer is the login they use. Without it, screenshare starts its own
Xvfb `:99` and Chrome. `SCREENSHARE_CHROME=own` forces that even when worker-chrome is up.

The agent side is the `screenshare-tunnel` skill: it takes a screenshot first, and only asks you
to look when the page actually needs a human.

## How it works

```
worker:      Xvfb :99  <- draws -  Chrome (persistent profile, CDP on 9222)
             (or worker-chrome's :90 and CDP 9230, adopted as is)
               |
             rustdesk --server   (DISPLAY=:99, direct IP on 21118)
               |  firewall: 21118 dropped on every interface except lo and tailscale0
controller:  rustdesk://connection/new/<worker tailnet IP>?password=<per-worker secret>
```

1. **Xvfb** is an X server with no hardware. It gives the worker a framebuffer.
2. **Chrome** draws into it with a persistent `--user-data-dir`. Cookies live in that directory,
   so a login done once through the viewer stays logged in for later headless runs.
3. **RustDesk** streams `:99` as video, which stays smooth where VNC lags. It is configured for
   direct IP only, so no public RustDesk server is involved.
4. **The firewall rule** is the security model. RustDesk listens everywhere; only the tailnet
   may reach it.
5. **The password** is random, generated on the worker at `~/.screenshare/password` on first run,
   and read over ssh by the viewer. It never leaves your machines.

Everything on the worker runs as transient systemd user units (`screenshare-xvfb`,
`screenshare-chrome`, `screenshare-rustdesk`), so it survives the ssh session dropping. The
script adopts an Xvfb or Chrome that is already running instead of starting a second one.

## Install

On the controller (the machine you sit at):

```bash
npm i -g screenshare-tunnel     # puts `screenshare` on PATH
```

Update with `npm i -g screenshare-tunnel@latest`. Nothing gets installed on the worker; the script
drives it over ssh. The worker only needs these:

| where | what |
| --- | --- |
| worker (Linux) | `rustdesk`, `xorg-server-xvfb` / `xvfb`, Chrome or Chromium, ImageMagick (`import`), `nftables` |
| controller | the RustDesk client (macOS: the script installs it with Homebrew) |
| both | Tailscale, and an ssh alias for the worker |

Point it at the worker with `export SCREENSHARE_HOST=<ssh alias>` (default `worker`). If
`tailscale ip -4 <alias>` can't resolve it, also set `SCREENSHARE_IP` to the worker's tailnet IP.

## Agent skill

`skills/screenshare-tunnel/SKILL.md` teaches an agent when and how to use this: screenshot
first, only ask a human when the page needs one. It ships in the npm package. Link it into
your harness:

```bash
ln -s "$(npm root -g)/screenshare-tunnel/skills/screenshare-tunnel" ~/.claude/skills/
```

[agent-fleet](https://github.com/Aryan-Saini/agent-fleet) does this for you when the
`screenshare` package is enabled.

## Verify

```bash
screenshare status
nc -z -w 3 <worker-tailnet-ip> 21118   # must connect
nc -z -w 3 <worker-lan-ip> 21118       # must be refused
```

## Gotchas

- **RustDesk overwrites its own config while running.** The script edits config only while the
  process is stopped. Do the same by hand.
- **The firewall rule disappears on boot** because tailscale rebuilds its table. Running
  `screenshare` re-adds it. Don't move it to a permanent table without checking chain order.
- **Black viewer:** the server is capturing another display. `screenshare` restarts it onto `:99`.
- **Password rejected:** delete `~/.screenshare/password` on the worker and run `screenshare`.
  It generates a new one and restarts the server with it.
- **Keeping an existing password:** a worker that already ran RustDesk can keep its password.
  Write it to `~/.screenshare/password` (mode 600) before the first run.
- Not Linux? The same chain works anywhere you can run a virtual display and a RustDesk server.
  Port the remote half; keep the commands.

## License

[MIT](LICENSE). Built by [Aryan Saini](https://github.com/Aryan-Saini) as part of
[agent-fleet](https://github.com/Aryan-Saini/agent-fleet).

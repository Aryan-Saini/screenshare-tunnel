# screenshare-tunnel

**This repo is public and published to npm as `screenshare-tunnel`.** Everything under `bin/`
and `skills/` is downloaded by anyone who runs `npm i -g screenshare-tunnel`.

Never commit hostnames, tailnet IPs, ssh aliases, passwords or usernames from a real machine.
The script reads all of that at runtime (`SCREENSHARE_HOST`, `SCREENSHARE_IP`,
`~/.screenshare/password` on the worker). Git history counts: check `git diff --cached` before
every commit, not after.

- `bin/screenshare` is the whole tool: plain bash, controller side, with the worker half sent
  over ssh as a heredoc. `bash -n` and `shellcheck -S warning` must pass.
- `skills/screenshare-tunnel/SKILL.md` is the agent skill. Keep its command table in sync with
  the script's usage comment.
- Release: bump `version` in `package.json` in the same PR, then follow the npm-publish steps
  (`npm pack --dry-run`, `npm publish`, tag `v<version>`).

# Working on this repository

Notes for Claude Code (and anyone else) working on the crosspoint-sync helper
script. The README describes what it does; this is the checklist.

## Ground rules

- **Start with `git pull`.** The owner works from more than one computer.
- **One branch, `master`, and pushing it ships it.** Every container's
  `update` downloads `crosspoint-sync.sh` from `master` and runs it, so a
  broken push breaks the next update everywhere. `update` only rejects a file
  that is empty or fails `bash -n`, so check more than that (below) before
  pushing.
- Every change gets a line in `CHANGELOG.md` (*Unreleased*; there are no
  tagged releases) and, if it changes what the script sets up or how it's
  used, the README. Rows in the README's variable table match the defaults
  in the script.
- **No secrets** in the repo or in what the script prints.

## The script

- One file, two halves: the host part (creates the LXC, runs as root on the
  Proxmox host) and the container installer between `<<'INNER_EOF'` and
  `INNER_EOF`. The installer is quoted, so `$` inside it stays literal; the
  `update` bootstrap inside it (`<<UPDATE_EOF`) is not, so its runtime
  variables are escaped (`\$`).
- The installer must stay **idempotent**: a run with nothing new upstream
  rebuilds nothing and restarts nothing. Rebuild only when the upstream
  commit changed or a build output is missing; restart only when the build,
  the unit or the env file changed.
- **The env file is the user's.** It's written once on install. When a newer
  upstream needs a setting, add it to older files only if it's missing
  (as with `LISTEN_ADDRESS`), never overwrite a value.
- Upstream (`crosspoint-reader/crosspoint-sync`, branch `main`) has no
  releases and changes without notice. Its `Dockerfile` and README's
  environment table are the reference for how it's built and run: when
  upstream changes, compare them with what the installer does. Settings
  the Docker image sets for itself (`LISTEN_ADDRESS=::`, `WEB_APP_DIR`) have
  to be handled here too.
- Builds have to fit the default 512 MB container (server plus web app,
  about a minute, 2026-11).

## Checking a change

1. `bash -n crosspoint-sync.sh`.
2. Run the changed installer on the live container **before** pushing, the
   way `update` runs it:
   ```bash
   scp crosspoint-sync.sh pve_thinklab:/tmp/cps-test.sh
   ssh pve_thinklab 'pct push 112 /tmp/cps-test.sh /root/cps-test.sh && rm /tmp/cps-test.sh &&
     pct exec 112 -- bash -c "bash /root/cps-test.sh --container-install; rm /root/cps-test.sh"'
   ```
   It must end with `crosspoint-sync is up`, and the server must answer
   from outside the container: `curl http://192.168.178.195:8080/healthz`
   (the installer's own check uses `127.0.0.1`, which can pass while the
   server listens on localhost only).
3. Run it a second time: nothing rebuilt, no restart.
4. A change to the host part (container creation) needs a throwaway
   container (`CTID=<free id>`), destroyed afterwards.

## The live container

LXC 112 `crosspoint-sync` on pve_thinklab, `http://192.168.178.195:8080`,
real data (both e-readers' reading progress). Back up the env file into HA
`/config/project-backups/crosspoint-sync/` before changing it by hand. Wiki
page `lxc/112-crosspoint-sync`; changes to what the container runs go there
too. thinklab has no internet 23:30–05:30, so `update` falls back to the
cached installer and can't fetch upstream at night.

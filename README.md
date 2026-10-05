# Alarm Clock System

A Linux alarm clock built for the *Operating Systems and Systems Programming* scenario:
**schedule an alarm, keep doing other work while it waits, then get notified (or cancel it).**

Everything, including the web server, is written in **C** and runs in user space (no kernel changes).
A browser dashboard sits on top and shows what the operating system is actually doing:
each alarm is its own process, and you can watch the kernel move it between the wait queue,
the ready queue and a CPU core.

## Features

- **One process per alarm**, created with `fork()`.
- **No busy-waiting:** the process blocks in `sigsuspend()` and uses 0% CPU until its timer fires.
- **CPU scheduler view:** alarm processes move through New, Wait queue, Ready queue, CPU core and Exited, based on events the worker records itself (real core, real timestamps).
- **Live process table** read from `/proc`: state, last CPU core, CPU time, context switches.
- **Notifications:**
  - Web UI: red "Alarm triggered" banner and a toast.
  - Linux desktop: `notify-send` popup with the process ID, parent PID, alarm ID, CPU core and latency.
  - Terminal: a line with the PID.
- **Loud alarm that rings until you stop it** (repeating beep pattern, with a Test sound button).
- **Cancel** an alarm before it fires, or **Stop** it while it rings.

## How it maps to the OS concepts

| Requirement | Implementation |
|---|---|
| Accept an alarm time or delay | Web form (delay or clock time), `POST /api/alarms` |
| Process to manage each alarm | `fork()`: one child per alarm (`server.c`) |
| Timer and signal for the alarm event | `alarm(2)` and `SIGALRM` |
| Suspend while waiting | `sigprocmask()` + `sigsuspend(2)` |
| Handle the alarm signal | `sigaction()` handlers in `alarm_worker.c` |
| Cancel an active alarm | `kill(pid, SIGUSR1)` |
| Stop a ringing alarm | `kill(pid, SIGUSR2)` |
| Multiple alarms | Independent processes; a `SIGCHLD` handler reaps them (no zombies) |
| IPC between server and workers | Shared SQLite database (`db/alarms.db`), WAL mode |
| Scheduler/CPU insight | `sched_getcpu()`, `/proc/self/schedstat`, `/proc/<pid>/stat`, `/proc/stat` |

## Process lifecycle

```
fork()        NEW
  |
first run     RUNNING    install handlers, arm alarm(N)
  |
sigsuspend()  WAITING    blocked, off the run queue, 0% CPU
  |  SIGALRM (timer expires)
  v
              READY      kernel put it on the run queue
  |  scheduler dispatches
  v
              RUNNING    on CPU n: handler runs, status -> ringing, notifications sent
  |
sigsuspend()  WAITING    ringing, blocked until SIGUSR2 (Stop)
  |  SIGUSR2
  v
              RUNNING -> EXITED   exit(0), parent reaps via SIGCHLD
```

Alarm status in the database: `active` -> `ringing` -> `triggered` (done), or `active` -> `cancelled`.

## Requirements

- Linux (Ubuntu / Debian), or **WSL** on Windows
- `gcc`, `make`, SQLite development headers
- `libnotify-bin` for desktop notifications (optional)
- A modern browser

## Build and run

```bash
sudo apt update
sudo apt install -y build-essential libsqlite3-dev libnotify-bin

cd alarm-clock-system
make
./alarm_server            # port 8080, db/alarms.db, public/
```

Open **http://localhost:8080**. Keep the terminal open: the server runs there and prints the PIDs.

Optional arguments: `./alarm_server <port> <db_path> <public_dir>`
Rebuild from scratch: `make clean && make`

### Running on WSL (Windows)

Windows drives appear under `/mnt/c/`. Copy the project to the Linux side before running it,
because SQLite can misbehave on `/mnt/c`:

```bash
cp -r "/mnt/c/Users/<you>/Downloads/Alarm-Clock-System" ~/alarm-clock-system
cd ~/alarm-clock-system
make && ./alarm_server
```

Then open http://localhost:8080 in your Windows browser. WSL has no notification daemon by default,
so the Linux popup usually will not appear there; the terminal line, UI banner, sound and CPU view still work.

## Using it

1. Enter a label, choose **Delay** or **Clock time**, and click **Set alarm**.
2. Watch the **CPU scheduler** panel. Your alarm appears as a chip and moves as the real process changes state.
3. When it fires, the banner and toast show **Alarm triggered**, the alarm sound repeats, and a Linux notification appears.
4. Click **Stop alarm** to silence it. Use **Cancel** on an alarm that has not fired yet.
5. **Clear finished** removes done and cancelled alarms.

Click **Set alarm** or **Test sound** once so the browser allows sound.

### Reading the scheduler panel

- The chips and the **event log** come from events each worker writes about itself, so the core number,
  timestamps, run-queue wait and timer latency are real measurements.
- Real transitions take microseconds, so **Slow motion** (on by default) only stretches the animation.
  The log always shows true timings. Untick it for near real-time playback.
- Load, process counts, per-core usage and the process table are live snapshots from `/proc`,
  refreshed about every 0.6 s, so they can already show a newer state than the animation.
- Per-core percentages are the busy share since the previous refresh. A core can show a small
  percentage and "idle" at the same time: the percentage covers the last fraction of a second,
  "idle" is a single instant.
- `alarm_server` nearly always appears as running, because the server must run to produce each snapshot.
- CPUs are **logical processors** numbered from 0, so CPU 0 to CPU 11 means 12 logical processors.
  Check with `nproc`. On WSL you only see the processors given to the VM, and only Linux processes are counted.

## Project layout

```
alarm-clock-system/
├── Makefile
├── include/
│   ├── alarm_worker.h
│   ├── db.h
│   ├── http.h
│   └── sysinfo.h
├── src/
│   ├── server.c         # HTTP server, routing, fork()
│   ├── alarm_worker.c   # per-alarm process: alarm(), SIGALRM, sigsuspend(), notifications
│   ├── sysinfo.c        # /proc reader: core usage, process states, context switches
│   ├── db.c             # SQLite layer: alarms, events, status transitions
│   └── http.c           # minimal HTTP helpers
├── public/              # web UI
│   ├── index.html
│   ├── style.css
│   └── app.js
└── db/                  # SQLite file is created here at runtime
```

## API

| Method | Path | Purpose |
|---|---|---|
| GET | `/api/alarms` | List alarms |
| POST | `/api/alarms` | Create `{"label": "...", "delay_seconds": N}` |
| POST | `/api/alarms/<id>/cancel` | Cancel an active alarm (`SIGUSR1`) |
| POST | `/api/alarms/<id>/dismiss` | Stop a ringing alarm (`SIGUSR2`) |
| POST | `/api/alarms/clear` | Delete finished alarms |
| GET | `/api/events?after=<id>` | Scheduler events newer than `id` |
| GET | `/api/system` | Live CPU and process snapshot from `/proc` |

## Useful commands during a demo

```bash
ps --forest -o pid,ppid,stat,psr,cmd      # one alarm_server child per alarm (psr = core)
pstree -p $(pgrep -o alarm_server)        # process tree
cat /proc/loadavg                         # load averages
nproc                                     # logical processors
grep ctxt /proc/<worker_pid>/status       # voluntary / nonvoluntary context switches
```

## Troubleshooting

| Problem | Fix |
|---|---|
| `bind: Address already in use` | Another server is on that port. Stop it, or run `./alarm_server 8081` |
| `sqlite3.h: No such file` | `sudo apt install libsqlite3-dev` |
| No Linux popup | Install `libnotify-bin`; run from a terminal inside a desktop session. Not available on plain WSL |
| No sound | Click **Set alarm** or **Test sound** once; check the browser tab is not muted |
| Alarm stuck on `ringing` | Click **Stop** in the list; the worker keeps running until dismissed |

## Notes and simplifications

- The HTTP server handles one request at a time, which is enough for a demo and keeps the code focused on OS concepts.
- The JSON parser in `http.c` is minimal and only reads the fixed request shape this project's UI sends.
- SQLite runs in WAL mode so the server and the short-lived workers can use the file concurrently.
- The alarm sound plays in the browser, so the page must be open to hear it.
- On startup, alarms whose worker process no longer exists are marked cancelled.

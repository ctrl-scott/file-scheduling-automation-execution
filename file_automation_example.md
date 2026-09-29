
I would structure it so the data file describes **what to execute**, while the Python program handles the scheduling and execution. That keeps the scheduler reusable.

### 1. `automator.py`

```python
#!/usr/bin/env python3

import os
import sys
import time
import shlex
import subprocess
from datetime import datetime
from pathlib import Path


LOG_FILE = "automator.log"


def log(message):
    timestamp = datetime.now().strftime("%Y-%m-%d %H:%M:%S")
    line = f"[{timestamp}] {message}"

    print(line)

    with open(LOG_FILE, "a", encoding="utf-8") as file:
        file.write(line + "\n")


def read_task_file(filename):
    """
    Reads a simple KEY=VALUE .txt or .dat task file.
    """

    task = {}

    with open(filename, "r", encoding="utf-8") as file:

        for line in file:

            line = line.strip()

            # Ignore comments and blank lines
            if not line or line.startswith("#"):
                continue

            if "=" not in line:
                continue

            key, value = line.split("=", 1)

            task[key.strip().upper()] = value.strip()

    return task


def wait_until(run_time_string):

    run_time = datetime.strptime(
        run_time_string,
        "%Y-%m-%d %H:%M:%S"
    )

    log(f"Scheduled execution time: {run_time}")

    while True:

        now = datetime.now()

        if now >= run_time:
            break

        remaining = (run_time - now).total_seconds()

        # Do not sleep longer than 30 seconds,
        # so shutdown signals remain responsive.
        time.sleep(min(30, max(1, remaining)))

    log("Scheduled time reached.")


def execute_local(command):

    log(f"Executing local command: {command}")

    try:

        args = shlex.split(command)

        result = subprocess.run(
            args,
            text=True,
            capture_output=True,
            check=False
        )

        if result.stdout:
            log("STDOUT:")
            print(result.stdout)

        if result.stderr:
            log("STDERR:")
            print(result.stderr)

        log(f"Exit code: {result.returncode}")

        return result.returncode

    except Exception as error:

        log(f"Local execution error: {error}")

        return 1


def execute_remote(host, user, command, port="22"):

    destination = f"{user}@{host}"

    ssh_command = [
        "ssh",
        "-p",
        str(port),
        destination,
        command
    ]

    log(
        f"Connecting to remote server: "
        f"{destination}:{port}"
    )

    try:

        result = subprocess.run(
            ssh_command,
            text=True,
            capture_output=True,
            check=False
        )

        if result.stdout:
            log("REMOTE STDOUT:")
            print(result.stdout)

        if result.stderr:
            log("REMOTE STDERR:")
            print(result.stderr)

        log(f"Remote exit code: {result.returncode}")

        return result.returncode

    except Exception as error:

        log(f"Remote execution error: {error}")

        return 1


def execute_file(filename):

    path = Path(filename)

    if not path.exists():

        log(f"Execution file does not exist: {filename}")

        return 1

    log(f"Executing file: {filename}")

    try:

        result = subprocess.run(
            [str(path.resolve())],
            text=True,
            capture_output=True,
            check=False
        )

        if result.stdout:
            print(result.stdout)

        if result.stderr:
            print(result.stderr)

        log(f"Program exit code: {result.returncode}")

        return result.returncode

    except PermissionError:

        log(
            f"Permission denied. "
            f"Try: chmod +x {filename}"
        )

        return 1

    except Exception as error:

        log(f"File execution error: {error}")

        return 1


def main():

    if len(sys.argv) != 2:

        print(
            "Usage:\n"
            "    python3 automator.py task.dat\n\n"
            "or:\n"
            "    python3 automator.py task.txt"
        )

        sys.exit(1)

    task_filename = sys.argv[1]

    if not os.path.exists(task_filename):

        print(f"Task file not found: {task_filename}")

        sys.exit(1)

    task = read_task_file(task_filename)

    mode = task.get("MODE", "LOCAL").upper()

    run_at = task.get("RUN_AT")

    command = task.get("COMMAND")

    execute_filename = task.get("FILE")

    # -------------------------------------
    # Scheduled execution
    # -------------------------------------

    if run_at:
        wait_until(run_at)

    # -------------------------------------
    # Local command
    # -------------------------------------

    if mode == "LOCAL":

        if command:

            exit_code = execute_local(command)

        elif execute_filename:

            exit_code = execute_file(execute_filename)

        else:

            log("No COMMAND or FILE was provided.")

            exit_code = 1

    # -------------------------------------
    # Remote SSH command
    # -------------------------------------

    elif mode == "REMOTE":

        host = task.get("HOST")
        user = task.get("USER")
        port = task.get("PORT", "22")

        if not host or not user or not command:

            log(
                "REMOTE mode requires "
                "HOST, USER, and COMMAND."
            )

            exit_code = 1

        else:

            exit_code = execute_remote(
                host,
                user,
                command,
                port
            )

    else:

        log(f"Unknown MODE: {mode}")

        exit_code = 1

    log("Automation task finished.")

    sys.exit(exit_code)


if __name__ == "__main__":
    main()
```

### 2. Local `.dat` example

For example, `backup.dat`:

```text
# Local scheduled task

MODE=LOCAL

RUN_AT=2026-09-29 14:30:00

COMMAND=/usr/bin/python3 backup.py
```

Run it with:

```bash
python3 automator.py backup.dat
```

The program waits until:

```text
September 29, 2026
2:30 PM
```

and then executes:

```bash
/usr/bin/python3 backup.py
```

You could use `.txt` identically:

```bash
python3 automator.py backup.txt
```

The extension is not important because the scheduler treats the contents as plain text.

---

### 3. Execute a program directly

For example:

`execute.dat`

```text
MODE=LOCAL

RUN_AT=2026-09-29 15:00:00

FILE=/Users/yourname/scripts/daily-backup.sh
```

Make the script executable:

```bash
chmod +x /Users/yourname/scripts/daily-backup.sh
```

The automation path becomes:

```text
task.dat
   |
   v
automator.py
   |
   +---- wait for date/time
   |
   v
daily-backup.sh
```

---

### 4. Remote server execution

The more interesting version is an SSH task.

`remote.dat`

```text
MODE=REMOTE

RUN_AT=2026-09-29 16:00:00

HOST=192.168.1.100
PORT=22
USER=scott

COMMAND=/usr/local/bin/backup.sh
```

Then:

```bash
python3 automator.py remote.dat
```

At 4:00 PM the program effectively performs:

```bash
ssh -p 22 scott@192.168.1.100 /usr/local/bin/backup.sh
```

The architecture is:

```text
                     task.dat
                        |
                        v
                  +-----------+
                  | automator |
                  |   .py     |
                  +-----------+
                        |
                 check RUN_AT
                        |
                        v
                scheduled time?
                   /        \
                 NO          YES
                 |            |
               sleep          |
                              v
                         MODE?
                      /          \
                   LOCAL        REMOTE
                     |             |
                     v             v
                 subprocess       SSH
                     |             |
                     v             v
              local program   remote server
```

### 5. SSH authentication

For automation, I recommend an SSH key rather than putting a password inside the `.dat` file.

On your Mac:

```bash
ssh-keygen -t ed25519
```

Then install the public key on your server:

```bash
ssh-copy-id scott@192.168.1.100
```

Test:

```bash
ssh scott@192.168.1.100
```

Once key authentication works, the Python scheduler doesn't need to contain your SSH password.

### 6. A useful next step: task directory

You can extend this design into something like:

```text
automator/
│
├── automator.py
│
├── automator.log
│
├── jobs/
│   ├── backup.dat
│   ├── webserver.txt
│   ├── remote-backup.dat
│   └── cleanup.dat
│
└── scripts/
    ├── backup.sh
    ├── cleanup.sh
    └── report.py
```

Then we could make `automator.py` scan:

```text
jobs/*.dat
jobs/*.txt
```

and maintain **multiple scheduled jobs simultaneously** instead of starting one Python process per job.

For example, a later format could support:

```text
NAME=Nightly Server Backup
MODE=REMOTE
HOST=server.example.com
USER=backupuser
PORT=22

RUN_AT=2026-09-29 23:30:00

COMMAND=/usr/local/bin/nightly-backup.sh
```

with the log showing:

```text
[2026-09-29 09:25:17] Loaded: Nightly Server Backup
[2026-09-29 09:25:17] Scheduled execution time: 2026-09-29 23:30:00

...

[2026-09-29 23:30:00] Scheduled time reached.
[2026-09-29 23:30:00] Connecting to remote server
[2026-09-29 23:30:01] Remote command started
[2026-09-29 23:30:07] Remote exit code: 0
```

One security improvement I'd add before using this for unattended remote execution is an **allowlist of permitted commands/scripts**, so a modified `.dat` file cannot cause arbitrary commands to run.
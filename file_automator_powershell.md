Basic Automator design rewritten for PowerShell. It reads either a `.dat` or `.txt` configuration file, waits for a specified date/time, and then either runs a local command/file or connects to a remote machine with SSH.

### `automator.ps1`

```powershell
param (
    [Parameter(Mandatory = $true)]
    [string]$TaskFile
)

$LogFile = "automator.log"

function Write-Log {
    param (
        [string]$Message
    )

    $Timestamp = Get-Date -Format "yyyy-MM-dd HH:mm:ss"
    $Line = "[$Timestamp] $Message"

    Write-Host $Line
    Add-Content -Path $LogFile -Value $Line
}


function Read-TaskFile {
    param (
        [string]$FileName
    )

    $Task = @{}

    Get-Content $FileName | ForEach-Object {

        $Line = $_.Trim()

        # Ignore blank lines
        if ([string]::IsNullOrWhiteSpace($Line)) {
            return
        }

        # Ignore comments
        if ($Line.StartsWith("#")) {
            return
        }

        # Process KEY=VALUE
        if ($Line.Contains("=")) {

            $Parts = $Line.Split("=", 2)

            $Key = $Parts[0].Trim().ToUpper()
            $Value = $Parts[1].Trim()

            $Task[$Key] = $Value
        }
    }

    return $Task
}


function Wait-Until {
    param (
        [string]$RunTimeString
    )

    $RunTime = [datetime]::ParseExact(
        $RunTimeString,
        "yyyy-MM-dd HH:mm:ss",
        $null
    )

    Write-Log "Scheduled execution time: $RunTime"

    while ((Get-Date) -lt $RunTime) {

        $Remaining = ($RunTime - (Get-Date)).TotalSeconds

        if ($Remaining -gt 30) {
            Start-Sleep -Seconds 30
        }
        elseif ($Remaining -gt 1) {
            Start-Sleep -Seconds ([int]$Remaining)
        }
        else {
            Start-Sleep -Milliseconds 500
        }
    }

    Write-Log "Scheduled time reached."
}


function Invoke-LocalCommand {
    param (
        [string]$Command
    )

    Write-Log "Executing local command: $Command"

    try {

        & powershell.exe `
            -NoProfile `
            -Command $Command

        $ExitCode = $LASTEXITCODE

        if ($null -eq $ExitCode) {
            $ExitCode = 0
        }

        Write-Log "Exit code: $ExitCode"

        return $ExitCode
    }
    catch {

        Write-Log "Local execution error: $_"

        return 1
    }
}


function Invoke-LocalFile {
    param (
        [string]$FileName
    )

    if (-not (Test-Path $FileName)) {

        Write-Log "Execution file does not exist: $FileName"

        return 1
    }

    Write-Log "Executing file: $FileName"

    try {

        $Extension = [System.IO.Path]::GetExtension($FileName)

        switch ($Extension.ToLower()) {

            ".ps1" {

                & powershell.exe `
                    -NoProfile `
                    -File $FileName
            }

            ".bat" {

                & cmd.exe /c $FileName
            }

            ".cmd" {

                & cmd.exe /c $FileName
            }

            ".exe" {

                & $FileName
            }

            default {

                Write-Log "Unsupported executable file type: $Extension"

                return 1
            }
        }

        $ExitCode = $LASTEXITCODE

        if ($null -eq $ExitCode) {
            $ExitCode = 0
        }

        Write-Log "Program exit code: $ExitCode"

        return $ExitCode
    }
    catch {

        Write-Log "File execution error: $_"

        return 1
    }
}


function Invoke-RemoteCommand {
    param (
        [string]$HostName,
        [string]$User,
        [string]$Command,
        [string]$Port = "22"
    )

    $Destination = "$User@$HostName"

    Write-Log "Connecting to remote server: ${Destination}:$Port"

    try {

        & ssh `
            -p $Port `
            $Destination `
            $Command

        $ExitCode = $LASTEXITCODE

        Write-Log "Remote exit code: $ExitCode"

        return $ExitCode
    }
    catch {

        Write-Log "Remote execution error: $_"

        return 1
    }
}


# -------------------------------------------------
# Main program
# -------------------------------------------------

if (-not (Test-Path $TaskFile)) {

    Write-Host "Task file not found: $TaskFile"

    exit 1
}


$Task = Read-TaskFile -FileName $TaskFile


$Mode = "LOCAL"

if ($Task.ContainsKey("MODE")) {
    $Mode = $Task["MODE"].ToUpper()
}


# -------------------------------------------------
# Scheduled execution
# -------------------------------------------------

if ($Task.ContainsKey("RUN_AT")) {

    Wait-Until -RunTimeString $Task["RUN_AT"]
}


# -------------------------------------------------
# Local execution
# -------------------------------------------------

if ($Mode -eq "LOCAL") {

    if ($Task.ContainsKey("COMMAND")) {

        $ExitCode = Invoke-LocalCommand `
            -Command $Task["COMMAND"]
    }
    elseif ($Task.ContainsKey("FILE")) {

        $ExitCode = Invoke-LocalFile `
            -FileName $Task["FILE"]
    }
    else {

        Write-Log "No COMMAND or FILE was provided."

        $ExitCode = 1
    }
}


# -------------------------------------------------
# Remote execution
# -------------------------------------------------

elseif ($Mode -eq "REMOTE") {

    if (
        -not $Task.ContainsKey("HOST") -or
        -not $Task.ContainsKey("USER") -or
        -not $Task.ContainsKey("COMMAND")
    ) {

        Write-Log "REMOTE mode requires HOST, USER, and COMMAND."

        $ExitCode = 1
    }
    else {

        $Port = "22"

        if ($Task.ContainsKey("PORT")) {
            $Port = $Task["PORT"]
        }

        $ExitCode = Invoke-RemoteCommand `
            -HostName $Task["HOST"] `
            -User $Task["USER"] `
            -Command $Task["COMMAND"] `
            -Port $Port
    }
}

else {

    Write-Log "Unknown MODE: $Mode"

    $ExitCode = 1
}


Write-Log "Automation task finished."

exit $ExitCode
```

For a local scheduled command, you could use `backup.dat`:

```text
# Local task

MODE=LOCAL

RUN_AT=2026-09-29 14:30:00

COMMAND=Write-Host "Scheduled task executed."
```

Run it with:

```powershell
.\automator.ps1 -TaskFile .\backup.dat
```

A more realistic example could start a program:

```text
MODE=LOCAL

RUN_AT=2026-09-29 14:30:00

COMMAND=python C:\Scripts\backup.py
```

Or run a PowerShell script directly:

```text
MODE=LOCAL

RUN_AT=2026-09-29 15:00:00

FILE=C:\Scripts\daily-backup.ps1
```

For remote SSH execution:

```text
# Remote Linux server

MODE=REMOTE

RUN_AT=2026-09-29 16:00:00

HOST=192.168.1.100
PORT=22
USER=scott

COMMAND=/usr/local/bin/backup.sh
```

Then:

```powershell
.\automator.ps1 -TaskFile .\remote.dat
```

At the scheduled time, the important part effectively becomes:

```powershell
ssh -p 22 scott@192.168.1.100 "/usr/local/bin/backup.sh"
```

The overall flow is:

```text
task.dat / task.txt
        |
        v
 automator.ps1
        |
        v
    RUN_AT?
     /   \
   no     yes
   |       |
   |     wait
   |       |
   +-------+
        |
        v
      MODE
     /    \
 LOCAL    REMOTE
   |        |
   v        v
command    SSH
or file     |
   |        |
   v        v
Windows   remote
program   server
```

On Windows 10/11 or modern Windows Server installations with OpenSSH Client installed, you can check SSH with:

```powershell
ssh -V
```

You can also check whether the Windows OpenSSH capability is installed with:

```powershell
Get-WindowsCapability -Online |
    Where-Object Name -Like 'OpenSSH.Client*'
```

For unattended automation, SSH public-key authentication is preferable to putting a password in the `.dat` file.

One important improvement for a production version would be to avoid allowing arbitrary commands from any writable `.dat` file. For example, we could make the configuration say:

```text
TASK=SERVER_BACKUP
```

and have PowerShell map `SERVER_BACKUP` to a predefined approved command. That would make the scheduler much safer for unattended operation.
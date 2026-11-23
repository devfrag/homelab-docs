# proxmox2

Documentation for the proxmox2 host and secure SSH access.

## Host

- Hostname: `proxmox2`
- LAN IP: `192.168.4.220`
- SSH user: `root`
- Proxmox web interface: `https://192.168.4.220:8006`

## SSH Quick-Connect

The US1 web dashboard provides a one-click button for opening an SSH session to Proxmox.

### Windows SSH Configuration

The Windows OpenSSH client uses the following host alias in:

`C:\Users\devfr\.ssh\config`

Host configuration:

    Host proxmox2
        HostName 192.168.4.220
        User root
        IdentityFile C:\Users\devfr\.ssh\id_ed25519
        IdentitiesOnly yes

The private SSH key must never be committed to this repository.

Manual connection test:

    ssh proxmox2

A successful connection opens a shell as:

    root@proxmox2

## HomeLab URI Handler

Windows has a custom URI protocol named:

    homelab://

The handler is registered under:

    HKCU\Software\Classes\homelab

The open command passes the requested URI to:

    C:\Tools\HomeLab\Open-HomeLab.cmd

The launcher supports:

    homelab://connect
    homelab://proxmox

Behavior:

- `homelab://connect` opens SSH to US1.
- `homelab://proxmox` opens SSH to proxmox2.

## Windows Launcher

`C:\Tools\HomeLab\Open-HomeLab.cmd` contains:

    @echo off

    echo %~1 | findstr /I "proxmox" >nul
    if not errorlevel 1 (
        start "" wt.exe -w new cmd.exe /k ssh proxmox2
        exit /b
    )

    start "" wt.exe -w new cmd.exe /k ssh homelab

No passwords or private SSH keys are stored in the launcher.

## US1 Dashboard

The US1 Docker web dashboard contains two SSH buttons:

- Open SSH Terminal
- Open Proxmox SSH

The Proxmox button uses:

    homelab://proxmox

The browser invokes the Windows HomeLab URI handler, which launches Windows Terminal and runs:

    ssh proxmox2

## Verified Working Path

The complete Proxmox SSH launch path was tested successfully:

    US1 dashboard
        -> homelab://proxmox
        -> Windows HomeLab URI handler
        -> Open-HomeLab.cmd
        -> Windows Terminal
        -> ssh proxmox2
        -> root@proxmox2

The original US1 SSH button was also retested successfully after adding the Proxmox button.

## Security

Do not store any of the following in Git:

- SSH private keys
- passwords
- API tokens
- WireGuard private keys
- unredacted configuration files containing credentials

SSH access uses public-key authentication.

# Oracle Free Tier Instance Creation

Automate the creation of Oracle Cloud Infrastructure (OCI) Free Tier ARM and AMD instances with Python. Retries automatically until capacity becomes available, with optional email, Discord, and Telegram notifications.

<div align="center">

<img src="ai-image.jpg" alt="Project Cover" width="600"/>

</div>

<div align="center">

[![Python](https://img.shields.io/badge/Python-3.8+-blue?logo=python&logoColor=white)](https://www.python.org/)
[![Shell](https://img.shields.io/badge/Shell-Bash-green?logo=gnu-bash&logoColor=white)](https://www.gnu.org/software/bash/)
[![OCI](https://img.shields.io/badge/Oracle%20Cloud-Infrastructure-orange?logo=oracle&logoColor=white)](https://www.oracle.com/cloud/)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)
[![PRs Welcome](https://img.shields.io/badge/PRs-welcome-brightgreen.svg)](CONTRIBUTING.md)

</div>

## Table of Contents

- [Features](#features)
- [How It Works](#how-it-works)
- [Prerequisites](#prerequisites)
- [Quick Start](#quick-start)
- [Configuration](#configuration)
- [Usage](#usage)
- [Project Structure](#project-structure)
- [Notifications](#notifications)
- [Instance Creation Flow](#instance-creation-flow)
- [Roadmap](#roadmap)
- [FAQ](#faq)
- [Contributing](#contributing)
- [License](#license)
- [References](#references)

## Features

- [x] Automated provisioning of Oracle Free Tier ARM (2 OCPU, 12 GB RAM) and AMD (1 OCPU, 1 GB RAM) instances
- [x] Retries on capacity errors ("Out of host capacity") until resources are available
- [x] Automatic SSH key pair generation for ARM instances
- [x] Configurable retry interval via `oci.env`
- [x] Gmail notifications on success or failure
- [x] Discord webhook notifications
- [x] Telegram bot notifications
- [x] Mermaid flowchart of the instance creation process
- [x] Setup scripts (`setup_env.sh`, `setup_init.sh`) for one-command installation and run

## How It Works

The script polls the OCI `LaunchInstance` API on a configurable interval. When capacity is unavailable, it catches the retryable errors ("Out of host capacity", `429 TooManyRequests`, `5xx`, network blips) and retries — rotating across availability domains if you list more than one. On a hard failure (bad OCID, auth failure, quota exceeded) it stops so you can fix the config rather than looping forever.

This is the same pattern used by other OCI instance hunters and matches the official [OCI Python SDK launch example](https://docs.oracle.com/en-us/iaas/tools/python-sdk-examples/latest/core/launch_instance.py.html).

> **Note (June 2026):** Oracle reduced the Always Free Ampere A1 allowance from 4 OCPU / 24 GB RAM to **2 OCPU / 12 GB RAM** (1,500 OCPU-hours + 9,000 GB-hours per month) on 15 June 2026. Pay-as-you-go tenancies may differ — check your own billing (Billing → Cost Analysis) before assuming. Set `OCPUS` / `MEMORY_GB` in your env to whatever your account actually allows.

## Prerequisites

Before you begin, make sure you have:

- An [Oracle Cloud](https://www.oracle.com/cloud/) account with Always Free eligibility
- An OCI API key pair (private key `.pem` and config file) — see [API Key-Based Authentication](https://docs.oracle.com/en-us/iaas/Content/API/Concepts/sdk%5Fauthentication%5Fmethods.htm)
- An Ubuntu environment (VM.Standard.E2.1.Micro or any machine with `bash`, `python3`, and `pip3`)
- A Gmail address and App Password (optional, for email notifications)
- A Discord webhook URL or Telegram bot token (optional, for alternative notifications)

### Required Keys and OCIDs

Per [Oracle's documentation](https://docs.oracle.com/en-us/iaas/Content/API/Concepts/apisigningkey.htm), you need:

1. An IAM user in at least one group with the required permissions
2. An RSA key pair in PEM format (minimum 2048 bits)
3. The fingerprint of the public key
4. Your tenancy's OCID and user's OCID

Both the private key and public key must be in PEM format. **Do not commit your private key to a public repository** — if you do, OCI will notify you to take corrective action.

## Quick Start

```bash
# 1. Clone the repository
git clone <your-repository-url>.git
cd oracle-freetier-instance-creation

# 2. Make setup scripts executable
chmod +x setup_env.sh setup_init.sh

# 3. Run the interactive environment setup
./setup_env.sh

# 4. Install dependencies and start the instance creation loop
./setup_init.sh
```

> The first run installs required Python packages (`oci`, etc.) and starts `main.py` in the background. The script retries instance creation every 60 seconds (configurable) until it succeeds.

## Configuration

Copy `sample_oci_config` to `oci_config` and fill in your OCI API details:

```
[DEFAULT]
user=ocid1.user.oc1..xxxx
fingerprint=xx:xx:xx:xx:xx:xx:xx:xx:xx:xx:xx:xx:xx:xx:xx:xx
tenancy=ocid1.tenancy.oc1..xxxx
region=us-ashburn-1
key_file=/home/ubuntu/oracle-freetier-instance-creation/oci_api_private_key.pem
```

Edit `oci.env` with your settings:

| Variable | Required | Description |
|----------|----------|-------------|
| `OCI_CONFIG` | Yes | Absolute path to your OCI config file |
| `OCT_FREE_AD` | Yes | Always-Free Tier availability domain(s), comma-separated |
| `DISPLAY_NAME` | No | Display name for the instance |
| `REQUEST_WAIT_TIME_SECS` | No | Seconds to wait between retries (default: 60) |
| `OCI_SUBNET_ID` | No | Subnet OCID (required when running locally) |
| `OCI_IMAGE_ID` | No | Specific image OCID; auto-discovered if omitted |
| `OCI_COMPUTE_SHAPE` | No | `VM.Standard.A1.Flex` or `VM.Standard.E2.1.Micro` |
| `SECOND_MICRO_INSTANCE` | No | Set to `True` only for a second Always-Free Micro |
| `OPERATING_SYSTEM` / `OS_VERSION` | No | Used to auto-discover the latest image |
| `ASSIGN_PUBLIC_IP` | No | Assign an ephemeral public IP |
| `BOOT_VOLUME_SIZE` | No | Boot volume size in GB (minimum 50) |
| `NOTIFY_EMAIL` | No | `True` to enable email notifications |
| `EMAIL` / `EMAIL_PASSWORD` | No | Gmail address and App Password |
| `DISCORD_WEBHOOK` | No | Discord webhook URL |
| `TELEGRAM_TOKEN` / `TELEGRAM_USER_ID` | No | Telegram bot token and user ID |

You can also generate `oci.env` interactively:

```bash
./setup_env.sh
```

> If `oci.env` already exists, a backup (`oci.env.bak`) is created before overwriting.

## Usage

Run from the project directory:

```bash
# Start (installs dependencies and runs in background)
./setup_init.sh

# Re-run (skips dependency installation)
./setup_init.sh rerun
```

Logs are written to:
- `launch_instance.log` — OCI API call logs
- `setup_and_info.log` — instance parameters (AD, compartment, subnet, image)
- `UNHANDLED_ERROR.log` — unhandled exceptions (emailed if enabled)

Once created, a file named `INSTANCE_CREATED` is generated with the instance details.

## Project Structure

```
oracle-freetier-instance-creation/
├── main.py                 # Core instance creation logic
├── setup_env.sh            # Interactive oci.env generator
├── setup_init.sh           # Dependency installer & runner
├── sample_oci_config       # OCI config template
├── oci.env                 # Environment variables (create this)
├── oci_config              # OCI API config (create this)
├── oci_api_private_key.pem # Your OCI API private key (create this)
├── ai-image.jpg            # Project cover image
├── email_content.html      # Email notification template
├── requirements.txt        # Python dependencies
├── LICENSE                 # MIT License
├── TROUBLESHOOTING.md      # Common errors and fixes
└── README.md               # This file
```

## Notifications

### Email (Gmail)
Set `NOTIFY_EMAIL=True` and provide your Gmail address and App Password in `oci.env`. You'll receive a notification on successful creation or on unhandled errors.

### Discord
Add `DISCORD_WEBHOOK=your_webhook_url` to `oci.env`. Notifications are sent on success and on errors.

### Telegram
Create a bot via [@BotFather](https://t.me/botfather), get your token and user ID, then add:

```bash
TELEGRAM_TOKEN=your_telegram_bot_token_here
TELEGRAM_USER_ID=your_telegram_user_id_here
```

## Instance Creation Flow

```mermaid
flowchart TD
    A([Start]) --> B[Load Environment Variables]
    B --> C[Initialize OCI Clients]
    C --> D{Instance Exists?}
    D -->|Yes| E[Notify Success]
    D -->|No| F[Generate/Read SSH Key]
    F --> G[Gather OCI Resources]
    G --> K[Launch Instance]
    K --> L{Launch Successful?}
    L -->|Yes| M[Check Instance State]
    L -->|No| N[Handle Errors]
    N --> K
    M -->|Running| E
    M -->|Not Running| O[Wait and Retry]
    O --> K
    E --> P([End])

    classDef oci fill:#FF9900,stroke:#FF6600,stroke-width:2px,color:white;
    classDef local fill:#66B2FF,stroke:#0066CC,stroke-width:2px,color:white;
    classDef env fill:#99CC00,stroke:#669900,stroke-width:2px,color:white;
    classDef error fill:#FF6666,stroke:#CC0000,stroke-width:2px,color:white;
    classDef startEnd fill:#4CAF50,stroke:#45a049,stroke-width:2px,color:white

    class A,P startEnd
    class B,F env;
    class C,G,K,M oci;
    class D,E,L local;
    class N,O error;
```

## Roadmap

- [ ] Make boot volume size configurable with free-tier limit handling
- [ ] Assign a public IP automatically through the script
- [ ] Interactive image/OS selection before launch
- [ ] Local VPC and subnet creation from the script

## FAQ

**Why do I keep seeing "Out of host capacity" errors?**

This is normal. Oracle doesn't have free capacity in your availability domain right now. The script retries every `REQUEST_WAIT_TIME_SECS` seconds until capacity opens up.

**Can I run this locally instead of on an OCI Micro instance?**

Yes. Set `OCI_SUBNET_ID` in `oci.env` to an existing subnet OCID. The script will use that subnet instead of auto-detecting one.

**How do I stop the script?**

Press `Ctrl+C` in the shell session, or kill the PID stored in `$SCRIPT_PID`. If you used `screen`, re-attach with `screen -r` then press `Ctrl+C`.

**Where can I find more error-specific help?**

See [TROUBLESHOOTING.md](TROUBLESHOOTING.md) for common errors and fixes.

## Contributing

Contributions are welcome! Please open an issue or submit a pull request.

1. Fork the repository
2. Create your feature branch (`git checkout -b feature/amazing-feature`)
3. Commit your changes (`git commit -m 'Add some amazing feature'`)
4. Push to the branch (`git push origin feature/amazing-feature`)
5. Open a pull request

## License

This project is licensed under the MIT License — see the [LICENSE](LICENSE) file for details.

## References

- [OCI Python SDK — Launch Instance Example](https://docs.oracle.com/en-us/iaas/tools/python-sdk-examples/latest/core/launch_instance.py.html)
- [OCI SDK Authentication Methods](https://docs.oracle.com/en-us/iaas/Content/API/Concepts/sdk%5Fauthentication%5Fmethods.htm)
- [Required Keys and OCIDs for API Signing](https://docs.oracle.com/en-us/iaas/Content/API/Concepts/apisigningkey.htm)
- [OCI Python SDK Quickstart](https://docs.oracle.com/en-us/iaas/tools/python/latest/quickstart.html)
- [OCI Python SDK Installation](https://docs.oracle.com/en-us/iaas/tools/python/latest/installation.html)
- [Oracle Cloud Free Tier](https://www.oracle.com/cloud/free/)

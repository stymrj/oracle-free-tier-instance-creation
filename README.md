<h1 align="center">
  <img src="ai-image.jpg" alt="Oracle Free Tier Instance Creation" width="70" style="border-radius:14px"/><br/>
  Oracle Free Tier Instance Creation
</h1>

<p align="center">
  <b>Automate Oracle Cloud Always-Free ARM &amp; AMD instance provisioning.</b><br/>
  A resilient Python worker that retries until capacity frees up.
</p>

<p align="center">
  <a href="https://www.python.org/"><img src="https://img.shields.io/badge/Python-3.8%2B-3776AB?style=flat-square&logo=python&logoColor=white" alt="Python"/></a>
  <a href="https://www.gnu.org/software/bash/"><img src="https://img.shields.io/badge/Shell-Bash-4EAA25?style=flat-square&logo=gnu-bash&logoColor=white" alt="Bash"/></a>
  <a href="https://www.oracle.com/cloud/"><img src="https://img.shields.io/badge/Oracle%20Cloud-F80000?style=flat-square&logo=oracle&logoColor=white" alt="OCI"/></a>
  <a href="LICENSE"><img src="https://img.shields.io/badge/License-MIT-yellow?style=flat-square" alt="License"/></a>
  <a href="https://github.com/stymrj/oracle-free-tier-instance-creation"><img src="https://img.shields.io/github/stars/stymrj/oracle-free-tier-instance-creation?style=flat-square" alt="Stars"/></a>
  <a href="https://github.com/stymrj/oracle-free-tier-instance-creation/network/members"><img src="https://img.shields.io/github/forks/stymrj/oracle-free-tier-instance-creation?style=flat-square" alt="Forks"/></a>
</p>

---

## ✨ Features

- 🖥️ Provision `VM.Standard.A1.Flex` (ARM) or `VM.Standard.E2.1.Micro` (AMD) Always-Free instances
- 🔁 Auto-retries on `Out of host capacity` until a slot opens
- 🔑 Auto-generates an SSH key pair if none exists
- 🌍 Rotates across multiple availability domains on each retry
- 📧 Gmail, 💬 Discord, and 📱 Telegram notifications
- 🛠️ One-command setup with `setup_env.sh` + `setup_init.sh`

---

## 🎯 Why This Exists

Oracle's Always-Free ARM hosts are in constant demand — launching an instance almost always fails with **"Out of host capacity"**. Capacity frees up unpredictably, and whoever polls at that second wins. This project polls the launch API on a configurable interval and grabs the slot the instant it opens, then notifies you.

> **Note (June 2026):** Oracle cut the Always-Free Ampere A1 allowance from 4 OCPU / 24 GB to **2 OCPU / 12 GB RAM**. Verify your tenancy's limits in **Billing → Cost Analysis**.

---

## 📋 Pre-Requisites

- An [Oracle Cloud](https://www.oracle.com/cloud/free/) account with Always-Free eligibility
- An OCI API key pair (`.pem` + config) — see the [API Key Generation guide](https://graph.org/Oracle-API-Key-Generation-12-11)
- An Ubuntu machine (`bash`, `python3`, `pip3`)
- *(Optional)* Gmail App Password, Discord webhook, or Telegram bot for notifications

> ⚠️ **Never commit your private key** — OCI actively detects leaked keys in public repos.

---

## 🚀 Quick Start

```bash
git clone https://github.com/stymrj/oracle-free-tier-instance-creation.git
cd oracle-free-tier-instance-creation

# 1. Configure OCI credentials
cp sample_oci_config oci_config   # edit with your user/fingerprint/tenancy/key_file

# 2. Generate environment config (interactive)
chmod +x setup_env.sh setup_init.sh
./setup_env.sh

# 3. Install dependencies & launch the retry loop
./setup_init.sh
```

---

## 🛠️ Setup

1. **SSH into your Micro instance**, clone the repo, and `cd` into it.
2. **Create `oci_api_private_key.pem`** and paste your API private key contents.
3. **Create `oci_config`** from [`sample_oci_config`](sample_oci_config) and set `key_file` to your key's absolute path:
   ```ini
   key_file=/home/ubuntu/oracle-free-tier-instance-creation/oci_api_private_key.pem
   ```
4. **Configure `oci.env`** — copy `oci.env.sample` or run `./setup_env.sh` interactively.

---

## 🔧 Environment Variables

**Required**
| Variable | Description |
|----------|-------------|
| `OCI_CONFIG` | Absolute path to your OCI config file |
| `OCT_FREE_AD` | Availability domain(s), comma-separated — rotates through them on retry |

**Optional**
| Variable | Description |
|----------|-------------|
| `DISPLAY_NAME` | Instance name |
| `REQUEST_WAIT_TIME_SECS` | Seconds between retries (default `60`) |
| `SSH_AUTHORIZED_KEYS_FILE` | SSH public key path (auto-generated if missing) |
| `OCI_SUBNET_ID` | Subnet OCID — **required when running locally**; auto-detected on a Micro instance |
| `OCI_IMAGE_ID` | Image OCID; auto-discovered from `OPERATING_SYSTEM` + `OS_VERSION` if empty |
| `OCI_COMPUTE_SHAPE` | `VM.Standard.A1.Flex` or `VM.Standard.E2.1.Micro` |
| `SECOND_MICRO_INSTANCE` | `True` only for your second Micro instance |
| `OPERATING_SYSTEM` / `OS_VERSION` | Used for image discovery |
| `ASSIGN_PUBLIC_IP` | Assign ephemeral public IP |
| `BOOT_VOLUME_SIZE` | Boot volume in GB (min `50`) |
| `NOTIFY_EMAIL` / `EMAIL` / `EMAIL_PASSWORD` | Gmail notifications (use an App Password with 2FA) |
| `DISCORD_WEBHOOK` | Discord webhook URL |
| `TELEGRAM_TOKEN` / `TELEGRAM_USER_ID` | Telegram bot credentials |

---

## 💻 Run

```bash
./setup_init.sh          # install deps + start loop
./setup_init.sh rerun    # skip deps, start loop
```

Logs: `launch_instance.log` (API calls), `setup_and_info.log` (params), `UNHANDLED_ERROR.log` (exceptions). On success, an `INSTANCE_CREATED` file is written.

---

## 📢 Notifications

**Gmail**
```bash
NOTIFY_EMAIL=True
EMAIL=you@gmail.com
EMAIL_PASSWORD=your16charapppassword
```

**Discord** — Server Settings → Integrations → Webhooks → copy URL:
```bash
DISCORD_WEBHOOK=https://discord.com/api/webhooks/...
```

**Telegram** — create a bot via [@BotFather](https://t.me/botfather), get your ID from [@myidbot](https://t.me/myidbot):
```bash
TELEGRAM_TOKEN=123456:ABC-DEF...
TELEGRAM_USER_ID=123456789
```

---

## 📊 Instance Creation Flow

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

---

## 🗺️ Roadmap

- [ ] Create VPC & subnet from the script when running locally
- [ ] Configurable boot volume size with limit handling
- [ ] Automatic public IP assignment
- [ ] Interactive image/OS picker before launch
- [ ] Docker image for zero-dependency runs

---

## ❓ FAQ

**Why do I keep seeing "Out of host capacity"?**
Normal — Oracle has no free ARM slot right now. The script retries until one opens (hours to days). Leave it running.

**Can I run this locally?**
Yes — set `OCI_SUBNET_ID` to an existing subnet OCID.

**How do I stop it?**
`Ctrl+C`, or kill the PID shown at startup.

**What does `LimitExceeded` mean?**
You hit a service limit — check whether an ARM instance already exists, or boot-volume storage is exhausted.

More help: [TROUBLESHOOTING.md](TROUBLESHOOTING.md)

---

## 🤝 Contributing

1. Fork → 2. Branch (`feat/...`) → 3. Commit → 4. Push → 5. Pull request

---

## 👤 Author

**Satyam Raj** — [@stymrj](https://github.com/stymrj)

---

## 📄 License

MIT — see [LICENSE](LICENSE).

---

## 🔗 References

- [Oracle Launch Instance Docs](https://docs.oracle.com/en-us/iaas/api/#/en/iaas/20160918/Instance/LaunchInstance)
- [OCI Python SDK Example](https://docs.oracle.com/en-us/iaas/tools/python-sdk-examples/latest/core/launch_instance.py.html)
- [OCI SDK Authentication](https://docs.oracle.com/en-us/iaas/Content/API/Concepts/sdk%5Fauthentication%5Fmethods.htm)
- [Required Keys & OCIDs](https://docs.oracle.com/en-us/iaas/Content/API/Concepts/apisigningkey.htm)
- [oci-arm-host-capacity](https://github.com/hitrov/oci-arm-host-capacity)

---

<p align="center">
  Made with ❤️ by <a href="https://github.com/stymrj">Satyam Raj</a>
</p>

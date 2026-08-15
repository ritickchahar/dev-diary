<div align="center">

# 💻 dev-diary

**A personal, no-fluff cheatbook of notes, shortcuts, and commands for everyday development.**

*Quick lookups for daily terminal work — the stuff you always end up re-Googling.*

<br />

[![PRs Welcome](https://img.shields.io/badge/PRs-welcome-brightgreen?style=flat-square)](#-contributing)
[![Made with Markdown](https://img.shields.io/badge/Made%20with-Markdown-1f425f?style=flat-square)](https://www.markdownguide.org/)
![Last Commit](https://img.shields.io/github/last-commit/ritickchahar/dev-diary?style=flat-square)
![Repo Size](https://img.shields.io/github/repo-size/ritickchahar/dev-diary?style=flat-square)
![Stars](https://img.shields.io/github/stars/ritickchahar/dev-diary?style=flat-square)

</div>

---

## 📖 Table of Contents

- [About](#-about)
- [Topics](#-topics)
- [Usage](#-usage)
- [Repository Structure](#-repository-structure)
- [Contributing](#-contributing)
- [License](#-license)

---

## 🧭 About

`dev-diary` is a growing collection of practical, copy-paste-ready notes gathered
from real day-to-day development — deploys, containers, cloud services, and the
occasional debugging saga. Each topic lives in its own Markdown file so you can
jump straight to what you need.

> **Design principle:** keep it simple. Every entry is a quick reference, not a
> tutorial — commands first, context only where it saves you a headache.

---

## 📚 Topics

### 🔧 Version Control
| Topic | Description | Link |
|-------|-------------|------|
| **Git** | Essential Git commands, stash usage, and reset tricks | [git.md](git.md) |

### 🐳 Containers & Deployment
| Topic | Description | Link |
|-------|-------------|------|
| **Docker** | Common Docker & Docker Compose usage and maintenance | [docker.md](docker.md) |
| **VPS Deployment** | VPS + Cloudflare + Caddy + Docker + GitHub Actions CI/CD | [vps_deployment.md](vps_deployment.md) |
| **AWS EC2 SSH** | Connecting to EC2 with a PEM key + `~/.ssh/config` shortcut | [aws_ec2_ssh.md](aws_ec2_ssh.md) |
| **Amplify SPA Static Files** | Debugging static files 404-ing on Amplify + CloudFront | [amplify_spa_static_files.md](amplify_spa_static_files.md) |
| **Mongo Backup to Google Drive** | Daily encrypted off-box backup with rclone, gpg + cron | [mongo_backup_gdrive.md](mongo_backup_gdrive.md) |

### 🐍 Python
| Topic | Description | Link |
|-------|-------------|------|
| **UV** | Fast Python package manager commands | [uv.md](uv.md) |
| **Async API** | Async APIs with background tasks | [async_api.md](async_api.md) |

### ☁️ Cloud & Storage
| Topic | Description | Link |
|-------|-------------|------|
| **Azure Blob Storage** | Guide to using Azure Blob Storage | [azure_blob_storage.md](azure_blob_storage.md) |

### 🧪 API Tooling
| Topic | Description | Link |
|-------|-------------|------|
| **Postman** | API testing tips and workflows | [postman.md](postman.md) |

---

## 🚀 Usage

No installation, no build step — it's just Markdown.

```bash
# Clone it
git clone https://github.com/ritickchahar/dev-diary.git
cd dev-diary

# Open a topic (example)
less docker.md
```

Or browse the topic files directly on GitHub and use them as a searchable
reference. Tip: press <kbd>t</kbd> on the repo page to fuzzy-find a file by name.

---

## 🗂️ Repository Structure

```
dev-diary/
├── README.md                     # You are here
├── git.md                        # Version control
├── docker.md                     # Containers
├── vps_deployment.md             # Deployment runbook
├── aws_ec2_ssh.md                # SSH into EC2 with a PEM key
├── amplify_spa_static_files.md   # Amplify/CloudFront static-file 404s
├── mongo_backup_gdrive.md        # Daily encrypted Mongo backup to Google Drive
├── uv.md                         # Python package manager
├── async_api.md                  # Async APIs & background tasks
├── azure_blob_storage.md         # Cloud storage
└── postman.md                    # API testing
```

---

## 🤝 Contributing

This is a personal reference, but improvements are welcome:

1. **Fork** the repository.
2. Add or edit a topic file (keep the no-fluff, commands-first style).
3. Link new topics from the [Topics](#-topics) table.
4. Open a **pull request** — or just fork and make it your own cheatbook.

---

## 📄 License

Free to use, copy, fork, and adapt. Attribution appreciated but not required.

<div align="center">
<br />
<sub>Built from real terminal sessions · Kept simple on purpose.</sub>
</div>

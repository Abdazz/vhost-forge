# 🔨 vhost-forge — Apache / Nginx Virtual Host Automation

Ansible automation tool to configure a **complete Virtual Host** with **Apache2** or **Nginx**, Let's Encrypt SSL, MySQL/MariaDB database, Git cloning, and automatic **Laravel** project setup on a Linux server (Debian/Ubuntu).

---

## 📋 What this project does

1. **Creates a Linux user** with a password and `sudo` access
2. **Creates the project directory** under `/var/www/<project_name>`
3. **Installs and configures the web server** (Apache2 or Nginx, your choice)
4. **VirtualHost / Server Block** with HTTP → HTTPS redirect
5. **Generates an SSL certificate** via Certbot (Let's Encrypt)
6. **Creates the MySQL/MariaDB database** and associated user
7. **Clones a Git repository** (optional, with SSH support)
8. **Full Laravel project setup** (optional):
   - Composer dependency installation
   - `.env` deployment with pre-filled database credentials
   - Application key generation
   - Storage symlink creation
   - Migrations and seeders
   - Cache clearing

---

## 📁 Project structure

```
vhost-forge/
├── README.md                     # This file
├── LICENSE                       # MIT
├── .gitignore                    # Excludes vars.yml, inventory.ini (contain secrets)
├── inventory.ini.example         # Template — copy to inventory.ini
├── vars.yml.example              # Template — copy to vars.yml
├── setup_vhost.yml               # Ansible playbook (orchestrator)
├── tasks/                        # Ansible modules (separate task files)
│   ├── prompts.yml               #   → Interactive prompts & derived variables
│   ├── user.yml                  #   → Linux user creation
│   ├── project.yml               #   → Project directory creation
│   ├── apache.yml                #   → Apache2 installation & config
│   ├── nginx.yml                 #   → Nginx installation & config
│   ├── ssl.yml                   #   → SSL certificate generation
│   ├── mysql.yml                 #   → MySQL database & user creation
│   ├── git.yml                   #   → Git repository cloning (optional)
│   └── laravel.yml               #   → Laravel project setup (optional)
└── templates/                    # Jinja2 templates
    ├── apache_vhost.conf.j2      #   → Apache VirtualHost (SSL)
    ├── apache_vhost_http.conf.j2 #   → Apache VirtualHost (HTTP-only, bootstrap)
    ├── nginx_vhost.conf.j2       #   → Nginx Server Block (SSL)
    ├── nginx_vhost_http.conf.j2  #   → Nginx Server Block (HTTP-only, bootstrap)
    └── laravel.env.j2            #   → Laravel .env file
```

---

## ⚙️ Requirements

- **Target server**: Debian/Ubuntu with root access
- **DNS**: The domain must point to the server IP before running (required for Certbot)
- **Control machine**: Ansible installed locally
- **MariaDB/MySQL**: Must be pre-installed on the target server
- **For Laravel**: PHP and Composer available on the target server

---

## 🚀 Usage

### 1. Configure variables

Copy the example file, then edit `vars.yml` with your values:

```bash
cp vars.yml.example vars.yml
cp inventory.ini.example inventory.ini
```

> **🔐 Both files are gitignored** — they contain plaintext secrets. For production, encrypt `vars.yml` with [Ansible Vault](#-security--secrets) (see below).

```yaml
# Web server: "apache2" or "nginx"
web_server: "apache2"

# Linux credentials
username: "myuser"
linux_password: "MyPassword"

# Project information
project_name: "myproject"
domain: "mydomain.com"

# MySQL credentials
mysql_password: "MySQLPassword"
mysql_db_name: ""        # Default: <project_name>_db
mysql_user_name: ""      # Default: <username>

# Email for SSL certificate
certbot_email: "email@example.com"

# Git (optional) — leave empty to disable
git_repo_url: ""
git_repo_branch: "main"
git_deploy_key: ""       # Absolute path to a private SSH key on the server

# Laravel (optional)
is_laravel: false              # true to enable Laravel setup
php_binary: "php"              # e.g. "php8.3" or "/usr/bin/php"
laravel_fresh_install: false   # ⚠️ true = migrate:fresh --seed (DROPS ALL TABLES)
```

> **💡 Tip**: If a variable is left empty, the playbook will prompt you to enter it interactively at runtime.

### 2. Configure inventory

Edit `inventory.ini` with your server IP:

```ini
[webservers]
<SERVER_IP> ansible_user=root
```

### 3. Run the playbook

```bash
ansible-playbook -i inventory.ini setup_vhost.yml
```

---

## 📝 Summary displayed at the end

```
✅ Full setup completed for mydomain.com

=========================================
          CONFIGURATION SUMMARY
=========================================
Web Server:          apache2
Domain:              mydomain.com
Project Directory:   /var/www/myproject
Git Repository:      git@github.com:user/repo.git
Laravel Project:     Yes

--- System User ---
Username:            myuser
Password:            (as provided)

--- MySQL Database ---
Database Name:       myproject_db
Database User:       myuser
Database Password:   (as provided)
=========================================
```

---

## 🔧 Variable reference

### Core variables

| Variable | Required | Description |
|---|---|---|
| `web_server` | ✅ | Web server: `apache2` or `nginx` |
| `username` | ✅ | Linux username to create |
| `linux_password` | ✅ | Linux user password |
| `project_name` | ✅ | Project name (no spaces) |
| `domain` | ✅ | Domain name (e.g. `mydomain.com`) |
| `mysql_password` | ✅ | MySQL password |
| `certbot_email` | ✅ | Email for Let's Encrypt registration |
| `mysql_db_name` | ➖ | Database name — default: `<project_name>_db` |
| `mysql_user_name` | ➖ | MySQL user — default: `<username>` |

### Git variables (optional)

| Variable | Description | Example |
|---|---|---|
| `git_repo_url` | Repository URL — **leave empty to disable** | `git@github.com:user/repo.git` |
| `git_repo_branch` | Branch to clone | `main`, `production` |
| `git_deploy_key` | Absolute path to the private SSH key on the server | `/root/.ssh/deploy_key` |

**Public repository (HTTPS):**
```yaml
git_repo_url: "https://github.com/user/my-project.git"
git_repo_branch: "main"
git_deploy_key: ""
```

**Private repository (SSH):**
```yaml
git_repo_url: "git@github.com:user/my-private-project.git"
git_repo_branch: "production"
git_deploy_key: "/root/.ssh/deploy_key"
```

> **💡** For a private SSH repository, the corresponding public key must be added as a **Deploy Key** in the GitHub repository settings.

### Laravel variables (optional)

| Variable | Default | Description |
|---|---|---|
| `is_laravel` | `false` | Enable Laravel setup after cloning |
| `php_binary` | `"php"` | Path to the PHP executable (e.g. `php8.3`) |
| `laravel_fresh_install` | `false` | ⚠️ `true` → `migrate:fresh --seed` (drops all tables). `false` → safe `migrate`. |

When `is_laravel: true`, the playbook automatically runs in order:

| Step | Command |
|---|---|
| Dependency installation | `composer install` |
| `.env` deployment | Jinja2 template with pre-filled database credentials |
| Application key | `php artisan key:generate` |
| Storage link | `php artisan storage:link` |
| Migrations | `php artisan migrate` *(or `migrate:fresh --seed` if `laravel_fresh_install: true`)* |
| Cache clearing | `php artisan optimize:clear` |
| Permissions | `storage/` and `bootstrap/cache/` → `www-data` |

> **⚠️ Destructive flag**: `laravel_fresh_install: true` runs `migrate:fresh --seed` — this **drops every table** before re-creating them. Use only for first deployments or dev/staging resets. Leave `false` for production and re-runs.

---

## 🧩 Modular architecture

The `setup_vhost.yml` playbook is an **orchestrator** that delegates each step to a dedicated file via `include_tasks`:

```
validate → prompts → user → project → apache|nginx → ssl → mysql → git → laravel → summary
```

```yaml
# Simplified
- include_tasks: tasks/prompts.yml
- include_tasks: tasks/user.yml
- include_tasks: tasks/project.yml
- include_tasks: tasks/apache.yml       # when: web_server == "apache2"
- include_tasks: tasks/nginx.yml        # when: web_server == "nginx"
- include_tasks: tasks/ssl.yml
- include_tasks: tasks/mysql.yml
- include_tasks: tasks/git.yml          # when: git_repo_url != ""
- include_tasks: tasks/laravel.yml      # when: is_laravel == true AND git_repo_url != ""
```

**Benefits**:
- Each module is **self-contained** (~20–50 lines), easy to read and modify
- **Extensible**: add `tasks/nodejs.yml`, `tasks/redis.yml`, etc. without touching anything else
- Optional features (Git, Laravel) are **strictly conditional**

---

## 🌐 VirtualHost templates

Templates are located in `templates/` and use the `web_root` variable:

- For a **standard** project: `web_root = /var/www/<project_name>`
- For a **Laravel** project: `web_root = /var/www/<project_name>/public`

| Template | Server | Used when |
|---|---|---|
| `apache_vhost.conf.j2` | Apache | SSL certificate exists |
| `apache_vhost_http.conf.j2` | Apache | Bootstrap (before SSL) |
| `nginx_vhost.conf.j2` | Nginx | SSL certificate exists |
| `nginx_vhost_http.conf.j2` | Nginx | Bootstrap (before SSL) |
| `laravel.env.j2` | — | `is_laravel: true` |

---

## 🔐 Security — secrets

`vars.yml` and `inventory.ini` contain plaintext secrets (passwords, server IPs, SSH key paths) and are **gitignored** by default. Treat them like you would treat a `.env` file.

### Encrypt with Ansible Vault (recommended for production)

```bash
# Encrypt the file in place
ansible-vault encrypt vars.yml

# Edit an encrypted file
ansible-vault edit vars.yml

# Run the playbook with vault password
ansible-playbook -i inventory.ini setup_vhost.yml --ask-vault-pass

# …or use a password file (also gitignored)
echo "my-vault-password" > .vault_pass
chmod 600 .vault_pass
ansible-playbook -i inventory.ini setup_vhost.yml --vault-password-file .vault_pass
```

See the [Ansible Vault documentation](https://docs.ansible.com/ansible/latest/vault_guide/index.html) for advanced usage (per-variable encryption, multiple vault IDs).

---

## ⚠️ Important notes

- **DNS**: The domain must point to the server IP **before** running, otherwise Certbot will fail.
- **MariaDB**: The MariaDB service must be installed and running on the target server.
- **Idempotency**: The playbook can be re-run safely — already-created resources are skipped. By default, `laravel_fresh_install: false` so re-runs use `migrate` (additive) and never drop existing data.

---

## 📄 License

MIT

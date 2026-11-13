# darkwiki

**Operational security knowledge base** — hardening, detection, defensive tactics, and incident
patterns from real lab work. Markdown-based, searchable, built for long-term reference.

---

## Structure

```
docs/
├── hardening/       # OS & app lockdown, least privilege, auditing
├── detection/       # Log analysis, network forensics, attack indicators
├── incident/        # Pattern matching: what to look for post-breach
├── tools/           # Reference guides: netstat, auditd, osquery, yara
├── ttps/            # Adversary techniques (MITRE ATT&CK mapped)
└── writeups/        # Real incidents analyzed
```

---

## Quick reference

### Hardening

| System | Key hardening | File |
|---|---|---|
| Linux (Ubuntu 22.04) | Audit policy, umask 0077, sudo logging, fail2ban | `hardening/linux-baseline.md` |
| SSH | Key-only auth, AllowUsers, PermitRootLogin no | `hardening/ssh-lockdown.md` |
| Firewall | UFW baseline, deny-default ingress | `hardening/ufw-rules.md` |
| Docker | Non-root user, read-only root FS, seccomp profile | `hardening/docker-security.md` |
| PostgreSQL | Row-level security, encrypted password hashing, audit log | `hardening/postgres-rbac.md` |

### Detection

| Signal | What it means | False positive rate |
|---|---|---|
| `sudo -i` no args in audit log | Privilege escalation attempt | ~5% (legitimate admin use) |
| Port scan from internal IP | Lateral movement recon | ~0% (almost never legitimate) |
| Process spawning /bin/bash from web app | Web shell or RCE | ~0% |
| SSH key permission >600 | Credential theft prep | ~2% (ansible, deployment tools) |
| Unusual cron job in /etc/cron.d | Persistence mechanism | ~1% (legitimate tool installs) |

### Incident patterns

**Pattern: Web shell + reverse shell**

Evidence trail:
1. Unexpected `.php` or `.jsp` in web root (file integrity check)
2. Web server user spawning `bash` (auditd rule)
3. Outbound connection from web process to attacker IP (netstat + IP reputation)

**Response:**
- Kill the web process
- Isolate the host
- Extract `.php` file + memory dump
- Review logs for entry point (1–3 days back)

---

## Knowledge base examples

### `hardening/linux-baseline.md` (excerpt)

```markdown
## Umask

Default: 0022 (files 644, dirs 755) — readable by everyone.
Hardened: 0077 (files 600, dirs 700) — readable only by owner.

Set in: /etc/profile, ~/.bashrc, /etc/default/useradd

Verify:
  $ umask
  0077
```

### `detection/auditd-rules.md`

```markdown
## Rules for web app RCE

-a exit,always -F arch=b64 -S execve -F uid=33 -k web_execve
# uid=33 is www-data on Ubuntu

Alert if www-data (uid 33) spawns anything other than:
  - /usr/sbin/apache2 (parent)
  - PHP interpreter (expected)
  - Database queries (mysql, psql processes)
```

### `ttps/persistence.md`

Mapped to MITRE ATT&CK T1547 (Boot or Logon Autostart Execution):

- **Cron jobs** (`/etc/cron.d`, user crontab)
- **Systemd services** (`.service` files in `/etc/systemd/system`)
- **SSH authorized_keys** (backdoor key)
- **Iptables rules** (persistence via network)
- **PAM modules** (kernel-level hooks)

Detection per method → Remediation steps → Lab validation

---

## Real incident writeup

### Case: Postgres auth bypass via misconfig

**Timeline:**

```
2026-03-15 02:14 UTC  postgres process spawns /bin/bash (auditd alert)
2026-03-15 02:15 UTC  Connection to attacker IP 1.2.3.4:4444 (netstat)
2026-03-15 02:16 UTC  /tmp/bc file created with execute bit (file integrity check)
2026-03-15 02:20 UTC  Database schema altered (audit log: CREATE USER attacker)
```

**Root cause:**

```sql
-- Postgres running as root (WRONG)
-- pg_hba.conf allowed password auth without password validation
-- CREATE FUNCTION with LANGUAGE C allowed loading arbitrary .so files
```

**Fix:**

```sql
-- postgres user, never root
-- pg_hba.conf: trust only localhost, md5+ for remote
-- Disable LANGUAGE C, audit CREATE FUNCTION
```

**Lesson:** Configuration is attack surface. Postgres defaults assume single-user labs, not
production. Audit every `CREATE FUNCTION` and every process permission.

---

## How to use this wiki

1. **Incident response:** Search by symptom (e.g., "bash spawned from web app") → TTP reference →
   Detection rules → Response steps
2. **Hardening checklist:** Read `hardening/*.md`, apply rules, use `auditd` to verify
3. **Threat hunting:** Patterns in `ttps/` → build queries in `tools/` → hunt your logs
4. **Learning:** Read writeups to see real chains of evidence

---

## Tools referenced

Each tool has a reference page:

| Tool | Purpose | Reference |
|---|---|---|
| `auditd` | Kernel-level syscall auditing | `tools/auditd.md` |
| `osquery` | System telemetry on all endpoints | `tools/osquery.md` |
| `yara` | Malware/webshell pattern matching | `tools/yara.md` |
| `netstat` / `ss` | Active connections | `tools/netstat-ref.md` |
| `jq` | JSON log parsing | `tools/jq-recipes.md` |

---

## Lab-scoped, defensive-only

This wiki documents **detection and hardening**. No active exploitation, no C2 channels, no
malware staging. All patterns are from authorized security research and lab exercises.

Use case: **Your environment security guide** — what to harden, what to look for, how to respond.

---

## Contributing

Add incident writeups (sanitized of real names/IPs), new hardening guides, or detection patterns.
Format: markdown, code blocks for CLI examples, clear **Timeline → Diagnosis → Fix** structure.

---

## License

MIT. Use for defense, threat hunting, and incident response. No warranty — security configurations
can brick systems if misconfigured.

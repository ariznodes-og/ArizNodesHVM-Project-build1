# ARIZNODES HVM — MASTER BUILD PROMPT

Version 1.0  |  Paste this whole document into your AI coding assistant  |  Build phase by phase

---

## SECTION 0 — DOCUMENT CONVENTIONS

0.1   MUST / MUST NOT = mandatory. The build is rejected if violated.
0.2   SHOULD = strong default. Deviate only with a one-line reason.
0.3   MAY = optional enhancement. Implement if the phase budget allows.
0.4   Requirements are numbered by section (for example 7.3.2). Cite these numbers in code comments, tests and commit messages.
0.5   "Panel" = the ArizNodes HVM web application (API + dashboards).
0.6   "Node" = a host (VPS, codespace, bare metal) running the ArizNodes agent, Docker and KVM.
0.7   "VPS" = a virtual server instance provisioned as a Docker container that runs QEMU/KVM.
0.8   "Leader" = the primary node (vms1) that coordinates all other nodes.
0.9   "Key portal" = the separate external website that issues ArizOfficial API keys.
0.10  "Admin" = a privileged account. "User" = a normal customer account.
0.11  Where this document is ambiguous, choose the most secure and highest-performance option, state the assumption in one line, and continue.
0.12  Where this document conflicts with security best practice, follow best practice and state the deviation in one line.

---

## SECTION 1 — ROLE AND MISSION

1.1   You are a principal full-stack engineer and security-focused software architect.
1.2   You write production-ready, modular, DRY, well-tested code. Delivered code never contains "TODO", "implement later" or "..." placeholders.
1.3   Mission: build ArizNodes HVM, a self-hostable, Proxmox-style VPS management panel.
1.4   The panel provisions KVM/QEMU-backed VPS instances as local Docker containers on registered nodes.
1.5   Admins create and assign VPS. Users manage the VPS assigned to them. Users never create VPS.
1.6   Anyone must be able to run the whole platform locally or on their own servers with one "docker compose up" command.
1.7   Priorities in order: security, correctness, reliability, performance, developer experience, visual polish.
1.8   Target audience: hobbyists, small hosting communities and homelab operators.
1.9   Product name: "ArizNodes HVM". Node hostnames follow vms1.ariznodes.com, vms2.ariznodes.com and so on.

---

## SECTION 2 — OPERATING RULES FOR YOU (THE ASSISTANT)

2.1   Work in the phases defined in SECTION 25. Finish one phase completely before starting the next.
2.2   At the start of every phase, restate in five lines or fewer: goal, files to be created, assumptions.
2.3   Deliver complete, copy-pasteable files. Put the full relative path as a header above each file.
2.4   Never truncate a file. If a file is too long for one response, split it into logical modules instead of cutting it.
2.5   If your output is cut off, resume from the last complete file when the user says "continue".
2.6   Comments explain WHY, not WHAT. Keep them concise.
2.7   Validate all input at the boundary with a schema library. Reject unknown fields.
2.8   Use parameterized queries or the ORM query builder only. Never build SQL by string concatenation.
2.9   Never build shell commands by string concatenation. Use argument arrays or the Docker SDK.
2.10  Never log secrets, passwords, tokens, API keys, verification codes or TOTP secrets.
2.11  Every state-changing action writes an audit log entry.
2.12  Every external call (SMTP, Cloudflare, Docker, node agent, key portal) has a timeout, retry with backoff and a typed error.
2.13  Prefer boring, well-maintained libraries. Pin versions. Justify each dependency in one line.
2.14  After each phase output: (a) what was built, (b) how to run it, (c) how to test it, (d) known limitations.
2.15  After each phase STOP and wait for the user to say "continue".
2.16  If a requirement would create a serious vulnerability, implement the safe variant and explain the change in one sentence.

---

## SECTION 3 — PRODUCT OVERVIEW

3.1   Architecture

      [Browser] --HTTPS--> [Panel API + Web UI] ---> [PostgreSQL]  [Redis]
                                    |
                                    v
                           [Job queue / Orchestrator]
                                    |   signed commands over Cloudflare Tunnel
                                    v
                     [Node agent on vms1 ... vmsN] ---> [Docker Engine]
                                                              |
                                                              v
                                                 [VPS containers running QEMU/KVM]

3.2   VPS creation flow (the "system overflow")
      Step 1  Admin submits a VPS creation package: plan, owner, resources, image, target node.
      Step 2  The Panel API validates the package: schema, permissions, quotas, node capacity.
      Step 3  The Panel stores the package in the database as "pending" and signs it (Ed25519).
      Step 4  The database record is the source of truth. The package is verified against it before anything runs.
      Step 5  The orchestrator sends the signed package to the target node agent.
      Step 6  The agent verifies the signature and confirms the package with the Panel before executing.
      Step 7  The agent creates and starts the container, reporting progress events.
      Step 8  The Panel marks the VPS "active", stores connection details encrypted, and notifies the owner by in-app message and email.
      Step 9  Any failure sets the VPS to "failed", rolls back partial resources and notifies the admin.

3.3   Login flow summary
      Credentials -> TOTP code -> emailed 6-digit code -> session. Max 5 failed attempts. Weekly device re-verification (SECTION 8).

3.4   Non-goals for v1: billing and invoices, multi-region scheduling, Kubernetes, Windows guests, live migration.

---

## SECTION 4 — TECH STACK AND ASSUMPTIONS

4.1   Backend language     : TypeScript on Node.js 22 LTS.
4.2   Backend framework    : Fastify with zod schemas (fast, schema-first).
4.3   ORM and migrations   : Drizzle ORM or Prisma, plus versioned SQL migrations.
4.4   Database             : PostgreSQL 16.
4.5   Cache and queue      : Redis 7 with BullMQ for jobs, rate limits and short-lived codes.
4.6   Frontend             : React 18, Vite, TypeScript, Tailwind CSS, TanStack Query, React Router.
4.7   Real-time            : Server-Sent Events (or WebSocket) for live status and job progress.
4.8   Node agent           : Go or Node.js, shipped as a static binary or container; talks to the Docker Engine API.
4.9   Email                : SMTP through nodemailer (Gmail, Outlook or any provider). HTML and plain-text templates.
4.10  Auth primitives      : Argon2id (passwords), otplib (TOTP), jose (JWT), crypto.randomInt and randomBytes (codes, tokens).
4.11  Reverse proxy        : Caddy or Nginx in docker compose; TLS terminates at Cloudflare or the proxy.
4.12  Tunnel               : cloudflared, managed through the Cloudflare API.
4.13  Testing              : Vitest, Supertest, Playwright, Testcontainers.
4.14  Lint and format      : ESLint, Prettier, strict TypeScript.
4.15  Assumption: Linux hosts with Docker Engine 24 or newer and /dev/kvm available for hardware virtualization.
4.16  Assumption: the Panel and the key portal are separate deployable services.
4.17  Assumption: single region and a single database for v1; horizontal scaling is a v2 goal.
4.18  If you prefer another stack (for example Python and FastAPI), say so in one line and keep every requirement below intact.

---

## SECTION 5 — REPOSITORY LAYOUT

5.1   Use a pnpm monorepo with exactly this structure (add files as needed, never remove folders):

```text
ariznodes-hvm/
├── README.md
├── LICENSE
├── package.json
├── pnpm-workspace.yaml
├── Makefile
├── docker-compose.yml
├── docker-compose.dev.yml
├── .env.example
├── .github/workflows/ci.yml        # optional, lint/test/build only, never provisions VPS
├── apps/
│   ├── panel-api/
│   │   ├── Dockerfile
│   │   ├── src/
│   │   │   ├── main.ts
│   │   │   ├── config/env.ts
│   │   │   ├── plugins/            # db, redis, auth, csrf, rate-limit, security-headers, audit
│   │   │   ├── modules/
│   │   │   │   ├── auth/           # signup, login, totp, email-code, sessions, devices
│   │   │   │   ├── users/
│   │   │   │   ├── vps/
│   │   │   │   ├── nodes/
│   │   │   │   ├── license/        # Ariz key verification and startup gate
│   │   │   │   ├── admin/          # biayp, user-control, status, live-control, settings
│   │   │   │   ├── abuse/          # mining detection, alt-account linking
│   │   │   │   ├── notifications/
│   │   │   │   └── audit/
│   │   │   ├── jobs/               # provisioning, reverify-scheduler, heartbeat-watch, cleanup
│   │   │   ├── lib/                # crypto, mailer, cloudflare, docker-command-builder, errors
│   │   │   └── db/                 # schema, migrations, seed
│   │   └── test/
│   ├── panel-web/
│   │   ├── src/
│   │   │   ├── routes/             # login, signup, verify, dashboard, admin/*
│   │   │   ├── components/         # otp-input, vps-card, node-table, charts
│   │   │   ├── features/
│   │   │   ├── lib/                # api client, auth store, sse
│   │   │   └── styles/
│   │   └── e2e/
│   ├── node-agent/
│   │   ├── src/                    # enroll, heartbeat, command-verifier, docker-driver, firewall
│   │   └── install/install.sh.tmpl
│   └── key-portal/
│       ├── Dockerfile
│       └── src/                    # issue, revoke, verify, admin-ui
├── packages/
│   ├── shared-types/               # zod schemas shared by API, web and agent
│   ├── crypto/                     # ed25519 signing, aes-gcm envelope, token helpers
│   └── config/                     # eslint, tsconfig, prettier presets
├── images/
│   └── arz-ubuntu24/
│       ├── Dockerfile
│       ├── entrypoint.sh
│       └── healthcheck.sh
├── scripts/                        # dev-setup, backup, restore, rotate-keys
└── docs/                           # architecture, runbooks, threat-model
```

5.2   Shared zod schemas live in packages/shared-types so the API, web app and agent can never drift apart.
5.3   Business logic lives in services, not in route handlers. Route handlers only parse, authorize, call a service and serialize.

---

## SECTION 6 — DATABASE MODEL (PostgreSQL 16)

6.1   Conventions: UUIDv7 primary keys, timestamptz everywhere, created_at and updated_at on every table, citext for usernames and emails, explicit ON DELETE rules, CHECK constraints for enums.
6.2   Migrations are versioned, reversible and reviewed. A seed script creates the first admin from environment variables and forces a password change on first login.
6.3   Tables (columns abbreviated; create the listed indexes):

      users
        id, username (unique), email (unique), email_verified_at, password_hash, role (admin|user),
        status (pending|active|locked|banned), failed_attempts, locked_until, ban_reason, banned_at,
        banned_by, password_changed_at, password_leaked_at, abuse_score, last_login_at, last_login_ip
        indexes: (status), (role), (last_login_ip)

      totp_credentials
        id, user_id (unique FK), secret_encrypted (AES-256-GCM), confirmed_at, last_used_step

      backup_codes
        id, user_id, code_hash, used_at

      email_codes
        id, user_id, purpose (login|verify_email|reverify|reset), code_hash, attempts, expires_at,
        consumed_at, ip, device_id
        index: (user_id, purpose, expires_at)

      devices
        id, user_id, fingerprint_hash, cookie_id_hash, label, status (active|pending_reverify|blocked),
        first_seen_at, last_seen_at, next_reverify_at, reverify_deadline_at, blocked_at, blocked_reason
        indexes: (user_id), (status, next_reverify_at), (fingerprint_hash)

      sessions
        id, user_id, device_id, family_id, refresh_hash, ip, user_agent, expires_at, revoked_at, rotated_from
        indexes: (user_id), (family_id), unique (refresh_hash)

      login_attempts
        id, identifier_hash, user_id (nullable), ip, device_hash, step (password|totp|email), success,
        failure_reason, created_at
        indexes: (ip, created_at), (user_id, created_at)

      license_state
        id, key_hash, key_prefix, status, panel_instance_id, license_token, expires_at, last_check_at, mode

      nodes
        id, name, slug, hostname, is_leader, status (pending|online|offline|draining|disabled),
        agent_version, public_key, cf_tunnel_id, cf_dns_record_id, total_cpu, total_ram_mb,
        total_disk_gb, overcommit_cpu, overcommit_ram, ssh_port_range, console_port_range,
        ssh_access_mode, config_version, last_heartbeat_at
        indexes: unique (slug), unique (hostname), partial unique (is_leader) WHERE is_leader

      node_enrollments
        id, node_id, token_hash, expires_at, used_at, created_by

      node_config_versions
        id, node_id, version, config jsonb, created_by, created_at, rolled_back_at

      node_heartbeats   (partition by day, keep 30 days)
        node_id, cpu_pct, ram_used_mb, disk_used_gb, net_rx, net_tx, vps_running, created_at

      plans
        id, name, cpu, ram_mb, disk_gb, image, is_active

      vps
        id, short_id (unique), owner_id, node_id, plan_id, name, status, cpu, ram_mb, disk_gb, image_digest,
        container_name, volume_name, ssh_port, console_port, secrets_encrypted, package_signature,
        idempotency_key (unique), suspended_reason, created_by, created_at, deleted_at
        indexes: (owner_id), (node_id, status), unique (node_id, ssh_port), unique (node_id, console_port)

      vps_events
        id, vps_id, type, payload jsonb, actor_id, created_at

      vps_metrics   (partition by day, keep 14 days)
        vps_id, cpu_pct, ram_used_mb, disk_used_gb, net_rx, net_tx, created_at

      jobs
        id, type, payload jsonb, status, attempts, last_error, run_at, created_at

      audit_logs   (append-only: revoke UPDATE and DELETE)
        id, actor_id, actor_ip, action, target_type, target_id, before jsonb, after jsonb, request_id, created_at
        indexes: (actor_id, created_at), (target_type, target_id)

      notifications
        id, user_id, type, title, body, read_at, created_at

      bans
        id, subject_type (user|ip|device|email_domain), subject_hash, reason, expires_at, created_by, lifted_at

      account_links
        id, user_a, user_b, signals jsonb, score, status (suggested|confirmed|dismissed), reviewed_by

      abuse_events
        id, vps_id, type (mining|spam|scan|ddos|proxy), score, evidence jsonb, action_taken, reviewed_by, created_at

      settings
        key (pk), value (json or encrypted), updated_by, updated_at

6.4   Query rules: no N+1 (use joins or batched loads), cursor pagination on large tables, EXPLAIN-check the ten hottest queries, statement timeout of 5 seconds.
6.5   Encrypt secrets at rest with AES-256-GCM using an envelope key from the environment or a KMS; support key rotation through a key_id column.

---

## SECTION 7 — AUTHENTICATION AND SESSIONS

7.1   Signup page (a "Sign up" button on the login page links here)
7.1.1 Fields: username (3-24 chars, a-z 0-9 underscore), email, password, confirm password, terms checkbox, Cloudflare Turnstile.
7.1.2 Password policy: 12-128 chars, zxcvbn score of 3 or more, rejected if found in a breach corpus (HIBP k-anonymity: only the first 5 SHA-1 hex characters leave the server).
7.1.3 Hashing: Argon2id (64 MiB, 3 iterations, parallelism 1, tunable by env) with a server-side pepper applied through HMAC-SHA256 before hashing.
7.1.4 Email verification: 6-digit code, 10 minute expiry, 5 attempts. The account stays "pending" until verified.
7.1.5 After verification the user must enroll TOTP before reaching the dashboard: QR code, manual key and 10 one-time backup codes shown once.
7.1.6 The TOTP secret is displayed ONLY during enrollment. Login never asks the user to type the secret, only the rotating 6-digit code.
7.1.7 Signup and error responses are uniform to prevent username and email enumeration.
7.1.8 An admin setting can disable public signup (invite-only mode). Signup never grants the admin role.

7.2   Login page
7.2.1 Title: "ArizNodes Login". Banner: "You have a maximum of 5 attempts."
7.2.2 Step 1 fields: username or email, and password. Links to signup and to password reset.
7.2.3 Step 2: two-factor code (TOTP) or a backup code.
7.2.4 Step 3: emailed verification code.
      Heading : "Please Enter Verification Code"
      Body    : "A 6-digit code was sent to your Gmail or Outlook inbox. Please check it (and your spam folder)."
      Input   : [ ] [ ] [ ] [ ] [ ] [ ]
7.2.5 Steps are bound together by a signed, single-use "login challenge" token (5 minute TTL) so steps cannot be skipped, replayed or reordered. The server records which steps passed.
7.2.6 A session is issued only after all three steps succeed.

7.3   Attempt limiting
7.3.1 Maximum 5 failed attempts per account per 15 minutes, counted across ALL steps (password, TOTP, email code).
7.3.2 On the 5th failure lock the account for 15 minutes. Each further lock doubles (30 min, 1 h, ...) up to 24 h. Email the owner about the lock.
7.3.3 Show a remaining-attempts counter in the UI without revealing which field was wrong.
7.3.4 Per-IP limits: 20 attempts per 15 minutes plus a global velocity alarm. Show Turnstile after 2 failures.
7.3.5 Equalize timing: run a dummy Argon2 verification for unknown users and use constant-time comparison for every secret.
7.3.6 Record every attempt in login_attempts with IP, device hash and outcome. Admins can search them.

7.4   Email verification codes
7.4.1 Generate with crypto.randomInt (never Math.random). Six digits, zero-padded.
7.4.2 Store only an HMAC-SHA256 of the code with a pepper. Single use. 10 minute TTL.
7.4.3 Max 5 verification attempts per code. Resend cooldown 60 s. Max 5 sends per hour. A new code invalidates older ones.
7.4.4 The email (HTML and plain text) shows the code, time, IP, approximate location, device and a "This wasn't me" link that revokes all sessions and forces a password reset.
7.4.5 Mail is sent through the job queue with retries. Login never waits on SMTP longer than a 10 second timeout.
7.4.6 UI: six inputs with inputmode="numeric" and autocomplete="one-time-code", paste-to-fill, auto-advance, backspace navigation, auto-submit on the 6th digit, resend countdown, ARIA labels and an error shake animation.

7.5   Sessions and tokens
7.5.1 Access token: JWT signed with EdDSA, 10 minute lifetime. Refresh token: opaque random 32 bytes, stored hashed, 7 day sliding window, 30 day absolute cap.
7.5.2 Refresh rotation with reuse detection: replaying an old refresh token revokes the whole token family.
7.5.3 Cookies: __Host- prefix, httpOnly, Secure, SameSite=Lax (Strict for the refresh path). Never store tokens in localStorage.
7.5.4 CSRF: double-submit token on every state-changing request plus Origin and Referer checks.
7.5.5 Users can list active sessions and devices and revoke any of them. "Log out everywhere" is available.
7.5.6 After login show this notice: "You are logged in. Your account is still in a verification check. Every 7 days you will receive an email code and must enter it within 24 hours, or this device will be permanently blocked as a suspected bot."

7.6   Password reset and 2FA recovery
7.6.1 Reset link: 32 random bytes, hashed in the database, 30 minute TTL, single use. Reset also requires a TOTP or backup code.
7.6.2 A successful reset revokes every session and notifies the owner by email.
7.6.3 Lost 2FA: use a backup code, or an admin-assisted recovery flow that is fully audited and requires approval from a second admin.
7.6.4 An admin-forced reset (SECTION 15) invalidates all sessions and requires a new password at next login.

---

## SECTION 8 — DEVICE TRUST AND WEEKLY RE-VERIFICATION

8.1   Device identity = a signed random device cookie (__Host-dvc) plus a server-side fingerprint hash (user agent, language, timezone, screen class). The fingerprint is a signal, never the sole identity.
8.2   A new device must pass the full three-step login, then is registered as "active" with next_reverify_at = now + 7 days.
8.3   A scheduler runs every 5 minutes using SELECT ... FOR UPDATE SKIP LOCKED so multiple API instances never double-process a device.
8.4   When next_reverify_at is reached the device becomes "pending_reverify", reverify_deadline_at = now + 24 hours, and the user receives a 6-digit email code plus a magic link.
8.5   While pending the user can keep working but sees a persistent banner with a live countdown and a "Verify now" button.
8.6   Successful verification returns the device to "active" and sets next_reverify_at = now + 7 days.
8.7   If the deadline passes: status becomes "blocked", all sessions on that device are revoked, the device cookie and fingerprint are blocklisted and the owner is emailed.
8.8   A blocked device stays blocked until an admin lifts it (Admin > User Control > Devices) or the owner completes account recovery (TOTP or backup code plus email link) from a new device.
8.9   Blocking a device never deletes the account and never stops running VPS.
8.10  Reverify attempts: max 5 wrong codes per challenge and max 3 resends per day. Exceeding either blocks the device immediately.
8.11  Repeated blocks across many devices, or many blocked devices sharing one IP, raise the abuse score and appear in User Control for review.
8.12  Every transition (active, pending_reverify, blocked, unblocked) is written to audit_logs.
8.13  The cadence (7 days) and window (24 hours) are configurable in Settings with those defaults.

---

## SECTION 9 — ROLES AND PERMISSIONS (RBAC)

9.1   Roles: admin and user. Deny by default. Enforce in three layers: route middleware, service layer and SQL ownership filters.
9.2   Permission matrix:

      Action                          User                    Admin
      ------------------------------  ----------------------  ----------------
      View own VPS                    yes                     yes
      View any VPS                    no                      yes
      Start / stop / restart own VPS  yes                     yes
      Open console on own VPS         yes                     yes
      Reinstall own VPS               yes (confirm + TOTP)    yes
      Create VPS                      NO                      yes
      Assign VPS to a user            no                      yes
      Delete VPS                      no (can request)        yes
      Ban / unban users               no                      yes
      Manage nodes and live control   no                      yes
      View audit logs                 no                      yes
      Change settings                 no                      yes
      Impersonate a user (view only)  no                      MAY, with reason

9.3   Every :id route has an IDOR test proving a user cannot read or modify another user's resources.
9.4   Admin actions that are destructive (delete VPS, ban, promote leader, rotate keys) require re-entering the admin's TOTP code.
9.5   The last remaining admin cannot be deleted, banned or demoted.

---

## SECTION 10 — ARIZ API KEY SYSTEM AND KEY PORTAL

10.1  Key format: arz-#010101-<20 random characters>
      Regex : ^arz-#010101-[A-Za-z0-9]{20}$
      "#010101" is the fixed series marker. Keep it in a constant (KEY_SERIES) so future series can be added.
10.2  Generation: CSPRNG with rejection sampling (no modulo bias) over A-Z a-z 0-9. The random part carries about 119 bits of entropy.
10.3  Storage in the key portal: only a SHA-256 hash plus a short display prefix. The plaintext is shown exactly once at issue time.
10.4  The key portal is a separate service with its own database, admin login (same auth rules as SECTION 7) and audit log.
10.5  Portal features: issue key, revoke, list, per-key metadata (owner, note, max activations, expires_at, status), usage view, rate limits.
10.6  Verification API: POST /v1/keys/verify with {key, panel_instance_id, panel_version, public_url}.
      The response is a signed (Ed25519) license token: {valid, expires_at, features, panel_instance_id, issued_at}.
10.7  The panel verifies the token signature with the portal's public key, caches it and re-checks every 6 hours.
10.8  Bind a key to one panel instance on first activation. Maximum activations per key is configurable (default 1). Re-binding requires a portal action.
10.9  Startup gate: without a valid key the panel serves ONLY the "Enter ArizOfficial Key" setup page and refuses every other route.
10.10 Offline grace period: 72 hours (configurable). After that the panel enters "restricted mode" (read-only, no new VPS, no new users).
10.11 Revocation: when the portal revokes a key, the panel enters restricted mode at its next check. BIAYP shows the license state.
10.12 Verification attempts are rate limited (5 per minute per IP) and use constant-time comparison.
10.13 The key is masked everywhere (arz-#010101-ab************) and never appears in logs.
10.14 Ship a "license" module in the panel and a separate portal package that share one signed-token specification.

---

## SECTION 11 — NODES, ENROLLMENT AND CLOUDFLARE TUNNELS

11.1  Naming: {prefix}{index}.{base_domain}. Defaults: prefix "vms", base "ariznodes.com" -> vms1.ariznodes.com, vms2.ariznodes.com, and so on. Custom slugs such as sv2.yourhvm.com are allowed. There is no node limit.
11.2  Prerequisites: a Cloudflare account, the domain on Cloudflare and an API token with least privilege (Zone DNS Edit plus Account Cloudflare Tunnel Edit).
11.3  Add-node wizard (admin): name, slug, label, resource limits, SSH access mode. The panel creates the node record as "pending", the enrollment token, the Cloudflare tunnel and the DNS record, then shows the unique install command.
11.4  Install command (unique per node, single use):
      curl -fsSL https://<panel-domain>/enroll/<enrollment-id>/install.sh | sudo bash -s -- --token <ENROLL_TOKEN>
11.5  Command security: the token is 32 random bytes, stored hashed, single use, expires after 15 minutes and is bound to one node record. The installer verifies the SHA-256 checksum of the agent binary. The secrecy of the command is NOT the security boundary; post-enrollment authentication (11.7) is.
11.6  Installer steps:
      a) require root, supported OS and architecture
      b) check CPU virtualization flags (vmx or svm), /dev/kvm existence and permissions
      c) install Docker from the official repository if missing
      d) fetch the agent, verify checksum, install as a systemd service
      e) generate an Ed25519 key pair locally (the private key never leaves the node)
      f) call the enrollment endpoint with the token, public key and hardware inventory
      g) receive the tunnel token and the panel's public key, start cloudflared
      h) run a self-test (Docker, KVM, tunnel, clock skew) and print a clear pass or fail report
11.7  Runtime authentication: every panel-to-agent and agent-to-panel request is signed (Ed25519) and carries a timestamp and nonce. Reject requests older than 60 seconds or with a reused nonce (Redis). mTLS is an acceptable alternative.
11.8  The installer works on any VPS or codespace that exposes KVM. If /dev/kvm is missing it aborts with an explicit message. TCG software emulation is available only behind an explicit flag and is labeled "very slow" in the UI.
11.9  Heartbeat every 15 seconds (jittered): CPU, RAM, disk, network, running VPS count, agent version, Docker version, KVM status. Three missed beats mark the node offline and raise an alert.
11.10 Leader: exactly one node has is_leader = true (vms1 by default), enforced by a partial unique index. The leader receives configuration from the panel and fans it out to the other nodes; the panel database stays the source of truth. Promote and demote are manual in v1, with confirmation and audit. Automatic failover is v2.
11.11 Node removal: drain (no new VPS), confirm, revoke keys, delete tunnel and DNS record, deregister the agent.
11.12 The agent accepts only these allow-listed commands: vps.create, vps.start, vps.stop, vps.restart, vps.kill, vps.delete, vps.reinstall, vps.stats, vps.console_token, node.config.apply, node.update, node.drain, node.ping.
11.13 There is NO generic "run shell command" endpoint on the agent. Ever.

---

## SECTION 12 — VPS PROVISIONING (LOCAL DOCKER)

12.1  VPS are created ONLY by local Docker containers on nodes. GitHub Actions and any other CI system are never used to create VPS.
12.2  Reference command the platform must be able to generate from validated parameters:

```bash
docker run -d \
  --name ariznodes-vs \
  --privileged \
  -p 6080:6080 \
  -p 2026:2222 \
  -e RAM=430080 \
  -e CPU=24 \
  -e DISK=2048 \
  -e VNC_PASS=admin \
  -e ROOT_PASS=admin \
  -v ariznodes-vm-data:/vm \
  --restart unless-stopped \
  ariznodes/arz-ubuntu24
```

12.3  Parameter mapping (every value comes from the validated package, never hardcoded):

      Flag or variable   Meaning                        Rule
      -----------------  -----------------------------  ------------------------------------------
      --name             container name                 ariznodes-vs-<short_id>
      -p 6080:6080       noVNC web console              host port from console pool, bind 127.0.0.1
      -p 2026:2222       SSH into the guest             host port from SSH pool
      RAM                guest memory in MB             512 up to node free capacity
      CPU                vCPU count                     1 up to node free capacity
      DISK               guest disk in GB               5 up to node free capacity
      VNC_PASS           console password               random 20 chars by default, never "admin"
      ROOT_PASS          guest root password            random 24 chars by default, never "admin"
      -v                 persistent data volume         ariznodes-vm-data-<short_id>:/vm
      --restart          restart policy                 unless-stopped
      image              VPS image                      allow-listed, pinned by digest

12.4  The values RAM=430080 (420 GB), CPU=24 and DISK=2048 (2 TB) are an example of a very large plan. Never hardcode them; the capacity check must reject any request the node cannot fit.
12.5  Use the Docker Engine API or SDK (dockerode, docker-py or the Go SDK), not shell strings. The command preview in the UI is generated from the same parameters for display only.
12.6  Privilege profile: implement a configurable privilege mode. Default to a reduced profile (--device /dev/kvm, --device /dev/net/tun, only the capabilities needed). Allow --privileged per image or plan for compatibility, with a clear warning in the UI.
12.7  Secrets: never place passwords on a command line. Use a 0600 env-file on tmpfs or Docker secrets; the image supports VNC_PASS_FILE and ROOT_PASS_FILE. Note that `docker inspect` exposes plain environment variables, so avoid them for secrets where possible.
12.8  Capacity accounting: allocatable = total x overcommit ratio, minus host reserve (2 cores, 4 GB RAM). Keep 10 percent disk free. Reject with 409 and a list of reasons.
12.9  Port allocation is transactional with unique (node_id, port) constraints and retries on conflict. Pools are configurable per node.
12.10 State machine: pending -> provisioning -> active <-> stopped; active -> suspended; any -> failed; active or stopped -> deleting -> deleted. Validate transitions in one function.
12.11 Provisioning steps, each with a rollback: reserve resources, allocate ports, create network, create volume, create container, start, health check (SSH banner and console reachable within 120 s), mark active.
12.12 Idempotency: creation requires an Idempotency-Key header; jobs are safe to retry; agent commands are idempotent per (vps_id, command_id).
12.13 Runtime limits: container memory limit = RAM + 512 MB overhead, CPU limit = CPU, pids-limit 4096, json-file logs capped at 10 MB x 3, restart unless-stopped.
12.14 Delete is a soft delete with a 7 day retention window; the volume is purged only after retention or on explicit admin purge.
12.15 Reinstall: stop, recreate the volume, re-provision with the same ports. Requires the typed VPS name plus TOTP (SECTION 16.4), or an admin.

---

## SECTION 13 — VPS IMAGE: ariznodes/arz-ubuntu24

13.1  Build from ubuntu:24.04. Install qemu-system-x86, qemu-utils, ovmf, novnc, websockify, openssh-client, cloud-image-utils, iproute2, procps, curl and tini.
13.2  entrypoint.sh (set -euo pipefail) reads RAM (MB), CPU (vCPUs), DISK (GB), VNC_PASS, ROOT_PASS and the *_FILE variants for secrets.
13.3  Validate every variable (integer ranges, password length) and exit with a clear message on bad input.
13.4  Create /vm/disk.qcow2 sparse at DISK GB on first boot only. Keep all state under /vm so the volume survives container recreation.
13.5  Bake in or download an Ubuntu 24.04 cloud image, verify its SHA-256 and seed it with cloud-init (root password from ROOT_PASS, SSH enabled).
13.6  Boot QEMU with -enable-kvm -cpu host -smp $CPU -m $RAM, machine q35, virtio disk and NIC, host forward tcp::2222-:22, and VNC bound to a local socket or 127.0.0.1 only.
13.7  Run websockify and noVNC on port 6080, protected by VNC_PASS, proxying to the local VNC endpoint.
13.8  If /dev/kvm is missing, exit with an explicit error. Allow TCG fallback only when ALLOW_TCG=1 and log a slowness warning.
13.9  Use tini as PID 1. On SIGTERM send an ACPI powerdown through the QEMU monitor, wait up to 60 seconds, then quit QEMU.
13.10 HEALTHCHECK: QEMU process alive, port 6080 answering and the guest SSH banner readable on 2222. Report "starting" for the first 120 seconds.
13.11 Never print passwords to logs. Unset secret variables after reading them.
13.12 Publish versioned tags, scan with Trivy in the build workflow and pin by digest in production.
13.13 Include a README documenting variables, ports, volumes and how to run the image standalone.

---

## SECTION 14 — CONSOLE, SSH AND NETWORKING

14.1  The noVNC port is never exposed publicly. Bind it to 127.0.0.1 on the node and reach it only through the authenticated console proxy.
14.2  Console flow: user clicks "Console" -> Panel issues a 60 second, single-use token bound to user, VPS and IP -> browser opens a WebSocket to the node through the tunnel -> the agent validates the token and proxies to the container.
14.3  SSH uses a host port from the SSH pool (example: 2026 -> 2222). The VPS page shows "ssh root@<host> -p <port>" with a copy button.
14.4  Important: Cloudflare Tunnel proxies HTTP(S) by default, not raw TCP. For SSH use the node's public IP, or Cloudflare Access with "cloudflared access tcp", or another documented method. The panel shows correct instructions per node based on an ssh_access_mode setting.
14.5  Firewall (nftables) on every node: default deny inbound except the SSH pool. The agent needs no inbound port because it dials out through the tunnel.
14.6  Outbound controls per VPS: block SMTP (25, 465, 587) by default, block the metadata address 169.254.169.254, cap connection rate and packet rate, optional bandwidth cap with tc.
14.7  No inter-VPS traffic: one isolated bridge network per VPS and no forwarding between bridges.
14.8  IPv6 is optional. If disabled, disable it inside the container network so it cannot bypass egress rules.
14.9  Record all port allocations in the database; the unique (node_id, port) constraint prevents conflicts.

---

## SECTION 15 — ADMIN DASHBOARD

15.0  Layout: left sidebar with the six sections below, top bar with node selector, global search, alerts bell and session menu. Every table supports search, filters, sort, server-side pagination, CSV/JSON export and bulk actions.

15.1  BIAYP — Basic Information About Your Panel
      - Panel version, build hash, update check against the release feed, changelog link.
      - Ariz key status: masked key, series, activations, expiry, last verification, mode (normal or restricted).
      - Every API in use with live health, latency and last error: key portal, Cloudflare API, SMTP, Docker on each node, HIBP, Turnstile.
      - Online servers: X of Y nodes online, leader node highlighted.
      - Users: total, active, pending, locked, banned and blacklisted (IP, device, email domain) counts.
      - VPS: total, running, stopped, suspended, failed, plus allocation versus capacity gauges.
      - Recent security events: last 20 bans, lockouts, blocked devices and abuse detections.

15.2  User Control
      - Users table: username, email, role, status, VPS count, last login, IP, abuse score.
      - User detail tabs: Overview, VPS, Sessions and Devices, Login History, Linked Accounts, Audit Trail.
      - Ban: required reason, duration (temporary or permanent), optional IP and device ban, optional VPS suspension, notify toggle. Revokes all sessions.
      - Unban: restores access. VPS resume only after the admin confirms.
      - Leaked-password check ("leak pass"): flags accounts whose password appears in known breach data, checked at login through HIBP k-anonymity, and sets password_leaked_at. The admin can force a reset for one user or all flagged users. Admins never see plaintext passwords.
      - Ban all alts: builds a candidate list of linked accounts from shared devices, IP history and behavioral signals with a confidence score. The admin reviews, deselects false positives and confirms. All selected accounts are banned in one audited, reversible transaction.
      - Crypto-mining check: per-VPS mining score with evidence (CPU pattern, pool connections). Actions: throttle, suspend, terminate, dismiss.
      - VPS status per user: power state, node, resources, uptime, recent events and start, stop, restart, suspend, delete buttons.
      - Impersonation (MAY): read-only "view as user", requires a reason, shows a red banner and is fully audited.

15.3  Status
      - Live board of every VPS, website and node (vms1.ariznodes.com, vms2.ariznodes.com, ...) with state (online, degraded, offline), latency, CPU, RAM, disk, network, agent version and last heartbeat.
      - 24 hour and 30 day uptime percentages with sparkline charts.
      - Incident timeline, auto-created when a node misses 3 heartbeats, with acknowledge and resolve actions.
      - Alert rules: node offline, disk above 90 percent, RAM above 95 percent, failed provisioning, mining detected. Channels: email and webhook (MAY: Discord, Telegram).
      - Optional public status page (MAY), toggled in Settings.

15.4  All Live Control
      - vms1 is the leader. A node selector lets the admin choose which node to configure.
      - Config surface: overcommit ratios, port pools, allowed images, maintenance mode, agent log level, firewall profile, heartbeat interval.
      - Node actions: drain, undrain, restart agent, update agent, rotate node keys, promote to leader (double confirm).
      - Every change shows a diff preview, requires confirmation, is pushed through the leader with per-node results (success, failed, pending) and supports one-click rollback.
      - Config versions are stored in the database with author and timestamp.

15.5  VPS Creation
      - Form: owner (searchable user picker), node (auto or manual), plan template or custom CPU, RAM and DISK, image, name, port assignment (auto), password mode (auto-generate by default).
      - Live capacity check against the chosen node before submit, with clear errors when the request cannot fit.
      - A "Command preview" panel shows the exact docker run equivalent (SECTION 12.2) with secrets redacted.
      - Submit creates the signed package (SECTION 3.2) and shows a live progress stepper fed by SSE.
      - On completion the admin sees the connection details once and can send them to the owner.
      - Bulk creation from CSV (MAY).

15.6  Settings
      - General: site name, logo, support email, base domain, signup mode.
      - Email: SMTP host, port, user, password, TLS, sender and a "Send test email" button.
      - Cloudflare: account id, zone id, API token (encrypted), tunnel naming pattern and "Test connection".
      - Security: attempt limit, lockout schedule, code TTLs, reverify cadence and window, session lifetimes, Turnstile keys.
      - Quotas and defaults: per-user VPS limit, default plans, port pools, overcommit ratios.
      - Licensing: view or replace the Ariz key, "Re-verify now".
      - Admins: list, invite, remove, force 2FA reset (needs a second admin).
      - Backups: schedule, retention, run now, download.
      - Danger zone: maintenance mode, rotate secrets, export all data.

---

## SECTION 16 — NORMAL USER DASHBOARD

16.1  Users can NOT create VPS. There is no create button, the API returns 403 and tests prove it.
16.2  Home: a card per assigned VPS with status badge, node, CPU, RAM and disk gauges and quick actions.
16.3  VPS detail: overview, live charts (CPU, RAM, disk, network), power controls (start, stop, restart), web console, SSH command with copy button, events timeline.
16.4  Reinstall is destructive: require typing the VPS name plus a TOTP code, rate limited.
16.5  Show connection details once at creation. Afterwards offer "Reset root password" (TOTP required) instead of revealing it.
16.6  Notifications center: in-app list with unread count and email preferences for non-security messages.
16.7  Account page: change password, manage TOTP and backup codes, view and revoke sessions and devices, verification countdown banner.
16.8  Request center: a form to request a new VPS or more resources; admins see requests in User Control.
16.9  Every list is scoped to the owner in SQL (WHERE owner_id = $1). Never rely on hiding UI elements.

---

## SECTION 17 — ABUSE PREVENTION AND CRYPTO-MINING DETECTION

17.1  Signals are collected per VPS on the node (cgroup stats, conntrack, DNS logs). Never inspect guest disk contents.
17.2  Mining heuristics, each adding to a score from 0 to 100:
      - CPU above 90 percent continuously for 30 minutes or more across the allocated vCPUs.
      - Connections to common stratum ports (3333, 4444, 5555, 7777, 14444) or stratum protocol signatures.
      - DNS lookups matching a maintained blocklist of mining pools.
      - Flat, non-bursty CPU pattern with very low disk and inbound network activity.
17.3  Thresholds: 40 = warning email, 60 = CPU throttle, 80 = suspend and alert the admin. The admin reviews evidence and can dismiss (false positive), terminate or ban.
17.4  Other abuse types: outbound spam attempts, port scanning, outbound DDoS (packet-rate spikes) and open proxy patterns. Each has its own rule, evidence record and default action.
17.5  Every detection stores an abuse_events row with score, evidence JSON, action taken and reviewer.
17.6  Users can appeal a suspension with a short form; appeals appear in User Control.
17.7  Blocklists update daily from a configurable source; failures are logged and alerted.
17.8  All automated actions are reversible and explained to the user in the notification.

---

## SECTION 18 — NOTIFICATIONS AND EMAIL

18.1  Channels: in-app (delivered live over SSE), email, and webhooks (admin only).
18.2  Events that notify: signup verification, login code, account lock, reverify request, device blocked, password reset, VPS created, VPS failed, VPS suspended, ban and unban, node offline, mining detected, appeal received.
18.3  Templates live in a templates folder as HTML plus plain text with i18n keys. Never build email HTML by string concatenation of user input; escape every variable.
18.4  Security emails (codes, locks, blocks, resets) are always sent. Marketing-style messages do not exist in v1.
18.5  Sending goes through the job queue with retries, exponential backoff and a dead-letter queue visible to admins.
18.6  Document SPF, DKIM and DMARC setup in docs/email.md, and test delivery to Gmail and Outlook inboxes.
18.7  Include the request IP, approximate location and device in every security email so users can spot misuse.
18.8  Rate limit outbound mail per user and per IP so the panel cannot be abused as a spam relay.

---

## SECTION 19 — API CATALOG (prefix /api/v1, JSON, zod-validated, OpenAPI generated)

19.1  Auth (public unless marked auth)
      POST    /auth/signup                    create account, send verification code
      POST    /auth/verify-email              confirm signup code
      POST    /auth/login                     step 1: credentials -> login challenge
      POST    /auth/login/totp                step 2: TOTP or backup code
      POST    /auth/login/email-code          step 3: emailed code -> session
      POST    /auth/email-code/resend         resend with cooldown
      POST    /auth/refresh                   rotate refresh token
      POST    /auth/logout                    end current session
      POST    /auth/logout-all                end every session
      POST    /auth/password/forgot           request reset link
      POST    /auth/password/reset            complete reset (TOTP required)
      POST    /auth/totp/enroll               begin TOTP setup (auth)
      POST    /auth/totp/confirm              confirm TOTP setup (auth)
      POST    /auth/reverify                  submit weekly verification code (auth)
      GET     /auth/sessions                  list sessions and devices (auth)
      DELETE  /auth/sessions/:id              revoke one (auth)

19.2  Account (auth)
      GET     /me                             profile
      PATCH   /me                             update profile
      POST    /me/password                    change password (TOTP required)
      GET     /me/notifications               list
      POST    /me/notifications/:id/read      mark read

19.3  VPS (user, owner-scoped)
      GET     /vps                            list own VPS
      GET     /vps/:id                        detail
      POST    /vps/:id/power                  body {action: start|stop|restart}
      POST    /vps/:id/console-token          60 second single-use token
      POST    /vps/:id/reinstall              destructive, TOTP and name confirmation
      POST    /vps/:id/reset-password         TOTP required
      GET     /vps/:id/metrics                time series
      GET     /vps/:id/events                 timeline
      GET     /vps/stream                     SSE live updates
      POST    /requests                       ask admins for a VPS or more resources

19.4  Admin (role admin only)
      GET     /admin/overview                 BIAYP data
      GET     /admin/users                    list, filter, paginate
      GET     /admin/users/:id                detail
      POST    /admin/users/:id/ban            ban with reason and duration
      POST    /admin/users/:id/unban          unban
      POST    /admin/users/:id/force-reset    force password reset
      POST    /admin/users/leak-check         run leaked-password scan
      GET     /admin/users/:id/links          linked-account candidates
      POST    /admin/users/ban-alts           confirm and ban selected linked accounts
      GET     /admin/devices                  list devices
      POST    /admin/devices/:id/unblock      lift a device block
      GET     /admin/vps                      all VPS
      POST    /admin/vps                      create (Idempotency-Key required)
      PATCH   /admin/vps/:id                  assign owner, change plan
      POST    /admin/vps/:id/suspend          suspend
      POST    /admin/vps/:id/unsuspend        resume
      DELETE  /admin/vps/:id                  soft delete
      GET     /admin/nodes                    list
      POST    /admin/nodes                    create node and enrollment command
      POST    /admin/nodes/:id/enroll-token   regenerate single-use token
      POST    /admin/nodes/:id/drain          drain or undrain
      POST    /admin/nodes/:id/promote        make leader
      DELETE  /admin/nodes/:id                remove node
      GET     /admin/status                   live status board
      GET     /admin/live-control/:nodeId     read config
      POST    /admin/live-control/:nodeId     apply config with diff token
      POST    /admin/live-control/:nodeId/rollback
      GET     /admin/abuse-events             list detections
      POST    /admin/abuse-events/:id/resolve
      GET     /admin/audit-logs               search audit trail
      GET     /admin/settings                 read settings
      PUT     /admin/settings                 update settings

19.5  License (public until activated, then admin)
      GET     /license/status                 current license mode
      POST    /license/activate               submit the Ariz key

19.6  Agent (signed requests only)
      POST    /agent/v1/enroll                one-time enrollment
      POST    /agent/v1/heartbeat             metrics every 15 seconds
      POST    /agent/v1/events                progress and abuse events
      WS      /agent/v1/commands              signed command channel

19.7  Error shape for every failure: {code, message, requestId}. Never leak stack traces or SQL errors.
19.8  Rate limits: auth endpoints 10 per minute per IP, general 120 per minute per user, admin 300 per minute. Respond 429 with Retry-After.
19.9  Pagination: cursor-based with limit (default 25, max 100). Sorting and filtering fields are allow-listed.

---

## SECTION 20 — SECURITY REQUIREMENTS (OWASP TOP 10 MAPPING)

20.1  A01 Broken access control: deny by default, role middleware plus owner-scoped queries, IDOR tests on every :id route.
20.2  A02 Cryptographic failures: TLS only, HSTS, AES-256-GCM at rest, Argon2id, Ed25519, no custom crypto, secrets from env or KMS.
20.3  A03 Injection: parameterized SQL, schema-validated input, no shell interpolation, output encoding in the UI.
20.4  A04 Insecure design: threat model in docs/, abuse cases for login, provisioning and node enrollment reviewed each phase.
20.5  A05 Security misconfiguration: non-root hardened images, read-only filesystems where possible, CSP, no default credentials, production config checks at startup.
20.6  A06 Vulnerable components: lockfile, Renovate or Dependabot, npm audit and Trivy in CI, fail the build on high severity.
20.7  A07 Authentication failures: SECTIONS 7 and 8, breached-password checks, mandatory MFA, session rotation.
20.8  A08 Integrity failures: signed node commands, signed license tokens, checksum-verified installers, signed release artifacts.
20.9  A09 Logging failures: structured audit trail, alerts on auth anomalies, tamper-evident logs (hash chain, MAY).
20.10 A10 SSRF: allow-list outbound hosts (Cloudflare, key portal, SMTP, HIBP). Block private ranges for any user-influenced URL such as webhooks.
20.11 Security headers: Content-Security-Policy with nonces and no inline scripts, X-Content-Type-Options, X-Frame-Options DENY, Referrer-Policy, Permissions-Policy, COOP and CORP.
20.12 CORS: exact origin allow-list; credentials only for the panel origin.
20.13 Request limits: JSON body 100 KB by default, strict content types, request timeouts.
20.14 Secrets never live in git. Provide .env.example only and run gitleaks as a pre-commit hook and in CI.
20.15 Backups are encrypted (age or GPG), have a tested restore script and documented RPO and RTO.
20.16 Node hardening guide: SSH keys only, unattended security updates, Docker daemon never exposed on TCP, agent runs with least privilege.
20.17 Threat: a compromised VPS attempts host escape. Mitigate with the reduced privilege profile, isolated networks, resource limits, dedicated nodes for untrusted users and fast patching.
20.18 Threat: a stolen enrollment command. Mitigate with single use, 15 minute expiry, node binding and audit alerts on failed enrollments.
20.19 Threat: a malicious or compromised admin. Mitigate with TOTP re-entry on destructive actions, complete audit logs and second-admin approval for 2FA resets.

---

## SECTION 21 — OBSERVABILITY

21.1  Structured JSON logs (pino) with request IDs and automatic redaction of secrets, tokens, codes and passwords.
21.2  Health endpoints: /healthz (process alive) and /readyz (database, Redis and queue reachable).
21.3  Prometheus metrics at /metrics, protected by a token or bound to an internal interface: request latency, error rates, queue depth, job failures, node heartbeat age, login failures.
21.4  Optional OpenTelemetry tracing across panel-api, agent calls and jobs (MAY).
21.5  Audit logs are append-only, searchable by actor, target, action and date, and exportable.
21.6  Retention: logs 30 days, metrics 14 days (VPS) and 30 days (nodes), audit logs 1 year, all configurable.
21.7  Ship Grafana dashboard JSON and alert rule examples in docs/ (MAY).

---

## SECTION 22 — TESTING AND QUALITY GATES

22.1  Unit tests: password policy, code generation and verification, lockout math, capacity accounting, state machine transitions, docker parameter builder, signature verification.
22.2  Integration tests with Testcontainers (Postgres and Redis): full login flow, weekly reverification scheduler, provisioning job with a fake agent, license gate.
22.3  End-to-end tests with Playwright: signup, TOTP enrollment, three-step login, admin creates VPS, user sees it, user cannot create a VPS.
22.4  Security tests: auth bypass attempts, step skipping in login, refresh token replay, CSRF, IDOR on every :id route, rate limit enforcement, SQL injection payloads, XSS payloads in usernames and VPS names.
22.5  Agent tests: reject unsigned, expired, replayed and tampered commands; reject unknown commands.
22.6  Load test with k6: 200 concurrent logins and 1000 status polls must stay under a p95 of 300 ms on a 2 vCPU host.
22.7  Coverage: at least 85 percent on auth, provisioning and crypto modules.
22.8  CI (optional GitHub workflow, never used to provision VPS): install, lint, typecheck, test, build, Trivy scan, gitleaks.
22.9  A change is not done until tests pass and every touched requirement number is referenced in the tests.

---

## SECTION 23 — DEPLOYMENT AND SELF-HOSTING

23.1  docker-compose.yml services: panel-api, panel-web (static files served by Caddy), postgres, redis, caddy (TLS), mailhog (dev only). The key portal ships in its own compose file.
23.2  Health checks and depends_on conditions on every service. Named volumes for postgres, redis and caddy data. No secrets inside the compose file.
23.3  .env.example documents every variable (see APPENDIX A) with safe defaults or an explicit "generate-me" marker.
23.4  scripts/dev-setup.sh generates the JWT key pair, pepper and encryption key, writes .env, starts compose, runs migrations and seeds the first admin.
23.5  First run order: the panel shows the "Enter ArizOfficial Key" page (SECTION 10), then admin login, then a guided setup: SMTP test, Cloudflare test, add the first node.
23.6  Local mode: works on a laptop with Docker. A node can be "localhost" for development with Cloudflare Tunnel disabled and a direct agent connection.
23.7  Nodes are installed with the generated enrollment command (SECTION 11.4) on any VPS, GitHub Codespace or bare-metal host that exposes KVM.
23.8  Upgrades: migrations run automatically under an advisory lock. Document the rollback procedure and always take a backup first.
23.9  Provide a Makefile with: make dev, make test, make lint, make migrate, make seed, make backup, make restore.
23.10 Release workflow (optional) publishes multi-arch images for panel-api, panel-web, node-agent, key-portal and arz-ubuntu24 with signed tags.
23.11 GitHub is used for source hosting and optional CI only. VPS are created by local Docker on nodes, never by GitHub Actions.
23.12 Resource sizing guide in docs/: 2 vCPU and 2 GB RAM is enough for the panel; node sizing depends on VPS plans.

---

## SECTION 24 — UI AND UX REQUIREMENTS

24.1  Brand: "ArizNodes HVM". Dark theme by default with a light theme toggle. A single accent color, defined as a design token.
24.2  Design tokens in Tailwind config: colors, spacing scale, radii, shadows, typography. No hardcoded hex values in components.
24.3  Layout: fixed sidebar on desktop, bottom navigation or drawer on mobile, sticky top bar. Fully responsive from 360 px up.
24.4  Every screen has designed loading (skeletons), empty and error states with a retry action.
24.5  Destructive actions use confirmation dialogs that name the target; irreversible ones require typing the name.
24.6  Accessibility: WCAG 2.1 AA contrast, full keyboard navigation, visible focus rings, ARIA labels on custom controls, reduced-motion support.
24.7  Forms: inline validation, disabled submit while pending, clear server error mapping, password visibility toggle, no autofill blocking.
24.8  Login page: centered card, product name, attempt banner, step indicator (1 Credentials, 2 Two-factor, 3 Email code), OTP boxes as in 7.2.4.
24.9  Live data: SSE-driven updates with a visible "live" indicator and graceful fallback to polling.
24.10 Charts: lightweight (uPlot or Recharts), consistent colors, tooltips, time range selector.
24.11 Toasts for success and non-blocking errors; never leak technical details or stack traces to users.
24.12 Copy: short, direct, friendly. All strings go through an i18n layer (English first).
24.13 Performance budgets: initial JS under 200 KB gzipped, route-level code splitting, Lighthouse performance above 90.
24.14 Console page: full-screen noVNC with a toolbar (ctrl-alt-del, clipboard, fullscreen, disconnect) and reconnect handling.

---

## SECTION 25 — DELIVERY PHASES

Build strictly in this order. At the end of each phase output the four items from 2.14 and wait for "continue".

25.0  PHASE 0 — Plan
      Goal   : confirm understanding and lock decisions.
      Output : architecture summary, assumptions, threat model outline, final directory tree, list of libraries with one-line justifications.
      Exit   : the user approves the plan.

25.1  PHASE 1 — Foundation
      Goal   : running skeleton.
      Output : monorepo, docker-compose, env validation, Fastify app, Postgres schema and migrations, Redis, logging, error handling, health endpoints, seed script, CI (optional).
      Exit   : "make dev" starts everything; /readyz is green; migrations and seed work.

25.2  PHASE 2 — Authentication and security core
      Goal   : SECTIONS 7, 8 and 9 fully working.
      Output : signup, email verification, TOTP enrollment, three-step login, lockouts, sessions, CSRF, device trust, weekly reverification scheduler, RBAC middleware, audit logging, login and signup UI with OTP boxes.
      Exit   : all auth and IDOR tests pass; a blocked device really is blocked.

25.3  PHASE 3 — License gate and key portal
      Goal   : SECTION 10.
      Output : key generator, key portal service and UI, verification API, signed license tokens, panel startup gate, restricted mode, BIAYP license card.
      Exit   : the panel refuses to run without a valid key and recovers when a valid key is entered.

25.4  PHASE 4 — Nodes and agent
      Goal   : SECTION 11.
      Output : Cloudflare client, add-node wizard, enrollment endpoint, installer template, node agent (enroll, heartbeat, signed command verifier), status board data.
      Exit   : a real or simulated node enrolls, appears online and rejects tampered commands.

25.5  PHASE 5 — VPS image and provisioning
      Goal   : SECTIONS 12, 13 and 14.
      Output : arz-ubuntu24 image, docker parameter builder, capacity accounting, port allocator, provisioning state machine with rollback, console proxy, egress firewall rules.
      Exit   : an admin-created VPS boots, is reachable by console and SSH, and can be stopped, started, reinstalled and deleted.

25.6  PHASE 6 — Admin dashboard
      Goal   : SECTION 15.
      Output : BIAYP, User Control (ban, unban, leak check, alt linking, device unblock), Status, All Live Control with diff and rollback, VPS Creation with command preview, Settings.
      Exit   : every admin action is audited and covered by an integration test.

25.7  PHASE 7 — User dashboard, notifications and abuse detection
      Goal   : SECTIONS 16, 17 and 18.
      Output : user dashboard, live charts, notifications, request center, mining and abuse detectors, appeals flow.
      Exit   : a user can never create a VPS; a simulated miner is throttled then suspended.

25.8  PHASE 8 — Hardening, tests and docs
      Goal   : SECTIONS 20 to 23.
      Output : security review against 20.x, load test, backup and restore scripts, threat model, runbooks, README, quick-start guide.
      Exit   : every item in SECTION 26 is checked.

---

## SECTION 26 — DEFINITION OF DONE

26.1  All MUST requirements are implemented and tested; SHOULD deviations are listed with reasons.
26.2  A new contributor can run the full platform locally with one command and the README.
26.3  No secrets, default passwords or test keys are present in the repository.
26.4  Every mutating action writes an audit log row with actor, target, before and after.
26.5  A non-admin cannot create a VPS through the UI, the API or a crafted request.
26.6  A login cannot be completed by skipping, replaying or reordering steps.
26.7  Five failed attempts lock the account and the lock is enforced across all login steps.
26.8  An unverified device is blocked exactly 24 hours after its weekly deadline and stays blocked until an admin lifts it or recovery is completed.
26.9  Node commands with a bad signature, stale timestamp or reused nonce are rejected.
26.10 The panel refuses to serve normal routes without a valid Ariz key.
26.11 Capacity checks prevent oversubscription, and port conflicts are impossible under concurrent creation.
26.12 Lint, typecheck, unit, integration and end-to-end tests pass in a clean environment.
26.13 Dependency and image scans report no high or critical findings.
26.14 Backups can be restored on a clean machine using only the documentation.

---

## SECTION 27 — ASSUMPTIONS, RISKS AND INTERPRETATIONS

27.1  "Leak pass" is interpreted as leaked-password detection with forced reset. The platform never reveals or stores plaintext passwords.
27.2  "Ban all alts" is interpreted as evidence-based account linking with mandatory admin review, because shared IPs and devices cause false positives.
27.3  "Permanent device block" is interpreted as a block on the device that only an admin or a completed recovery flow can lift, so real owners are never locked out of their account forever.
27.4  A "unique command per panel" is implemented as a per-node, single-use enrollment command plus strong runtime authentication. Secrecy of the command alone is not treated as security.
27.5  The 2FA "secret key" is entered once during enrollment. Login uses the rotating code.
27.6  Running QEMU inside Docker with --privileged weakens isolation. Prefer the reduced privilege profile and dedicated nodes for untrusted users.
27.7  GitHub Codespaces and many cloud VPS plans do not expose /dev/kvm (nested virtualization is often disabled). The installer must detect this and explain it.
27.8  Cloudflare Tunnel is HTTP(S) oriented. Raw SSH needs the node's public IP or Cloudflare Access (SECTION 14.4).
27.9  Operators are responsible for their own terms of service, abuse handling, data retention and legal compliance in their jurisdiction.
27.10 All resource figures in examples (RAM=430080, CPU=24, DISK=2048) are illustrative and must be validated against real node capacity.

---

## APPENDIX A — .env.example (generate every "generate-me" value with scripts/dev-setup.sh)

```env
# --- Core ---
APP_URL=https://panel.example.com
NODE_ENV=production
PORT=3000
LOG_LEVEL=info

# --- Data stores ---
DATABASE_URL=postgres://ariz:generate-me@postgres:5432/ariznodes
REDIS_URL=redis://redis:6379

# --- Crypto ---
JWT_PRIVATE_KEY=generate-me
JWT_PUBLIC_KEY=generate-me
PANEL_SIGNING_PRIVATE_KEY=generate-me
PEPPER=generate-me
ENCRYPTION_KEY=generate-me
ENCRYPTION_KEY_ID=1

# --- Email (Gmail, Outlook or any SMTP provider) ---
SMTP_HOST=smtp.example.com
SMTP_PORT=587
SMTP_USER=generate-me
SMTP_PASS=generate-me
SMTP_FROM="ArizNodes HVM <no-reply@example.com>"

# --- Cloudflare ---
CF_API_TOKEN=generate-me
CF_ACCOUNT_ID=generate-me
CF_ZONE_ID=generate-me
BASE_DOMAIN=ariznodes.com
NODE_PREFIX=vms

# --- Licensing ---
KEY_PORTAL_URL=https://keys.example.com
KEY_PORTAL_PUBLIC_KEY=generate-me
KEY_SERIES=#010101

# --- Bot protection ---
TURNSTILE_SITE_KEY=generate-me
TURNSTILE_SECRET=generate-me

# --- First admin (used once by the seed script) ---
FIRST_ADMIN_USERNAME=admin
FIRST_ADMIN_EMAIL=admin@example.com
FIRST_ADMIN_PASSWORD=generate-me
```

---

## APPENDIX B — ERROR CODES (stable, machine-readable)

B.1   AUTH_INVALID_CREDENTIALS       generic message, never says which field was wrong
B.2   AUTH_ACCOUNT_LOCKED            includes retryAfterSeconds
B.3   AUTH_CHALLENGE_EXPIRED         login challenge older than 5 minutes
B.4   AUTH_STEP_OUT_OF_ORDER         a login step was skipped or replayed
B.5   AUTH_CODE_INVALID              wrong or expired verification code
B.6   AUTH_CODE_RATE_LIMITED         too many attempts or resends
B.7   AUTH_TOTP_REQUIRED             action needs a fresh TOTP code
B.8   AUTH_DEVICE_BLOCKED            device failed weekly verification
B.9   AUTH_EMAIL_UNVERIFIED          signup not confirmed yet
B.10  PERM_FORBIDDEN                 role or ownership check failed
B.11  LICENSE_REQUIRED               no valid Ariz key
B.12  LICENSE_RESTRICTED             restricted mode, read-only
B.13  NODE_OFFLINE                   target node not reachable
B.14  NODE_CAPACITY_EXCEEDED         includes the list of failing resources
B.15  NODE_SIGNATURE_INVALID         bad signature, stale timestamp or reused nonce
B.16  VPS_INVALID_STATE              transition not allowed from the current state
B.17  VPS_PORT_CONFLICT              port pool exhausted or conflicting
B.18  VPS_PROVISION_FAILED           includes the failing step name
B.19  RATE_LIMITED                   generic 429 with Retry-After
B.20  VALIDATION_FAILED              includes field-level messages

---

## APPENDIX C — SIGNED VPS PACKAGE CONTRACT

C.1   The panel signs this JSON with its Ed25519 key. The agent verifies it before running anything.

```json
{
  "packageId": "uuid",
  "idempotencyKey": "string",
  "ownerId": "uuid",
  "nodeId": "uuid",
  "name": "3-40 chars",
  "image": "ariznodes/arz-ubuntu24@sha256:<digest>",
  "resources": { "cpu": 24, "ramMb": 430080, "diskGb": 2048 },
  "ports": { "ssh": 2026, "console": 6080 },
  "privilegeMode": "reduced | privileged",
  "secretsRef": "one-time secret fetch handle (never the secrets themselves)",
  "issuedAt": "ISO-8601",
  "expiresAt": "issuedAt + 5 minutes",
  "nonce": "base64, 16 random bytes",
  "signature": "base64 Ed25519 signature over the canonical JSON of all other fields"
}
```

C.2   Canonical JSON: keys sorted alphabetically, no whitespace, UTF-8, numbers without trailing zeros. Sign and verify the same bytes.
C.3   The agent rejects a package when: signature invalid, expired, nonce already seen, nodeId not its own, image not allow-listed, resources above node limits, or the panel does not confirm packageId as "pending".
C.4   Secrets are fetched by the agent over the signed channel using secretsRef, written to a tmpfs env-file with mode 0600 and deleted right after the container starts.
C.5   The agent reports each step (reserved, network, volume, container, started, healthy) as a signed event so the UI stepper can show real progress.

---

## APPENDIX D — RESPONSE FORMAT FOR EVERY PHASE

D.1   Begin with the "Phase plan" (five lines or fewer: goal, files, assumptions).
D.2   Then list the directory tree of files created or changed in this phase.
D.3   Then output every file in full, each under a header with its relative path.
D.4   Then output the four closing items: what was built, how to run it, how to test it, known limitations.
D.5   Finish with one line: "Reply 'continue' for the next phase."
D.6   Keep prose minimal. Prefer code, then a brief architectural justification only where a design choice is non-obvious.
D.7   State every assumption explicitly in one line instead of asking questions, unless the answer would change the architecture.

---

## FINAL INSTRUCTION

Start with PHASE 0 now. Do not write application code yet.
Reply with: (1) the architecture summary, (2) your assumptions, (3) the threat model outline, (4) the final directory tree, and (5) the library list with one-line justifications.
Then wait for the user to say "continue".

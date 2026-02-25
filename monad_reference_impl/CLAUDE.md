# CLAUDE.md — Monad Full Node Integration for eth2-quickstart

## What This Is

This is a one-time implementation task. You are working inside the `eth2-quickstart` repository.
Your job is to add Monad full node support alongside the existing Ethereum support.

This file is your complete specification. Do not infer, improvise, or fill gaps from general
knowledge. Every decision has already been made and is documented here. If something is not
covered, stop and ask rather than guessing.

---

## Repository You Are Working In

**eth2-quickstart** (`github.com/chimera-defi/eth2-quickstart`).

This is a two-phase server setup tool that turns a bare Ubuntu server into a running blockchain
node:

- **Phase 1** (`run_1.sh`): Runs as root. OS hardening, SSH lockdown, firewall, creates the
  operator user. Ends with a reboot.
- **Phase 2** (`run_2.sh`): Runs as the non-root operator user after reboot. Installs Ethereum
  clients (execution + consensus + MEV). This is the existing Ethereum-specific install.

**Critical constraint: do not rename, restructure, or modify `run_2.sh`.** It is tested and
working for Ethereum. You are adding Monad support alongside it, not replacing it.

### Key files you will read before writing anything

| File | Purpose |
|------|---------|
| `exports.sh` | Single source of all config variables. Sourced by every script. |
| `config/user_config.env.example` | Example user-facing config. Variables here override `exports.sh`. |
| `config/user_config.env` | Actual user config (may or may not exist). |
| `lib/common_functions.sh` | ~1000 lines of shared functions. Use these. Do not rewrite them. |
| `install/security/consolidated_security.sh` | Called by `run_1.sh`. Contains the `setup_firewall()` function you will modify. |
| `run_1.sh` | Phase 1 entrypoint. You will not modify its structure, only what `consolidated_security.sh` does. |

### Shared functions in `lib/common_functions.sh` you must use

- `log_info`, `log_warn`, `log_error` — all logging goes through these
- `require_root` / `require_non_root` — enforce execution context
- `setup_firewall_rules <port> [port...]` — opens UFW ports
- `create_systemd_service` — installs a systemd unit
- `enable_and_start_systemd_service` — enables and starts a unit
- `ensure_directory` — mkdir -p with logging
- `install_dependencies` — apt install with logging

---

## The Two-User Model

This is the most important architectural concept to understand before touching anything.

**Operator user** (`eth` by default, set via `LOGIN_UNAME` in `exports.sh`):
- Created in Phase 1 (`run_1.sh`) by `setup_secure_user()`
- This is the SSH login user the human uses after reboot
- Runs Phase 2 scripts
- Has sudo access

**Service user** (`monad`):
- A non-login system account (`useradd -r -s /sbin/nologin`)
- Created in `monad_install.sh` (Phase 2)
- The `monad-bft`, `monad-execution`, `monad-rpc` systemd services run as this user
- Owns `/home/monad/` and all node data directories
- **This is a different user from the operator user.** Do not conflate them.

The operator user (`eth`) runs `monad_install.sh`. That script then creates the `monad` service
user and installs services that run as it. This mirrors exactly how the Monad official docs
structure things.

---

## Change 1: Add `CHAIN` Variable to Config

### What to do

Add a `CHAIN` variable to `exports.sh` with a default value, and document it in
`config/user_config.env.example`.

**In `exports.sh`**, add in the "User Configuration" section near `LOGIN_UNAME`:

```bash
# Chain selection: which blockchain node to install in Phase 2
# Valid values: 'ethereum' | 'monad'
# Default is ethereum to preserve existing behaviour.
export CHAIN='ethereum'
```

**In `config/user_config.env.example`**, add a commented-out block:

```bash
# Chain selection (uncomment and set to 'monad' for Monad full node)
# Valid values: ethereum, monad
# export CHAIN='ethereum'
```

The variable must be exported so all sourced scripts can read it. No other changes to
`exports.sh` are needed.

---

## Change 2: Parameterize Firewall in `consolidated_security.sh`

### Context for this change

`install/security/consolidated_security.sh` is called by `run_1.sh`. It contains a
`setup_firewall()` function that currently hardcodes Ethereum-specific UFW rules:
- Opens: 30303 (Ethereum P2P), 13000/tcp (Prysm), 12000/udp (Prysm), SSH, 443
- Blocks: 4000, 3500, 8551, 8545 (Ethereum internal ports)

Monad requires completely different ports. The `CHAIN` variable (set in Change 1) drives
the branching.

### Monad ports — these are the ONLY correct values

Source: Official Monad documentation (https://docs.monad.xyz/node-ops/full-node-installation)

| Port | Protocol | Purpose |
|------|----------|---------|
| 8000 | TCP+UDP | Consensus P2P traffic |
| 8001 | TCP | Additional P2P |
| SSH port | TCP | Operator access (reads `YourSSHPortNumber` from exports.sh) |

**Do not use ports 26656 or 26657.** Those appear in `config/config.toml.example` in the
reference directory. They are stale Cosmos-era placeholders that were copy-pasted into that
template. They have no relation to actual Monad. The correct ports are 8000 and 8001.

### What to do

Inside `setup_firewall()` in `consolidated_security.sh`, replace the current hardcoded
port block with a chain-branching block. Keep all other logic (default deny incoming,
allow outgoing, Docker detection, private network blocks) **exactly as-is**. Only the
port-opening section changes.

The structure must be:

```bash
local SSH_PORT="${YourSSHPortNumber:-22}"

# TODO: firewall setup is no longer chain-agnostic now that multiple chains are
# supported. Consider extracting into a separate parameterized firewall phase
# (requiring sudo) in a future refactor, so Phase 1 can remain chain-agnostic.
local CHAIN_VAR="${CHAIN:-ethereum}"

if [[ "$CHAIN_VAR" == "monad" ]]; then
    log_info "Opening ports for Monad full node and SSH (port $SSH_PORT)..."
    # Monad P2P: 8000 TCP+UDP, 8001 TCP (source: docs.monad.xyz/node-ops/full-node-installation)
    setup_firewall_rules "$SSH_PORT/tcp" 8000/tcp 8001/tcp
    ufw allow 8000/udp

    # Anti-spam iptables rule recommended by official Monad docs.
    # Drops small UDP packets on port 8000 (spam mitigation).
    # WARNING: This rule does NOT persist across reboots. It is the operator's
    # responsibility to persist it (e.g. via iptables-persistent). This is
    # documented intentionally and left non-persistent per the official docs pattern.
    log_info "Applying Monad iptables anti-spam rule (non-persistent, resets on reboot)..."
    iptables -I INPUT -p udp --dport 8000 -m length --length 0:1400 -j DROP || \
        log_warn "iptables anti-spam rule failed — install iptables if missing"

    log_info "Monad firewall: opened SSH ($SSH_PORT), 8000/tcp, 8000/udp, 8001/tcp"
    log_info "Note: iptables anti-spam rule is not persistent across reboots"

elif [[ "$CHAIN_VAR" == "ethereum" ]]; then
    log_info "Opening ports for Ethereum clients and SSH (port $SSH_PORT)..."
    setup_firewall_rules 30303 13000/tcp 12000/udp "$SSH_PORT/tcp" 443/tcp
    # ... rest of existing Ethereum block unchanged ...
else
    log_error "Unknown CHAIN value: '$CHAIN_VAR'. Valid values: ethereum, monad"
    exit 1
fi
```

Preserve all log messages in the existing Ethereum branch verbatim. Do not simplify or
restructure the existing Ethereum logic — only wrap it in the `elif` branch.

---

## Change 3: Write `monad_install.sh`

This is the main deliverable. It is Phase 2 for Monad: analogous to `run_2.sh` for Ethereum.
Place it at the repository root alongside `run_1.sh` and `run_2.sh`.

### Execution context

Must run as the **non-root operator user** (with sudo available). Use `require_non_root` from
`common_functions.sh` at the top. Steps that need root elevation use `sudo` explicitly.

Source both config files at the top, same as every other script in the repo:

```bash
SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
source "$SCRIPT_DIR/exports.sh"
source "$SCRIPT_DIR/lib/common_functions.sh"
```

### Step-by-step implementation — execute in this exact order

**Step 0: Guard — CHAIN must be set to monad**

```bash
if [[ "${CHAIN:-}" != "monad" ]]; then
    log_error "This script is for Monad only. CHAIN is set to '${CHAIN:-unset}'."
    log_error "Set CHAIN='monad' in config/user_config.env and re-run."
    exit 1
fi
```

**Step 1: Guard — MONAD_TRIEDB_DRIVE must be set**

```bash
if [[ -z "${MONAD_TRIEDB_DRIVE:-}" ]]; then
    log_error "MONAD_TRIEDB_DRIVE is not set."
    log_error "This must be the path to a DEDICATED, EMPTY NVMe device (e.g. /dev/nvme1n1)."
    log_error "WARNING: The device will be completely reformatted. Do NOT set this to your OS drive."
    log_error "Set MONAD_TRIEDB_DRIVE in config/user_config.env and re-run."
    exit 1
fi
```

Also add `MONAD_TRIEDB_DRIVE` to `config/user_config.env.example` with a prominent comment:

```bash
# REQUIRED for Monad: path to a dedicated, empty NVMe device for TrieDB state storage.
# WARNING: This device will be completely reformatted. Do NOT use your OS drive.
# Use 'lsblk' to identify the correct device. Verify it has no mountpoints.
# export MONAD_TRIEDB_DRIVE='/dev/nvme1n1'
```

**Step 2: Verify drive has no mountpoints before touching it**

```bash
log_info "Verifying MONAD_TRIEDB_DRIVE=${MONAD_TRIEDB_DRIVE} is safe to format..."
if lsblk -o MOUNTPOINT "${MONAD_TRIEDB_DRIVE}" 2>/dev/null | grep -qv '^$\|MOUNTPOINT'; then
    log_error "Drive ${MONAD_TRIEDB_DRIVE} has active mountpoints. Aborting to protect data."
    log_error "Run 'lsblk -o NAME,SIZE,TYPE,MOUNTPOINT' to inspect drives."
    exit 1
fi
log_info "Drive ${MONAD_TRIEDB_DRIVE} has no mountpoints. Safe to proceed."
```

**Step 3: System update and dependencies**

```bash
sudo apt-get update -y
sudo install_dependencies curl nvme-cli aria2 jq iptables-persistent
```

Note: `iptables-persistent` is installed here so the iptables anti-spam rule set during
Phase 1 (if Monad was selected then) can be made persistent by the operator. Installing the
package does not automatically persist the rule — that is a documented manual step.

**Step 4: Add Monad APT repository and install package**

```bash
log_info "Configuring Monad APT repository..."
sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://pkg.category.xyz/keys/public-key.asc \
    | sudo gpg --dearmor --yes -o /etc/apt/keyrings/category-labs.gpg

sudo tee /etc/apt/sources.list.d/category-labs.sources > /dev/null << 'EOF'
Types: deb
URIs: https://pkg.category.xyz/
Suites: noble
Components: main
Signed-By: /etc/apt/keyrings/category-labs.gpg
EOF

sudo apt-get update -y
sudo apt-get install -y monad
sudo apt-mark hold monad
log_info "Monad package installed and held at current version."
```

`apt-mark hold` is mandatory. It prevents accidental upgrades that would break the pinned
node version. Do not remove it.

**Step 5: Create `monad` service user**

```bash
log_info "Creating monad system service user..."
if ! id -u monad &>/dev/null; then
    sudo useradd -r -m -s /bin/bash monad
    log_info "Created monad user."
else
    log_info "monad user already exists, skipping."
fi

sudo mkdir -p /home/monad/monad-bft/config \
              /home/monad/monad-bft/ledger \
              /home/monad/monad-bft/config/forkpoint \
              /home/monad/monad-bft/config/validators
sudo chown -R monad:monad /home/monad/
```

**Step 6: Configure TrieDB NVMe device**

```bash
log_info "Partitioning TrieDB device: ${MONAD_TRIEDB_DRIVE}"
log_info "THIS IS DESTRUCTIVE. The device will be completely reformatted."
sudo parted "${MONAD_TRIEDB_DRIVE}" mklabel gpt
sudo parted "${MONAD_TRIEDB_DRIVE}" mkpart triedb 0% 100%

PARTUUID=$(sudo lsblk -o PARTUUID "${MONAD_TRIEDB_DRIVE}" | tail -n 1)
log_info "TrieDB partition UUID: ${PARTUUID}"

echo "ENV{ID_PART_ENTRY_UUID}==\"${PARTUUID}\", MODE=\"0666\", SYMLINK+=\"triedb\"" \
    | sudo tee /etc/udev/rules.d/99-triedb.rules > /dev/null

sudo udevadm trigger
sudo udevadm control --reload
sudo udevadm settle

if [[ ! -e /dev/triedb ]]; then
    log_error "/dev/triedb symlink was not created. Check udev rules."
    exit 1
fi
log_info "TrieDB device configured at /dev/triedb"

log_info "Formatting TrieDB partition via monad-mpt service..."
sudo systemctl start monad-mpt
sudo journalctl -u monad-mpt -n 20 -o cat --no-pager
```

**Step 7: Download official Monad testnet config files**

```bash
MF_BUCKET="https://bucket.monadinfra.com"
log_info "Downloading Monad testnet configuration files..."
sudo curl -fsSL -o /home/monad/.env \
    "${MF_BUCKET}/config/testnet/latest/.env.example"
sudo curl -fsSL -o /home/monad/monad-bft/config/node.toml \
    "${MF_BUCKET}/config/testnet/latest/full-node-node.toml"
sudo chown monad:monad /home/monad/.env /home/monad/monad-bft/config/node.toml
log_info "Config files downloaded to /home/monad/"
```

These URLs are the official Monad infrastructure bucket. Do not substitute them.

**Step 8: Generate keystore password**

```bash
log_info "Generating keystore password..."
sudo sed -i "s|^KEYSTORE_PASSWORD=$|KEYSTORE_PASSWORD='$(openssl rand -base64 32)'|" \
    /home/monad/.env
sudo mkdir -p /opt/monad/backup/
KEYSTORE_PASS=$(sudo grep KEYSTORE_PASSWORD /home/monad/.env | cut -d= -f2)
echo "Keystore password: ${KEYSTORE_PASS}" | sudo tee /opt/monad/backup/keystore-password-backup
sudo chmod 600 /opt/monad/backup/keystore-password-backup
log_info "Keystore password generated and backed up to /opt/monad/backup/keystore-password-backup"
```

**Step 9: Generate BLS and SECP keys**

```bash
log_info "Generating monad keystore (BLS + SECP keys)..."
sudo bash << 'KEYEOF'
set -e
source /home/monad/.env
monad-keystore create \
    --key-type secp \
    --keystore-path /home/monad/monad-bft/config/id-secp \
    --password "${KEYSTORE_PASSWORD}"
monad-keystore create \
    --key-type bls \
    --keystore-path /home/monad/monad-bft/config/id-bls \
    --password "${KEYSTORE_PASSWORD}"
chown monad:monad /home/monad/monad-bft/config/id-secp \
                  /home/monad/monad-bft/config/id-bls
KEYEOF
log_info "Keystores generated."
```

**Step 10: Apply Monad performance sysctl**

These are Monad-specific kernel tuning values. They are **separate from** the security-focused
sysctl applied in Phase 1 (`run_1.sh`). They operate in different kernel namespaces and do not
conflict. Write them to a dedicated file:

```bash
log_info "Applying Monad performance sysctl settings..."
sudo tee /etc/sysctl.d/99-monad.conf > /dev/null << 'EOF'
# Monad full node kernel tuning
# Source: docs.monad.xyz + monad-bft README
# These are performance settings, separate from security sysctl in Phase 1.

# Hugepages required by monad-bft consensus process
vm.nr_hugepages = 2048

# UDP/TCP buffer sizes for consensus and RPC stability
net.core.rmem_max = 62500000
net.core.rmem_default = 62500000
net.core.wmem_max = 62500000
net.core.wmem_default = 62500000
net.ipv4.tcp_rmem = 4096 62500000 62500000
net.ipv4.tcp_wmem = 4096 62500000 62500000
EOF

sudo sysctl -p /etc/sysctl.d/99-monad.conf
log_info "Monad sysctl applied."
```

**Step 11: Install OTEL collector (metrics)**

```bash
OTEL_VERSION="0.139.0"
OTEL_PACKAGE="https://github.com/open-telemetry/opentelemetry-collector-releases/releases/download/v${OTEL_VERSION}/otelcol_${OTEL_VERSION}_linux_amd64.deb"
log_info "Installing OTEL collector v${OTEL_VERSION}..."
curl -fsSL "${OTEL_PACKAGE}" -o /tmp/otelcol_linux_amd64.deb
sudo dpkg -i /tmp/otelcol_linux_amd64.deb
sudo cp /opt/monad/scripts/otel-config.yaml /etc/otelcol/config.yaml
sudo systemctl restart otelcol
log_info "OTEL collector installed and configured."
```

**Step 12: Enable and start all Monad systemd services**

```bash
log_info "Enabling and starting Monad systemd services..."
for svc in monad-bft monad-execution monad-rpc monad-cruft otelcol; do
    enable_and_start_systemd_service "$svc"
done
```

**Step 13: Smoke check**

Wait up to 30 seconds for the RPC to come up, then query it:

```bash
log_info "Waiting for Monad RPC to become available..."
RPC_UP=false
for i in $(seq 1 6); do
    sleep 5
    RESULT=$(curl -sf -X POST http://localhost:8545 \
        -H "Content-Type: application/json" \
        -d '{"jsonrpc":"2.0","method":"eth_blockNumber","params":[],"id":1}' 2>/dev/null || true)
    if echo "${RESULT}" | grep -q '"result"'; then
        log_info "Monad RPC responded: ${RESULT}"
        RPC_UP=true
        break
    fi
    log_info "RPC not yet available (attempt $i/6)..."
done

if [[ "$RPC_UP" == "false" ]]; then
    log_warn "Monad RPC did not respond within 30 seconds."
    log_warn "This may be normal during initial sync. Check with:"
    log_warn "  journalctl -fu monad-bft"
    log_warn "  journalctl -fu monad-execution"
else
    log_info "Smoke check PASSED. Monad node is responding to RPC."
fi
```

Note: a non-response at this stage does not mean failure. The node may still be starting up
or beginning initial sync. The script should not exit 1 on RPC timeout — it logs and continues.

**Step 14: Print completion summary**

```bash
cat << EOF

=== Monad Full Node Installation Complete ===

Services installed:
  monad-bft        (consensus client)
  monad-execution  (execution client)
  monad-rpc        (RPC server)
  monad-cruft      (hourly cleanup)
  otelcol          (metrics collector)

To check status:
  sudo systemctl status monad-bft
  sudo systemctl status monad-execution
  sudo systemctl status monad-rpc

To follow logs:
  journalctl -fu monad-bft
  journalctl -fu monad-execution

To verify sync progress (block height should increase over time):
  curl -sf -X POST http://localhost:8545 \\
    -H "Content-Type: application/json" \\
    -d '{"jsonrpc":"2.0","method":"eth_blockNumber","params":[],"id":1}'

Config files:
  /home/monad/.env                        (env vars including KEYSTORE_PASSWORD)
  /home/monad/monad-bft/config/node.toml (node config)
  /opt/monad/backup/keystore-password-backup  (KEEP THIS SAFE)

EOF
```

---

## What Is Out of Scope — Do Not Implement

The following are explicitly excluded from this implementation round. Do not add them even if
they seem like good ideas or the reference scripts include them:

- Monitoring stack (Prometheus, Grafana, Loki, Alertmanager) — reference scripts have it,
  we are not building it
- Caddy reverse proxy — not needed for full node testing
- Validator key registration — requires MON tokens and VDP approval, not an infra task
- `monad-status` service (the Python HTTP status server from reference scripts) — not needed
- SSH hardening in `monad_install.sh` — already handled by `run_1.sh`
- fail2ban in `monad_install.sh` — already handled by `run_1.sh`
- Any changes to `run_2.sh` — do not touch it
- Any client selection wizard — Monad has one binary, no selection needed

---

## Reference Directory — What the Files Are and Their Caveats

The `monad_reference_impl/` directory alongside this file contains reference scripts from a
prior research phase. They are **read-only reference material**, not code to copy directly.

| File | What it is | Caveat |
|------|-----------|--------|
| `install_sysctl.sh` | Reference for sysctl values | Use values verbatim, they match official docs |
| `create_monad_user.sh` | Reference for user creation pattern | The official docs use `useradd -m` not `-r -m`; follow the official docs pattern in Step 5 above |
| `install_firewall_ufw.sh` | Reference for UFW pattern | Ports shown here are wrong — use 8000/8001 from official docs |
| `config.toml.example` | Old config template | **Port values 26656/26657 in this file are wrong.** Ignore them entirely. They are stale Cosmos placeholders. |
| `validator.env.example` | Reference for env file structure | Usable as structural reference |
| `setup_server.sh` | Reference for overall script flow | Uses old binary install pattern — follow apt install in Step 4 above instead |
| `preflight_check.sh` | Reference for preflight pattern | Can reference the check patterns |
| `check_rpc.sh` | Reference for RPC check invocation | The curl pattern is valid |
| `e2e_smoke_test.sh` | Reference for smoke test structure | Adapt the structure but use the RPC endpoint from Step 13 above |
| `RUNBOOK.md` | Operational runbook | For operator reference, not agent use |
| `DEPLOY_CHECKLIST.md` | Deploy checklist | For operator reference, not agent use |

---

## Acceptance Criteria

The implementation is complete when all of the following are true:

1. `shellcheck` passes on `monad_install.sh` (same standard as other scripts in the repo —
   run `shellcheck -x --exclude=SC2317,SC1091,SC1090,SC2034,SC2031,SC2181 monad_install.sh`)

2. `bash -n monad_install.sh` passes (syntax check)

3. `config/user_config.env.example` contains the `CHAIN` variable and `MONAD_TRIEDB_DRIVE`
   variable with the documented comments

4. `exports.sh` contains `export CHAIN='ethereum'` as default

5. `install/security/consolidated_security.sh` branches on `CHAIN`: opens 8000/8001 for monad,
   existing ports for ethereum, hard exits for unknown values

6. When deployed on a bare metal server with Ubuntu 24.04 and a dedicated NVMe drive:
   - `run_1.sh` completes without error with `CHAIN=monad` set
   - After reboot, `monad_install.sh` completes without error
   - `systemctl is-active monad-bft monad-execution monad-rpc` all return `active`
   - Block height returned by `eth_blockNumber` increases over time

---

## Coding Standards — Match the Existing Repo

Read at least the top 50 lines of `run_2.sh` and `lib/common_functions.sh` before writing any
code. Match:
- Shebang: `#!/bin/bash`
- Safety flags: `set -Eeuo pipefail` (sourced from `exports.sh`)
- All logging via `log_info` / `log_warn` / `log_error`
- No `echo` for status messages — use the logging functions
- Comments above non-obvious blocks explaining the why, not the what
# ANMA Quickstart

Get a working ANMA project in 5 minutes.

## Prerequisites

- Python 3.8+
- `pip install pyyaml`

## Step 1: Clone

```bash
git clone <this-repo> my-project
cd my-project
```

## Step 2: Verify the scaffold

```bash
python3 tools/lint_contracts.py
```

Expected output: 3 modules checked, 0 errors, 0 warnings. The repo ships with three example modules (`user-auth`, `todo-api`, `notifications`) to demonstrate the contract system.

## Step 3: Explore an existing contract

```bash
cat modules/user-auth/CONTRACT.yaml
```

Notice the structure:
- `purpose` — one sentence describing the module
- `provides` — interfaces this module exposes (with inputs, outputs, errors, invariants)
- `consumes` — interfaces this module depends on from other modules
- `contract_rules` — what changes are allowed without a breaking-change flag

Each module also has STATE.yaml (work status), MEMORY.yaml (institutional knowledge), CHANGELOG.yaml, TESTS.yaml, and ASSUMPTIONS.yaml.

## Step 4: Scaffold a new module

```bash
python3 tools/new_module.py billing --manager core-manager --consumes user-auth
```

This creates:
```
modules/billing/
  CONTRACT.yaml   # Empty template — fill this in
  STATE.yaml      # Status: idle
  MEMORY.yaml     # Empty
  CHANGELOG.yaml  # Empty
  TESTS.yaml      # Empty
  ASSUMPTIONS.yaml # Empty
  BUS/
    requests/
    deltas/
```

It also updates MANIFEST.yaml to include the new module.

## Step 5: Define your contract

Edit `modules/billing/CONTRACT.yaml`:

```yaml
module: billing
version: 1
status: draft
type: regular

purpose: "Subscription management and payment processing"

provides:
  - id: create_subscription
    input: { user_id: uuid, plan: enum(free|pro|enterprise) }
    output: { subscription_id: uuid, started_at: timestamp }
    errors: [USER_NOT_FOUND, ALREADY_SUBSCRIBED, PAYMENT_FAILED]
    invariants:
      - "free plan requires no payment method"
      - "downgrades take effect at end of billing period"

  - id: get_subscription
    input: { user_id: uuid }
    output: { subscription_id: uuid, plan: string, status: string, expires_at: timestamp }
    errors: [USER_NOT_FOUND, NO_SUBSCRIPTION]
    invariants:
      - "returns current active subscription only"

  - id: cancel_subscription
    input: { user_id: uuid, subscription_id: uuid }
    output: { cancelled_at: timestamp, effective_end: timestamp }
    errors: [SUBSCRIPTION_NOT_FOUND, ALREADY_CANCELLED]
    invariants:
      - "access continues until effective_end date"

consumes:
  - module: user-auth
    interfaces: [verify_token]

contract_rules:
  adding_interface: allowed
  modifying_interface: notify
  removing_interface: breaking
```

## Step 6: Regenerate the graph and lint

```bash
python3 tools/gen_graph.py
python3 tools/lint_contracts.py --strict
```

Zero errors, zero warnings = your contract is valid and ready for implementation.

## Step 7: Start building

Your AI agent (or a human developer) can now read `modules/billing/CONTRACT.yaml` and implement the module with complete knowledge of:
- What interfaces to build
- What inputs and outputs each expects
- What errors to handle
- What invariants must hold
- What other modules it depends on

No guessing. No reading through source code. ~250 tokens of context.

## What's Next

- [Architecture Overview](ARCHITECTURE.md) — understand the full ANMA structure
- [Contract Guide](CONTRACT-GUIDE.md) — best practices for writing contracts
- [CONTRIBUTING.md](../CONTRIBUTING.md) — how to contribute to ANMA

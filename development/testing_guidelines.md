Cracktrader Testing Guidelines
These are the official structural and naming conventions for Cracktrader’s test suite.

Goal:
Ensure that all test code is:

🔁 Idempotent (filenames, classes, and test names will not change later)

🧭 Behavioral (organized by what is tested, not how it’s implemented)

🧼 Minimal (no duplication, no testing internals directly)

📚 Self-documenting (clear from names and structure what’s being verified)

🔒 Core Principles
Structure follows interface, not implementation.

Tests go in folders based on the public-facing behavior they test (e.g. broker, order, feed), not which class defines the method.

Submodules are merged into their owning interface.

E.g. position tracking and commission routing are tested in the broker/ folder, even if implemented via mixins or helper classes.

Test organization should reflect real-world usage boundaries.

Ask: “From the user’s perspective, where would this be tested?”

No internal state manipulation.

Tests should only use public interfaces, not rely on internal attributes like broker._balances or order.status = … unless mocking the whole object.

🗂 Folder and File Layout
bash
Copy
Edit
tests/unit/
├── broker/              # Tests anything part of the broker interface
│   ├── test_broker_initialization.py
│   ├── test_broker_cash_and_value.py
│   ├── test_broker_order_submission.py
│   ├── test_broker_order_cancellation.py
│   ├── test_broker_position_tracking.py
│   ├── test_broker_commission_routing.py
│   ├── test_broker_concurrency.py
│   ├── test_broker_store_callbacks.py
│   ├── test_broker_oco_behavior.py
│   └── test_broker_factory.py

├── order/               # Tests the CCXTOrder lifecycle (parsing, updates, validation)
│   ├── test_order_state_parsing.py
│   ├── test_order_update_logic.py
│   └── test_order_submission_behavior.py

├── commission/          # Tests actual CommInfo classes and cost formulas
│   └── test_commission_info.py

├── feed/                # Tests for the data feed module
├── store/               # Tests for the store (e.g. balance and order syncing)
├── config/              # Tests for configuration manager
├── system/              # High-level integration tests

└── conftest.py
🧠 Naming Conventions
Folder Names
Lowercase, singular

Reflect the interface (e.g. broker, order, store)

File Names
Start with test_

Use flat, behavior-based names, e.g.:

✅ test_broker_order_cancellation.py

✅ test_broker_commission_routing.py

❌ test_comm_info_router_mixin.py (too implementation-focused)

Class Names
Begin with Test

CamelCase

Reflect a unit of responsibility

E.g. TestOrderLifecycleAndFills, TestPositionTracking, TestBrokerConcurrency

Test Function Names
All lowercase with underscores

Describe behavior or condition

E.g. test_order_rejected_on_insufficient_cash

🧾 Docstring Metadata (Required)
Every test function should include a docstring like:

python
Copy
Edit
"""
Cash is reduced after a filled buy order.

Target: broker
Type: unit
"""
These metadata lines allow automated test matrix generation.

Target: must be one of: broker, order, feed, store, comm_info, factory, system

Type: is usually unit, integration, or e2e

🧱 What Goes Where?
Test Category	Folder	Examples
Broker public behavior	broker/	getcash(), submit(), cancel(), getposition(), getcommissioninfo()
CCXTOrder lifecycle	order/	update(), from_exchange_dict(), order.ref
CommInfo formulas	commission/	getcommission(), getoperationcost()
Feed behavior	feed/	reset(), live reconnect, data validation
Store syncing / callbacks	store/	balance and order subscription logic
Config fallback and hierarchy	config/	file override logic, default config generation
Broker + feed + store interactions	system/	multi-component tests (without spinning up full system)

🧼 Guidelines for Adding New Tests
When writing a new test:

Ask: “Which public interface does this behavior belong to?”

Choose the matching folder (broker/order/feed/etc)

Check if an appropriate file exists; if not, create a new one that reflects the tested behavior.

Use clear test and class names that reflect the purpose.

Include docstring metadata for Target: and Type:

Use only public-facing logic — not internal variables or callbacks unless explicitly mocked.

📥 Contributor Notes
All new tests must follow this layout.

Avoid implementation leakage — don’t reference mixins, helper classes, or internal attributes in names or logic.

Use pytest.mark.parametrize() to cover multiple modes (live, back).

Keep class files flat and scoped, not overly fragmented (don’t create test_order_cancel_live.py).

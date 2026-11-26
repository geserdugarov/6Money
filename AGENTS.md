# Working on 6Money

Use the plan for the screen inventory, CSV contract, ledger rules, and build paths. Update it when implementation establishes a different behavior or resolves an open question. [Architecture and delivery plan](plans/001-iphone-architecture.md)

## Reference material

- Inspect the user's screenshots and CSV locally in `tmp/`. Keep personal names, notes, balances, images, and exports out of source, logs, issues, fixtures, and release materials. Create small fictional examples for tests and previews. A fresh checkout should build and test using tracked inputs alone.
- Use `../4money` to understand behavior. Its sources carry GPL-3.0-or-later notices. Write the Swift implementation independently and use original or appropriately licensed visual assets.

## Implementation

- Keep the ledger, CSV parsing, and SQLite operations buildable with SwiftPM on Linux. Put SwiftUI, document picking, Keychain access, and iOS lifecycle handling in the app target.
- Resolve accounts and categories by stable ledger-scoped IDs. Treat names, appearance, ordering, and archive status as user data. Preserve historical references when a user edits them.
- Keep exact decimal values, their currencies, and the original ledger day. Save both sides of a transfer in one transaction. Derive balances and reports from committed entries.
- Route every ledger mutation through the store, including imports, edits, and deletes. Commit its revision or deletion marker with the data so a future sync worker can identify changes after a restart.

## Verification

Once the package exists, run `swift test` from the root for core, import, and storage changes. Exercise money and migration changes with synthetic cases that reproduce the issue, including re-import or rollback where relevant. For app changes, run the shared iOS scheme on a Mac and exercise the affected user flow. Record which platform you verified when handing off work.

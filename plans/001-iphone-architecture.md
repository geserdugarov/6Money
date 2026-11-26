# 001 · iPhone ledger architecture and delivery

Proposed on 2026-11-26. This is the first planning milestone. The local references are nine private iPhone screenshots and one CSV export. The original app database is unavailable. The first usable release should let someone import a ledger, reconcile it, and use it every day offline. Automatic sync is a later milestone.

## What the iPhone references show

| Local reference in `tmp/` | Behavior to carry forward |
| --- | --- |
| `IMG_4923.PNG`, `IMG_4925.PNG` | Accounts shows an all-accounts total, a list with native-currency balances, distinct colors and icons, an add button, and a starred account. The account sheet offers Edit, Balance, Transactions, Recharge, Withdraw, and Transfer. |
| `IMG_4924.PNG` | Settings exposes theme, budget, startup screen, passcode, notifications, default currency, number format, first days of week and month, backup, CSV export, data deletion, and About. |
| `IMG_4926.PNG`, `IMG_4927.PNG` | Categories surrounds a spending ring with category icons and period totals. The period sheet offers all time, today, a selected day, week, month, year, and a custom range, with arrows to step periods. |
| `IMG_4928.PNG` | Transactions groups entries by day and shows the starting and ending balances for the period. Rows show an account or category, note, and amount. Search and the add button are available here. |
| `IMG_4929.PNG`, `IMG_4930.PNG`, `IMG_4931.PNG` | The entry sheet switches between Income, Expense, and Transfer. The first two show different category grids. Transfer offers accounts to choose from. |

The four tabs are Accounts, Categories, Transactions, and Overview. Only the entry points for Overview and My Finances are visible. For the first Overview, propose income, spending, and net change for the selected period, with drill-down to the matching entries. The contents of My Finances, the meaning of the star, and the detailed budget, passcode, and notification flows need further evidence. Settings and screenshots are design references. Personal labels and values belong to the user's ledger.

## What can be recovered from the CSV

The UTF-8, comma-delimited export has two tables. The first header is `DATE, TYPE, FROM ACCOUNT, TO ACCOUNT / TO CATEGORY, AMOUNT, CURRENCY, AMOUNT 2, CURRENCY 2, TAGS, NOTES`. After comma-only separator records comes `NAME, BALANCE, CURRENCY`. Read quoted CSV records and recognize these headers. Accept a UTF-8 BOM, LF or CRLF, quoted commas, escaped quotes, and embedded newlines. Keep Unicode text. The sample uses `dd.MM.yyyy` calendar days and a dot for decimals regardless of the iPhone's display format.

| Type in this export | Effect of `AMOUNT / CURRENCY` on `FROM ACCOUNT` | Meaning of the second amount and destination |
| --- | --- | --- |
| `Income` | Credit the named account | The destination label identifies the income category. Preserve `AMOUNT 2 / CURRENCY 2` for the category side. |
| `Expense` | Debit the named account | The destination label identifies the expense category. Preserve its amount and currency. |
| `Transfer` | Debit the source account | Credit the destination account with `AMOUNT 2 / CURRENCY 2`. The two amounts can have different currencies. |

The sample has 5,605 transaction rows (4,397 expenses, 219 incomes, and 989 transfers), 15 account snapshots, and six currency codes. Some converted amounts have 19 fractional digits. Tags are empty in this export. Notes include Unicode and commas. Comparing all ten fields produces 42 repeat occurrences beyond the first copy of each distinct row.

1. **Preview.** Show counts by type, currencies, proposed mappings, and row-specific errors before writing. Match accounts by label and currency, including accounts found only in the closing table. Interpret a category label of the form `Parent (Child)` as a proposed hierarchy within income or expense. Let the user correct ambiguous mappings and preserve the original label. Keep notes, tag text, both amounts, and row order within each day. Day is the finest time unit available in this file.
2. **Reconcile.** Treat the second table as closing balances in each account's own currency. Seven of the 15 balances differ from replaying the entries from zero. For a new ledger, propose an opening balance equal to the exported closing balance minus the signed account movements through the snapshot cutoff. Show the difference for confirmation. Record an accepted opening before the imported history and mark it as inferred. For an existing ledger, preview its balance including proposed rows at the confirmed cutoff and offer a dated adjustment for any remaining difference.
3. **Commit once.** The CSV has labels and row positions for identity. Assign stable IDs on import. Save the accepted rows, mappings, reconciliation entries, and import receipt in one SQLite transaction. Key the receipt by ledger ID and file digest, and identify a source row by its record ordinal within that file. A retry, including after a crash, either imports the whole accepted draft or finds its completed receipt. For a different file with overlapping history, compare normalized rows with occurrence counts and let the user resolve matches. Preserve identical purchases within a single export.

Full recovery of unused categories, appearance, display order, archive flags, preferences, and the historical exchange-rate series would require another source. Use editable defaults for missing configuration. A versioned 6Money backup should include these records and restore their IDs. Keep CSV available for interchange.

## App and package layout

Use native SwiftUI for iPhone and a SwiftPM package rooted at `Package.swift` for the portable code. Plan for iOS 17 as the initial deployment target and confirm it on the intended phone during the first device build. The proposed paths below are implementation milestones.

| Location | Responsibility |
| --- | --- |
| `Sources/LedgerCore/` | Value types, entry commands, account and category rules, period boundaries, report calculations, and the store interface. Pure Swift with portable Foundation APIs. |
| `Sources/LedgerStore/`, `Sources/CSQLite/` | SQLite implementation and a SwiftPM system-library target linking the platform's SQLite. Own schema versions, transactional migrations, foreign keys, writes, and queries. |
| `Sources/OneMoneyCSV/` | Decode the two tables and produce typed import drafts using LedgerCore. The store commits an accepted draft through the same write path used by manual entry. |
| `ios/SixMoney.xcodeproj`, `ios/SixMoney/` | Shared `SixMoney` scheme and SwiftUI app. Connect screens to the package, present Files import/export, and handle iOS lifecycle, file protection, and Keychain. |
| `Tests/` | SwiftPM tests with invented ledgers, CSV fixtures, and temporary databases. Simulator UI tests live with the iOS project. |

On Save, the view submits one command. The store commits it and publishes the change. The account list, daily history, ring, and Overview query the same committed state. Keep the selected tab, account filter, period, and unsubmitted draft in view state. Opening the app after a restart reads the ledger from SQLite.

## Persistent state and accounting rules

| Record | Fields and behavior |
| --- | --- |
| Ledger and preferences | A stable ledger UUID, reporting currency, and week/month boundaries. Device-local preferences include theme, startup tab, and formatting. Store any device-lock credentials in Keychain. |
| Account | UUID, name, native currency, icon, color, sort position, star, and archive state. Let users add and edit every account. A currency change must preserve the denomination of past entries. |
| Category | UUID, income/expense kind, optional parent ID, name, icon, color, sort position, and archive state. Support adding, renaming, reordering, and archiving parents and children. Equal names can occur under different parents or kinds. |
| Entry | UUID, ledger day, kind, account postings (account ID, direction, exact amount and currency), optional category ID and category amount, note, tags, and optional import provenance. Save an entire transfer, including edits and reversals, atomically. |
| Valuation rate | Source currency, target currency, effective day, and exact target units per source unit. Allow a user-supplied rate when the ledger needs a conversion for a report. |
| Change metadata | Ledger-scoped record IDs, local revision and change sequence, deletion markers, and a schema version. Import receipts and their mappings survive restarts. |

Represent a money value as a base-10 coefficient and scale plus currency. Store the coefficient as decimal digit text so high-precision imports survive SQLite and JSON. Perform checked decimal arithmetic in the core. SQLite numeric aggregation on decimal text can coerce it to floating point, so sum through the exact arithmetic path. Round for display or an explicitly requested conversion. Flag unsupported precision or overflow in the import preview. [SQLite aggregate functions](https://www.sqlite.org/lang_aggfunc.html)

An account balance is its opening entry plus signed movements through the requested day. A balance correction is a dated adjustment. Income and spending totals include only their corresponding categorized entries. An internal transfer moves value between accounts and contributes zero to both totals. Build every account picker and category grid from these records, so a new item is immediately available. Archiving hides it from new-entry pickers and preserves its reference in history.

Calculate the period's starting balance immediately before its first day and the ending balance through its last day. Apply the same account selection and calendar boundaries across history, categories, and Overview. For spending and income, use the recorded category-side amount when it matches the reporting currency. A combined account balance values each native balance at the applicable boundary date. If a report needs a conversion and lacks a rate, show the affected currency subtotal and identify the missing rate.

## Adding automatic sync later

On first launch, create a local ledger with a device-generated UUID. The ledger works offline and needs no login. Commit each mutation and its change metadata together, including imports and deletions. A future sync worker can read this store and use an injected transport to push changes and pull updates after a server cursor.

1. **Pair a ledger.** On opt-in, capture a consistent snapshot at local change sequence S and start queuing subsequent writes at that boundary. Seed a new remote ledger from the snapshot. An empty second device downloads a snapshot plus a server cursor. For two independently populated ledgers, offer an explicit merge before pairing. Use authenticated access scoped to the selected ledger.
2. **Send changes.** Once paired, each local write also queues its changes in an outbox in the same transaction. Give each group an immutable operation ID, affected record IDs, expected remote revisions, and a versioned payload. A large import can queue several groups in dependency order. Both transfer postings travel in one group. The server applies a group atomically and remembers its ID, so replay returns the original acknowledgement. Remove the pending group after recording the acknowledgement.
3. **Receive and resolve.** Apply incoming changes, including tombstones, together with the next cursor in one local transaction. Mark their origin as remote so the outbox queues only local edits. Use remote revisions to detect concurrent edits and edit/delete conflicts. Preserve both versions of a conflicting financial entry for user resolution as one new revision. Keep tombstones until peers have acknowledged them or require a fresh snapshot for a stale cursor.
4. **Schedule work.** Sync on foreground entry, after a save, and when connectivity returns. Persist progress and retry transient failures with backoff. Use iOS background refresh as an additional opportunity subject to system scheduling. Surface pending changes, last successful sync, and conflicts. Keep credentials in Keychain. [Apple background task guidance](https://developer.apple.com/documentation/backgroundtasks/choosing-background-strategies-for-your-app)

Choose the remote service when the sharing requirements are known. A private CloudKit database is a candidate for one person's Apple devices. If household or cross-platform sharing is needed, evaluate a service with authenticated per-ledger access. [CloudKit private database](https://developer.apple.com/documentation/cloudkit/ckcontainer/privateclouddatabase)

The Android reference combines PowerSync and Supabase. Its `OneMoneyConvert.kt`, `DbSchema.kt`, and `AtomicCrudSupabaseConnector.kt` illustrate two-sided entries and grouped writes. The converter expects Russian type labels. This export uses English ones. Use the local CSV as the format reference and write the importer independently.

## Build and delivery milestones

| Milestone | Completion check |
| --- | --- |
| 1. Save the first expense on Kubuntu | Build the root package with `swift build`. In a `swift test` integration test, create an account and a category, record an expense, close and reopen the SQLite database, and verify the balance. |
| 2. Recover the ledger on Kubuntu | Test the three entry kinds, mixed currencies, 19-place decimals, quoted fields, duplicate rows, subcategories, snapshot reconciliation, and interrupted imports using synthetic data. Privately run the import preview against the reference file and check the counts above. Confirm a second import changes nothing. |
| 3. Build and verify the iPhone screens | With access to local or hosted macOS, link the package to the SwiftUI app and run the shared scheme on an iPhone simulator. On an iPhone in airplane mode, edit accounts and categories, record and edit a cross-currency transfer, search history, and check period totals. Exercise archive, Dynamic Type, VoiceOver labels, and distinct income/expense cues. |
| 4. Prepare the local release | Round-trip a versioned backup into an empty ledger and verify balances, IDs, settings intended for backup, and history. Test schema upgrades and CSV export. Validate a signed archive and install the TestFlight build on an iPhone. |
| 5. Add optional sync | Against two local stores and a test transport, exercise offline edits, interrupted upload, duplicate delivery, pull after restart, concurrent edit/delete, and a stale device. Both devices must converge with transfer integrity and matching balances after conflict resolution. |

Run portable tests on Linux CI. For milestone 3, a hosted macOS runner can build the app and run simulator tests from a workflow triggered on Kubuntu. Use a Mac desktop for interactive Simulator sessions or a signed TestFlight build for hands-on testing on the phone. Commit the Xcode project, shared scheme, migrations, and package resolution file. Once the app target exists, list simulator destinations on macOS and substitute an available simulator ID in the test command. [GitHub-hosted runners](https://docs.github.com/en/actions/reference/runners/github-hosted-runners) · [TestFlight](https://developer.apple.com/testflight/)

```sh
xcodebuild -project ios/SixMoney.xcodeproj -scheme SixMoney -showdestinations
xcodebuild -project ios/SixMoney.xcodeproj -scheme SixMoney -destination 'platform=iOS Simulator,id=<simulator-id>' test
```

For distribution, enroll in the Apple Developer Program, choose an available product name and unique bundle ID, configure signing on the Mac, and create the matching App Store Connect record. Archive a Release build for a generic iOS device, validate and upload it through Xcode Organizer, test it with TestFlight, then select the build for App Review. Keep private signing material in the Mac keychain or the release runner's secret store. Apple's April 2026 upload minimum is Xcode 26 with the iOS 26 SDK or newer. Check the active requirement when choosing the release toolchain. The deployment target controls which phones can run that build. [Apple SDK requirements](https://developer.apple.com/news/upcoming-requirements/?id=04282026a) · [Submitting to the App Store](https://developer.apple.com/app-store/submitting/)

Create original branding, icons, store copy, and screenshots using a fictional ledger. Include a support URL, an accessible privacy policy in the app and listing, and App Privacy disclosures that reflect actual data handling and SDKs. Review required-reason API and privacy manifest obligations for the shipped build. Explain the offline import and reconciliation flow in review notes. If a later sync service introduces user registration, include in-app account deletion and revisit the disclosures. [App Review Guidelines](https://developer.apple.com/app-store/review/guidelines/) · [App Privacy Details](https://developer.apple.com/app-store/app-privacy-details/)

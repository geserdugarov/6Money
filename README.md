# 6Money

6Money is the working name for an offline personal finance app for iPhone. Its first job is to carry an existing 1Money ledger forward. You should be able to import the CSV, check each account against its exported balance, then keep recording income, expenses, and transfers. Accounts, categories, and subcategories are yours to create, rename, style, reorder, and archive.

The project is at the planning stage. The first plan maps the screens and CSV to a ledger model, defines what the export can recover, and sets milestones through TestFlight. [Implementation plan](plans/001-iphone-architecture.md) · [Repository conventions](AGENTS.md)

## Developing on Kubuntu

Kubuntu can be the main development machine. The proposed stack is a SwiftUI iPhone app backed by a portable Swift package and SQLite. You can write the app source on Kubuntu and compile and test the accounting rules, database migrations, CSV importer, and reports there. Use a Linux Swift toolchain compatible with the one in your chosen Xcode release, plus the Ubuntu packages `libsqlite3-dev` and `pkg-config`. [Swift installation on Linux](https://www.swift.org/install/linux/)

Once the first milestone adds `Package.swift`, build and test from the repository root.

```sh
swift build
swift test
```

The first two milestones run entirely on Kubuntu. They cover saving an expense, checking its balance after reopening SQLite, and importing the legacy CSV. For the native screens, Apple's iPhone Simulator runs with Xcode on macOS. A hosted macOS runner can build the app and run automated simulator tests from a workflow you trigger on Kubuntu. For hands-on testing, use a hosted Mac desktop or install a signed TestFlight build on your iPhone. TestFlight and App Store distribution require Apple Developer Program membership. [Xcode system requirements](https://developer.apple.com/xcode/system-requirements/) · [GitHub-hosted runners](https://docs.github.com/en/actions/reference/runners/github-hosted-runners) · [Apple developer membership](https://developer.apple.com/programs/whats-included/)

## Working with the references

The screenshots and CSV in `tmp/` are private local inputs. Git ignores this directory. Tests, previews, and App Store screenshots should use invented data. The neighboring Android checkout, `../4money`, provides a behavioral reference for transfers, category ordering, and offline sync. The iPhone project will use its own implementation, artwork, and product identity.

---
lang: en
---

# About This Series

**Telegram iOS Internals** is a deep-dive blog series explaining the Telegram iOS codebase — one of the most sophisticated open-source iOS applications ever built.

The series covers 274 Swift/ObjC modules, a custom MTProto protocol implementation, a custom reactive framework (SwiftSignalKit), a custom SQLite persistence layer (Postbox), a custom high-performance UI framework (AsyncDisplayKit + ComponentFlow), and the Bazel build system that ties it all together.

## Who Is This For?

Experienced iOS developers who want to understand every major architectural decision in a production-grade, performance-optimized messaging app — and potentially reuse these patterns in their own projects.

## How to Read

- **Sequential**: Posts are ordered bottom-up by dependency. Start at Post 1 and read in order.
- **Reference**: Each post is self-contained enough to look up a specific module or pattern.
- **By interest**: Jump to the part that matches what you're building (see the roadmap on the home page).

## Source Code

All code references point to paths in a local checkout of the Telegram iOS repository. Clone it to follow along:

```
git clone --recursive -j8 https://github.com/nicegram/Telegram-iOS-Old.git Telegram-iOS
```

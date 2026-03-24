# Swift Package Manager — Complete Developer Guide

> **SPM = resolver + builder + linker.**
> Everything is declared in `Package.swift` — no Xcode project file, no Ruby, no separate install.

---

## Table of Contents

1. [Core Mental Model](#1-core-mental-model)
2. [Basic Package Structure](#2-basic-package-structure)
3. [Targets vs Products — The Critical Distinction](#3-targets-vs-products--the-critical-distinction)
4. [All Target Types](#4-all-target-types)
5. [Declaring & Wiring Dependencies](#5-declaring--wiring-dependencies)
6. [Versioning — SemVer and Version Rules](#6-versioning--semver-and-version-rules)
7. [Resources — Bundling Assets with Targets](#7-resources--bundling-assets-with-targets)
8. [Platform Support & Conditional Compilation](#8-platform-support--conditional-compilation)
9. [Real-World Multi-Product Architecture](#9-real-world-multi-product-architecture)
10. [Package.resolved — The Lock File](#10-packageresolved--the-lock-file)
11. [Diamond Dependency Conflicts](#11-diamond-dependency-conflicts)
12. [Common Mistakes & How to Avoid Them](#12-common-mistakes--how-to-avoid-them)
13. [Decision Cheat Sheet](#13-decision-cheat-sheet)
14. [CLI Command Reference](#14-cli-command-reference)
15. [Final Mental Compression](#15-final-mental-compression)

---

## 1. Core Mental Model

A Swift package is three things at once: a **modular code unit**, a **dependency manager** that fetches external libraries, and a **build system** that compiles, links, and runs your code. The entire configuration lives in one file.

```
Package
 ├── Targets      (internal modules — the code)
 ├── Products     (what you expose to the outside world)
 └── Dependencies (external packages you pull in)
```

| Concept | What it is |
|---|---|
| `Package.swift` | The manifest. Declares everything — name, platforms, targets, products, dependencies. |
| **Target** | One compiled module. Maps to a folder under `Sources/`. Internal unless wrapped in a Product. |
| **Product** | The public face of your package. A library or executable that consumers can import. |
| **Dependency** | An external package fetched by Git URL or local path. Must be declared at top level **and** wired into the target that needs it. |

---

## 2. Basic Package Structure

SPM uses convention-over-configuration for folder layout. Each subfolder under `Sources/` becomes a target automatically — no explicit path declaration needed if you follow the convention.

```
MyPackage/
 ├── Package.swift          ← manifest (must be at root)
 ├── Package.resolved       ← lock file (commit for apps)
 ├── Sources/
 │    └── MyTarget/
 │         └── MyFile.swift
 └── Tests/
      └── MyTargetTests/
           └── MyTests.swift
```

### Minimal `Package.swift`

```swift
// swift-tools-version: 5.9
import PackageDescription

let package = Package(
    name: "MyLibrary",
    products: [
        .library(name: "MyLibrary", targets: ["MyLibrary"])
    ],
    dependencies: [],
    targets: [
        .target(name: "MyLibrary"),
        .testTarget(name: "MyLibraryTests",
                    dependencies: ["MyLibrary"])
    ]
)
```

> [!NOTE]
> The `swift-tools-version` comment is not decorative — SPM reads it to determine which manifest API features are available. Always set it to the lowest toolchain version your team actually uses.

---

## 3. Targets vs Products — The Critical Distinction

> **Target** = internal module (code lives here).
> **Product** = exposed API (what consumers import).
> A target not listed in any product is **invisible** to dependents.

This is the most commonly misunderstood concept in SPM. You can have ten targets and expose only two as products — the rest are private implementation details that consumers cannot import, no matter what.

### Exposing multiple targets in one product

```swift
products: [
    .library(
        name: "AppCore",
        targets: ["Core", "Networking", "Utils"]
        // All three bundled — consumer imports one module
    )
]
```

### Keeping helpers internal

```swift
products: [
    .library(name: "DesignSystem", targets: ["Components"])
    // DSUtilities is NOT listed here — consumers cannot import it
],
targets: [
    .target(name: "Components",
            dependencies: ["DSUtilities"]),  // still wired internally
    .target(name: "DSUtilities")             // private helper
]
```

---

## 4. All Target Types

### 4.1 Regular target — `.target`

```swift
.target(
    name: "Networking",
    dependencies: [
        .product(name: "Alamofire", package: "Alamofire")
    ],
    resources: [.process("Resources")]
)
```

Use for: business logic, API layer, UI modules, any reusable Swift code.

---

### 4.2 Test target — `.testTarget`

```swift
.testTarget(
    name: "NetworkingTests",
    dependencies: ["Networking"]
)
```

- One test target per main target
- Not included in the app bundle
- Run with `swift test`
- Use for unit tests, snapshot tests, regression safety

---

### 4.3 Executable target — `.executableTarget`

```swift
.executableTarget(
    name: "CodeGenTool",
    dependencies: [
        .product(name: "ArgumentParser",
                 package: "swift-argument-parser")
    ]
)
```

- Must have exactly one `@main` entry point
- Run with `swift run CodeGenTool`
- Use for CLI tools, server-side apps, build scripts

---

### 4.4 Binary target — `.binaryTarget`

```swift
// Remote — hosted as a zip with checksum verification
.binaryTarget(
    name: "AnalyticsSDK",
    url: "https://example.com/AnalyticsSDK.xcframework.zip",
    checksum: "sha256checksum..."
)

// Local — XCFramework on disk
.binaryTarget(
    name: "LocalSDK",
    path: "Frameworks/LocalSDK.xcframework"
)
```

- Distributes pre-compiled XCFrameworks
- Use for closed-source SDKs or when build-time matters
- Generate checksum: `swift package compute-checksum file.zip`

> [!TIP]
> XCFramework bundles separate slices per platform/architecture (iOS device, iOS Simulator, macOS) in a single artifact. SPM picks the right slice at link time.

---

### 4.5 System library target — `.systemLibrary`

```swift
.systemLibrary(
    name: "COpenSSL",
    pkgConfig: "openssl",
    providers: [
        .brew(["openssl"]),
        .apt(["libssl-dev"])
    ]
)
```

- Wraps a C/C++ library installed on the system
- Requires a `module.modulemap` file in the target directory
- Use for `libcurl`, `libssl`, `libxml2`, etc.

---

### 4.6 Plugin target — `.plugin`

```swift
// Build tool plugin — runs during compilation
.plugin(
    name: "SwiftGenPlugin",
    capability: .buildTool()
)

// Command plugin — run manually
.plugin(
    name: "AWSLambdaPackager",
    capability: .command(
        intent: .custom(verb: "archive",
                        description: "Build and archive")
    )
)
```

- **Build tool plugins** run during compilation and generate Swift code from other files (SwiftGen, protobuf)
- **Command plugins** are invoked manually via `swift package <verb>` or right-click in Xcode

---

## 5. Declaring & Wiring Dependencies

> [!WARNING]
> Two steps every time: (1) declare the package at top level, (2) wire the specific product into the target that needs it. Forgetting step 2 is the most common cause of "no such module" errors.

### Dependency sources

```swift
// Remote Git (most common)
.package(url: "https://github.com/Alamofire/Alamofire.git",
         from: "5.6.0"),

// Local path (for simultaneous development)
.package(path: "../LocalPackage"),

// Private repo via SSH
.package(url: "git@github.com:org/PrivateLib.git",
         from: "1.0.0"),
```

### Wiring into a target

```swift
targets: [
    .target(
        name: "MyApp",
        dependencies: [
            .product(name: "Alamofire", package: "Alamofire"),
            .product(name: "ArgumentParser",
                     package: "swift-argument-parser")
        ]
    )
]
```

### Platform-conditional dependencies

```swift
.product(
    name: "UIComponents",
    package: "UIComponents",
    condition: .when(platforms: [.iOS, .tvOS])
)
```

---

## 6. Versioning — SemVer and Version Rules

SPM uses Semantic Versioning: `MAJOR.MINOR.PATCH`. A major bump signals breaking changes. Minor adds features backward-compatibly. Patch fixes bugs only.

| Rule | Resolved range | Notes |
|---|---|---|
| `from: "5.6.0"` | `5.6.0 ..< 6.0.0` | **Recommended default** |
| `.upToNextMajor(from: "5.6.0")` | `5.6.0 ..< 6.0.0` | Same as `from:` — explicit form |
| `.upToNextMinor("5.6.0")` | `5.6.0 ..< 5.7.0` | Use when minor versions break APIs |
| `.exact("5.6.1")` | `5.6.1` only | Blocks security patches — document why |
| `"5.2.0".."5.8.0"` | `5.2.0 ..< 5.8.0` | Custom range, exclusive upper bound |
| `.branch("main")` | branch tip | Not reproducible — development only |
| `.revision("abc123")` | specific SHA | Maximum reproducibility, manual updates |

> [!TIP]
> Always use `from: "x.y.z"` (upToNextMajor) unless you have a specific documented reason not to. It gives SPM the most room to find compatible versions and prevents diamond conflicts.

---

## 7. Resources — Bundling Assets with Targets

Since swift-tools-version 5.3, targets can bundle resources. Each target gets its own `Bundle.module` at runtime — a generated accessor that SPM creates automatically.

### Two processing modes

```swift
.target(
    name: "UI",
    resources: [
        .process("Assets"),      // optimised — images compressed, Core Data compiled
        .copy("config.json")     // copied as-is, preserving folder structure
    ]
)
```

| Mode | Use for |
|---|---|
| `.process` | Images, asset catalogs, Core Data models, storyboards |
| `.copy` | JSON, fonts, raw files where you need the exact directory structure |

### Accessing resources at runtime

```swift
// SPM generates Bundle.module for each target automatically
let image = UIImage(named: "logo", in: .module, compatibleWith: nil)
let url   = Bundle.module.url(forResource: "config", withExtension: "json")
```

> [!CAUTION]
> Never use `Bundle.main` in a package target — it points to the host app bundle, not the package bundle. Always use `Bundle.module`.

---

## 8. Platform Support & Conditional Compilation

### Declaring platform minimums

```swift
platforms: [
    .iOS(.v16),
    .macOS(.v13),
    .watchOS(.v9),
    .tvOS(.v16),
    .visionOS(.v1)
]
```

All dependencies must support a deployment target equal to or lower than yours. SPM emits an error if a dependency raises the floor above what you declared.

### Conditional compilation in source

```swift
#if canImport(UIKit)
import UIKit
// iOS / tvOS specific code
#elseif canImport(AppKit)
import AppKit
// macOS specific code
#endif

#if os(Linux)
// Linux-only implementation
#endif
```

---

## 9. Real-World Multi-Product Architecture

A well-structured package exposes multiple products with clearly scoped targets. Consumers who only need tokens should not be forced to pull in SwiftUI components. The dependency chain flows in one direction only.

### Example: `DesignSystem` package

```swift
let package = Package(
    name: "DesignSystem",
    platforms: [.iOS(.v16), .macOS(.v13)],
    products: [
        // Full UI library
        .library(name: "DesignSystem",
                 targets: ["Components"]),
        // Lightweight — tokens only, no SwiftUI
        .library(name: "DesignTokens",
                 targets: ["Tokens"]),
        // Build tool — generates asset code
        .plugin(name: "SwiftGenPlugin",
                targets: ["SwiftGenPlugin"]),
    ],
    targets: [
        .target(name: "Components",
                dependencies: ["Foundations", "DSUtilities"]),
        .target(name: "Foundations",
                dependencies: ["Tokens"]),
        .target(name: "Tokens"),           // no deps — foundation layer
        .target(name: "DSUtilities"),      // internal only, NOT in products
        .plugin(name: "SwiftGenPlugin",
                capability: .buildTool()),
        .testTarget(name: "ComponentsTests",
                    dependencies: ["Components"]),
        .testTarget(name: "FoundationsTests",
                    dependencies: ["Foundations"]),
    ]
)
```

### Dependency flow

```
Components
 ├── Foundations
 │    └── Tokens        (leaf — no deps)
 └── DSUtilities        (private helper — not exposed)

Products exposed:
  DesignSystem   → Components (includes transitive: Foundations, Tokens)
  DesignTokens   → Tokens only (lightweight import)
  SwiftGenPlugin → build tool
```

### Ideal app-level architecture

```
App
 ├── Core           (shared models, protocols)
 ├── Networking     (depends on Core)
 ├── UIComponents   (depends on Core)
 ├── FeatureHome    (depends on Networking + UIComponents)
 └── FeatureProfile (depends on Networking + UIComponents)
```

Each = separate target. Only `Core`, `Networking`, `UIComponents` are products if consumed across packages. Feature targets can stay internal to the app package.

---

## 10. `Package.resolved` — The Lock File

> `Package.swift` says *"I want Alamofire between 5.0 and 6.0"* — a range.
> `Package.resolved` says *"we agreed on 5.8.1, commit `86d2c66`"* — an exact contract.

Every time SPM resolves dependencies, it writes the exact chosen versions — plus the full Git commit SHA — into `Package.resolved`. On subsequent builds, SPM reads this file and uses those exact versions without hitting the network.

### What it records per dependency

```json
{
  "pins": [
    {
      "identity": "Alamofire",
      "kind": "remoteSourceControl",
      "location": "https://github.com/Alamofire/Alamofire.git",
      "state": {
        "revision": "86d2c66f8d5e3cc3...",
        "version": "5.8.1"
      }
    }
  ]
}
```

| Field | What it means |
|---|---|
| `identity` | Canonical name SPM uses to refer to this package |
| `kind` | How it was fetched — `remoteSourceControl`, `localSourceControl`, or `registry` |
| `location` | Exact URL or path used to fetch |
| `revision` | Full 40-character Git SHA — **immutable**, the true lock |
| `version` | Human-readable SemVer tag |

> [!NOTE]
> The `revision` SHA is more trustworthy than the version tag. Git tags can be force-pushed to a different commit. The SHA cannot be changed. If you suspect a dependency has been tampered with, compare the `revision` field against the actual commit history on the repository.

### Commit it or ignore it?

| Context | Rule |
|---|---|
| **Apps & executables** | ✅ **Commit** `Package.resolved` — every developer and CI run gets identical versions. Updates are deliberate, reviewable diffs. |
| **Reusable libraries** | ❌ **Ignore** it (add to `.gitignore`) — consumers must resolve their own graph. Committing your pins silently constrains them. |

### The "works on my machine" problem

Without a committed lock file:

```
Dev A  (Monday)   → resolves Alamofire 5.8.0  ✓
Dev B  (Wednesday) → resolves Alamofire 5.8.1  ← regression shipped in between
CI     (Wednesday) → resolves Alamofire 5.8.1  ✗ build fails
```

With `Package.resolved` committed: all three get `5.8.0` until someone explicitly runs `swift package update` and reviews the diff.

### Managing the lock file

```bash
# Intentionally update — resolves fresh, overwrites Package.resolved
swift package update

# Update one specific package
swift package update Alamofire

# Restore known-good state after a bad update
git checkout Package.resolved
swift package resolve

# Full reset (clears .build and Package.resolved)
swift package reset
```

> [!WARNING]
> Never edit `Package.resolved` by hand. Always use `swift package update` to advance pins, then review the `git diff` before committing.

---

## 11. Diamond Dependency Conflicts

A diamond occurs when two packages in your graph both depend on the same third package with incompatible version constraints. SPM must find a single version satisfying both — when it can't, the build fails with a constraint graph error.

### The shape of a diamond

```
     MyApp
    /     \
  LibA   LibB
    \     /
    Alamofire

LibA requires: Alamofire >= 5.6.0
LibB requires: Alamofire .exact("5.4.1")

→ No version satisfies both. Build fails.
```

### Detection

```bash
# Map the full graph before touching anything
swift package show-dependencies

# Machine-readable for scripting
swift package show-dependencies --format json
```

SPM's error output names the conflicting package and every package demanding it. Read it bottom-up — the package at the bottom is your fix target.

```
error: Dependencies could not be resolved because
no versions of 'Alamofire' match the required versions:
  'MyApp' requires 'Alamofire' from '5.6.0'
  'LibB' requires 'Alamofire' from '5.4.0'..'5.5.0'
```

### Fix strategies — in order of preference

**1. Upgrade the transitive dependency** ← do this first

If `LibB` has a newer release that relaxes its Alamofire constraint, update it:

```swift
// Before
.package(url: "…/LibB", from: "1.0.0"),

// After — newer LibB allows Alamofire 5.6+
.package(url: "…/LibB", from: "1.4.0"),
```

**2. Relax your own constraint**

If your code doesn't actually need 5.6+, widen your constraint to overlap with LibB's range:

```swift
// Too strict
.package(url: "…/Alamofire", .upToNextMajor(from: "5.6.0")),

// Relaxed to overlap with LibB
.package(url: "…/Alamofire", .upToNextMajor(from: "5.4.0")),
```

**3. Fork / override** ← last resort

Fork `LibB`, bump its Alamofire requirement, point at your fork. Maintenance burden — switch back when upstream ships the fix. Document the reason clearly with a comment in `Package.swift`.

> [!TIP]
> Prevention: always use `.upToNextMajor`, never `.exact` on shared dependencies. Run `swift package show-dependencies` before adding any new library to spot conflicts before they break your build.

---

## 12. Common Mistakes & How to Avoid Them

### One giant target

**Problem:** Everything in a single target means every consumer compiles your entire codebase, including code they never use. Incremental builds can't skip unchanged modules.

**Fix:** Split by functional boundary — `Networking`, `UI`, `Core`, `Analytics` each get their own target.

---

### Wrong product configuration

**Problem:** Declaring a target but forgetting to include it in a product makes it completely invisible to dependent packages. They get a `no such module` error with no clear indication why.

**Fix:** Every target intended for external use must appear in a `product`.

---

### Missing `Bundle.module` for resources

**Problem:** Using `Bundle.main` in a package target points to the host app bundle. Images, JSON, and Core Data models will not be found at runtime.

**Fix:** Always use `Bundle.module`.

---

### Exact version pins on shared deps

**Problem:** `.exact("x.y.z")` on a widely-used dependency is a time bomb. Any other package requiring even a patch-level variation causes an unsolvable conflict.

**Fix:** Use `.upToNextMajor` unless you have a documented, specific reason not to.

---

### Not committing `Package.resolved` for apps

**Problem:** Without the lock file committed, every clone resolves independently. A dependency shipping a bug between two clones causes "works on my machine" failures.

**Fix:** Commit `Package.resolved` for apps, add it to `.gitignore` for libraries.

---

### Circular target dependencies

**Problem:** Target A depends on B, B depends on A. SPM enforces a directed acyclic graph and refuses to build.

**Fix:** Extract the shared type into a third target C that both A and B can import without creating a cycle.

```
Before (invalid):   A ↔ B
After (valid):      A → C ← B
```

---

### Exposing internal helpers as products

**Problem:** Listing every target in `products` makes internal implementation details public API. Consumers depend on them and you can never change them without a breaking change.

**Fix:** Only expose targets that are intentional, stable, public API.

---

## 13. Decision Cheat Sheet

### Which target type?

| Need | Target type | When to use |
|---|---|---|
| App / library logic | `.target` | Business logic, UI modules, API layer |
| Tests | `.testTarget` | Unit tests, regression safety — 1 per target |
| CLI tool | `.executableTarget` | Scripts, code-gen tools, runnable binaries |
| Pre-built SDK | `.binaryTarget` | Closed-source SDKs, faster build times |
| C / system library | `.systemLibrary` | Low-level C integration, libxml2, openssl |
| Build automation | `.plugin` | Code generation, linting, archiving |

### Which version rule?

| Rule | When to use |
|---|---|
| `from: "x.y.z"` | **Default.** Use this unless you have a reason not to. |
| `.upToNextMinor` | When minor versions have historically broken your code. |
| `.exact` | Only when a specific version is required for a known bug. Document why. |
| `.branch` | During active development on a dependency. Switch to a tag before shipping. |

### When something breaks — 90% of the time it's one of:

- ❌ **Wrong dependency declaration** — declared at package level but not wired into the target
- ❌ **Wrong product exposure** — target exists but isn't listed in products
- ❌ **Wrong target linkage** — two targets that need to share code aren't connected
- ❌ **Missing `Bundle.module`** — resource not found at runtime in a package target
- ❌ **Version conflict** — `.exact` or `.upToNextMinor` pin causes unsolvable diamond

---

## 14. CLI Command Reference

| Command | What it does |
|---|---|
| `swift build` | Build the package and all dependencies |
| `swift test` | Run all test targets |
| `swift run <Target>` | Build and run an executable target |
| `swift package resolve` | Fetch dependencies without building |
| `swift package update` | Update all deps to latest compatible versions |
| `swift package update Lib` | Update one specific package |
| `swift package clean` | Remove build artifacts from `.build/` |
| `swift package reset` | Remove `.build` and `Package.resolved` — full reset |
| `swift package purge-cache` | Clear the shared package download cache |
| `swift package show-dependencies` | Print the full dependency graph as a tree |
| `swift package show-dependencies --format json` | Machine-readable graph for scripting and diffing |
| `swift package init --type library` | Scaffold a new library package |
| `swift package init --type executable` | Scaffold a new CLI tool package |
| `swift package compute-checksum file.zip` | Generate SHA-256 checksum for a binary target zip |

---

## 15. Final Mental Compression

```
SPM = resolver + builder + linker

Target     = code
Product    = exposed module
Dependency = external code

Package.swift   = what you want
Package.resolved = what you got
```

The three questions to ask for any SPM problem:

```swift
// 1. Is the dependency declared at package level?
dependencies: [ .package(url: "...", from: "1.0.0") ]

// 2. Is the product wired into the target?
.target(name: "MyApp", dependencies: [
    .product(name: "SomeLib", package: "SomeLib")
])

// 3. Is the target exposed as a product (if it needs to be)?
products: [ .library(name: "MyApp", targets: ["MyApp"]) ]
```

### Best practices summary

- ✅ Use `upToNextMajor` for all version constraints
- ✅ Commit `Package.resolved` for apps, ignore it for libraries
- ✅ One focused target per functional boundary
- ✅ Never expose internal helpers as products
- ✅ Run `show-dependencies` before adding any new library
- ✅ Always use `Bundle.module` for resources in package targets
- ✅ Review the `git diff` of `Package.resolved` on every dependency update PR

---

*Swift Package Manager Complete Guide — covers SPM as of swift-tools-version 5.9+*

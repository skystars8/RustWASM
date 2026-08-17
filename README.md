# WebAssembly and Rust for Portable, No-Install Local Apps

> A beginner-friendly guide to what WebAssembly is, how Rust fits into it, what “portable” really means, which personal-computer apps are a good fit, and where a native program is still the better choice.

**Last technical review:** August 17, 2026  
**Audience:** Beginners who know what an application, browser, and file are, but do not need prior Rust, JavaScript, compiler, or operating-system knowledge.

---

## Contents

1. [The short answer](#the-short-answer)
2. [The problem WebAssembly solves](#the-problem-webassembly-solves)
3. [A simple mental model](#a-simple-mental-model)
4. [What a `.wasm` file contains](#what-a-wasm-file-contains)
5. [WebAssembly is not JavaScript](#webassembly-is-not-javascript)
6. [How Rust fits into WebAssembly](#how-rust-fits-into-webassembly)
7. [The host is half of the application](#the-host-is-half-of-the-application)
8. [The main ways to run a Wasm app](#the-main-ways-to-run-a-wasm-app)
9. [What “no install” and “portable” really mean](#what-no-install-and-portable-really-mean)
10. [The strengths of WebAssembly](#the-strengths-of-webassembly)
11. [The limitations of WebAssembly](#the-limitations-of-webassembly)
12. [The best types of local apps for Wasm](#the-best-types-of-local-apps-for-wasm)
13. [Apps that are a mixed fit](#apps-that-are-a-mixed-fit)
14. [Apps that are usually a poor fit](#apps-that-are-usually-a-poor-fit)
15. [A detailed example: local file encryption](#a-detailed-example-local-file-encryption)
16. [Performance: when Wasm helps and when it does not](#performance-when-wasm-helps-and-when-it-does-not)
17. [Security: what the sandbox does and does not guarantee](#security-what-the-sandbox-does-and-does-not-guarantee)
18. [Recommended architectures](#recommended-architectures)
19. [Distribution choices for instant local apps](#distribution-choices-for-instant-local-apps)
20. [A tiny Rust-to-Wasm example](#a-tiny-rust-to-wasm-example)
21. [A practical decision guide](#a-practical-decision-guide)
22. [Design and testing checklist](#design-and-testing-checklist)
23. [Common myths](#common-myths)
24. [Glossary](#glossary)
25. [Recommended learning path](#recommended-learning-path)
26. [Final recommendations](#final-recommendations)
27. [One-page recap](#one-page-recap)
28. [Official references](#official-references)

---

## The short answer

WebAssembly, usually shortened to **Wasm**, is a portable binary instruction format. A compiler—software that translates source code into another executable form—can translate code written in Rust, C, C++, and several other languages into a `.wasm` file. A compatible host—such as a web browser or a standalone Wasm runtime—then validates and executes that file.

Wasm is **not JavaScript**, and Rust compiled to Wasm does not become JavaScript. In a browser application, however, the Rust/Wasm portion lives inside the browser and normally works alongside HTML, CSS, JavaScript bindings, and browser APIs—the interfaces through which page code asks the browser to do things.

In this guide, **local processing** means the user’s data is processed on the user’s device. It does not necessarily mean the application was loaded from a local disk, works offline, or arrived through a trustworthy delivery channel. Those are separate properties.

For the kind of application considered in this guide—an individual utility that somebody opens locally on a Windows, macOS, or Linux PC—Wasm is especially attractive when all of the following are true:

- The app should open quickly without a traditional installer.
- Most work can happen inside the browser sandbox.
- The app has a web-style graphical interface.
- Its important work is computation: parsing, converting, compressing, encrypting, decoding, rendering, calculating, or simulating.
- The user can explicitly choose input and output files.
- The developer values one common application build across operating systems.

The most practical design is usually:

```text
HTML and CSS            present the interface
small JavaScript layer  connects browser events and Web APIs
Rust compiled to Wasm   performs the substantial, reusable computation
browser                 supplies the window, files, network, timers, and sandbox
```

This can feel like an instant desktop utility: open a URL, select a file, do the work locally, and save the result. It can also be cached for offline use as a Progressive Web App (PWA).

But one important correction is necessary:

> A `.wasm` file by itself is usually not a complete desktop application and is not normally something a user can double-click on every operating system.

It needs a **host**. A browser is the most widely available cross-platform host with a built-in user-interface system. A standalone Wasm runtime that implements the required WASI interfaces is another possible host, but the user must already have that runtime or receive it with the app.

### A one-paragraph recommendation

Use **browser-hosted Rust/Wasm** for portable, no-install utilities whose work can be expressed as “the user gives the app some data, the app transforms or analyzes it, and the user receives a result.” Keep the interface and browser integration in normal web technology, place CPU-heavy or reusable logic in Rust, and run heavy work in a worker so the screen stays responsive. If the app needs broad direct disk access, background services, device drivers, deep operating-system integration, or the strongest possible control over large-file I/O, build a native application or Rust command-line interface (CLI) instead. It is often best to share one Rust core between both versions.

---

## The problem WebAssembly solves

Rust, C, and C++ source is commonly compiled ahead of time into native machine code for a particular target. Browsers normally parse JavaScript and may interpret or just-in-time compile it while the app runs. In every case, the physical processor ultimately executes low-level machine instructions. A native application is commonly built for a particular combination of:

- processor family, such as x86-64 or ARM64;
- operating system, such as Windows, macOS, or Linux;
- system interfaces and binary conventions;
- available libraries and platform features.

A normal native build therefore produces different executables for different targets:

```text
Rust source
   ├──> Windows x86-64 executable
   ├──> Windows ARM64 executable
   ├──> macOS ARM64 executable
   ├──> macOS x86-64 executable
   └──> Linux x86-64 executable
```

That model provides excellent access to the operating system, but distribution has costs. The developer may have to build, sign, package, test, and update several files. Users may face installers, warnings, administrator permissions, missing libraries, or architecture mismatches.

Web technology took a different route. Developers send portable HTML, CSS, and JavaScript to a browser, and the browser adapts it to the local machine. That gives the web extraordinary reach, but developers with substantial existing Rust or C/C++ code historically could not simply use that code in a browser.

WebAssembly provides a portable compilation target that is lower-level than JavaScript and suitable for many languages. The developer compiles source code to Wasm, and the Wasm engine in the host compiles or interprets it for the current processor.

```text
Rust source
    │
    │ rustc + LLVM
    ▼
portable `.wasm` module
    │
    ├──> browser on Windows ──> local machine instructions
    ├──> browser on macOS   ──> local machine instructions
    ├──> browser on Linux   ──> local machine instructions
    └──> standalone runtime ──> local machine instructions
```

The developer ships the portable Wasm module, and each host turns it into instructions suitable for the current machine.

### What Wasm was designed to be

At its core, WebAssembly is designed to be:

- **portable:** the same module format can be supported on many processors and operating systems;
- **compact:** binary encoding is efficient to download and decode;
- **fast to validate and execute:** the format is structured for efficient compilation by the host;
- **sandboxable:** code runs in an isolated environment and reaches the outside world only through interfaces supplied by the host;
- **language-neutral:** it is a target for compilers, not a replacement source language that everybody must write by hand;
- **usable on and off the web:** browsers are a major host, but not the only possible host.

The official WebAssembly site calls it a binary instruction format for a stack-based virtual machine and a portable compilation target. “Virtual machine” here does not mean a complete emulated Windows or Linux computer. It means the host implements a well-defined abstract instruction machine.

---

## A simple mental model

Imagine an app as a skilled worker inside a locked workshop.

- The **Wasm module** is the worker and its internal tools.
- The **sandbox** is the locked workshop.
- The **host** owns every door, service hatch, and telephone.
- **Imports** are services the host agrees to provide to the worker.
- **Exports** are jobs the worker agrees to perform for the host.

The worker can calculate and modify materials already inside the workshop. It cannot wander through the building and open arbitrary filing cabinets. If it needs a file, the host must deliberately pass the file—or an approved file handle—through a door.

This is the central WebAssembly idea:

```text
                       HOST
    ┌───────────────────────────────────────────┐
    │ browser or standalone Wasm runtime        │
    │                                           │
    │  files  clock  random  network  window    │
    │     │      │      │       │       │       │
    │     └──────┴──────┴── imports ────┘       │
    │                    ▼                      │
    │           ┌──────────────────┐            │
    │           │  WASM SANDBOX    │            │
    │           │                  │            │
    │           │ compiled Rust    │            │
    │           │ functions        │            │
    │           │ linear memory    │            │
    │           └────────┬─────────┘            │
    │                    │ exports              │
    │                    ▼                      │
    │           results returned to host        │
    └───────────────────────────────────────────┘
```

This model explains both Wasm’s power and its limitations. Pure computation is highly portable. Interaction with the real computer depends on the host.

### An even shorter analogy

Think of Wasm as a **portable engine**, not a complete car.

The engine may be used in different vehicles, but it does not provide a steering wheel, doors, dashboard, road rules, or fuel station. In an app, those missing parts include the graphical user interface (GUI), filesystem, network, clipboard, notifications, and operating-system integration.

---

## What a `.wasm` file contains

The `.wasm` extension can identify either a **core Wasm module** or a **Component Model component**. They are different binary formats even though both commonly use the same extension. This subsection first describes a core module, which can contain items such as:

- compiled function bodies;
- numeric constants;
- type information;
- global values;
- one or more linear memories, depending on the module and supported features;
- tables used for indirect function calls;
- declarations of imported functions, tables, memories, globals, or tags it expects an external instance to supply;
- declarations of exported functions, tables, memories, globals, or tags an external instance may use;
- optional custom sections, such as names or debugging information.

A Component Model component can contain core modules or nested components and describes richer typed imports and exports using WIT. Those richer interfaces can include records, variants, strings, lists, and resource handles. The host must understand the correct binary kind and interfaces.

It does **not automatically contain**:

- a desktop window;
- buttons, menus, or text fields;
- the ability to open any local path;
- a universal operating-system filesystem API;
- a universal network API;
- a package installer;
- an update mechanism;
- a complete JavaScript engine;
- the Rust compiler;
- source code in its original form.

### WebAssembly Text format

Wasm also has a human-readable text representation, normally called **WAT**. A tiny function might look roughly like this:

```wat
(module
  (func (export "add") (param i32 i32) (result i32)
    local.get 0
    local.get 1
    i32.add))
```

Developers normally write Rust or another higher-level language instead of WAT. The text format is useful for teaching, inspection, testing, and debugging.

### Linear memory

Many Wasm programs use a contiguous byte array called **linear memory**. Rust data such as strings, vectors, structs, and file buffers can be represented inside it. WebAssembly checks that an access stays inside the overall linear-memory bounds, but that does not make every program logically correct.

For example, a Wasm load or store traps when it goes outside the addressed linear memory’s bounds. That memory may be defined by the module, imported from a host, exported to a host, or shared under supported conditions; it is not necessarily private “owned” memory. Wasm also does not inherently know that bytes 1,000 through 1,100 belong to one Rust object while bytes 1,101 through 1,200 belong to another. Rust’s normal type and ownership rules add another valuable layer of safety, provided the code does not misuse `unsafe` operations.

### Imports and exports

A module’s **exports** are functions, memory, tables, or globals made available to its host. An image-processing module might export a function named `resize`.

Its **imports** are typed external dependencies. A core module can import functions, tables, memories, globals, or tags. Some imported functions or handles confer capabilities—for example, a random-number source or access to an approved file—while others are ordinary computation or shared state. The host decides what an imported function actually does; Wasm validates compatible types, not the truth of a function’s name or intended semantics.

This is why portability has a condition:

> The Wasm instructions may be portable, but the module only runs when its host supplies compatible imports.

A pure mathematics module with no imports is easy to move. A module expecting one browser’s particular APIs is portable only to hosts that provide or emulate those APIs.

---

## WebAssembly is not JavaScript

This misconception is understandable because most people first encounter Wasm inside a web browser.

### JavaScript

JavaScript is a high-level, dynamically typed programming language. Browsers can parse and execute JavaScript source code. JavaScript is deeply connected to browser APIs and can manipulate the Document Object Model (DOM), respond to clicks, create elements, call `fetch`, and use many other web-platform services.

### WebAssembly

WebAssembly is a lower-level binary instruction format with a small, explicitly typed machine model. It is usually produced by a compiler. It does not directly know what an HTML button, browser tab, or local document is.

### They cooperate in a browser

A typical browser application looks like this:

```text
user clicks button
      │
      ▼
JavaScript event handler or generated binding
      │
      ▼
Rust function compiled to Wasm
      │
      ▼
result returned through the binding
      │
      ▼
JavaScript updates the HTML interface
```

Tools such as `wasm-bindgen` generate bindings that translate between browser/JavaScript values and Rust/Wasm values. The generated JavaScript may be small, but it still matters.

### Four accurate statements

1. **Wasm is not JavaScript.** It has a different format and execution model.
2. **A browser Wasm app is still a web app.** It inherits browser permissions, lifecycle, storage, and delivery concerns.
3. **Wasm can run without JavaScript.** Standalone runtimes can host Wasm directly.
4. **Wasm in a browser normally uses web APIs through bindings.** The browser remains responsible for the DOM, file picker, networking, and other platform features.

### Is Wasm merely a faster JavaScript replacement?

No. That description misses the more important benefits:

- compiling languages such as Rust into a web-compatible format;
- reusing non-JavaScript libraries;
- predictable low-level numeric and memory operations;
- portable sandboxed components;
- sharing a computational core between web, command-line, server, plugin, and embedded hosts.

Wasm is not automatically faster than well-written JavaScript. It is often strongest for sustained, CPU-heavy work using arrays, binary data, parsers, codecs, numerical algorithms, and existing native-language libraries. JavaScript can be faster or simpler for DOM manipulation, small event handlers, ordinary application coordination, and code dominated by browser API calls.

---

## How Rust fits into WebAssembly

Rust is a source programming language. WebAssembly is one target to which Rust can be compiled.

```text
Rust is what the developer writes.
Wasm is one format the compiler can produce.
The browser or runtime is what executes that format.
```

Rust is not required for Wasm. Toolchains for C, C++, Go, C#, and many other languages can produce Wasm. JavaScript ecosystems commonly host and interoperate with Wasm, while Python can participate through implementations and toolchains with different approaches and levels of support. Rust is nevertheless an especially useful pairing.

### Why Rust and Wasm work well together

#### 1. Memory safety without requiring a garbage collector

Rust’s ownership and borrowing rules catch many memory errors during compilation. This is valuable for parsers, codecs, cryptographic code, and binary data processing. Rust generally does not require a tracing garbage collector, which maps well to Wasm’s lower-level execution model.

#### 2. Strong control over data layout and allocation

Rust allows precise control when performance or binary formats matter, while still offering safe high-level abstractions such as vectors, iterators, enums, and pattern matching.

#### 3. A mature package and build ecosystem

Cargo manages Rust packages, builds, tests, features, and dependency metadata. A Rust package is commonly called a **crate**. Many crates are pure Rust and can compile to Wasm, although not every crate is compatible.

#### 4. Reusable core logic

A well-designed Rust library can often be used by several front ends:

```text
                    ┌── browser UI + Wasm adapter
shared Rust core ───┼── native Rust CLI
                    ├── native desktop wrapper
                    └── server or test harness
```

This can give users an instant browser version and a native CLI for large files or high-assurance workflows without duplicating the most important algorithm.

#### 5. Good performance for computational code

Rust compiles through LLVM to efficient Wasm instructions. This is useful for loops over large byte buffers, compression, parsing, image transforms, simulations, and similar work.

### What Rust features still work?

Much of ordinary Rust works well:

- structs, enums, traits, and generics;
- `Vec`, `String`, slices, and many collections;
- iterators and algorithms;
- most pure computation;
- many pure-Rust libraries;
- unit tests, with target-aware setup;
- explicit error types and pattern matching.

### What changes when Rust targets Wasm?

The target environment matters. A Rust program compiled for `wasm32-unknown-unknown`, commonly used for the browser, does not automatically have normal operating-system services. Official Rust documentation notes that operations such as ordinary `std::fs` access and spawning normal OS threads are not generally available on that minimal target.

Browser-specific projects commonly use:

- `wasm-bindgen` to describe Rust/JavaScript boundaries;
- `web-sys` or similar generated bindings to browser APIs;
- JavaScript or framework code to manage the interface;
- web workers for CPU-heavy work off the main interface thread;
- target-specific code selected with Rust configuration attributes.

Standalone WASI projects use a Rust target and runtime matched to the required WASI generation, which can offer standardized system-style interfaces. Browser and WASI targets are related, but they are not interchangeable application environments.

### Not every Rust crate works in Wasm

A crate may assume it can:

- open arbitrary operating-system paths;
- start native threads;
- create processes;
- use platform sockets directly;
- call C or C++ libraries that were not also built for Wasm;
- use architecture-specific assembly;
- dynamically load native libraries;
- access environment variables or system configuration;
- memory-map files.

If so, it may need feature flags, replacement dependencies, an adapter layer, or a native-only implementation.

### Rust is not a security certificate

Rust reduces important classes of memory mistakes, but an application can still have:

- incorrect cryptographic design;
- authentication or authorization errors;
- data loss bugs;
- denial-of-service problems;
- unsafe-code defects;
- dependency compromise;
- malicious update logic;
- incorrect file-format handling;
- privacy leaks through the host or interface layer.

Language choice helps; design, review, tests, and distribution are still essential.

---

## The host is half of the application

Core WebAssembly deliberately does not define a universal desktop operating system. It defines computation and an import mechanism. The **host** determines what the module can do beyond calculation.

### In a browser, the host provides

- the window and rendered interface;
- DOM APIs;
- mouse, keyboard, touch, and accessibility events;
- user-mediated file selection;
- downloads and browser storage;
- network requests subject to web security rules;
- clocks and cryptographically secure randomness;
- audio, canvas, WebGL, WebGPU, and other supported APIs;
- workers and, under the right conditions, shared-memory features;
- origin-based isolation (separation among web security identities), permissions, and lifecycle rules.

### In a standalone runtime, the host may provide

- standard input and output;
- selected directories or file handles;
- clocks and random numbers;
- network interfaces;
- environment-like configuration;
- application-specific functions;
- logging, databases, or plugin APIs.

### The host can grant less authority than the user has

This is a major security feature. A browser page does not automatically inherit the user’s complete filesystem permissions. A capability-oriented WASI host can expose one chosen directory while hiding everything else.

### The host also limits portability

Suppose a module imports a function named `read_customer_database`. It will instantiate only when an external value resolves that import with a compatible function type. Core Wasm cannot verify that the function actually reads the intended database, enforces the intended policy, or behaves honestly. The CPU instructions inside the module are portable; the surrounding typed and semantic contract is application-specific.

There are therefore at least four layers of portability:

| Layer | Question | Wasm’s contribution |
|---|---|---|
| Processor portability | Can the same module run on x86-64 and ARM64? | Strong, when the runtime supports required Wasm features. |
| OS portability | Can it run on Windows, macOS, and Linux? | Strong for computation; the host must exist on each OS. |
| Host-API portability | Are files, network, clocks, and other imports compatible? | Depends on browser APIs, WASI version, or a custom contract. |
| Experience portability | Does the app behave and integrate identically everywhere? | Not guaranteed; browsers, permissions, fonts, file dialogs, and OS conventions differ. |

“Write once, run anywhere” is most accurate for a small pure function. It becomes less absolute as an app reaches farther into the machine.

---

## The main ways to run a Wasm app

There is no single universal “Wasm app” package. Several models use WebAssembly in different ways.

### Model 1: A normal website with a Wasm engine

The user opens an HTTPS address. HTML and CSS create the interface, JavaScript loads the module, and Rust/Wasm performs important work. Files selected by the user can be processed locally without uploading them, if the app is designed that way.

**Advantages**

- Almost no user setup.
- One deployment can reach modern browsers on several operating systems.
- Updates are immediate.
- Excellent for occasional utilities.
- The browser supplies a mature GUI, accessibility model, and sandbox.

**Tradeoffs**

- The first load normally needs a network connection.
- Users must trust the code delivered on that visit.
- Browser extensions, a compromised site, or compromised dependencies can affect the app.
- Local file and device access is permission-mediated.
- The tab may be closed, suspended, refreshed, or discarded.

**Best for:** instant converters, analyzers, editors, calculators, compressors, cryptographic utilities, visualizers, and educational tools.

### Model 2: An offline-capable Progressive Web App

A PWA is still a web app. It adds a web app manifest and may use a **service worker**, an event-driven browser worker that can manage requests and cached assets. The user can often install the app into an app-like window, and the cached app can work offline.

**Advantages**

- Web reach with an optional app icon and standalone-looking window.
- Offline use after the required assets are cached.
- A single web codebase.
- Updates can be managed through the web delivery path.

**Tradeoffs**

- Installation and integration details vary by browser and OS.
- A PWA still runs under browser rules.
- Service-worker updates and caches add state that must be tested carefully.
- “Works offline” must be deliberately implemented and verified; it is not automatic.
- Browser background execution is constrained and may be stopped.

**Best for:** frequently used personal utilities that benefit from an icon and offline operation but do not need broad direct system access.

### Model 3: A downloaded static web-app folder

The developer distributes HTML, CSS, JavaScript, and `.wasm` files together. The user keeps the folder locally. This avoids depending on the developer’s server after download.

```text
portable-tool/
├── index.html
├── app.js
├── app.wasm
├── styles.css
└── assets/
```

**Advantages**

- The exact files can be archived, hashed, scanned, and kept offline.
- The same web assets can be used on several operating systems.
- If the bundle is self-contained and loads no remote code, a central server cannot silently change the downloaded bytes.

**Tradeoffs**

- Double-clicking `index.html` through a `file://` URL is not consistently reliable for module loading, fetching `.wasm`, workers, or service workers.
- A small local HTTP server is often required, which creates a bootstrapping problem for nontechnical users.
- Updating is manual unless an updater is added.
- Some browser APIs require a **secure context**, normally a trustworthy HTTPS origin; `localhost` receives special treatment, while arbitrary local-file origins do not behave like HTTPS sites.

**Best for:** technically comfortable users, controlled environments, archived tools, and high-trust offline copies.

### Model 4: A single self-contained HTML file

Build tools can sometimes embed JavaScript, CSS, images, and even encoded Wasm bytes into one HTML file. The user opens one document in a browser.

**Advantages**

- Extremely easy to copy, email, archive, or place on removable storage.
- No installer and no folder of assets.
- The file can be hashed or signed as one unit.

**Tradeoffs**

- Embedded Wasm may require special loading code.
- Browser `file://` restrictions still apply to some features.
- The file can be larger and cannot cache individual resources efficiently.
- Workers, dynamic imports, and strict security policies can become awkward.
- This packaging pattern needs cross-browser testing; it is not guaranteed merely because ordinary HTML opens.

**Best for:** small, self-contained calculators, viewers, demonstrations, and simple file transformations.

### Model 5: A standalone WASI program or artifact

WASI means **WebAssembly System Interface**. It is a family of standardized interfaces that hosts can provide to Wasm modules or components. A command-style Rust program can be compiled for a WASI target and run by a compatible runtime such as Wasmtime or another WASI implementation.

The word **artifact** is useful here because different WASI generations produce different kinds of `.wasm` binaries. WASI 0.1/Preview 1 (P1) uses core Wasm modules. WASI 0.2/Preview 2 (P2) and WASI 0.3/Preview 3 (P3) use Component Model components.

**Status snapshot for August 17, 2026:** Rust’s `wasm32-wasip1` and `wasm32-wasip2` targets are Tier 2. The former produces legacy P1 core modules; the latter produces P2 components. WASI 0.3 is a stable WASI milestone, but Rust’s `wasm32-wasip3` target remains Tier 3 without official prebuilt standard-library artifacts and may still import some P2 APIs during its transition. A newer WASI specification therefore does not automatically mean every Rust toolchain and runtime is ready for that generation. Pin and test the exact combination.

Conceptually:

```text
tool.wasm + WASI runtime + granted capabilities
```

**Advantages**

- No browser or JavaScript is inherently required.
- The same Wasm artifact may run in compatible runtimes on several operating systems.
- Capabilities can restrict access to selected files, directories, or services.
- Strong fit for plugins, command tools, automation, server functions, and embedded execution.

**Tradeoffs**

- The operating system does not normally launch `.wasm` files by itself; a runtime is required.
- WASI is evolving, and runtime/toolchain support for particular versions and interfaces must match.
- Standard desktop GUI APIs are not the central strength of WASI.
- Shipping the runtime with the app may require a different native runtime binary for each OS and processor.
- A standalone `.wasm` file is therefore not automatically a universal double-clickable desktop executable.

**Best for:** portable command-line tools, sandboxed plugins, batch processors, developer utilities, and application components.

### Model 6: Wasm embedded in a native desktop shell

A native desktop app can embed a Wasm runtime and use Wasm for plugins or business logic. Alternatively, a web-view shell can display a web interface while native Rust code supplies deeper OS integration.

**Advantages**

- Native filesystem, menus, windows, notifications, and other integrations are possible.
- Wasm components can remain portable and sandboxed inside the larger app.
- The user gets a conventional desktop experience.

**Tradeoffs**

- Native shells normally require OS-specific builds and packaging.
- It is no longer a truly universal single artifact.
- Signing, installers, updates, and platform testing return.
- The architecture is more complex.

**Best for:** polished desktop products that value Wasm internally but need stronger system integration.

### Model 7: Portable native Rust executables

This is not WebAssembly, but it belongs in the comparison. A developer can build one native Rust executable for each target and distribute a portable ZIP without an installer.

**Advantages**

- Direct, efficient OS access.
- Strong file I/O and command-line behavior.
- Easy to support large streams, atomic renames, and operating-system conventions.
- No browser lifecycle or web delivery path.

**Tradeoffs**

- Separate builds are required for Windows, macOS, Linux, and processor architectures.
- Code signing and OS warnings may affect trust and usability.
- A GUI still needs a cross-platform toolkit and platform testing.

**Best for:** high-assurance file tools, automation, system utilities, backup/sync tools, and large-file pipelines.

### Comparison table

| Packaging model | Traditional install required? | One app artifact across OSes? | GUI convenience | Local filesystem access | Offline use | Main prerequisite |
|---|---:|---:|---:|---:|---:|---|
| Hosted browser app | No | Usually yes | Excellent | User-mediated | After caching, if designed | Modern browser; network initially |
| PWA | Optional browser-level install | Usually yes | Excellent | User-mediated | Yes, if implemented | Supporting browser |
| Static local folder | No | Usually yes | Excellent | User-mediated | Yes | Browser; often local HTTP server |
| Single HTML bundle | No | Usually yes | Good | User-mediated | Yes | Compatible browser behavior |
| Standalone WASI artifact | Runtime may need installation | Potentially | Weak by default | Capability-granted | Yes | Wasm runtime with compatible WASI support |
| Native desktop shell | Often, though portable bundles exist | No; build per target | Excellent | Broad, with OS permission | Yes | Correct platform package |
| Native Rust CLI | No installer required if statically portable | No; build per target | Terminal only | Broad, with OS permission | Yes | Correct platform binary |

---

## What “no install” and “portable” really mean

These phrases are useful, but several different promises hide inside them.

### “No install” can mean

1. **Open a URL:** no application package is deliberately installed, although the browser may cache assets and retain site data.
2. **Use a PWA:** the browser performs a lightweight, optional installation.
3. **Unzip and run:** files are copied but no system installer changes the machine.
4. **Run one file:** the application is self-contained.
5. **Use an existing runtime:** the app itself is not installed, but a browser or WASI runtime already is.

The first option is the easiest for nontechnical users. The third and fourth can be better for offline ownership and version pinning.

### “Portable” can mean

1. **Portable source:** the project can be recompiled for each operating system.
2. **Portable module:** the same `.wasm` bytes run in compatible hosts.
3. **Portable package:** the same folder opens on each operating system.
4. **Portable data:** the user’s documents move cleanly between systems.
5. **Portable experience:** the UI and behavior are nearly identical everywhere.
6. **Portable on removable storage:** the app can run from a USB drive without leaving important state behind.

Wasm strongly supports the second meaning. A browser-based package often supports the third and fifth. Removable-storage portability needs extra attention because browsers may store permissions, cache data, databases, or settings in the browser profile rather than beside the application.

### There is always a runtime somewhere

No application runs without supporting software. A native executable depends on the operating system and CPU. A Java application depends on a JVM. A browser app depends on the browser. A WASI artifact depends on a Wasm runtime implementing the interfaces it imports.

Wasm makes the runtime boundary portable and explicit; it does not eliminate the boundary.

### The browser is the practical universal runtime

For consumer-facing, no-install GUI tools, the browser has a unique advantage: most users already have one, and it supplies a cross-platform interface, sandbox, network stack, fonts, accessibility tree, input handling, and graphics APIs.

This is why “Rust/Wasm app” often really means “web app with a Rust/Wasm computational core.” That is not a compromise or deception. It is an effective division of responsibilities.

---

## The strengths of WebAssembly

Wasm is not the best answer to every software problem. Its value becomes clearer when its strengths are stated precisely.

### 1. Portable computation

The same module can often execute on different processors and operating systems without recompiling the module itself. The runtime handles translation to local machine instructions.

This is most powerful when the module is mostly self-contained:

```text
input bytes ──> parse / calculate / transform ──> output bytes
```

The fewer host-specific services it assumes, the easier it is to move.

### 2. A route for Rust and other languages into the browser

Wasm allows developers to use libraries, algorithms, and programming-language features that are not JavaScript. A mature Rust parser or codec may be adapted for browser use without rewriting its entire core in TypeScript.

This reduces duplicated logic and can keep behavior consistent among browser, native, and server versions.

### 3. Strong sandboxing boundary

A Wasm module cannot simply issue arbitrary operating-system system calls. It can access only its own state plus functions and resources exposed by the host. Browsers add their existing origin, permission, and process-isolation rules. WASI hosts can grant narrow capabilities.

This makes Wasm useful for running code that should be constrained, such as:

- third-party plugins;
- user-submitted computations;
- document parsers;
- extension modules;
- game logic;
- database functions;
- code in multi-tenant services.

The sandbox is not a promise that the module has no bugs. It is a boundary intended to limit what those bugs can reach.

### 4. Efficient binary representation

The binary format is structured for compact encoding, fast validation, and efficient compilation. Hosts can verify a module’s structure and types before executing it.

A particular Rust/Wasm bundle can still be large. Generic code, formatting support, panic messages, allocators, debug information, and unused dependencies all affect size. Release builds, dead-code elimination, and Wasm-specific optimization can help.

### 5. Good performance for sustained computation

Wasm is designed to map efficiently to common hardware. It often performs well for:

- integer and floating-point loops;
- binary parsing;
- pixel and sample manipulation;
- compression and decompression;
- cryptographic primitives and password derivation;
- geometry;
- physics and simulation;
- emulation;
- language runtimes;
- regular-expression engines and search;
- checksums and hashes.

“Performs well” is more accurate than “always runs at native speed.” Runtime implementation, browser, processor, memory behavior, enabled Wasm features, algorithm, and data movement all matter.

### 6. Predictable boundary and explicit dependencies

Imports make outside dependencies visible. In a well-designed host, a component can be given only what it requires. A thumbnail generator may need an image and a result buffer, not the user’s home directory or network.

This can make architectures easier to reason about and test.

### 7. Fast startup compared with heavier isolation methods

Wasm modules can often instantiate with far less machinery than a complete operating-system virtual machine or container. This is one reason Wasm is useful in serverless functions, plugins, edge systems, and short-lived tools.

It is not guaranteed to beat a tiny native executable or small JavaScript file in cold-start time. Compilation, download, decompression, binding initialization, and module size still matter.

### 8. Language-neutral components

The WebAssembly Component Model extends core modules with richer, machine-described interfaces. The **Wasm Interface Type (WIT)** language can describe records, variants, strings, lists, resources, and functions in a language-neutral way. Components written in different languages can then interact through generated bindings.

For an ordinary beginner browser app, core modules plus `wasm-bindgen` are currently the simpler mental model. The Component Model is important for the broader future of portable libraries, plugins, command tools, and composable applications.

### 9. Easy trial for the user

This is a product strength rather than an instruction-set feature. When Wasm is delivered through a browser, a user can try sophisticated local computation by opening a link. There may be:

- no administrator prompt;
- no installer;
- no architecture choice;
- no permanent commitment;
- no server upload of the selected data;
- no native executable warning.

For small personal utilities, removing setup friction can matter more than maximizing theoretical performance.

### 10. A useful division between host-controlled interface and constrained engine

An application can expose a narrow contract to a Wasm engine. The host can validate inputs, set limits, cancel work, and decide which files or services to provide. This pattern is valuable even inside a native application.

---

## The limitations of WebAssembly

Understanding the limits prevents disappointing architecture decisions.

### 1. Core Wasm has no desktop GUI

There is no core instruction meaning “create a window,” “draw a button,” or “open a native menu.” A browser host uses HTML, CSS, canvas, and Web APIs. A native host needs its own GUI integration. WASI does not presently function as a universal cross-platform desktop-windowing standard.

If the main goal is a polished no-install GUI, the browser—not bare Wasm—is doing much of the portability work.

### 2. Core Wasm has no general filesystem

A module does not inherently know about drive letters, `/home`, file permissions, or directories. Browser apps usually receive files chosen or dropped by the user. WASI programs receive access explicitly granted by the runtime.

This is safer, but it means Wasm is not a drop-in replacement for a native file manager, backup agent, or disk scanner.

### 3. Browser access is intentionally mediated

A browser app normally cannot:

- scan the entire disk;
- silently overwrite arbitrary files;
- launch any local executable;
- install drivers;
- bind every kind of network socket;
- remain running forever after the tab closes;
- read other applications’ memory;
- bypass permission prompts.

Those restrictions generally give an unfamiliar web page less ambient authority over the machine than an unfamiliar native executable would receive. They do not make the page trustworthy or harmless.

### 4. A `.wasm` file is not normally double-clickable

Windows, macOS, and Linux do not provide one universal built-in WASI command runner and GUI shell. File associations can be configured, but that is not the same as universal deployment.

If the user must first install a runtime, the app is not completely dependency-free. If the developer bundles the runtime, platform-specific packaging returns.

### 5. Boundary crossings have a cost

JavaScript and Wasm use different value and memory models. Moving strings, objects, and buffers between them can require conversion, copying, allocation, or generated adapter work.

An inefficient design might call Wasm once per pixel or once per DOM element. A better design sends one image buffer, performs a large operation inside Wasm, and returns one result.

```text
Poor boundary design:
JS ─> Wasm ─> JS ─> Wasm ─> JS ... millions of tiny calls

Better boundary design:
JS ─────────> Wasm performs substantial batch ─────────> JS
```

### 6. The UI thread can still freeze

Wasm running in the browser’s main thread can block clicks, animation, and rendering just like long-running JavaScript. Heavy work should normally run in a web worker, with progress and cancellation messages sent to the interface.

Wasm does not make synchronous work magically asynchronous.

### 7. Threads need extra architecture

Browser parallelism normally uses workers. Shared-memory Wasm threads require compatible browser support and a securely configured, **cross-origin-isolated** page—one served with headers that isolate it from incompatible cross-origin content. Libraries cannot assume that native thread behavior is available exactly as it is on a desktop target.

Many apps should begin with one worker and only add parallel threads after measurement.

### 8. Memory and very large files need care

A simple browser implementation may read an entire file into memory, copy it into Wasm memory, allocate a complete output, copy that output back, and then create a downloadable `Blob`. Peak memory can become several times the file size.

For a 4 GB input, that approach is not merely inefficient; it may be impossible on a target browser or machine. Chunking and streaming must be part of the algorithm and file format from the beginning.

Native programs have more direct, platform-specific control over streaming files, temporary paths, flushing operations such as `fsync`, rename behavior, memory mapping, and recovery after interruption. Exact durability, locking, and replacement guarantees still vary by operating system and filesystem.

### 9. Browser APIs are not perfectly uniform

The basic file-input and download model is widely portable. More advanced direct file handles, streaming output, GPU features, codecs, and installed-PWA integrations can vary across browsers and operating systems.

A compatibility fallback may be necessary. “Works in my browser” is not a cross-platform test plan.

### 10. Debugging can cross several layers

A failure may involve:

- Rust source;
- generated Wasm;
- generated JavaScript bindings;
- bundler configuration;
- browser security policy;
- worker messaging;
- the host API;
- target-specific dependency behavior.

Source maps, browser developer tools, Rust logging hooks, unit tests, and small interface boundaries help, but native debugging can still be simpler.

### 11. Distribution remains part of security

An HTTPS page can be changed every time it loads. That is convenient for updates and dangerous if the server or build pipeline is compromised. Wasm does not protect a password or plaintext from malicious JavaScript on the same page or from a malicious host.

Downloaded, signed, versioned bundles can improve pinning and auditability, but lose seamless updates.

### 12. Wasm is not automatically smaller

A tiny utility written in plain JavaScript might be a few kilobytes. Adding a Rust allocator, panic machinery, bindings, and `.wasm` module can make it much larger. Wasm shines when it carries enough valuable computation to justify the toolchain and payload.

### 13. Wasm is not automatically faster

For DOM-heavy interfaces, network-bound apps, small scripts, and work dominated by host APIs, JavaScript or TypeScript may be equal or better. Native Rust normally has the broadest optimization and OS-access opportunities.

### 14. WASI is not an operating system or runtime

WASI is a set of interface specifications. A runtime implements some version and set of those interfaces. As of this document’s review date, WASI 0.3 is the newest milestone and adds native asynchronous concepts to the Component Model, while real toolchains and runtimes may support different subsets or earlier versions.

Version claims should therefore be treated like compatibility requirements, not abstract labels. Pin, document, and test the exact target and runtime combination.

---

## The best types of local apps for Wasm

The ideal no-install Wasm app usually has a simple shape:

```text
explicit user input
        │
        ▼
substantial self-contained computation
        │
        ▼
clear result the user views or saves
```

The following categories are especially strong.

### File format converters

Examples:

- image format conversion;
- subtitle conversion;
- structured-data conversion among JSON, CSV, and other formats;
- font subsetting or conversion where licensing allows it;
- document or archive inspection;
- audio sample conversion;
- 3D model conversion.

Why Wasm fits:

- input and output are naturally byte buffers or streams;
- parser and encoder libraries often already exist in Rust or C/C++;
- users benefit from files staying local;
- the interface can be a simple drop zone and download button;
- the algorithm is mostly independent from the operating system.

### Image processing

Examples:

- resizing and cropping;
- metadata inspection and removal;
- color conversion;
- filters;
- sprite-sheet generation;
- QR or barcode processing;
- pixel-level analysis.

Why Wasm fits:

- CPU-heavy loops over typed buffers are a natural workload;
- canvas can preview the result;
- WebGPU or other browser graphics facilities may help advanced tools;
- files can remain on the user’s machine.

### Audio processing

Examples:

- waveform analysis;
- trimming and normalization;
- format decoding or encoding where supported libraries are available;
- effects;
- spectrogram generation;
- instrument and synthesizer engines.

Why Wasm fits:

- predictable numeric computation;
- existing DSP code can be reused;
- browser audio APIs provide input/output and scheduling;
- an instant interface is valuable for occasional tools.

Real-time audio has strict timing requirements. Allocation, garbage collection in the surrounding page, worker/audio-thread rules, and browser support must be considered carefully.

### Compression and archive tools

Examples:

- compressing selected data;
- decompressing an archive in memory or by chunks;
- listing archive contents without uploading them;
- checksum calculation;
- comparing compressed formats.

Why Wasm fits:

- algorithms are computational and platform-independent;
- established libraries may compile cleanly;
- privacy and immediate access are useful.

Very large archives, random-access extraction, symbolic links, permissions, and exact filesystem reconstruction are easier in a native tool.

### Parsers, validators, linters, and formatters

Examples:

- source-code formatting;
- configuration validation;
- log parsing;
- regular-expression testing;
- schema validation;
- binary file inspection;
- syntax highlighting and language services;
- data cleanup.

Why Wasm fits:

- many developer tools are pure transformations;
- Rust parsers can be shared with CLIs and servers;
- the browser supplies a capable editor UI;
- untrusted input is processed inside a sandbox boundary.

### Cryptographic utilities

Examples:

- hashing selected files;
- verifying signatures;
- generating keys or secure random values through appropriate APIs;
- encrypting and decrypting user-selected files;
- password-based key derivation;
- inspecting certificates or encoded keys.

Why Wasm can fit:

- users value local processing;
- algorithms operate on bytes;
- audited Rust libraries may be reusable;
- the browser supplies secure randomness and Web Crypto primitives;
- no installer is attractive for occasional use.

Security depends on much more than compiling Rust to Wasm. See the dedicated encryption section below.

### Scientific calculators and simulations

Examples:

- statistical calculators;
- physics simulations;
- signal analysis;
- numerical solvers;
- educational models;
- astronomy tools;
- optimization algorithms.

Why Wasm fits:

- sustained numerical work;
- repeatable inputs and outputs;
- interactive visualization in the browser;
- easy distribution to students or collaborators.

Floating-point results can differ subtly among algorithms and environments. Numerical correctness still needs test cases and error analysis.

### Emulators and interpreters

Examples:

- retro console or computer emulators;
- virtual machines for small languages;
- SQL engines;
- regular-expression engines;
- compilers and playgrounds;
- sandboxed scripting languages.

Why Wasm fits:

- interpreters are computation-heavy;
- existing C/C++ or Rust implementations can be reused;
- the sandbox is valuable for untrusted programs;
- browser graphics, audio, and keyboard input can form the shell.

### Games

Examples:

- puzzle and strategy games;
- emulated games;
- physics-heavy browser games;
- portable game engines;
- offline casual games.

Why Wasm fits:

- large simulation loops and asset decoders benefit from compiled code;
- a URL removes installation friction;
- graphics and audio APIs are available through the browser;
- game logic can be shared with other targets.

### Local data exploration

Examples:

- opening a CSV and calculating summaries;
- querying a local embedded database loaded into memory;
- visualizing logs;
- deduplicating lists;
- comparing documents;
- offline search over a user-selected dataset.

Why Wasm fits:

- data need not leave the machine;
- SQL engines and parsers can be compiled to Wasm;
- browser charting and layout provide a rich interface;
- the app can be discarded after use.

Memory limits and persistence strategy matter for large datasets.

### PDF and document utilities

Examples:

- rendering previews;
- extracting selected information;
- merging modest documents;
- page rearrangement;
- metadata inspection;
- redaction interfaces, if implemented with great care;
- generating documents from local inputs.

Why Wasm fits:

- complex parsing and rendering libraries can be reused;
- local-only handling protects privacy better than mandatory upload;
- canvas supplies preview rendering.

Documents are hostile-input territory. Parsers need fuzzing, limits, and regular security updates. A “redaction” tool must remove underlying content, not merely draw a black rectangle.

### Sandboxed plugins

Examples:

- user-installed filters in a desktop editor;
- transformations inside a data pipeline;
- game modifications;
- application extensions;
- tenant-supplied business rules.

Why Wasm fits:

- the host can expose a narrow API;
- modules are portable and language-neutral;
- execution can be metered or limited by the runtime;
- capabilities make authority explicit.

This is one of Wasm’s strongest non-browser uses.

### App-fit summary

| App idea | Browser Rust/Wasm fit | Standalone WASI fit | Native Rust fit | Important reason |
|---|---:|---:|---:|---|
| Image resizer | Excellent | Good for CLI | Excellent | Byte-buffer transformation |
| CSV analyzer | Excellent | Good | Excellent | Local parsing and calculation |
| Hash/checksum tool | Excellent | Excellent | Excellent | Simple file-in/result-out model |
| Code formatter | Excellent | Excellent | Excellent | Reusable pure logic |
| Archive viewer | Excellent | Good | Excellent | Parser reuse; extraction details vary |
| Scientific simulator | Excellent | Good | Excellent | CPU-heavy with web visualization |
| Retro emulator | Excellent | Conditional | Excellent | Compute plus browser graphics/audio |
| Sandboxed plugin | Good | Excellent | Conditional | Capability boundary is central |
| Large backup program | Poor | Conditional | Excellent | Broad, reliable filesystem access |
| Always-running sync agent | Poor | Conditional | Excellent | Background lifecycle and OS integration |
| Hardware driver | Inappropriate | Inappropriate | Possible/platform-specific | Native privileged platform code is required; Rust itself is optional |

---

## Apps that are a mixed fit

These applications can work in browser-hosted Wasm, but the details determine whether that is wise.

### Very large file processors

Wasm can process chunks efficiently, but a portable browser UI also needs a reliable way to stream input and output. Basic browser downloads may accumulate the entire result in memory. Advanced file-handle APIs are not equally available everywhere.

Good fit when:

- files are moderate;
- the algorithm supports independent or carefully framed chunks;
- a fallback can hold the result in memory;
- interruption is acceptable or resumable;
- target browsers are controlled.

Prefer native when:

- inputs routinely exceed memory;
- exact streaming behavior is critical;
- in-place or atomic replacement is required;
- partial results must survive crashes;
- the tool must recurse through directories.

### Video editing and transcoding

Wasm can run codecs, filters, timelines, and format parsers. Browsers also have evolving media and GPU APIs. Yet video files are huge, encoding is expensive, and codec licensing/support can be complicated.

Light clipping, inspection, preview, and short conversions can be excellent. Professional multi-hour transcoding pipelines often benefit from native tools and hardware-specific integrations.

### Full text and code editors

Browser interfaces can be excellent, and parsers or language servers can run in Wasm. The challenge is filesystem integration, projects with thousands of files, external tools, terminals, Git credentials, and native extensions.

A single-file editor is a strong fit. A complete IDE is possible but is a major platform project, not a simple consequence of choosing Wasm.

### Local databases

An embedded database engine can run in Wasm and persist through browser storage. This is excellent for demos, offline-first tools, and moderate datasets. Browser storage quotas, eviction rules, backup, concurrency, and data export require deliberate design.

### Peer-to-peer and network tools

Browsers offer HTTPS, WebSocket, WebRTC, and other controlled networking APIs. They do not expose arbitrary raw networking in the same way a native app can. A protocol that maps to supported web APIs may fit; a low-level network scanner does not.

### Password managers and secret stores

The browser provides useful cryptographic and storage primitives, but a serious password manager also needs a strong update channel, secure autofill model, clipboard policy, locking behavior, recovery design, and possibly browser extensions or OS integration. Wasm can be one implementation component; it is not the security architecture.

### CAD, 3D, and graphics tools

Wasm is strong for geometry and file parsing, while WebGL/WebGPU can render complex scenes. Browser memory, large assets, GPU compatibility, local project storage, and professional device integration may become limiting. Viewer and light-editor use cases are particularly good.

---

## Apps that are usually a poor fit

Browser-hosted Wasm is usually the wrong primary platform when the app’s value depends on broad or privileged access to the local computer.

### Deep operating-system utilities

Examples:

- device drivers;
- antivirus engines scanning the entire system;
- disk partitioning;
- registry editors;
- system configuration tools;
- kernel monitors;
- process managers;
- low-level performance profilers.

These require authority the browser intentionally does not provide.

### Backup, synchronization, and filesystem watchers

Reliable backup software needs directory traversal, metadata preservation, long-running background execution, change notification, retries, locking, crash recovery, and often privileged access. A native service or application is the normal fit.

### Always-running agents

Browsers can suspend workers and discard pages. Service workers are event-driven and may be stopped between events. A program that must run continuously, start at login, or guarantee a schedule should use an operating-system service or native agent.

### Apps that launch arbitrary processes

Build orchestrators, terminal applications, local automation launchers, and system administration tools often need to start and supervise other executables. Browser Wasm cannot do this freely.

### Apps dominated by ordinary DOM work

A settings form, simple CRUD interface, note page, or network dashboard may gain little from Rust/Wasm. TypeScript is often easier to build, debug, hire for, and maintain when little CPU-heavy computation exists.

### Tiny calculations

If the entire engine is ten lines of JavaScript, a Rust toolchain, generated bindings, allocator, and `.wasm` payload may add complexity without meaningful benefit.

### Native accessibility or platform integration requirements

Web accessibility can be excellent when semantic HTML is used correctly. But an app needing specialized native controls, deep assistive-technology integration, global shortcuts, system trays, shell extensions, Finder/Explorer integration, or platform automation may need a native shell.

### Workflows requiring identical filesystem guarantees everywhere

If correctness depends on exact rename, locking, durability, sparse-file, permission, symbolic-link, extended-attribute, or memory-mapping behavior, a native platform abstraction with explicit per-OS testing is more suitable.

---

## A detailed example: local file encryption

File encryption is a useful case study because it combines computation, local files, secrets, reliability, and distribution trust.

### Browser-based encryption can be legitimate

JavaScript source code does not imply that a cipher is implemented as slow, improvised JavaScript arithmetic. Browsers expose the standardized **Web Cryptography API**, which calls cryptographic functionality implemented by the browser. It includes a secure random interface and a set of common low-level cryptographic operations.

Rust compiled to Wasm is another option. It may be appropriate when:

- an audited Rust cryptography library must be reused;
- the required construction is not conveniently exposed by Web Crypto;
- password derivation or chunk processing benefits from Rust code;
- the same file format must be shared with a native Rust CLI;
- the code benefits from Rust’s type system and testing ecosystem.

Neither choice is automatically secure. A secure application needs a secure **whole design**.

### Three different questions

It helps to separate these questions:

1. **Are the cryptographic primitives sound?**  
   This concerns the algorithms, libraries, randomness, nonce rules, key derivation, and authentication.

2. **Is the program operationally reliable?**  
   This concerns large-file streaming, cancellation, partial outputs, disk-full errors, atomic replacement, and recovery.

3. **Can the user trust the exact code being run?**  
   This concerns the website, executable signature, build pipeline, dependencies, updates, and reproducibility.

Rust improves some implementation risks. Wasm provides a sandbox. A browser provides useful cryptographic and file APIs. None of them answers all three questions alone.

### A sensible browser architecture

```text
┌──────────────────────────────────────────────────────────────┐
│ Browser page                                                 │
│                                                              │
│  password ───────┐                                           │
│  random salt ────┼──> reviewed KDF + stored parameters       │
│  KDF settings ───┘                 │                         │
│                                     ▼                         │
│                                derived key                    │
│                                     │                         │
│  chosen input file ──> chunk reader                          │
│                           │         │                         │
│                           ▼         ▼                         │
│                  reviewed encryption core                    │
│                  (Web Crypto and/or Rust/Wasm)                │
│                           │                                  │
│                           ▼                                  │
│                  authenticated output chunks                 │
│                           │                                  │
│                           ▼                                  │
│                  download or granted output handle           │
└──────────────────────────────────────────────────────────────┘
```

Heavy work should run away from the main interface thread. The app should report progress, allow cancellation, bound memory, and never destroy the original file.

### The central hosted-page risk

A hosted encryption page is an application delivered from a server. If the server, deployment account, build pipeline, dependency, analytics script, or page is compromised, the delivered code could capture the password or plaintext before it reaches Wasm.

HTTPS protects the connection between the user and the origin. It does not prove that the origin itself is serving honest code.

Likewise, the Wasm sandbox primarily protects the host computer **from the module**. It does not protect a secret inside the module from a malicious host page that controls inputs, imports, outputs, and potentially accessible memory.

This leads to different trust profiles:

| Delivery model | Helpful property | Main concern |
|---|---|---|
| Hosted web app | Instant access and automatic updates | Code may change on every load; origin compromise can steal secrets |
| Cached PWA | Can work offline after a trusted load | Update/service-worker path still matters |
| Downloaded offline web bundle | Exact version can be hashed and archived | User needs a dependable way to launch and verify it |
| Signed native CLI | Pin-able executable with strong file control | Per-OS builds, signatures, update chain, broad native authority |
| Source build | User can inspect and compile | Build dependencies and toolchain must also be trusted |

For valuable data, publishing source is helpful but not sufficient. The user still needs confidence that a delivered binary or web bundle corresponds to that source. Signed releases, checksums, reproducible-build work, minimal dependencies, and independent review improve assurance.

### Cryptographic design requirements

Do not invent a cipher or casual file format. Use a mature, reviewed library or established format appropriate to the application. A serious design normally needs all of the following:

- cryptographically secure randomness supplied by a supported host API;
- when passwords are used, a reviewed password-based key derivation method with a fresh per-file salt stored with the file;
- authenticated encryption, so tampering is detected rather than producing silent garbage;
- correct nonce generation and uniqueness rules for the chosen authenticated cipher;
- authentication of important headers and metadata;
- explicit file-format versioning;
- recorded algorithm and parameter identifiers needed for future decoding;
- a safe chunking construction for streaming files;
- detection of truncation, missing chunks, duplicated chunks, and reordered chunks;
- strict input-size and allocation limits;
- malformed-input fuzzing;
- published test vectors where practical;
- cross-version round-trip and corruption tests;
- a migration plan before old algorithms or parameters become unacceptable.

The Web Crypto interface is intentionally low-level; its name `SubtleCrypto` is a useful warning that safe composition is subtle. A provided algorithm is not automatically appropriate for every protocol.

### Randomness

Never use `Math.random()` for keys, salts, or nonces. Browser projects should use the browser’s cryptographically secure random source directly or ensure that the Rust random library is correctly connected to it.

A library that obtains secure randomness automatically on native Rust may require a different target feature or host binding under `wasm32-unknown-unknown`. This must be tested, not assumed.

### Secret memory

Rust libraries can make a best effort to erase secret buffers, but browser secret handling has extra layers:

- a password may first exist as a JavaScript string;
- bytes may be copied while crossing into Wasm memory;
- allocation or binding code may create temporary copies;
- the host controls the runtime and memory implementation;
- browser extensions or malicious same-origin code may observe page data;
- crash dumps, swap, and operating-system behavior are outside the module’s control.

Zeroization is still worthwhile, but it is not a guarantee that every historical copy vanished. The app should minimize copies, retain secrets for as little time as possible, avoid logging them, and load no unrelated third-party scripts.

### Large-file reliability

A naive browser encryptor might do this:

```text
read entire 2 GB file
copy 2 GB into Wasm
allocate 2+ GB encrypted output
copy output back to JavaScript
construct 2+ GB Blob
start download
```

Peak memory could greatly exceed the original file size. The tab might fail even on a machine with plenty of total RAM because browser and Wasm limits also apply.

A streaming design instead reads bounded chunks, encrypts and authenticates each chunk under a carefully reviewed construction, and writes or accumulates output through an appropriate sink. It also authenticates the sequence and final length so an attacker cannot remove or reorder chunks undetected.

The browser output path is the hard part. A widely compatible fallback may need to build a final `Blob`, while more direct writable-file APIs vary by browser. State supported browsers and maximum recommended sizes honestly.

### Protect the original

An encryption tool should never overwrite or delete the only plaintext copy before the encrypted result is completely finalized and reported successful.

A native CLI can often:

1. create a temporary output in the destination filesystem;
2. write and finalize it;
3. flush data as required by its durability promise;
4. close it;
5. optionally reopen and validate its structure;
6. rename it into place according to the platform’s guarantees;
7. leave the original untouched unless the user separately requests removal.

A browser download does not necessarily offer equivalent control. This is one major reason a native Rust CLI can be operationally stronger.

### Browser extensions, malware, and the threat model

No normal application can defend secrets from a fully compromised operating system. A malicious browser extension with sufficient page access may also be able to observe inputs. A native executable avoids the web page’s script environment but has its own update, dependency, and native-malware risks.

The README for an actual encryption product should state its threat model explicitly:

- Is the server trusted?
- Must the app work after the server disappears?
- Are browser extensions considered hostile?
- Are very large files supported?
- Is metadata such as filename or size protected?
- What happens after interruption?
- How is the exact app version verified?
- What recovery is possible if a password is lost?

### Browser app versus Rust CLI

| Concern | Browser app with Web Crypto/Rust-Wasm | Native Rust CLI |
|---|---|---|
| Initial use | Open a page; extremely easy | Download correct binary; terminal knowledge may be needed |
| Cross-OS artifact | Usually one web build | Separate builds per OS/CPU |
| File selection | User-mediated | Paths, pipes, directory traversal |
| Very large files | Possible but browser-dependent and architecturally demanding | Natural streaming model |
| Atomic output workflow | Limited/uniformity varies | Stronger control through OS APIs |
| Automation | Weak | Excellent scripts, pipes, exit codes |
| Version pinning | Harder for a live site | Straightforward with signed release and checksum |
| Sandbox | Strong browser restrictions | Native process has broader user-level authority |
| UI | Easy graphical experience | Terminal unless a GUI is added |
| Code provenance | Trust current origin or verified offline bundle | Trust signed binary/build chain |

### Balanced conclusion

A carefully reviewed browser encryptor can be valuable for moderate, user-selected files and immediate cross-platform access when its complete threat model is addressed. A carefully reviewed native Rust CLI is often the better high-assurance operational tool for huge or important data because it gives stronger control over streaming, errors, output durability, automation, and version pinning.

The strongest product design may provide both, backed by the same reviewed Rust file-format core.

---

## Performance: when Wasm helps and when it does not

Performance is a complete-system property. A microbenchmark of one inner loop does not tell the user how quickly an app opens, reads a file, responds, and saves a result.

### Work Wasm often handles well

- long loops over numbers or bytes;
- predictable allocation patterns;
- parsing binary formats;
- decompression and compression;
- image, audio, and geometry algorithms;
- cryptographic or hashing work;
- code ported from optimized Rust or C/C++ libraries;
- algorithms that stay inside Wasm for substantial batches.

### Work that may gain little

- changing DOM elements;
- waiting for network responses;
- small event handlers;
- simple validation already handled by browser features;
- code making thousands of tiny host calls;
- tasks dominated by string/object conversion;
- one-time calculations too small to offset loading and initialization.

### Costs to include in a benchmark

An honest benchmark measures:

1. asset download or disk load;
2. decompression of web resources;
3. module validation and compilation;
4. Wasm instantiation and binding initialization;
5. file reading;
6. copies into or out of linear memory;
7. the algorithm itself;
8. copies or encoding of the result;
9. output creation and writing;
10. peak memory;
11. interface responsiveness;
12. later cached launches as well as first launch.

### Keep the boundary coarse

Prefer this:

```text
resize_rgba(input_buffer, width, height, new_width, new_height)
```

over this:

```text
for every pixel:
    call Wasm to transform one channel
```

Similarly, parse an entire logical record batch inside Rust rather than asking JavaScript to fetch every individual field through a binding.

### Avoid unnecessary copies

Depending on the binding and API, a `Uint8Array` may be copied into Wasm memory. Returning a Rust `Vec<u8>` may create another view or copy. Measure and document the actual path.

Possible strategies include:

- allocating a buffer inside Wasm and letting the host fill it;
- processing chunks rather than complete files;
- returning compact metadata instead of copied structures;
- using transferable buffers between workers where appropriate;
- maintaining long-lived state inside one worker/module instance;
- designing one binary interface instead of many string-heavy calls.

Safety and maintainability come first. A fragile zero-copy trick is not automatically better than one clear bounded copy.

### Keep the interface responsive

An instant app feels broken if the button freezes for ten seconds. Put sustained CPU work in a worker. Send periodic progress messages at a reasonable rate, not on every byte. Define cancellation points. Decide what partial state is safe to discard.

### Native comparison

Native Rust normally has:

- direct filesystem APIs;
- fewer browser/host boundaries;
- access to platform threads and synchronization;
- mature profilers and debuggers;
- architecture-specific optimization opportunities;
- better control over memory mapping and large streams.

Wasm’s advantage is often **distribution plus adequate high performance**, not defeating the best native build in every benchmark.

### Size and startup matter for instant tools

A 20 MB module that saves 50 milliseconds of processing may feel worse than a 20 KB JavaScript implementation. Optimize for the actual interaction:

- remove unused dependencies;
- use release builds;
- strip debug information from distributed modules while preserving symbols separately;
- use size optimization where appropriate;
- compress assets during HTTPS delivery;
- lazy-load engines not needed on the first screen;
- show useful UI before a heavy module finishes loading;
- cache assets safely for repeat use.

---

## Security: what the sandbox does and does not guarantee

### What the Wasm sandbox helps protect

Under a conforming, correctly implemented Wasm engine, a valid core module executes under defined memory and structured control-flow rules. Its own instructions cannot directly jump to arbitrary host machine code or make an undeclared OS system call. Interaction happens through host-provided imports.

The browser or standalone runtime, its compiler, and host import implementations remain part of the trusted computing base. They can contain vulnerabilities. A native host that unsafely interprets guest-controlled pointers, lengths, or handles can undermine the boundary even when the guest module itself validates correctly.

In a browser, normal web policies also apply, including origin and permission controls. In a capability-oriented standalone runtime, the host may grant a module access to one directory and no network, for example.

This can reduce the damage from:

- a buggy parser;
- a malicious plugin;
- untrusted user computation;
- a library that should not see the whole machine.

### What it does not prove

The sandbox does not prove that:

- output is correct;
- input cannot trigger a denial of service;
- the algorithm is secure;
- secrets are hidden from the host;
- code has no memory bugs inside its own linear memory;
- a Rust `unsafe` block is correct;
- the host imports are honest;
- the website is uncompromised;
- dependencies are trustworthy;
- timing or other side channels do not exist;
- the user cannot be tricked by the interface.

### Protecting the host versus protecting the guest

This distinction is essential:

```text
Wasm sandbox: primarily constrains what the guest module can do to the host.
It is not: a trusted vault protecting the guest from the host that runs it.
```

A host can choose the input, observe results, deny resources, and implement imports maliciously. In a browser page, same-origin script coordinates the app. Do not put a secret in Wasm and assume JavaScript or the page can never reach it.

### Rust and Wasm provide complementary layers

Safe Rust aims to prevent many invalid references, use-after-free errors, and data races at the source-language level. Wasm checks module-level memory bounds and control-flow rules at runtime. The browser or standalone runtime controls external authority.

```text
Rust type/ownership safety
          +
Wasm validation and sandbox
          +
host capability/permission policy
          +
secure application design and distribution
          =
useful layered defense
```

Remove any layer and the others still help, but no single layer is a complete security story.

### Treat all files as hostile

Local does not mean trusted. A document or archive can be deliberately malformed. Parsers should have:

- strict bounds checks;
- integer-overflow handling;
- limits on recursion, dimensions, records, and decompressed size;
- timeouts or cancellation;
- fuzz testing;
- a policy for unsupported features;
- prompt security updates;
- clear error messages that do not leak secrets.

### Local processing and privacy

A browser app can process data entirely on the device, but that is a design property, not an automatic Wasm property. JavaScript or imported host functions can send data over the network if allowed.

A privacy-sensitive tool should:

- state whether any file contents, filenames, sizes, hashes, or telemetry leave the device;
- avoid unnecessary analytics and third-party scripts;
- restrict allowed network destinations with a strong Content Security Policy where applicable;
- make offline operation possible and visible;
- document browser storage and cache use;
- allow users to verify downloadable releases;
- test that error reporting does not include sensitive inputs.

### No install does not mean no traces

A browser may retain:

- cached HTML, JavaScript, and Wasm;
- service-worker data;
- IndexedDB or other site storage;
- download history;
- recent-file metadata;
- permissions associated with an origin;
- browser history and logs.

A portable native app may also leave recent-file entries, temporary files, crash reports, OS logs, and filesystem metadata. Make trace behavior explicit if it matters.

### Source visibility

Wasm binary format is compact, not secret. Tools can inspect, disassemble, instrument, and reverse-engineer modules. Do not place a permanent API secret, private encryption key, or irreplaceable proprietary secret in client-side Wasm.

---

## Recommended architectures

Good architecture isolates portable logic from environment-specific code. The goal is not “put everything in Wasm.” The goal is to put each responsibility in the layer that handles it well.

### Architecture A: Browser UI plus Rust/Wasm worker

This is the default recommendation for an instant, no-install personal utility with a graphical interface.

```text
┌──────────────── MAIN BROWSER THREAD ────────────────┐
│ HTML/CSS interface                                 │
│ - file picker                                      │
│ - options                                          │
│ - progress bar                                     │
│ - preview and result                               │
│                                                    │
│ small TypeScript/JavaScript controller             │
└──────────────────────┬─────────────────────────────┘
                       │ messages / transferable data
                       ▼
┌──────────────── WEB WORKER ────────────────────────┐
│ generated bindings                                 │
│ Rust compiled to Wasm                              │
│ - parser                                           │
│ - transformation                                   │
│ - encoder                                          │
│ - validation                                       │
└──────────────────────┬─────────────────────────────┘
                       │ bounded result chunks
                       ▼
                 browser output path
```

Use this when:

- the user selects a limited set of inputs;
- the core performs substantial computation;
- the browser’s permission model is acceptable;
- a web interface is desirable;
- instant cross-OS access matters.

Design rules:

- keep DOM access on the main side;
- send coarse jobs to the worker;
- use structured progress and error messages;
- make cancellation explicit;
- put size limits at the boundary;
- keep output streaming needs in mind from the first file-format design;
- do not make the Rust core depend directly on UI concepts.

### Architecture B: One pure Rust core with browser and native front ends

This is often the strongest long-term design for a serious file utility.

```text
portable-tool/
├── core/
│   ├── parser.rs
│   ├── format.rs
│   ├── transform.rs
│   └── error.rs
├── web/
│   ├── Rust-to-Wasm bindings
│   ├── worker
│   ├── HTML/CSS/TypeScript
│   └── browser file adapter
├── cli/
│   ├── native file adapter
│   ├── streaming and temporary-output logic
│   └── terminal arguments/progress
└── tests/
    ├── shared vectors
    ├── malformed inputs
    └── cross-front-end compatibility
```

The core accepts abstract readers, writers, byte slices, or domain values rather than browser `File` objects or OS paths. Each front end adapts its environment to that core.

Benefits:

- one definition of the important file format;
- browser version for convenience;
- native CLI for huge files and automation;
- shared test vectors;
- easier comparison of results;
- platform-specific code remains small and visible.

Potential difficulty:

An API designed around fully synchronous native `Read` and `Write` traits may not map naturally to asynchronous browser streams. Define an incremental state machine or chunk-oriented core when both environments need streaming.

### Architecture C: WASI command component

```text
terminal or host application
          │
          ▼
WASI runtime with explicit grants
          │
          ├── input directory capability
          ├── output directory capability
          ├── clock/random capability
          └── optional network capability
          │
          ▼
Rust/Wasm command or component
```

Use this when:

- users or the enclosing product already have a compatible runtime;
- a terminal interface is enough;
- sandboxed plugins or commands are desired;
- the same module must run inside several controlled hosts;
- explicit capabilities are part of the design.

Document the exact Wasm target, WASI interface version, runtime versions, enabled features, and invocation. “Supports WASI” alone is not a sufficiently precise compatibility statement.

### Architecture D: Native shell with portable Wasm plugins

```text
OS-specific native application
├── native window and menus
├── filesystem and device integration
├── update/signing system
├── embedded Wasm runtime
└── portable sandboxed plugin modules
```

Use this when the main application needs native powers but third-party or user-supplied extensions should receive narrower authority.

The native host can expose domain-specific imports such as:

- `read_current_document()`;
- `replace_selected_pixels()`;
- `emit_diagnostic()`;
- `request_user_confirmation()`.

This is safer and easier to evolve than giving every plugin arbitrary OS access, provided the host API is carefully designed.

### Architecture E: Web UI plus native Rust backend

Sometimes the right conclusion is that the Rust code should remain native.

A desktop framework can display HTML/CSS in a web view and call a native Rust backend for direct file streaming, system integration, and background operations within the user’s and operating system’s permissions. That gives a cross-platform web-style UI but requires per-OS builds.

Use this when:

- the interface benefits from web technology;
- the backend needs native files, processes, sockets, or devices;
- OS-specific packaging is acceptable;
- a double-clickable desktop product matters more than a single cross-OS artifact.

Compiling native backend logic to Wasm merely to put it inside the same product may add an unnecessary boundary unless sandboxing or portable plugins are explicit requirements.

### Architecture F: Plain TypeScript without Wasm

This is a valid and often excellent answer.

Use plain web technology when:

- almost all work is UI and browser API coordination;
- the computation is small;
- no valuable Rust library is being reused;
- the Wasm boundary would mainly pass strings and objects back and forth;
- the smallest payload and simplest debugging path matter.

Adding Wasm should solve a real problem: reuse, performance, isolation, predictable binary processing, or component portability.

### Three concrete examples

#### Image converter: browser Rust/Wasm is ideal

```text
user selects image
      ↓
browser reads selected bytes
      ↓
worker passes buffer to Rust/Wasm codec
      ↓
Rust decodes, transforms, and encodes
      ↓
browser previews and downloads result
```

The app needs explicit input, explicit output, and CPU-heavy portable logic. It does not need ambient filesystem access.

#### Sandboxed log converter: WASI is ideal in a managed environment

```text
administrator grants one input and output directory
      ↓
WASI runtime starts converter
      ↓
converter sees only granted resources
      ↓
portable result is written to approved output
```

The app is command-oriented, sandboxing is useful, and the organization can standardize the runtime.

#### Folder backup: native Rust is better

The tool must recurse, preserve metadata, understand symbolic links, detect changes, handle permissions, stream indefinitely, recover from interruption, and run in the background. Those are operating-system responsibilities, not merely computation.

### The portability triangle

It is difficult to maximize all three of these at once:

```text
                  SAME ARTIFACT ACROSS OSES
                            ▲
                           / \
                          /   \
                         /     \
                        /       \
      NO EXTRA RUNTIME ◄─────────► DEEP OS INTEGRATION
```

- **Browser app:** same web artifact and an already-present browser, but limited OS integration.
- **WASI artifact:** the same Wasm artifact and controlled integration, but a compatible runtime is required.
- **Native app:** deep integration and no separately installed language runtime, but separate builds are required.

Pick the two properties most important to the product rather than pretending the tradeoff does not exist.

---

## Distribution choices for instant local apps

Distribution is part of the product architecture. The word “local” can describe where computation happens even when code is delivered from the web.

### Choice 1: Hosted HTTPS app

The user opens a link and the app processes chosen data locally.

Recommended when:

- the lowest possible trial friction matters;
- users accept trusting the hosted origin;
- automatic updates are valuable;
- online first launch is acceptable.

Good release practices:

- use no third-party scripts in sensitive tools;
- publish a clear privacy/network statement;
- use a restrictive Content Security Policy;
- version the application and show the version in the UI;
- preserve release notes and source tags;
- test old cached assets against new assets;
- ensure `.wasm` is served with the correct MIME type;
- offer an offline package for users who need pinning.

### Choice 2: Offline-capable PWA

The first trusted visit caches an application shell and Wasm assets. The app can later open offline, possibly from an icon.

Recommended when:

- users return frequently;
- offline operation matters;
- browser installation behavior is acceptable;
- the team can test service-worker lifecycle and update cases.

Important questions:

- Does the app clearly show when it is offline?
- Is every required Wasm, worker, font, and asset cached?
- What happens if a new HTML file is paired with an old `.wasm` file?
- Does an update wait until current jobs finish?
- Can the user pin a reviewed version?
- What happens if browser storage is cleared?

### Choice 3: Downloaded static ZIP

The package contains the complete web app and can be archived.

Recommended when:

- exact-version ownership matters;
- users need an air-gapped copy;
- the audience can run a local server or launcher;
- a signed checksum manifest or an artifact signature is useful.

Do not assume double-clicking `index.html` works. Choose and document a supported launcher or local-server path. Test `file://` only if the project intentionally promises to support it; ES modules, workers, fetch, and service workers can make that promise impractical.

### Choice 4: Single HTML document

Recommended when the application and data are small and one-file copying is the dominant requirement. Treat this as a deliberate build target. Test it separately rather than assuming the normal multi-file build can simply be concatenated.

Potential costs:

- encoded Wasm grows relative to its raw bytes;
- initial parsing may be slower;
- resource caching is all-or-nothing;
- workers may need generated blob URLs;
- strict CSP becomes more difficult;
- source maps and debugging become awkward.

### Choice 5: `.wasm` plus a documented WASI runtime

Recommended for technical audiences, developer tools, or managed environments. Publish:

- the exact target and component/module type;
- supported runtime names and minimum versions;
- hashes or signatures;
- the capability grants the command needs;
- example invocations;
- expected exit codes;
- input/output format versions.

The runtime should not receive more directory or network access than the tool needs.

### Choice 6: Per-OS native portable packages

Recommended when users need direct files and strong reliability but an installer is unnecessary. A release can contain:

```text
tool-windows-x86_64.zip
tool-windows-arm64.zip
tool-macos-universal-or-per-arch.tar.gz
tool-linux-x86_64.tar.gz
tool-linux-arm64.tar.gz
```

Code signing, notarization, executable reputation, libc compatibility, and platform testing still matter. “Portable ZIP” means no installer; it does not mean one executable format works on every OS.

### Choice 7: Native desktop application

Recommended when the app is used often, needs file associations or system integration, or must behave as a first-class OS application. An installer can actually improve reliability by placing files correctly, registering uninstallation, verifying signatures, and managing updates.

“No installer” is a usability goal, not a moral virtue. Choose it when it helps the user.

### Browser file behavior for beginners

#### Selecting a file

A normal file input or drag-and-drop gives the page a `File` object for the item the user selected. The app can read its bytes. It does not thereby gain permission to scan neighboring files.

#### Paths

Browsers normally avoid revealing a useful absolute operating-system path. Application logic should use the supplied file object, not expect `C:\Users\...` or `/home/...`.

#### Saving a result

The most portable approach creates a download. More advanced browsers may offer user-approved writable file handles, but support and behavior need testing. A download may not give the app confirmation of the same durability properties a native program can request.

#### Reading by chunks

`File`/`Blob` data can be sliced or streamed. Rust/Wasm code must expose an incremental API or process bounded buffers. A Rust function that accepts the entire `Vec<u8>` is not a streaming design just because the source is Wasm.

#### Memory planning

A useful first estimate is:

```text
peak memory ≈
    resident input chunks
  + JavaScript copies
  + Wasm input/storage
  + algorithm workspace
  + Wasm output
  + JavaScript output/Blob
  + browser and UI overhead
```

Measure real peak memory on low-end supported machines. Do not publish a maximum file size based only on the nominal Wasm memory ceiling.

#### Closing and interruption

The user can close or refresh a tab at any time. The browser can crash or reclaim resources. Design jobs so that an incomplete result is clearly incomplete and the original remains safe.

### Distribution decision table

| Approach | User opens | Extra prerequisite | Same payload across OSes | GUI | File access | Large-file reliability | OS integration | Best use |
|---|---|---:|---:|---:|---|---|---|---|
| JavaScript browser app | URL/page | Browser | Usually | Excellent | Selected/restricted | Conditional | Low | Simple web utilities |
| Browser Rust/Wasm | URL/page | Browser | Usually | Excellent | Selected/restricted | Conditional | Low | CPU-heavy instant tools |
| Downloaded web bundle | Local page/folder | Browser; possibly local server | Often | Excellent | Browser-restricted | Conditional | Low | Pinned offline utility |
| PWA | App icon/browser | PWA support | Mostly | Excellent | Browser-dependent | Conditional | Low–medium | Repeated offline-first use |
| WASI artifact | Runtime command | Wasm runtime with matching WASI support | Often | Primarily terminal | Explicit grants | Good with suitable interfaces | Medium | Portable sandboxed command |
| Native shell + embedded Wasm | OS executable | None separately | No | Excellent | Native by host policy | Good | High | Desktop app with plugins |
| Web UI + native Rust | OS application | None separately | No | Excellent | Native backend | Very good | High | Cross-platform desktop GUI |
| Native Rust CLI | OS executable | None separately | No | Terminal | User-level native | Excellent | High | Serious file/system utility |

“Same payload” always assumes compatible host features. “Native access” remains limited by the user account, OS security controls, and any application sandbox.

---

## A tiny Rust-to-Wasm example

This example counts zero-valued bytes in a selected file. It is intentionally harmless and simple; it demonstrates the boundary without pretending to be an encryption tutorial.

**Before copying it:** this teaching example reads the complete file into memory and runs the count on the browser’s main thread. Choose only a small, non-sensitive test file. The page below enforces a 10 MiB demonstration limit. It is not a pattern for large-file production work.

Prerequisites:

- a current Rust toolchain installed through the official Rust instructions;
- `wasm-pack` installed according to its maintained documentation;
- a terminal in which `rustc --version`, `cargo --version`, and `wasm-pack --version` work;
- a modern browser.

Create the library project and enter its directory:

```console
cargo new --lib wasm-byte-counter
cd wasm-byte-counter
cargo add wasm-bindgen
```

The following files belong inside that new `wasm-byte-counter` directory.

### 1. The Rust library

`src/lib.rs`:

```rust
use wasm_bindgen::prelude::*;

#[wasm_bindgen]
pub fn count_zero_bytes(bytes: &[u8]) -> u32 {
    bytes
        .iter()
        .filter(|&&byte| byte == 0)
        .count()
        .try_into()
        .unwrap_or(u32::MAX)
}
```

The `#[wasm_bindgen]` attribute asks the binding tool to expose the function across the JavaScript/Wasm boundary.

Relevant `Cargo.toml` pieces:

```toml
[package]
name = "wasm-byte-counter"
version = "0.1.0"
edition = "2024"

[lib]
crate-type = ["cdylib"]

[dependencies]
wasm-bindgen = "0.2"
```

In a real project, pin and update dependencies according to its release policy.

### 2. Build for the browser

One common workflow uses `wasm-pack`:

```console
rustup target add wasm32-unknown-unknown
wasm-pack build --target web --release
```

The build creates a `.wasm` module plus generated JavaScript bindings in `pkg/`. Cargo package names may contain hyphens, but Rust identifiers and generated filenames use underscores; that is why the package named `wasm-byte-counter` produces an import named `wasm_byte_counter.js` below.

Tooling changes over time. Consult the maintained `wasm-bindgen` documentation for current setup and bundler choices rather than treating these two commands as a complete production build system.

### 3. A minimal page

`index.html`:

```html
<!doctype html>
<html lang="en">
  <head>
    <meta charset="utf-8">
    <meta name="viewport" content="width=device-width, initial-scale=1">
    <title>Wasm byte counter</title>
  </head>
  <body>
    <h1>Count zero bytes locally</h1>
    <label for="file">Choose a small file (maximum 10 MiB)</label>
    <input id="file" type="file">
    <p id="result" role="status" aria-live="polite">No file chosen.</p>

    <script type="module">
      import init, { count_zero_bytes } from "./pkg/wasm_byte_counter.js";

      await init();

      const picker = document.querySelector("#file");
      const result = document.querySelector("#result");
      const demoLimit = 10 * 1024 * 1024;

      picker.addEventListener("change", async () => {
        const file = picker.files[0];
        if (!file) return;

        if (file.size > demoLimit) {
          result.textContent = "That file is too large for this small demo.";
          picker.value = "";
          return;
        }

        try {
          result.textContent = `Reading ${file.name}…`;
          const bytes = new Uint8Array(await file.arrayBuffer());
          const count = count_zero_bytes(bytes);
          result.textContent = `${file.name} contains ${count} zero bytes.`;
        } catch (error) {
          console.error(error);
          result.textContent = "The file could not be read or processed.";
        }
      });
    </script>
  </body>
</html>
```

### 4. What each layer did

| Layer | Responsibility |
|---|---|
| HTML | Displayed a file picker and result paragraph |
| JavaScript | Loaded Wasm, responded to selection, read browser `File`, updated DOM |
| Generated binding | Converted the `Uint8Array` call into the module’s expected memory/interface form |
| Rust/Wasm | Counted bytes |
| Browser | Supplied the window, file permission, JavaScript engine, Wasm engine, and rendering |

The selected file was not automatically uploaded. The example contains no network call after its assets load. A real privacy claim should still be verified against the complete application and deployed page.

### 5. Why double-click may fail

The page uses an ES module and fetches/instantiates a `.wasm` asset. Browser security rules around `file://` can block that flow. During development, serve the directory over localhost with a suitable development server and open the shown localhost URL.

For example, run this from the demonstration directory where `index.html` lives when an appropriate Python installation is already available:

```console
python3 -m http.server --bind 127.0.0.1 8000
```

Then open `http://127.0.0.1:8000/`. Python’s `http.server` is a basic development convenience, not a production server. A production project should use its normal build/development server, send the correct MIME types and headers, and test the deployment directly.

### 6. Why this is not a large-file design

`file.arrayBuffer()` reads the whole file into memory, and the binding may copy its bytes. That is acceptable for a tiny demonstration, not a proof that multi-gigabyte files are supported.

A real processor would expose an incremental Rust state object or chunk function, run it in a worker, report progress, bound all allocations, and use a carefully tested output strategy.

### 7. What a native version would change

The counting function could live in a pure Rust core. A native CLI would open a file path and feed fixed-size buffers through that same core. The algorithm stays shared; only the host adapter changes.

---

## A practical decision guide

Start with the app’s required authority, not with a preferred language.

```text
Does the app need arbitrary filesystem access, device access,
process launching, guaranteed background execution, or deep OS integration?
│
├── Yes
│   └── Prefer native Rust or a web-style UI with a native Rust backend.
│
└── No
    │
    └── Is “open instantly with no traditional installer” a primary goal?
        │
        ├── Yes
        │   └── Use a browser app.
        │       └── Add Rust/Wasm if computation, reuse, or isolation justifies it.
        │
        └── No
            │
            └── Is a standalone Wasm runtime an acceptable prerequisite?
                │
                ├── Yes
                │   └── Consider WASI for a sandboxed command/component.
                │
                └── No
                    └── Ship per-OS native builds.
```

### Ask these questions in order

#### 1. What does the app need permission to do?

- one selected file;
- a selected directory;
- arbitrary directory traversal;
- background monitoring;
- raw networking;
- device or driver access;
- child processes;
- global shortcuts;
- clipboard, notifications, or camera;
- full desktop integration.

This answer frequently determines browser versus native before performance is discussed.

#### 2. What is the core workload?

- DOM and forms: favor TypeScript;
- byte transformation or parser: Wasm may be excellent;
- reused Rust library: Wasm is compelling;
- system orchestration: favor native;
- sandboxed plugin: Wasm/WASI is compelling;
- network wait time: Wasm may add little.

#### 3. What does “portable” mean for these users?

- same URL;
- offline browser package;
- one `.wasm` for managed runtimes;
- one ZIP per OS;
- runs from USB;
- preserves user data across OSes;
- identical interface;
- no administrator rights.

Write the required promise explicitly.

#### 4. What is the maximum real input?

Do not answer “unlimited.” Determine typical, 95th-percentile, and supported-maximum sizes. Estimate peak memory, processing time, output size, and failure recovery.

#### 5. What is the trust model?

- Is code fetched fresh from a server?
- Can the server be trusted with updates?
- Is an offline pin needed?
- Are plugins or input documents hostile?
- Does the app handle secrets?
- Are signed artifacts required?

#### 6. What happens when interrupted?

- Is the original safe?
- Is partial output recognizable?
- Can work resume?
- Are temporary resources cleaned up?
- Does the user receive an actionable error?

### Fast recommendations by goal

| Primary goal | Recommended starting point |
|---|---|
| Instant GUI utility for selected files | Browser + TypeScript; add Rust/Wasm for the engine |
| Reuse a Rust parser in the browser | Rust/Wasm bindings plus a web UI |
| Very small browser calculator | Plain TypeScript/JavaScript |
| Multi-gigabyte dependable file transformation | Native Rust CLI, optionally plus browser version for smaller files |
| Portable sandboxed command in a managed environment | WASI artifact: P1 module or P2/P3 component |
| User-written plugins in a native app | Embedded Wasm runtime with narrow host API |
| Full desktop integration with web-style UI | Native Rust backend plus web-view UI |
| Background sync or filesystem watch | Native service/application |
| Public demo of scientific engine | Browser UI plus Rust/Wasm worker |
| High-trust encryption workflow | Reviewed native tool or verified offline package; choose from threat model |

### The simplest reliable rule

> Use Wasm when the heart of the app is portable computation. Use native Rust when the heart of the app is operating-system integration.

---

## Design and testing checklist

Use this as a release checklist for an actual local utility.

### Product definition

- [ ] State exactly what “local” means.
- [ ] State whether any content, metadata, telemetry, or errors leave the PC.
- [ ] Define supported browsers, runtimes, operating systems, and processors.
- [ ] Define typical and maximum supported input sizes.
- [ ] Define offline behavior.
- [ ] Define what persists after closing the app.
- [ ] Explain the update and version-pinning model.
- [ ] Offer a native alternative if important workloads exceed browser limits.

### Architecture

- [ ] Keep pure business/format logic separate from browser and OS adapters.
- [ ] Decide whether Wasm solves a concrete problem over plain TypeScript.
- [ ] Keep JavaScript/Wasm calls coarse-grained.
- [ ] Put CPU-heavy work in a worker.
- [ ] Define progress, cancellation, error, and completion messages.
- [ ] Design streaming before implementing large-file support.
- [ ] Do not expose more host functions or capabilities than needed.
- [ ] Version interfaces and persistent file formats.

### Rust target compatibility

- [ ] Pin the Rust toolchain according to project policy.
- [ ] Pin and audit dependencies.
- [ ] Check every crate for target-specific OS, thread, socket, C-library, and assembly assumptions.
- [ ] Test browser-target code in actual browsers, not only native unit tests.
- [ ] Test WASI code in every claimed runtime/version.
- [ ] Make native-only and web-only features explicit.
- [ ] Verify the random source on each target.
- [ ] Decide and test panic behavior.
- [ ] Preserve useful symbols/source maps separately for diagnosis.

### Browser behavior

- [ ] Test Chrome/Chromium, Firefox, Safari, and Edge as claimed.
- [ ] Test Windows, macOS, and Linux as claimed.
- [ ] Test file selection cancellation.
- [ ] Test permission denial and revoked handles.
- [ ] Test offline first launch versus offline repeat launch.
- [ ] Test service-worker update transitions.
- [ ] Test refresh, tab close, browser crash, and worker termination.
- [ ] Test storage clearing and quota exhaustion.
- [ ] Test browser zoom, high contrast, keyboard use, and screen readers.
- [ ] Detect optional APIs and provide fallbacks or clear messages.

### Performance and memory

- [ ] Measure first load and cached load.
- [ ] Measure complete workflow, not only the inner Rust function.
- [ ] Measure peak memory on the least powerful supported machine.
- [ ] Count or profile major buffer copies.
- [ ] Test smallest, typical, large, and deliberately oversized inputs.
- [ ] Keep the interface responsive during worst-case work.
- [ ] Rate-limit progress messages.
- [ ] Benchmark JavaScript-only, Wasm, and native alternatives where the choice matters.
- [ ] Set decompression, recursion, dimension, record, and runtime limits.

### Data safety and reliability

- [ ] Never overwrite the only input before complete success.
- [ ] Mark or isolate partial output.
- [ ] Test disk-full and quota-full behavior.
- [ ] Test malformed, empty, truncated, and adversarial files.
- [ ] Test duplicate names and output collisions.
- [ ] Test cancellation at many points.
- [ ] Test cross-version compatibility.
- [ ] Provide recovery instructions.
- [ ] Make every “success” state mean the result is actually finalized.

### Security and privacy

- [ ] Treat all input as hostile.
- [ ] Fuzz important parsers and decoders.
- [ ] Minimize host capabilities and browser permissions.
- [ ] Avoid unrelated third-party scripts.
- [ ] Use a restrictive Content Security Policy where applicable.
- [ ] Document every network destination.
- [ ] Prevent errors and telemetry from containing secrets.
- [ ] Publish checksums and signatures for downloadable releases.
- [ ] Consider reproducible builds for high-trust tools.
- [ ] Track dependency and compiler security advisories.
- [ ] Do not treat Wasm as source-code protection.
- [ ] Have an update and vulnerability-response process.

### Cryptographic utilities

- [ ] Use mature reviewed designs and libraries.
- [ ] Obtain cryptographically secure randomness from the host.
- [ ] Use authenticated encryption where confidentiality and integrity are required.
- [ ] Review KDF, salt, nonce, and chunking rules.
- [ ] Authenticate important headers and sequence information.
- [ ] Publish or maintain test vectors.
- [ ] Test every kind of corruption and truncation.
- [ ] Minimize secret copies and lifetime.
- [ ] Do not log passwords, keys, plaintext, or sensitive filenames.
- [ ] State the server, browser-extension, and malware threat assumptions.
- [ ] Explain that lost keys/passwords may mean permanent data loss.

### Release engineering

- [ ] Assign an application version visible to users.
- [ ] Record source revision, compiler, target, and build dependencies.
- [ ] Produce a software bill of materials where risk warrants it.
- [ ] Optimize the release build and strip only what can be recovered for debugging.
- [ ] Serve `.wasm` with correct content type and compression settings.
- [ ] Test cache invalidation across every asset.
- [ ] Sign native or offline release artifacts as appropriate.
- [ ] Publish support and deprecation policies.

### User experience

- [ ] Explain that the app processes locally before users select sensitive data.
- [ ] Explain every permission prompt in plain language.
- [ ] Show progress and a safe cancel action.
- [ ] Make errors specific and actionable.
- [ ] Preserve the user’s original data.
- [ ] Provide keyboard and assistive-technology support.
- [ ] Avoid claiming “unlimited file size.”
- [ ] Make offline and version status visible.
- [ ] State where output will be saved.

---

## Common myths

| Myth | Reality |
|---|---|
| “Wasm is just JavaScript.” | Wasm is a separate binary instruction format. In a browser, both are hosted by the browser and commonly cooperate. |
| “Rust becomes JavaScript.” | Rust can compile into Wasm instructions. Binding tools may also generate JavaScript adapter code. |
| “Wasm replaces JavaScript.” | Usually not in browser apps. JavaScript/TypeScript and web APIs remain excellent for UI and host coordination. |
| “One `.wasm` file runs everywhere.” | Only where the host supports the required Wasm features and supplies compatible imports. |
| “A `.wasm` file is a portable desktop executable.” | It needs a browser, standalone runtime, or embedding application and normally is not universally double-clickable. |
| “Wasm means no install.” | A host is still required. Browsers happen to be widely preinstalled. |
| “Wasm always runs at native speed.” | It can be fast, especially for sustained computation, but performance is workload- and host-dependent. |
| “Wasm is always faster than JavaScript.” | JavaScript may be faster or simpler for UI-heavy, small, or host-API-dominated work. |
| “Wasm has native filesystem access.” | Core Wasm has none. The host grants selected browser files, WASI capabilities, or custom imports. |
| “Wasm includes a standard desktop GUI.” | It does not. Browsers, native shells, or custom hosts provide the GUI. |
| “The sandbox makes the program secure.” | It limits authority. It does not correct bad logic, weak cryptography, vulnerable dependencies, or a malicious host. |
| “Rust makes any program secure.” | Safe Rust prevents many memory errors, not design, protocol, dependency, or operational failures. |
| “Browser-local means data cannot leave.” | Data stays local only when the full application makes no transmission. Wasm itself does not block network-capable host code. |
| “Browser encryption is weak because JavaScript is involved.” | Browser cryptographic implementations can be strong. Delivery provenance, protocol design, and file reliability are separate concerns. |
| “Native is automatically secure.” | Native software can be vulnerable and usually has broader authority if compromised. |
| “Portable means one file.” | It may mean common source, one Wasm module, one web folder, or equivalent per-OS packages. |
| “No installer means the app leaves no traces.” | Browser caches, storage, history, downloads, temp files, logs, and OS metadata may remain. |
| “WASI is a runtime.” | WASI is a family of host-interface specifications implemented by runtimes. |
| “WASI is the browser API.” | Browser Web APIs and WASI are different host environments, even though adapters can bridge some use cases. |
| “Wasm hides proprietary code.” | Wasm can be inspected and reverse-engineered. It is a delivery format, not a secret vault. |
| “Any Rust crate can compile to browser Wasm.” | Crates depending on OS services, native libraries, threads, assembly, or sockets may need changes or replacements. |
| “Using a worker makes code parallel.” | A worker moves work off the UI thread. True parallel algorithms and shared memory require additional design and support. |
| “A PWA is automatically offline.” | Assets and data must be deliberately cached, versioned, and tested for offline use. |
| “Opening `index.html` always launches an offline Wasm app.” | `file://` restrictions can block module loading, fetches, workers, service workers, and other features. |

---

## Glossary

### ABI

**Application Binary Interface.** A low-level agreement about how compiled pieces call each other and represent data. The Component Model’s canonical ABI helps different languages exchange richer values consistently.

### API

**Application Programming Interface.** A defined way for one piece of software to request services from another. Browser APIs include files, networking, cryptography, workers, graphics, and the DOM.

### Browser runtime

The browser machinery that parses web assets, provides Web APIs, validates and executes Wasm, enforces permissions, and renders the interface.

### Capability

An explicit authority to use a resource or operation. A runtime might grant one directory handle rather than giving a module access to the entire filesystem.

### CLI

**Command-line interface.** An interface used by typing commands in a terminal. CLIs are especially useful for automation, scripts, pipes, repeatable options, and clear exit statuses.

### Compiler

Software that translates code from one language or form into another. Rust’s compiler can produce native machine code for a particular platform or Wasm for a compatible host.

### Component Model

An extension around core WebAssembly for describing and composing components with richer, language-neutral interfaces.

### Core module

A binary following the WebAssembly Core Specification. It contains functions, types, memory declarations, imports, exports, and other low-level sections.

### Crate

A Rust package or compilation unit. A library crate provides reusable code; a binary crate produces a program. Cargo manages crates and their dependencies.

### Cross-origin isolation

A browser security configuration established with particular response headers. It separates a page from incompatible cross-origin content and is required for features such as shared memory in current browsers.

### DOM

**Document Object Model.** The browser representation of the HTML page. JavaScript and web bindings manipulate it; core Wasm has no built-in DOM instructions.

### Export

A function or other item the Wasm guest makes available to its host or another component.

### GUI

**Graphical user interface.** The visible windows, buttons, forms, menus, previews, and other graphical controls through which a person uses an app.

### Guest

Code being executed inside a host. A Wasm module or component is commonly called the guest.

### Host

The environment that loads Wasm and supplies external capabilities. A browser, standalone runtime, server, database, or native application can be a host.

### Import

A function, memory, table, global, or component interface a Wasm guest expects another part of the system to provide.

### JavaScript glue or binding

Adapter code that loads a browser Wasm module, translates values, exposes exported functions, and connects to browser APIs. It may be generated automatically.

### JIT and AOT

**Just-in-time** compilation translates code around execution time. **Ahead-of-time** compilation performs translation earlier. Wasm runtimes may use either or a mixture.

### Linear memory

The byte-addressable memory region used by many Wasm programs for stacks, heaps, strings, arrays, and objects.

### Native executable

A binary built for a particular operating system, processor architecture, and binary interface, such as a Windows `.exe` or a Mach-O executable on macOS.

### Origin

A web security identity based mainly on a URL’s scheme, host, and port—for example, `https://example.com`. Browser storage, permissions, script access, and network rules often depend on origins.

### PWA

**Progressive Web App.** A web application with metadata and optional offline/background facilities that can often be installed or launched in an app-like way.

### Runtime

Software that validates, loads, and executes code while providing an environment around it. Browsers contain Wasm runtimes; standalone Wasm runtimes also exist.

### Sandbox

An isolation boundary intended to constrain code. Wasm code reaches host resources through supplied interfaces rather than unrestricted direct system calls.

### Secure context

A browser context considered trustworthy enough for sensitive Web APIs, normally an HTTPS origin. Loopback addresses such as `http://127.0.0.1` and `http://localhost` receive special treatment for development.

### Service worker

An event-driven browser worker associated with a web origin. It can intercept requests and manage cached resources for offline behavior, but the browser may stop it when idle; it is not a continuously running background service.

### Target triple

A compiler label describing a build target. Rust’s `wasm32-unknown-unknown` is a common minimal browser-oriented Wasm target. Rust also has WASI-oriented targets such as `wasm32-wasip1` and `wasm32-wasip2`.

### Virtual machine

In this context, an abstract instruction machine implemented by a host. A Wasm VM is not necessarily a complete virtualized PC or guest operating system.

### WASI

**WebAssembly System Interface.** A family of standardized host interfaces for capabilities such as files, clocks, randomness, streams, networking, and command-style operation. WASI is not itself a runtime.

### Wasm

The conventional abbreviation for WebAssembly. It refers to the standardized instruction/module ecosystem, not only to browser use.

### WAT

**WebAssembly Text format.** A human-readable representation used for inspection, learning, testing, and tooling.

### Web API

An interface supplied by browsers, such as DOM, File, Web Crypto, Canvas, Worker, Fetch, or Web Audio APIs.

### Web worker

A browser execution context separate from the main page thread. It is commonly used to keep a CPU-heavy Wasm job from freezing the interface.

### WIT

**Wasm Interface Type.** The language used to define Component Model interfaces and “worlds” of imports and exports.

---

## Recommended learning path

### Step 1: Learn the host/module distinction

Be able to say: “Wasm performs computation; the host supplies access to the outside world.” This prevents most beginner misconceptions.

### Step 2: Build a pure Rust library

Write a function that accepts a byte slice or ordinary value and returns a result without opening files, printing, starting threads, or accessing the network. Test it natively first.

### Step 3: Expose one function to the browser

Use maintained Rust/Wasm binding documentation. Pass a string or byte array, call one function, and display its result.

### Step 4: Learn browser files and workers

Build a file-picker app. Then move its CPU work into a worker. Observe how data is copied or transferred and what happens when the user cancels.

### Step 5: Measure

Compare:

- plain JavaScript;
- Rust/Wasm on the main thread;
- Rust/Wasm in a worker;
- native Rust for the same core.

Measure startup, complete task time, peak memory, and responsiveness.

### Step 6: Add offline behavior carefully

Learn service-worker caching, versioning, and secure-context requirements. Test first visit, repeat visit, update, offline start, storage clearing, and partial cache failures.

### Step 7: Explore WASI separately

Compile a command-style Rust example for a documented P1 or P2 Rust target and run it in a compatible runtime with a narrowly granted directory. Notice that this host model differs from browser APIs. Treat P3 support as a separately checked toolchain/runtime choice rather than assuming the newest milestone is universally available.

### Step 8: Share a real core

Create browser and native adapters around one Rust library. Use the same test vectors and confirm byte-for-byte compatible results.

### Step 9: Study threat modeling and release engineering

If the app processes secrets or valuable files, learning the instruction format is the easy part. Study input hardening, dependency security, cryptographic protocol design, signing, reproducible builds, updates, recovery, and incident response.

---

## Final recommendations

For the specific goal of **instant personal apps that run locally, need no traditional installer, and move easily among operating systems**, the best default is:

1. Build a normal accessible web interface in HTML and CSS.
2. Use a small TypeScript/JavaScript layer for browser events and Web APIs.
3. Put substantial, reusable, byte-oriented, or CPU-heavy logic in a pure Rust library.
4. Compile that engine to Wasm for the browser.
5. Run heavy jobs in a worker.
6. Let the user explicitly select inputs and outputs.
7. Add tested PWA caching if offline reuse matters.
8. Publish an offline, versioned package when code pinning matters.
9. Provide a native Rust CLI for very large, automated, or high-assurance workflows.
10. Keep both front ends compatible through shared formats and test vectors.

Use **plain TypeScript** when the app is mostly interface and browser coordination.

Use **browser Rust/Wasm** when the app’s heart is a portable computation and link-like distribution is the main advantage.

Use **standalone WASI** when a compatible runtime is acceptable and sandboxed command/component portability is the goal.

Use **native Rust** when the app’s heart is the operating system: filesystems, devices, processes, background execution, crash-safe output, or deep integration.

The most important conclusion is not that Wasm is better or worse than native software:

> WebAssembly is a portable, sandboxed computation engine. The browser turns it into an unusually convenient no-install application platform. Native Rust remains the stronger tool when direct control of the computer is the essential requirement.

---

## One-page recap

```text
WHAT WASM IS
    A portable binary instruction format and execution model.

WHAT RUST IS
    A source language that can compile to native code or Wasm.

WHAT WASM IS NOT
    JavaScript, an operating system, a GUI toolkit, an installer,
    or a universally double-clickable desktop executable.

WHY THE BROWSER MATTERS
    It is the commonly available host that supplies UI, files,
    graphics, network, permissions, caching, and a Wasm engine.

BEST BROWSER WASM APPS
    Explicit input → substantial computation → explicit output.
    Converters, parsers, image/audio tools, compression, emulators,
    calculators, simulations, local analysis, and selected-file tools.

BEST WASI USES
    Sandboxed commands, plugins, components, server and embedded work
    where a compatible runtime is available.

WHEN NATIVE RUST WINS
    Broad filesystem access, huge streams, atomic output, background
    services, devices, process launching, and deep OS integration.

BEST GENERAL ARCHITECTURE
    Shared pure Rust core
        ├── browser adapter compiled to Wasm
        └── native CLI or desktop adapter

RULE OF THUMB
    Wasm for portable computation.
    Native Rust for operating-system integration.
```

---

## Official references

These references were checked during the technical review dated at the top of this guide. Web platform, Rust target, Component Model, and WASI details continue to evolve, so check current compatibility before starting or releasing a project.

### WebAssembly fundamentals and specifications

- [WebAssembly official site](https://webassembly.org/) — concise definition and high-level properties.
- [WebAssembly high-level goals](https://webassembly.org/docs/high-level-goals/) — portability, efficiency, web integration, and non-web embeddings.
- [WebAssembly portability](https://webassembly.org/docs/portability/) — explains that core Wasm has imports rather than built-in APIs or syscalls.
- [WebAssembly security model](https://webassembly.org/docs/security/) — sandbox goals, memory/control-flow rules, and limitations.
- [WebAssembly use cases](https://webassembly.org/docs/use-cases/) — browser and non-browser application categories.
- [WebAssembly specifications index](https://webassembly.org/specs/) — core, JavaScript, Web, and WASI specifications.
- [WebAssembly Core module structure](https://webassembly.github.io/spec/core/syntax/modules.html) — core imports, exports, memories, tables, globals, tags, and functions.
- [WebAssembly non-web embeddings](https://webassembly.org/docs/non-web/) — Wasm without a JavaScript VM and the role of non-web hosts.

### Rust and browser Wasm

- [Rust `wasm32-unknown-unknown` target documentation](https://doc.rust-lang.org/stable/rustc/platform-support/wasm32-unknown-unknown.html) — explains the minimal target and unavailable OS-dependent standard-library behavior.
- [Rust platform support](https://doc.rust-lang.org/rustc/platform-support.html) — current target list and support tiers.
- [Rust `wasm32-wasip3` target documentation](https://doc.rust-lang.org/stable/rustc/platform-support/wasm32-wasip3.html) — current experimental/Tier 3 limitations.
- [`wasm-bindgen` guide](https://wasm-bindgen.github.io/wasm-bindgen/) — maintained documentation for Rust/JavaScript/browser bindings.
- [`wasm-pack` quick start](https://wasm-bindgen.github.io/wasm-pack/book/quickstart.html) — setup and the browser-oriented build command used in the example.
- [MDN WebAssembly documentation](https://developer.mozilla.org/en-US/docs/WebAssembly) — browser-oriented guides and API reference.

### WASI and components

- [WASI.dev introduction](https://wasi.dev/) — current WASI overview, runtimes, and capability model.
- [WASI releases](https://wasi.dev/releases) — P1, P2, and P3 status and binary-model distinctions.
- [WebAssembly Component Model guide](https://component-model.bytecodealliance.org/) — components, interfaces, WIT, composition, and language support.
- [Component Model concepts](https://component-model.bytecodealliance.org/design/component-model-concepts.html) — modules, components, hosts, platforms, and WASI relationships.
- [WIT reference](https://component-model.bytecodealliance.org/design/wit.html) — the Wasm Interface Type language.

### Browser application capabilities

- [MDN File API](https://developer.mozilla.org/en-US/docs/Web/API/File_API) — files selected or dropped by the user.
- [MDN Web Workers API](https://developer.mozilla.org/en-US/docs/Web/API/Web_Workers_API) — moving computation away from the interface thread.
- [MDN Progressive Web Apps](https://developer.mozilla.org/en-US/docs/Web/Progressive_web_apps) — installability, service workers, and offline operation.
- [MDN `showOpenFilePicker`](https://developer.mozilla.org/en-US/docs/Web/API/Window/showOpenFilePicker) — advanced file-handle API and compatibility considerations.
- [File System Access specification](https://wicg.github.io/file-system-access/) — evolving specification for user-approved file and directory access.
- [Service Workers specification](https://www.w3.org/TR/service-workers/) — lifecycle and network/cache interception model.
- [Python `http.server` documentation](https://docs.python.org/3/library/http.server.html) — the loopback-bound development server used in the tiny example.

### Cryptography

- [W3C Web Cryptography API](https://www.w3.org/TR/WebCryptoAPI/) — secure randomness, key objects, and low-level cryptographic operations.
- [W3C cryptography usage guidance](https://www.w3.org/TR/security-guidelines-cryptography/) — current standards-oriented guidance and cautions; check its publication status and updates.

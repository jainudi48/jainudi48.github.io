---
layout: post
title: "Chapter 1 — What is Python?"
date: 2025-05-29
categories: [python, software-engineering, learning]
tags: [python, cpython, internals, jit, gil, numba, mypyc, pypy, performance, beginners]
description: "The mental model you need before writing a single line of Python — execution pipeline, memory management, the GIL, speed tools, and the packaging ecosystem."
---

Most people start learning Python without understanding what Python actually is.

They write code, it runs, and they assume it's "just interpreted." Then they hit a performance wall, hear "Python is slow," and have no mental model for why — or what to do about it.

Before writing a single line of Python code, here's what you should actually know:

**Python is not just interpreted.**
It compiles your `.py` file to bytecode first, then a virtual machine (PVM) executes that bytecode. CPython and PVM are not the same thing — one is the whole program, the other is just the execution engine inside it.

**"Python is slow" is a half-truth.**
NumPy, Pandas, TensorFlow — the libraries you'll actually use for heavy computation — are written in C. You're already running near-native speed. The slowness is in pure Python loops, not the ecosystem.

**When you do need more speed, you have precise tools:**
- Numba: surgically JIT-compile numerical hotspots with one decorator
- mypyc: AOT-compile your whole module to a C extension using type hints
- PyPy: swap the runtime entirely for a tracing JIT
- multiprocessing: bypass the GIL for true CPU parallelism

**The GIL isn't the bogeyman it sounds like.**
It only hurts CPU-bound multithreaded code. I/O-bound code — most web apps, API clients, file processing — is unaffected. And Python 3.13 ships an experimental no-GIL build.

Every question that typically surfaces when learning Python internals is addressed in the chapter below — before you can even ask it.

---

# Chapter 1 — What is Python?

---

## 1. The Big Picture

Python is a high-level, dynamically typed programming language. But calling it "interpreted" is an oversimplification. Python actually compiles your code — just not to native machine code. Here is the full execution pipeline:

```
Your .py file
     ↓  [CPython Compiler — parses and compiles]
  Bytecode (.pyc files stored in __pycache__)
     ↓  [PVM — interprets bytecode]
  C functions execute
     ↓  [those C functions are already native machine code]
  Result on your CPU
```

---

## 2. CPython vs PVM — The Precise Distinction

This is where most people are sloppy. The terms are not interchangeable.

**CPython** is the whole program — the default Python implementation, written in C. It contains two components:

```
CPython
  ├── Compiler     → parses .py → compiles to bytecode
  └── PVM          → executes that bytecode
```

**PVM (Python Virtual Machine)** is just the execution engine inside CPython — a loop that reads bytecode instructions one by one and runs them.

```
Who compiles .py to bytecode?   → CPython's compiler
Who executes the bytecode?      → PVM
Who contains both?              → CPython
```

Mapping to .NET:

```
.NET SDK    →  CPython       (the whole thing)
Roslyn      →  CPython's compiler
CLR         →  PVM           (the runtime/executor)
```

---

## 3. The Compilation Stages in Detail

```
Stage 1 — Lexing (Tokenization)
  "x = 5 + 3" → [NAME:'x'] [OP:'='] [NUMBER:5] [OP:'+'] [NUMBER:3]

Stage 2 — Parsing
  Tokens → AST (Abstract Syntax Tree)
  Same concept as Roslyn in C#

Stage 3 — Compilation
  AST → Bytecode (.pyc)
  Stored in __pycache__ folder
  Not native code — still needs PVM to run

Stage 4 — Execution
  PVM reads and executes bytecode instruction by instruction
```

---

## 4. What "Interpreting" Actually Means

Interpreting does NOT mean converting to machine code. That is the most common misconception.

Interpreting means: **read one instruction, act on it directly, repeat. No translation to machine code.**

The PVM is essentially this loop in C:

```c
while (true) {
    instruction = fetch_next_bytecode_instruction();

    switch (instruction) {
        case LOAD_FAST:
            push(local_variables[operand]);
            break;
        case BINARY_ADD:
            b = pop();
            a = pop();
            push(a + b);   // calls CPython's C function for addition
            break;
        case RETURN_VALUE:
            return pop();
    }
}
```

The PVM itself is already compiled native machine code (compiled from C when CPython was installed). It reads each bytecode instruction and directly executes the corresponding C function. It never generates new machine code.

```
INTERPRETING                      COMPILING (JIT)
────────────────────────────────  ────────────────────────────────
Read instruction                  Read instructions
        ↓                                 ↓
Execute it directly               Translate to machine code
        ↓                                 ↓
Read next instruction             CPU runs that machine code
        ↓
(never produces machine code)
```

The CPU itself is a hardware interpreter of machine code. The PVM is a software interpreter of bytecode. That is why the PVM is slower — software interpretation vs hardware.

```
CPU   → hardware interpreter of machine code  (fast)
PVM   → software interpreter of bytecode      (slow)
JIT   → converts bytecode TO machine code,
        CPU then runs it directly             (fast)
```

---

## 5. Memory Management

Python manages memory in two layered mechanisms.

### Layer 1 — Reference Counting (Primary)

Every Python object has a hidden counter — refcount — tracking how many things point to it. The moment refcount hits 0, memory is freed **immediately**.

```python
x = [1, 2, 3]   # refcount = 1
y = x            # refcount = 2
del x            # refcount = 1
del y            # refcount = 0 → freed immediately
```

This is fundamentally different from C#. In C#, object destruction is non-deterministic — the GC decides when. In Python, destruction via refcount is deterministic — the moment nothing points to it, it dies.

Internally, every Python object is a C struct called PyObject:

```c
typedef struct _object {
    Py_ssize_t ob_refcnt;    // the reference count
    PyTypeObject *ob_type;   // pointer to the type
} PyObject;
```

You can inspect the refcount:

```python
import sys
x = [1, 2, 3]
print(sys.getrefcount(x))  # prints 2
# Why 2? Passing x to getrefcount() temporarily increments it
```

### The Problem — Circular References

```python
a = {}
b = {}
a['ref'] = b   # a points to b
b['ref'] = a   # b points to a
del a
del b
# Both still have refcount = 1. Neither reaches 0. Memory leak.
```

Reference counting alone cannot handle circular references.

### Layer 2 — Cyclic Garbage Collector (Secondary)

Python supplements reference counting with a cyclic GC (the `gc` module) specifically to detect and collect circular references.

It uses generational collection — same concept as .NET's GC — but only for cycle detection, not primary memory management:

```
Generation 0 → checked most frequently (new objects)
Generation 1 → checked less frequently
Generation 2 → checked least frequently (old objects)
```

```python
import gc
gc.collect()        # manually trigger cycle collection
gc.get_threshold()  # (700, 10, 10) — generation thresholds
gc.disable()        # disable if you're certain no cycles exist
```

---

## 6. The GIL — Global Interpreter Lock

### What It Is

The GIL is a mutex lock inside CPython that ensures only one thread executes Python bytecode at any moment — even across multiple CPU cores.

```
Thread 1: executing bytecode  ← GIL held here
Thread 2: waiting...
Thread 3: waiting...

(GIL releases every ~5ms or at I/O)

Thread 2: executing bytecode  ← GIL held here
Thread 1: waiting...
```

### Why It Exists

Because reference counting is not thread-safe. Without the GIL:

```
Thread 1: reads ob_refcnt = 5, about to write 6
Thread 2: reads ob_refcnt = 5, about to write 6
Thread 1: writes ob_refcnt = 6
Thread 2: writes ob_refcnt = 6
# Should be 7. Object freed too early. Crash.
```

The GIL is the price Python pays for having simple, fast reference counting.

### Real World Impact

```python
# CPU-BOUND — GIL HURTS
# Threads fight for GIL, effectively run serially
# Slower than single-threaded due to GIL contention

# I/O-BOUND — GIL does NOT hurt
# Thread releases GIL while waiting for network/disk
# True concurrency achieved because GIL is voluntarily released
```

### Escaping the GIL

```
threading         → still GIL-bound (use for I/O only)
multiprocessing   → separate processes, each has own GIL ✓
asyncio           → single-threaded cooperative concurrency (I/O)
PyPy              → different runtime
C extensions      → can release GIL manually
```

### GIL in Python 3.13+

Python 3.13 introduced experimental free-threaded mode — you can build CPython with `--disable-gil`. This is a major ongoing change in the Python ecosystem.

---

## 7. Python Runtimes

CPython is the default but not the only runtime:

| Runtime | Written In | Key Feature |
|---|---|---|
| CPython | C | Default, reference implementation |
| PyPy | Python/RPython | Tracing JIT, 5-10x faster for CPU tasks |
| Jython | Java | Runs on JVM, no GIL, true threading |
| IronPython | C# | Runs on CLR, no GIL, uses .NET GC |
| MicroPython | C | Tiny, runs on microcontrollers |
| GraalPy | Java | GraalVM polyglot, high performance |

---

## 8. Why PyPy Is Faster — and Why It's Not The Default

### How PyPy's Tracing JIT Works

CPython interprets every bytecode instruction every time. PyPy watches code as it runs and identifies hot paths — code executing repeatedly:

```
Phase 1 — Interpretation
  PyPy interprets like CPython, but profiles what runs often

Phase 2 — Tracing (loop runs ~1000 times)
  PyPy identifies this as a hot path and starts tracing it

Phase 3 — JIT Compilation
  PyPy compiles that hot path to native machine code

Phase 4 — Fast Execution
  CPU runs native code directly — no interpreter loop
```

PyPy also does type specialization — it learns from runtime behavior that a function always receives integers, and compiles a version with no type-checking overhead.

This is a Tracing JIT (selectively JITs hot paths) vs .NET's Method JIT (JITs entire methods upfront).

### Why PyPy Hasn't Replaced CPython

**Reason 1 — C Extension Incompatibility (the biggest reason)**

NumPy, Pandas, TensorFlow, scikit-learn are written in C against CPython's internal API — the PyObject struct and CPython's memory layout. PyPy has a completely different internal object model optimized for JIT. These C extensions are binary incompatible with PyPy.

The deepest irony: the scientific ecosystem that needs speed most (NumPy, Pandas, PyTorch) is the exact ecosystem PyPy can't run well.

**Reason 2 — Warmup Time**

JIT compilation has a cold start cost. PyPy takes time to trace and compile hot paths. For short scripts, CLI tools, Lambda functions — Python's most common use cases — PyPy never warms up enough to pay off.

**Reason 3 — Memory Usage**

PyPy uses 2-3x more RAM than CPython. The JIT compiler, trace data, and compiled native code all sit in memory.

**Reason 4 — The NumPy Paradox**

When you use NumPy, you're already running C code. CPython is just the orchestrator. PyPy's JIT optimizes Python bytecode — but the heavy computation isn't in Python bytecode, it's in C extensions. PyPy solves a problem scientific Python users don't actually have.

**Reason 5 — Ecosystem Momentum**

CPython is maintained by the Python Software Foundation with backing from Google, Meta, Microsoft, Amazon. PyPy has a small team with limited funding. Every tutorial, Docker image, and Stack Overflow answer assumes CPython.

---

## 9. LLVM — The Universal Compiler Backend

### The Problem LLVM Solved

Before LLVM, every language had to build its own optimizer and code generator for every CPU architecture. Building an x86 backend is enormously complex. Then ARM. Then RISC-V. Multiply by every language — massively duplicated effort.

### LLVM's Solution

LLVM provides a universal intermediate representation — LLVM IR — that any language can compile to. LLVM then handles all optimization and native code generation:

```
Python (Numba)  ─┐
C/C++ (Clang)   ─┤
Rust            ─┼──→  LLVM IR  ──→  [LLVM Optimizer]  ──→  x86
Swift           ─┤                                      ──→  ARM
Julia           ─┘                                      ──→  WebAssembly
```

### LLVM IR

LLVM IR is a low-level, strongly typed, architecture-independent assembly-like language:

```llvm
define i64 @add(i64 %a, i64 %b) {
entry:
    %result = add i64 %a, %b
    ret i64 %result
}
```

### The LLVM Pipeline

```
Stage 1 — Frontend (language-specific)
  Language compiler parses code → generates LLVM IR

Stage 2 — Middle End / Optimizer (LLVM)
  Runs optimization passes: dead code elimination,
  loop unrolling, inlining, constant folding,
  SIMD vectorization — all language-agnostic

Stage 3 — Backend (LLVM)
  Translates optimized IR → native machine code
  for target CPU architecture
```

LLVM vs .NET IL:

```
IL       → lives at runtime, JIT compiled by CLR
LLVM IR  → typically compiled ahead-of-time to native binary
           (but can also JIT — which is what Numba does)
```

---

## 10. Numba — Surgical JIT for Python

### The Problem

Pure Python loops are slow because CPython:
- Checks types of every variable every iteration
- Allocates new Python objects for every intermediate result
- Increments/decrements refcounts constantly

```python
# Slow — CPython overhead on every iteration
def sum_squares(n):
    total = 0
    for i in range(n):
        total += i * i
    return total
```

### What Numba Does

```python
from numba import jit

@jit(nopython=True)
def sum_squares(n):
    total = 0
    for i in range(n):
        total += i * i
    return total
```

On first call, Numba:
1. Inspects argument types (n is int64)
2. Generates LLVM IR specialized for int64
3. LLVM optimizes it (loop unrolling, SIMD vectorization)
4. LLVM emits native machine code
5. Caches it — all subsequent calls hit native code directly

Result: ~100x faster than pure Python for numerical loops.

### Type Specialization

```python
sum_squares(10)    # compiles version for int64
sum_squares(10.0)  # compiles separate version for float64
sum_squares(10)    # uses cached int64 — no recompilation
```

### Numba vs PyPy vs mypyc

```
PyPy    → replaces the entire Python runtime
          JITs general Python code via tracing
          incompatible with C extensions (NumPy, Pandas)
          scope: whole program

Numba   → sits on top of CPython
          only JITs functions you explicitly mark @jit
          no whole-file mode — function-level opt-in by design
          nopython=True only supports numerical/array code,
          would fail on strings, dicts, or general Python objects
          works with NumPy, compatible with CPython ecosystem
          can also compile to CUDA for GPU execution
          scope: per function

mypyc   → sits on top of CPython
          compiles entire module ahead-of-time to .so
          works on general Python (with type hints)
          scope: whole file
```

### Where Numba Works and Doesn't

```
✓ Numerical loops over arrays
✓ Mathematical computations
✓ NumPy array operations
✓ GPU computing (CUDA)

✗ General Python (strings, dicts, OOP)
✗ Pandas DataFrames (see below)
✗ Short functions called once
```

---

## 11. Why Numba Doesn't Work With Pandas

Numba's `nopython=True` mode generates code that runs completely outside the Python interpreter — no Python objects, no GIL, pure native execution.

For this to work, Numba must fully represent every object in native code at the type level. A NumPy ndarray is simply a pointer to contiguous memory + shape + dtype — Numba understands this perfectly.

A Pandas DataFrame is a deeply complex Python object graph:

```
DataFrame
  ├── BlockManager (internal storage manager)
  │     ├── numpy arrays per dtype block
  │     └── metadata dictionaries
  ├── Index objects
  └── Python dicts, lists, strings internally
```

Numba cannot represent this in native code without the Python interpreter. It cannot generate LLVM IR for it.

**The workaround:** extract the underlying NumPy arrays and pass those to Numba:

```python
@jit(nopython=True)
def compute(values):   # values is a numpy ndarray
    total = 0.0
    for v in values:
        total += v * v
    return total

result = compute(df['price'].values)  # .values extracts the ndarray
```

---

## 12. mypyc — Compiling Type-Annotated Python to C

mypyc takes type-annotated Python code and compiles it ahead-of-time to a C extension. Not every variable needs to be typed — mypyc will compile the whole file regardless; untyped parts just get no speedup.

```python
def add(x: int, y: int) -> int:
    return x + y
```

```
your_module.py  +  type hints
        ↓
      mypyc
        ↓
your_module.so / .pyd   (C extension)
        ↓
CPython imports it at near-C speed
```

mypyc is suited for pure Python libraries wanting a speed boost without rewriting in C. The mypy type checker itself is compiled with mypyc — making it 4x faster.

NumPy does not use mypyc because NumPy's core was always written in C directly, requiring manual SIMD control and BLAS integration that no Python-to-C compiler can generate automatically.

### How to Trigger mypyc — It Is Not Automatic

mypyc does **not** run when you run `python script.py`. CPython never invokes it on its own. It is an explicit ahead-of-time compilation step you run separately, before distribution or deployment.

**Install mypyc** (it ships bundled with mypy):

```bash
pip install mypy
```

**Compile a single file:**

```bash
mypyc your_module.py
# produces: your_module.cpython-311-x86_64-linux-gnu.so  (Linux/Mac)
#           your_module.pyd                               (Windows)
```

**After compilation, CPython auto-selects the compiled version:**

```
your_module.py   ← original source (still there)
your_module.so   ← compiled C extension

import your_module   # Python's import system automatically prefers .so over .py
```

You do not change any `import` statements. Python picks up the compiled extension silently.

**Wire mypyc into a build for distribution** (so users get the compiled version when they `pip install`):

```python
# setup.py
from mypyc.build import mypycify
from setuptools import setup

setup(
    name="mypackage",
    ext_modules=mypycify(["mypackage/your_module.py"]),
)
```

```bash
python setup.py build_ext --inplace  # compile in place during development
python setup.py bdist_wheel          # build distributable wheel with compiled .so inside
```

The wheel uploaded to PyPI will contain the compiled `.so`. Users who `pip install` get the fast version automatically — no mypyc needed on their machine.

```
Development workflow:
  1. Write type-annotated .py files
  2. Run: mypyc yourmodule.py         ← explicit step, not automatic
  3. Test: python -c "import yourmodule"  (imports .so silently)
  4. Ship: build wheel → .so bundled inside

CPython's role:
  CPython does NOT trigger mypyc.
  CPython only imports the .so after you've already compiled it.
```

### What Happens to Untyped Variables and Functions?

mypyc compiles the whole file — typed and untyped code alike. It does not skip untyped functions or refuse to compile.

For untyped code, mypyc falls back to treating everything as `object` — the most general Python type. The compiled code is still correct, but `object` forces dynamic dispatch on every operation, which gives roughly the same speed as CPython.

```python
def fast(x: int, y: int) -> int:   # fully typed
    return x + y                    # mypyc emits direct C int addition

def slow(x, y):                     # no annotations
    return x + y                    # mypyc treats x, y as object
                                    # still goes through Python's dynamic dispatch
                                    # ~same speed as CPython
```

```
Annotated parameter   →  mypyc knows the type at compile time
                          eliminates type checks, uses C-level operations
                          near-C speed

Unannotated parameter →  mypyc treats as `object`
                          every operation still does dynamic dispatch
                          correct, but no speedup over CPython
```

**One important distinction — mypyc runs mypy first:**

```
Type error (wrong type passed to annotated param)  →  mypyc REFUSES to compile
Missing annotation (no type hint at all)           →  mypyc compiles, falls back to object
```

Missing annotations are not type errors — mypy allows them by default. So `mypyc your_module.py` on a partially annotated file will succeed; you just won't get a speedup for the unannotated parts.

---

## 13. The Speed Problem — Full Solution Landscape

```
Problem: Python is slow for CPU-bound numerical work

CPython alone       → slow for pure Python loops
NumPy               → fast (drops into C), but only for array ops
PyPy                → fast for pure Python, breaks C extension ecosystem
Numba + LLVM        → fast for numerical Python, works with CPython + NumPy
mypyc               → compiles type-annotated pure Python to C extension
Cython              → compile Python → C, more manual, more control
C extension         → maximum control, maximum complexity
multiprocessing     → bypass GIL for CPU-bound parallelism
```

The trend is not replacing CPython but making it faster directly:

```
Python 3.11  → ~25% faster than 3.10
Python 3.12  → further ~5-10% improvement
Python 3.13  → experimental free-threaded (no-GIL) build
```

---

## 14. The Python Ecosystem

### pip and PyPI

PyPI (pypi.org) is Python's NuGet — 500,000+ packages. pip is the package installer bundled with Python 3.4+.

```bash
pip install requests            # install latest
pip install requests==2.28.0   # specific version
pip freeze > requirements.txt  # save dependencies
pip install -r requirements.txt # restore dependencies
```

### The Problem With requirements.txt

It only lists your direct dependencies — not the full dependency tree. Sub-dependencies are unlocked, causing "works on my machine" problems.

### pyproject.toml — The Modern Standard

```toml
[project]
name = "my-app"
version = "1.0.0"
requires-python = ">=3.11"
dependencies = [
    "requests>=2.28",
    "flask>=2.3",
]
```

```
requirements.txt  →  rough, manual, no sub-dependency locking
pyproject.toml    →  modern standard (PEP 517/518), project metadata + deps
poetry.lock       →  full deterministic lock of entire dependency tree
```

### Poetry — Modern Dependency Management

```bash
poetry new myproject      # scaffold new project
poetry add requests       # add + resolve full tree
poetry install            # install from lock file
poetry run python app.py  # run inside venv automatically
poetry build              # build distributable
poetry publish            # publish to PyPI
```

### venv — Virtual Environments (Critical)

In .NET, NuGet packages are isolated per project automatically. In Python, `pip install` installs globally by default — causing version conflicts across projects. `venv` creates an isolated Python environment per project.

```bash
python -m venv myenv          # create
source myenv/bin/activate     # activate (Mac/Linux)
myenv\Scripts\activate        # activate (Windows)
pip install requests          # installs only in this venv
deactivate                    # exit
```

### pyenv — Python Version Management

Multiple Python versions can coexist on one machine. pyenv manages this:

```bash
pyenv install 3.12.0    # install a version
pyenv global 3.11.0     # set system default
pyenv local 3.12.0      # set version for this folder only
```

`.python-version` file in your project root pins the Python version. Commit it to git.

### The .NET Equivalents

```
.python-version   →  global.json (SDK version pinning)
pyproject.toml    →  .csproj
poetry.lock       →  packages.lock.json
.venv/            →  bin/obj/ (git-ignored)
pip               →  dotnet CLI + NuGet
PyPI              →  NuGet Gallery
venv              →  (no direct equivalent — .NET handles this automatically)
```

### Professional Project Structure

```
myproject/
├── .python-version        ← pyenv: Python version
├── pyproject.toml         ← project metadata + dependencies
├── poetry.lock            ← full locked dependency tree
├── .venv/                 ← virtual environment (git-ignored)
├── src/
│   └── myproject/
│       ├── __init__.py
│       └── main.py
└── tests/
    └── test_main.py
```

---

## Chapter Summary

```
Python compiles .py → bytecode, then PVM interprets bytecode
CPython = compiler + PVM (the whole program)
PVM = just the execution engine inside CPython

Memory: reference counting (primary) + cyclic GC (secondary)
GIL: one thread executes bytecode at a time — protects refcounting
     I/O-bound: GIL released, threading works fine
     CPU-bound: use multiprocessing to bypass GIL

Speed solutions:
  PyPy      → tracing JIT, fast but breaks C extension ecosystem
  Numba     → LLVM-based JIT for numerical hotspots, works with CPython
  mypyc     → AOT compile type-annotated Python to C extension
  NumPy     → already C under the hood

LLVM: universal compiler backend used by Clang, Rust, Swift, Numba
      any language compiling to LLVM IR gets world-class optimization free

Ecosystem:
  pip + PyPI       → package management
  venv             → project isolation (always use it)
  pyproject.toml   → modern project config
  poetry           → full dependency + build management
  pyenv            → Python version management
```
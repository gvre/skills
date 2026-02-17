# Python 3.13 Features

**Status:** Released (October 7, 2024) | Latest: 3.13.11 (December 5, 2025)
**Requires:** Python 3.13+

This document covers features and changes specific to Python 3.13. For core Python 3.12+ features, see [SKILL.md](../SKILL.md).

---

## Overview

Python 3.13 brings significant improvements in performance, developer experience, and the type system, while removing deprecated "dead battery" modules.

**Key Highlights:**
- New and improved interactive interpreter based on PyPy's implementation
- Experimental free-threaded build mode (disables the GIL)
- Preliminary experimental JIT compiler
- Type system enhancements (TypeIs, type defaults, TypedDict improvements)
- Modified version of mimalloc memory allocator included
- Improved `locals()` builtin with well-defined semantics
- Official iOS (Tier 3) and Android (Tier 3) support
- WASI as Tier 2 supported platform
- Removal of deprecated "dead battery" modules (PEP 594)

---

## New Interactive Interpreter

### Enhanced REPL (Based on PyPy)

Python 3.13 includes a new and improved interactive interpreter based on PyPy's implementation:

**Features:**
- Multi-line editing with proper indentation
- Syntax highlighting in the terminal
- Colorized exception tracebacks
- Better auto-completion
- Improved history navigation
- Color-coded prompts

**Example:**
```python
# The new REPL provides:
>>> def greet(name: str) -> str:
...     return f"Hello, {name}!"
...
# ↑ With syntax highlighting and colorized output
```

### Colorized Exception Tracebacks

Exception tracebacks now feature color highlighting for better readability:

```python
# Tracebacks now display with:
# - Colored file paths
# - Highlighted error messages
# - Better visual distinction between stack frames
# - More intuitive error attribution
```

---

## Type System Enhancements

### TypeIs - Better Type Narrowing

`TypeIs` provides more accurate type narrowing than `TypeGuard`:

```python
from typing import TypeIs

def is_str_list(val: list[object]) -> TypeIs[list[str]]:
    """Check if all items in list are strings."""
    return all(isinstance(x, str) for x in val)

# Usage
items: list[object] = ["a", "b", "c"]
if is_str_list(items):
    # Type checker knows items is list[str] here
    result = ",".join(items)  # ✅ No type error
```

**TypeIs vs TypeGuard:**
- `TypeGuard`: Narrows the type but doesn't guarantee accuracy
- `TypeIs`: Stricter - the return value must actually match the narrowed type

### Type Defaults in Type Parameters

Python 3.13 adds support for default values in type parameters:

```python
from typing import TypeVar

T = TypeVar('T', default=int)

class Container[T = int]:
    """Container with default type parameter."""
    def __init__(self, value: T):
        self.value = value

# Uses default type (int)
container1 = Container(42)

# Explicitly specified type
container2: Container[str] = Container("hello")
```

### TypedDict Read-Only Items

Mark individual TypedDict items as read-only:

```python
from typing import TypedDict, ReadOnly

class User(TypedDict):
    name: str
    email: ReadOnly[str]  # Cannot be modified after creation
    age: int

user: User = {"name": "Alice", "email": "alice@example.com", "age": 30}
user["name"] = "Bob"  # ✅ OK
# user["email"] = "new@example.com"  # ❌ Type error
```

### Deprecation Annotations

New annotation for marking deprecations in the type system:

```python
from typing import deprecated

@deprecated("Use new_function() instead")
def old_function() -> None:
    """This function is deprecated."""
    pass
```

**Use Cases:**
- Mark deprecated functions, classes, and methods
- Provide clear migration paths
- Type checkers can warn about deprecated usage

---

## Performance Improvements

Python 3.13 includes significant performance enhancements:

### Experimental JIT Compiler

Python 3.13 introduces a preliminary, experimental Just-In-Time (JIT) compiler:

**Features:**
- Provides groundwork for significant performance improvements
- Experimental and opt-in
- Lays foundation for future optimization work

**Note:** The JIT is experimental in 3.13 and will continue to evolve in future releases.

### Free-Threaded Build Mode (Experimental)

Python 3.13 offers an experimental free-threaded build mode that disables the Global Interpreter Lock (GIL):

**Key Points:**
- Allows threads to run more concurrently
- Available as experimental feature in Windows and macOS installers
- Requires the mimalloc memory allocator
- Opens possibilities for true parallel execution in Python

```python
# With free-threaded build, CPU-bound threads can run in parallel
import threading

def cpu_intensive_task():
    # This can now run truly concurrently with other threads
    result = sum(i * i for i in range(10_000_000))
    return result

# Multiple threads can execute Python code simultaneously
threads = [threading.Thread(target=cpu_intensive_task) for _ in range(4)]
```

**Important:** This is an experimental feature. Test thoroughly before using in production.

### mimalloc Memory Allocator

A modified version of mimalloc is now included:

**Features:**
- Optional but enabled by default on supported platforms
- Required for free-threaded build mode
- Improves memory allocation performance
- Better memory usage patterns

### General Performance Improvements

```python
# Improved performance for:
# - Dictionary operations (create, lookup, update)
# - String operations (concatenation, formatting)
# - Import system (faster module loading)
# - Integer operations
# - List comprehensions
```

---

## Deprecated Module Removals (PEP 594)

### Removed Modules ("Dead Batteries")

Per PEP 594, the following deprecated modules have been removed in Python 3.13:

**Complete List of Removed Modules:**
- `aifc` - Audio Interchange File Format support
- `audioop` - Audio operations
- `chunk` - IFF chunk data
- `cgi` - Common Gateway Interface support
- `cgitb` - CGI traceback handler
- `crypt` - Unix password hashing
- `imghdr` - Image format detection
- `mailcap` - Mailcap file handling
- `msilib` - Windows Installer
- `nis` - Network Information Service
- `nntplib` - NNTP protocol client
- `ossaudiodev` - OSS audio device access
- `pipes` - Shell command pipelines
- `sndhdr` - Sound file format detection
- `spwd` - Shadow password database
- `sunau` - Sun AU sound format
- `telnetlib` - Telnet client
- `uu` - UUencode/decode
- `xdrlib` - XDR data encoding
- `lib2to3` - Python 2 to 3 conversion tool

**Note:** `asyncore`, `asynchat`, and `imp` were removed in earlier Python 3.x versions.

### Migration Guide

#### CGI Migration

```python
# Old (removed in 3.13)
import cgi

form = cgi.FieldStorage()

# New (Python 3.13+)
# Use modern web frameworks:
from fastapi import FastAPI, Form

app = FastAPI()

@app.post("/submit")
async def submit(name: str = Form(...)):
    return {"name": name}
```

#### Telnet Migration

```python
# Old (removed in 3.13)
import telnetlib

tn = telnetlib.Telnet("hostname")

# New (Python 3.13+)
# Use third-party libraries like telnetlib3
# pip install telnetlib3
```

#### Audio Format Detection Migration

```python
# Old (removed in 3.13)
import imghdr
import sndhdr

image_type = imghdr.what('image.png')
sound_type = sndhdr.what('audio.wav')

# New (Python 3.13+)
# Use third-party libraries:
# - For images: Pillow, imageio
# - For audio: pydub, soundfile
from PIL import Image

img = Image.open('image.png')
format = img.format  # 'PNG'
```

---

## Standard Library Updates

### Improved `locals()` Builtin

Python 3.13 provides well-defined semantics for the `locals()` builtin when mutating the returned mapping:

```python
def example():
    x = 1
    y = 2
    
    # locals() now has consistent behavior
    local_vars = locals()
    local_vars['z'] = 3  # Mutation behavior is now well-defined
    
    # Allows debuggers to operate more consistently
    # Better introspection and debugging capabilities
```

**Benefits:**
- Debuggers can operate more consistently
- Predictable behavior when mutating local variables
- Improved introspection capabilities
- Better support for development tools

**C API Equivalent:**
The C equivalent also has well-defined semantics, making it easier to write tools that interact with Python's local namespace.

### dbm.sqlite3 Backend

The `dbm` module now includes a new SQLite3-based backend:

```python
import dbm

# dbm.sqlite3 is now used by default when creating new files
with dbm.open('mydb', 'c') as db:
    db['key'] = 'value'
    # Stored in SQLite format by default

# Benefits:
# - Better performance for many workloads
# - More reliable than older dbm backends
# - Better cross-platform compatibility
# - ACID compliance
```

**Key Features:**
- Used by default when creating new database files
- Better reliability and performance
- Cross-platform compatibility
- Modern, actively maintained backend

### Docstring Memory Optimization

Docstrings now have their leading indentation stripped automatically:

```python
def example():
    """
    This docstring will have its leading indentation stripped.
    This reduces memory usage and .pyc file size.
    
    Most tools already handle this, but now it's built-in.
    """
    pass
```

**Benefits:**
- Reduced memory usage
- Smaller `.pyc` files
- No behavior change for most code (tools already strip indentation)

---

## Platform Support Updates

### Mobile Platform Support

**iOS - Tier 3 Support:**
- Official support for iOS as a Tier 3 platform
- Python can be embedded in iOS applications
- Better integration with iOS development tools

**Android - Tier 3 Support:**
- Official support for Android as a Tier 3 platform
- Python can be embedded in Android applications
- Enables native Android app development with Python

### WASI Platform Support

**WASI - Tier 2 Support:**
- WebAssembly System Interface (WASI) is now Tier 2 supported
- Better support for WebAssembly environments
- Improved portability and security

**Emscripten:**
- No longer officially supported by Python
- Pyodide continues to support Emscripten independently

### macOS Requirements

**Minimum macOS Version Updated:**
- Minimum supported version: macOS 10.13 (High Sierra)
- Previous minimum: macOS 10.9
- Older versions are no longer supported

**Example Use Cases:**
- Kivy applications for mobile
- BeeWare mobile apps
- Embedded Python scripting in native apps
- WebAssembly deployment via WASI

**Resources:**
- [Python Mobile Development](https://python-mobile.dev/)
- [BeeWare Project](https://beeware.org/)

---

## Configuration Updates

### pyproject.toml

```toml
[project]
requires-python = ">=3.13"

[tool.ruff]
target-version = "py313"

[tool.mypy]
python_version = "3.13"

[tool.ty]
python_version = "3.13"
```

### .python-version

```text
3.13
```

---

## Adoption Checklist

When upgrading to Python 3.13:

- [ ] Check for use of removed modules (PEP 594: aifc, cgi, cgitb, imghdr, etc.)
- [ ] Migrate removed modules to modern alternatives (see Migration Guide above)
- [ ] Update type hints to use TypeIs where beneficial
- [ ] Consider using type defaults in generic classes
- [ ] Use ReadOnly for TypedDict items that shouldn't be modified
- [ ] Update all tool configurations to target Python 3.13
- [ ] Verify macOS 10.13+ for macOS deployments
- [ ] Test thoroughly in 3.13 environment
- [ ] Review and update dependencies for 3.13 compatibility
- [ ] Update CI/CD pipelines to test against 3.13
- [ ] Consider testing experimental JIT and free-threaded mode for performance
- [ ] Enjoy improved REPL with colorized output and better error messages

---

## Resources

**Official Documentation:**
- [Python 3.13 Release Notes](https://docs.python.org/3.13/whatsnew/3.13.html)
- [Python 3.13 Documentation](https://docs.python.org/3.13/)
- [Python 3.13.11 Release](https://www.python.org/downloads/release/python-31311/)

**PEPs (Python Enhancement Proposals):**
- [PEP 719](https://peps.python.org/pep-0719/) - Python 3.13 Release Schedule
- [PEP 594](https://peps.python.org/pep-0594/) - Removing dead batteries from the standard library
- [PEP 744](https://peps.python.org/pep-0744/) - JIT Compilation
- [PEP 703](https://peps.python.org/pep-0703/) - Making the Global Interpreter Lock Optional
- [PEP 742](https://peps.python.org/pep-0742/) - TypeIs narrowing
- [PEP 696](https://peps.python.org/pep-0696/) - Type defaults for type parameters
- [PEP 705](https://peps.python.org/pep-0705/) - TypedDict: Read-only items

**Migration Guides:**
- Check third-party package compatibility before upgrading
- Review breaking changes in release notes before upgrading production systems
- Test free-threaded mode thoroughly in non-production environments first
- [Issue Tracker](https://github.com/python/cpython/issues) - Report bugs and issues

---

## Summary

Python 3.13 is a significant release focusing on:
- **Developer Experience**: New PyPy-based REPL with colorized tracebacks
- **Performance**: Experimental JIT compiler and free-threaded build mode (no GIL)
- **Type Safety**: TypeIs, type defaults, ReadOnly TypedDict items, and deprecation annotations
- **Memory**: mimalloc allocator and optimized docstring storage
- **Modernization**: PEP 594 "dead battery" removal and dbm.sqlite3 backend
- **Platform Support**: iOS and Android (Tier 3), WASI (Tier 2), macOS 10.13+ minimum
- **Standard Library**: Improved `locals()` semantics for better debugging

**Recommendation**: Upgrade to 3.13 for new projects to take advantage of the new REPL, type system improvements, and platform support. For existing projects, plan migration after thorough testing, especially if using removed modules. The experimental JIT and free-threaded mode offer exciting performance possibilities but should be tested carefully before production use.

# audio_feature_extractor

A simple audio analysis app for audio files (MP3, WAV, etc).

User uploads a file to output JSON object of extracted audio features: BPM, spectral features and more.

JSON can be downloaded as a file or copied directly from logged display.

## Setup & Installation

```
git clone https://github.com/MDW-94/audio_feature_extractor.git
```

## MVP

- take an audio file
- load it with Librosa
- extract the tempo and some spectral features
- print them out or save them to a JSON file

### MVP+

• Tempo — the BPM of a piece of audio
• Spectral features — what frequencies are dominant, essentially the “colour” of the sound
• Onset detection — where beats or events occur in time
• Mel-frequency cepstral coefficients (MFCCs) — a compact representation of the sound that’s widely used in audio ML and speech recognition

## Resources

- [Librosa](https://github.com/librosa/librosa)
- ...

## Error Log

### Librosa installation issues: M1 & LLVM

Great question. Let me break down exactly what happened and give you a mental model for debugging this class of problem yourself.

---

## What actually happened

**The dependency chain was the root cause:**

```
librosa → numba → llvmlite → LLVM (system C library)
```

llvmlite is unusual — it's not pure Python. It's a Python wrapper around LLVM, a compiler infrastructure written in C++. When there's no pre-built wheel for your exact platform, pip/uv has to _compile it from source_, which requires LLVM to be installed on your system. You didn't have it, and couldn't install it because LLVM 15 is incompatible with your macOS version (Sonoma/Sequoia).

**Why was 0.45.1 fine but 0.46/0.47 wasn't?**

PyPI hosts pre-built "wheels" (`.whl` files) for popular platform/Python combinations. llvmlite 0.45.1 happened to have a wheel for your exact setup (`cp311-cp311-macosx_10_15_x86_64`). The newer versions didn't — so pip fell back to building from source, which failed.

**Why did uv keep ignoring the working version?**

uv resolves the _entire_ dependency graph before installing anything. librosa's `numba` dependency kept pulling in newer numba versions, which in turn required newer llvmlite versions that had no wheels. Your overrides in `pyproject.toml` had syntax errors (wrong TOML types) so they were silently ignored or caused parse failures.

**Why did pip install to the wrong place?**

When you ran plain `pip install` outside of `uv run` or an activated virtualenv, it installed to the system Python at `/usr/local/lib/python3.11`, not into your project's `.venv`. uv manages its own isolated environment, so those two don't talk to each other.

---

## The mental model to carry forward

**Three layers of Python packaging:**

| Layer                   | What it is                                 | Can break when...                       |
| ----------------------- | ------------------------------------------ | --------------------------------------- |
| Pure Python             | Just `.py` files                           | Almost never                            |
| Python + C extension    | Compiled `.so`/`.dylib`, has wheels        | No wheel for your platform              |
| Python + system library | Wraps something like LLVM, OpenSSL, ffmpeg | System library missing or wrong version |

llvmlite is the third kind — the hardest. Whenever you see a package invoking `cmake` or `gcc` during install, you're in this territory.

---

## How to debug this yourself next time

**1. Read the error bottom-up**

Tracebacks are printed root-cause-last. Scroll to the very bottom first. In your case the smoking gun was always:

```
CMake Error: Could not find a package configuration file provided by "LLVM"
```

Everything above it was noise.

**2. Identify whether it's a build-from-source failure**

If you see any of these words in the error, you're compiling from source and need a system dependency:

- `cmake`, `make`, `gcc`, `clang`
- `building wheel`, `running bdist_wheel`
- `subprocess-exited-with-error`

Your first move should always be:

```bash
pip install <package> --only-binary=:all:
```

If that fails with "no matching distribution", there's genuinely no wheel for your platform and you need another approach (conda, or finding a compatible older version).

**3. Trace the dependency chain**

```bash
pip install pipdeptree
pipdeptree -p librosa
```

This shows you exactly why each package is being pulled in, so you can find the problematic link.

**4. Check what wheels actually exist before installing**

Go to `https://pypi.org/project/llvmlite/#files` — you can see every available wheel. Check whether your platform tag (`cp311-macosx_...`) is listed for the version being requested. If it's not there, you'll need to find a version that does have it.

**5. Understand your platform tag**

```bash
python -c "import platform; print(platform.platform())"
python -c "import sys; print(sys.version)"
```

The wheel filename format is `{package}-{version}-{python}-{abi}-{platform}.whl`. Matching all three of the last parts is required for a wheel to install without building.

**6. When using uv specifically**

uv resolves the full graph before installing, which is stricter than pip. When overrides aren't working, check the TOML syntax carefully — uv will sometimes warn but proceed with a broken config. Use:

```bash
uv tree          # shows the full resolved dependency tree
uv lock --dry-run  # shows what would change without doing it
```

**7. conda as a last resort for scientific packages**

For the numpy/scipy/numba/llvmlite ecosystem specifically, conda is the right tool — it manages system-level binary dependencies (like LLVM) that pip simply can't. If you ever find yourself fighting pip to install anything from that stack, switch to conda immediately and save yourself an hour.

---

The core skill is learning to distinguish _which layer_ is failing: Python resolver, wheel availability, or system dependency. Once you can identify that quickly, you'll know which tool to reach for.

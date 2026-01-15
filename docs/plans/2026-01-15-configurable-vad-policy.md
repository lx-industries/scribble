# Configurable VadPolicy Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Make VadPolicy configurable through scribble's public API so callers can tune VAD sensitivity (threshold) and other parameters.

**Architecture:** Export VadPolicy struct with Default impl, add optional policy field to Opts, wire it through VadProcessor and WhisperBackend. Callers pass custom policy via Opts; if None, use VadPolicy::default().

**Tech Stack:** Rust, scribble library

**Branching Strategy:**
- Scribble: Create `feat/configurable-vad-policy` from `friday` branch → merge into `friday` when complete

---

## Part 1: Setup

### Task 1: Create feature branch in scribble

**Step 1: Switch to friday branch and create feature branch**

```bash
cd /home/jmlx/Projects/github.com/lx-industries/scribble
git checkout friday
git pull origin friday
git checkout -b feat/configurable-vad-policy
```

**Step 2: Verify branch**

Run: `git branch --show-current`
Expected: `feat/configurable-vad-policy`

---

## Part 2: Implement Default for VadPolicy

### Task 2: Add Default impl for VadPolicy

**Files:**
- Modify: `src/vad/to_speech.rs`

**Step 1: Add Default impl for VadPolicy**

Find the `DEFAULT_VAD_POLICY` constant (around line 235-243):

```rust
/// Default policy tuned for "keep speech, drop/attenuate silence".
pub const DEFAULT_VAD_POLICY: VadPolicy = VadPolicy {
    threshold: 0.5,
    pre_pad_ms: 250,
    post_pad_ms: 250,
    min_speech_ms: 250,
    gap_merge_ms: 300,
    non_speech_gain: 0.0,
};
```

Replace with:

```rust
impl Default for VadPolicy {
    /// Default policy tuned for "keep speech, drop/attenuate silence".
    fn default() -> Self {
        Self {
            threshold: 0.5,
            pre_pad_ms: 250,
            post_pad_ms: 250,
            min_speech_ms: 250,
            gap_merge_ms: 300,
            non_speech_gain: 0.0,
        }
    }
}
```

**Step 2: Update usages of DEFAULT_VAD_POLICY in same file**

Search for `DEFAULT_VAD_POLICY` in `to_speech.rs` and replace with `VadPolicy::default()`.

**Step 3: Run tests to verify compilation**

Run: `cargo test -p scribble --lib`
Expected: PASS (or FAIL if other files use DEFAULT_VAD_POLICY)

**Step 4: Commit**

```bash
git add src/vad/to_speech.rs
git commit -m "feat(vad): implement Default for VadPolicy"
```

---

### Task 3: Update VadProcessor to use VadPolicy::default()

**Files:**
- Modify: `src/vad/processor.rs`

**Step 1: Update import and usage**

Find:
```rust
use super::to_speech::{DEFAULT_VAD_POLICY, VadPolicy, to_speech_only_with_policy};
```

Replace with:
```rust
use super::to_speech::{VadPolicy, to_speech_only_with_policy};
```

Find in `new()`:
```rust
            policy: DEFAULT_VAD_POLICY,
```

Replace with:
```rust
            policy: VadPolicy::default(),
```

**Step 2: Run tests to verify compilation**

Run: `cargo test -p scribble --lib`
Expected: PASS

**Step 3: Commit**

```bash
git add src/vad/processor.rs
git commit -m "refactor(vad): use VadPolicy::default() in VadProcessor"
```

---

## Part 3: Export VadPolicy

### Task 4: Export VadPolicy from vad module

**Files:**
- Modify: `src/vad/mod.rs`

**Step 1: Add VadPolicy to vad module exports**

In `src/vad/mod.rs`, add the re-export:

```rust
//! Voice Activity Detection (VAD) utilities.
//!
//! Scribble applies VAD (when enabled) in the high-level pipeline before passing audio into a
//! backend. This keeps backends focused on ASR and makes preprocessing behavior consistent.
//!
//! Exposes a receiver-like adapter (`VadStreamReceiver`) rather than the lower-level buffering
//! machinery. This keeps public APIs unsurprising and preserves explicit control flow at call
//! sites.

mod processor;
mod stream;
mod to_speech;

pub use processor::VadProcessor;
pub use stream::{VadStream, VadStreamReceiver};
pub use to_speech::VadPolicy;
```

**Step 2: Run tests to verify compilation**

Run: `cargo test -p scribble --lib`
Expected: PASS

**Step 3: Commit**

```bash
git add src/vad/mod.rs
git commit -m "feat(vad): export VadPolicy from vad module"
```

---

### Task 5: Export VadPolicy from lib.rs

**Files:**
- Modify: `src/lib.rs`

**Step 1: Add VadPolicy to public exports**

Find the line:
```rust
pub use crate::vad::{VadProcessor, VadStream, VadStreamReceiver};
```

Replace with:
```rust
pub use crate::vad::{VadPolicy, VadProcessor, VadStream, VadStreamReceiver};
```

**Step 2: Run tests to verify compilation**

Run: `cargo test -p scribble --lib`
Expected: PASS

**Step 3: Commit**

```bash
git add src/lib.rs
git commit -m "feat: export VadPolicy from public API"
```

---

## Part 4: Add VadPolicy to Opts

### Task 6: Add vad_policy field to Opts

**Files:**
- Modify: `src/opts.rs`

**Step 1: Import VadPolicy at top of file**

Add after existing imports:
```rust
use crate::output_type::OutputType;
use crate::vad::VadPolicy;
```

**Step 2: Add vad_policy field to Opts struct**

Add after the `emit_single_segments` field:

```rust
    /// Custom VAD policy for tuning speech detection behavior.
    ///
    /// When `None`, uses `VadPolicy::default()`. Key tunable fields:
    /// - `threshold`: VAD confidence threshold (lower = more sensitive, default 0.5)
    /// - `pre_pad_ms` / `post_pad_ms`: Padding around speech segments
    /// - `min_speech_ms`: Minimum speech duration to keep
    /// - `gap_merge_ms`: Merge segments separated by less than this gap
    ///
    /// Only used when `enable_voice_activity_detection` is `true`.
    pub vad_policy: Option<VadPolicy>,
```

**Step 3: Run tests to verify compilation**

Run: `cargo test -p scribble --lib`
Expected: FAIL (tests that construct Opts will be missing the new field)

**Step 4: Fix any test compilation errors**

Search for Opts construction in tests and add `vad_policy: None`:

Run: `grep -rn "Opts {" src/ tests/`

For each occurrence, add `vad_policy: None,` field.

**Step 5: Run tests again**

Run: `cargo test -p scribble --lib`
Expected: PASS

**Step 6: Commit**

```bash
git add src/opts.rs src/
git commit -m "feat(opts): add vad_policy field for custom VAD configuration"
```

---

## Part 5: Wire VadPolicy through VadProcessor

### Task 7: Add with_policy constructor to VadProcessor

**Files:**
- Modify: `src/vad/processor.rs`

**Step 1: Add with_policy constructor**

After the existing `new()` method, add:

```rust
    /// Create a new VAD processor with a custom policy.
    pub fn with_policy(model_path: &str, policy: VadPolicy) -> Result<Self> {
        let params = WhisperVadContextParams::default();
        let ctx = WhisperVadContext::new(model_path, params)?;
        Ok(Self { ctx, policy })
    }
```

**Step 2: Run tests to verify compilation**

Run: `cargo test -p scribble --lib`
Expected: PASS

**Step 3: Commit**

```bash
git add src/vad/processor.rs
git commit -m "feat(vad): add VadProcessor::with_policy constructor"
```

---

## Part 6: Wire VadPolicy through WhisperBackend

### Task 8: Use vad_policy from Opts in WhisperBackend

**Files:**
- Modify: `src/backends/whisper/mod.rs`

**Step 1: Update create_stream_anyhow to use opts.vad_policy**

Find the `create_stream_anyhow` method (around line 270-288), locate:

```rust
        let vad = if opts.enable_voice_activity_detection {
            let vad_processor = VadProcessor::new(&self.vad_model_path)?;
            Some(VadStream::new(vad_processor))
        } else {
            None
        };
```

Replace with:

```rust
        let vad = if opts.enable_voice_activity_detection {
            let policy = opts.vad_policy.unwrap_or_default();
            let vad_processor = VadProcessor::with_policy(&self.vad_model_path, policy)?;
            Some(VadStream::new(vad_processor))
        } else {
            None
        };
```

**Step 2: Run tests to verify compilation**

Run: `cargo test -p scribble --lib`
Expected: PASS

**Step 3: Run clippy**

Run: `cargo clippy -p scribble -- -D warnings`
Expected: No errors

**Step 4: Commit**

```bash
git add src/backends/whisper/mod.rs
git commit -m "feat(whisper): use vad_policy from Opts when creating VAD processor"
```

---

## Part 7: Merge and Push

### Task 9: Push feature branch and merge into friday

**Step 1: Push feature branch**

```bash
git push -u origin feat/configurable-vad-policy
```

**Step 2: Merge into friday branch**

```bash
git checkout friday
git merge feat/configurable-vad-policy
git push origin friday
```

**Step 3: Verify with git log**

Run: `git log --oneline -8`
Expected: Shows commits for VadPolicy configurability

---

## Summary

| Task | Description |
|------|-------------|
| 1 | Create feature branch from friday |
| 2 | Implement Default for VadPolicy |
| 3 | Update VadProcessor to use VadPolicy::default() |
| 4 | Export VadPolicy from vad module |
| 5 | Export VadPolicy from lib.rs public API |
| 6 | Add vad_policy field to Opts |
| 7 | Add VadProcessor::with_policy constructor |
| 8 | Wire vad_policy through WhisperBackend |
| 9 | Push feature branch and merge into friday |

After completion, Friday can configure VAD sensitivity like this:

```rust
use scribble::{Opts, VadPolicy};

let opts = Opts {
    enable_voice_activity_detection: true,
    vad_policy: Some(VadPolicy {
        threshold: 0.3,  // More sensitive than default 0.5
        ..VadPolicy::default()
    }),
    // ... other fields
};
```

# How to Manually Apply the Timeout Feature

If you need to use the timeout feature before the official release (v0.29.0) is merged, you can apply it as a patch to your existing ShellSpec installation.

## Prerequisites

- `curl` or `wget`
- `patch` utility (standard on most Unix-like systems)
- Access to your ShellSpec installation directory (e.g., `~/.local/lib/shellspec` or wherever it was cloned/extracted)

## Instructions

1.  **Navigate to your ShellSpec installation directory:**

    ```bash
    # Common location if installed via installer
    cd ~/.local/lib/shellspec
    # OR if you cloned the repo manually
    # cd /path/to/shellspec
    ```

2.  **Download the Patch:**

    We recommend using the pull request patch from the fork (which includes the latest bug fixes):
    
    ```bash
    # Download patch for PR #357 (Release 0.29.0)
    curl -L https://github.com/shellspec/shellspec/pull/357.patch -o shellspec-timeout.patch
    ```

3.  **Apply the Patch:**

    ```bash
    patch -p1 < shellspec-timeout.patch
    ```

4.  **Handle Critical Rejections:**

    Some hunks may be rejected. Check for critical rejections:
    
    ```bash
    find . -name "*.rej"
    ```
    
    **Critical files that need manual attention:**
    
    - `lib/core/dsl.sh.rej` - Contains the main timeout execution logic
    - `lib/core/outputs.sh.rej` - Contains the TIMEOUT output handler
    
    See the "Manual Fixes for Critical Rejections" section below.

5.  **Verify the Installation:**

    Check if the version or help output reflects the change.

    ```bash
    ./shellspec --help | grep timeout
    ```
    
    You should see:
    ```
    --timeout SECONDS           Specify the default timeout for each test [default: 60]
    --no-timeout                Disable timeout for all tests
    ```

## Manual Fixes for Critical Rejections

### lib/core/outputs.sh

If `lib/core/outputs.sh.rej` exists, add the following function to `lib/core/outputs.sh` after the `shellspec_output_NOT_IMPLEMENTED` function (around line 60):

```sh
shellspec_output_TIMEOUT() {
  shellspec_output_statement "tag:timeout" "note:TIMEOUT" "fail:y" \
    "timeout:$1" \
    "failure_message:${SHELLSPEC_LINENO:+<$SHELLSPEC_LINENO>}Test exceeded timeout" \
    "message:Test exceeded timeout of $1 seconds"
}
```

### lib/core/dsl.sh

If `lib/core/dsl.sh.rej` exists, you need to modify the `shellspec_example` function (around line 190). The key changes are:

1. **Add timeout setup** after the profile_start and before test execution:

```sh
# Timeout setup
SHELLSPEC_TIMEOUT_SIGNAL_FILE="$SHELLSPEC_STDIO_FILE_BASE.timeout_signal"
SHELLSPEC_TIMEOUT_RESULT_FILE="$SHELLSPEC_STDIO_FILE_BASE.timeout_result"
shellspec_effective_timeout="${SHELLSPEC_EXAMPLE_TIMEOUT:-${SHELLSPEC_TIMEOUT:-60}}"
shellspec_timeout_seconds=$(shellspec_parse_timeout "$shellspec_effective_timeout")

if [ "$shellspec_timeout_seconds" -gt 0 ]; then
  : > "$SHELLSPEC_TIMEOUT_SIGNAL_FILE"
  : > "$SHELLSPEC_TIMEOUT_RESULT_FILE"
fi
```

2. **Replace the test execution block** with timeout-aware version (see the `.rej` file for the full code).

**IMPORTANT:** Use `shellspec_rm -f` instead of `rm -f` for timeout file cleanup to avoid PATH issues.

## Potential Conflicts

When applying the patch to **version 0.28.1** (or earlier), you may see warnings about `Hunk #1 FAILED` or rejected hunks. This is expected due to minor version differences.

### Expected/Harmless Rejections

The following files may fail to patch cleanly, but these failures are generally **harmless** for the functionality of the timeout feature:

1.  `package.json` & `shellspec`:
    -   **Reason:** The patch attempts to bump the version to `0.29.0`.
    -   **Impact:** Your installation will remain on version `0.28.1` (or whatever you had). The actual logic for timeouts is separate and should still work.

### Known Bug Fix

The timeout feature requires these fixes to work correctly in shellspec's modified PATH environment:

1. **Watchdog PATH restoration**: The watchdog script needs to restore `SHELLSPEC_PATH` to access system utilities like `sleep` and `rm`. This should be at the top of `libexec/shellspec-timeout-watchdog.sh`:

```sh
# Restore original PATH to access system utilities like sleep, rm
# SHELLSPEC_PATH contains the original PATH before shellspec modified it
if [ "${SHELLSPEC_PATH:-}" ]; then
  PATH="$SHELLSPEC_PATH"
  export PATH
fi
```

2. **Use shellspec_rm**: In `lib/core/dsl.sh`, use `shellspec_rm -f` instead of `rm -f` for timeout file cleanup.

## Rolling Back

To revert the changes:

```bash
patch -R -p1 < shellspec-timeout.patch
rm shellspec-timeout.patch
```

## Troubleshooting

### Tests hang instead of timing out

If tests hang instead of timing out, check:

1. Is the watchdog script executable? `chmod +x libexec/shellspec-timeout-watchdog.sh`
2. Does the watchdog have the PATH restoration code? (See Known Bug Fix section)

### "rm: command not found" or "sleep: command not found"

This means the PATH restoration fix wasn't applied. Ensure:
1. `libexec/shellspec-timeout-watchdog.sh` has the SHELLSPEC_PATH restoration at the top
2. `lib/core/dsl.sh` uses `shellspec_rm -f` instead of `rm -f`

### TIMEOUT message not displayed

This means `lib/core/outputs.sh` wasn't patched correctly. Add the `shellspec_output_TIMEOUT` function manually (see Manual Fixes section).

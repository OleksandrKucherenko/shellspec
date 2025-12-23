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

    We recommend using the pull request patch.
    
    ```bash
    # Download patch for PR #357 (Release 0.29.0)
    curl -L https://github.com/shellspec/shellspec/pull/357.patch -o shellspec-timeout.patch
    ```

3.  **Apply the Patch:**

    ```bash
    patch -p1 < shellspec-timeout.patch
    ```

4.  **Verify the Installation:**

    Check if the version or help output reflects the change.

    ```bash
    ./shellspec --help | grep timeout
    ```
    
    You should see:
    ```
    --timeout SECONDS           Specify the default timeout for each test [default: 60]
    --no-timeout                Disable timeout for all tests
    ```

## Potential Conflicts

When applying the patch to **version 0.28.1** (or earlier), you may see warnings about `Hunk #1 FAILED` or rejected hunks. This is expected due to minor version differences.

### Expected/Harmless Rejections

The following files may fail to patch cleanly, but these failures are generally **harmless** for the functionality of the timeout feature:

1.  `package.json` & `shellspec`:
    -   **Reason:** The patch attempts to bump the version to `0.29.0`.
    -   **Impact:** Your installation will remain on version `0.28.1` (or whatever you had). The actual logic for timeouts is separate and should still work.

2.  `lib/core/outputs.sh`:
    -   **Reason:** Minor context differences in surrounding functions (e.g., `shellspec_output_NO_EXPECTATION`).
    -   **Impact:** The `shellspec_output_TIMEOUT` function might not be registered correctly if this fails completely. You can check the `.rej` file or manually add the function if needed, but often the patch tool is smart enough to apply the important parts.

### How to Fix Rejections

If critical logic fails to apply:

1.  Check for `.rej` files: `find . -name "*.rej"`
2.  Manually apply the missing code blocks from the `.rej` files to the corresponding source files.


## Rolling Back

To revert the changes:

```bash
patch -R -p1 < shellspec-timeout.patch
rm shellspec-timeout.patch
```

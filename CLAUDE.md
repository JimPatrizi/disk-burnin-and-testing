# CLAUDE.md

## Project Overview

**disk-burnin-and-testing** is a single-script POSIX-compliant shell tool for burn-in testing of disk drives. It sequentially runs SMART short tests, destructive `badblocks` write tests, and SMART extended tests, logging all results. It is designed for new or empty disks only — **all data on the target disk is erased**.

## Repository Structure

```
.
├── disk-burnin.sh      # Main executable script (625 lines, POSIX sh)
├── README.md           # User-facing documentation
├── .editorconfig       # Editor settings (2-space indent, UTF-8, LF)
├── license.txt         # MIT license (Keith Nash + Michael Schnerring)
└── burnin-*.log        # Example log output from a dry-run test
```

This is a single-script project. There is no build system, no package manager, and no test suite.

## Key Script: disk-burnin.sh

### Test Sequence

1. **SMART short self-test** — quick electrical/mechanical check (2–10 min)
2. **badblocks** — destructive write test with patterns `0xaa, 0x55, 0xff, 0x00` (hours to days). Skipped for SSDs.
3. **SMART extended self-test** — full surface scan (hours to days)

### Safety Mechanisms

- **Dry-run by default**: without `-f`, the script only simulates commands
- **Root required**: exits immediately if not running as root
- **Dependency check**: validates all required commands exist before starting
- **Disk existence check**: verifies the target device exists in `/dev/`

### Command-Line Options

| Flag | Purpose |
|------|---------|
| `-h` | Show help |
| `-e` | Show extended help (version history, tested hardware) |
| `-f` | Force destructive mode (disables dry-run) |
| `-b <size>` | Override badblocks block size (default: 8192) |
| `-c <count>` | Override badblocks concurrent block count (default: 64) |
| `-x` | Full badblocks pass (don't exit on first error) |
| `-o <dir>` | Log output directory (default: working directory) |

### Script Organization

1. **Pre-execution validation** — root check, dependency check
2. **Usage text** — `USAGE` and `USAGE_2` readonly constants
3. **Option parsing** — `getopts` loop
4. **Constants** — readonly variables for drive info, test durations, paths
5. **Functions** — documented helper functions
6. **`main()`** — orchestrates the full test sequence
7. **Entrypoint** — single `main` call

## Shell Coding Conventions

### General Rules

- **Shebang**: `#!/bin/sh` — strict POSIX compliance, no bash-isms
- **Validation tool**: code is checked with [shellcheck.net](https://www.shellcheck.net)
- **Indentation**: 2 spaces (per `.editorconfig`)
- **Line endings**: LF (Unix)

### Variable Conventions

- **Constants**: `UPPERCASE` names, declared with `readonly`
- **Local/temporary variables**: `l_` prefix (e.g., `l_poll_duration_seconds`)
- **All variables quoted**: `"${VARIABLE}"` — always use braces and double quotes
- **Command substitution**: `$()` syntax, never backticks

### Function Documentation

Every function uses this header format:

```sh
##################################################
# Brief description of function.
# Globals:
#   GLOBAL_VAR_READ
#   GLOBAL_VAR_MODIFIED
# Arguments:
#   arg1 - description
# Outputs:
#   Write value to stdout.
# Returns:
#   0 on success, 1 on failure.
##################################################
function_name() {
  ...
}
```

### Error Handling

- Errors go to stderr: `echo "ERROR: message" >&2`
- Error messages prefixed with `ERROR:`
- Exit code `0` = success, exit code `2` = error
- Dependency failures, permission errors, and invalid args all exit with code 2
- Critical operations use `|| exit 2` (e.g., `mkdir -p -- "${LOG_DIR}" || exit 2`)

### Patterns

- **Conditional defaults**: `[ -z "${VAR}" ] && VAR="default"`
- **POSIX arithmetic**: `$(( MINUTES * 60 ))`
- **Pipe chains for text extraction**: `printf '%s' "${VAR}" | grep "pattern" | awk '...'`
- **Dry-run wrapper**: `dry_run_wrapper()` logs commands instead of executing when in dry-run mode; uses `eval` in live mode

## Dependencies

**Required system packages:**
- `smartmontools` (provides `smartctl`)

**Required utilities (typically pre-installed):**
- `awk`, `badblocks`, `grep`, `sed`, `sleep`

**System requirements:**
- Root/sudo privileges
- SATA or SAS drives with SMART support
- On FreeBSD/FreeNAS: run `sysctl kern.geom.debugflags=0x10` first

## Platform Compatibility

**Tested operating systems:**
- FreeNAS 9.10.2-U1, 11.1-U7, 11.2-U8 (FreeBSD)
- Ubuntu Server 16.04.2 LTS
- CentOS 7.0
- Tiny Core Linux 11.1
- Fedora 33 Workstation

**Platform-specific code:**
- `cleanup_log()` uses `sed -i` on Linux vs `sed -i ''` on FreeBSD (different in-place edit syntax)

## Development Guidelines

### When Modifying the Script

1. **Maintain POSIX compliance** — no bash-specific syntax (`[[ ]]`, `local`, arrays, etc.)
2. **Validate with shellcheck** — run the script through [shellcheck.net](https://www.shellcheck.net) or `shellcheck disk-burnin.sh`
3. **Keep `readonly` discipline** — new constants must use `readonly`
4. **Use the `l_` prefix** for any temporary variables inside functions
5. **Quote everything** — all variable references must be double-quoted with braces
6. **Document new functions** using the existing header format (Globals, Arguments, Outputs, Returns)
7. **Preserve dry-run support** — any new destructive commands must go through `dry_run_wrapper()`
8. **Test on multiple platforms** if changing sed, grep, or awk behavior

### What NOT to Do

- Do not add bash-isms (this is `/bin/sh`)
- Do not remove the dry-run safety default
- Do not add dependencies without updating the `DEPENDENCIES` check and documentation
- Do not use `local` keyword (not POSIX-portable in all shells)

### Log Output

- Log files: `burnin-[MODEL]_[SERIAL].log`
- Bad blocks files: `burnin-[MODEL]_[SERIAL].bb`
- Output goes to both stdout and log file via `tee -a`
- Headers are formatted with `+-----` borders and timestamps

## Git Workflow

- Main branch: `master`
- Contributions via pull requests
- 3 main contributors: Keith Nash (KN), Michael Schnerring (MS), K. Younger (KY)
- Commit messages should be clear and descriptive
- Version history is maintained in the `USAGE_2` constant inside the script

## License

MIT License — see `license.txt`

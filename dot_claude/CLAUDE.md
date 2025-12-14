# Claude Code Best Practices

This file contains general best practices and patterns to follow across all projects.

## Python Async Patterns

### Cooperative Shutdown Pattern

**DO**: Use `asyncio.Event()` for cooperative shutdown

```python
async def monitor(stop: asyncio.Event):
    """Long-running task that checks stop event."""
    try:
        while not stop.is_set():
            # Do work
            await asyncio.sleep(1)
    except asyncio.CancelledError:
        logger.info("Monitor cancelled, cleaning up")
        raise

async def main():
    stop = asyncio.Event()

    # Setup signal handlers
    loop = asyncio.get_running_loop()
    for sig in (signal.SIGINT, signal.SIGTERM):
        try:
            # Note: Use default arg to capture loop variable!
            loop.add_signal_handler(sig, lambda s=sig: stop.set())
        except NotImplementedError:
            signal.signal(sig, lambda *_, s=sig: stop.set())

    # Start background tasks
    monitor_task = asyncio.create_task(monitor(stop), name="monitor")

    try:
        # Wait for shutdown signal
        await stop.wait()
        logger.info("Shutdown signal received")

        # Graceful shutdown with timeout
        try:
            await asyncio.wait_for(monitor_task, timeout=10)
        except TimeoutError:
            logger.warning("Task didn't finish, cancelling")
            monitor_task.cancel()
            await monitor_task  # Still await after cancel!
    finally:
        # Cleanup
        pass
```

### Task Lifecycle Management

**DO**: Always await cancelled tasks

```python
# Good
task.cancel()
try:
    await task
except asyncio.CancelledError:
    pass

# Bad - leaves task dangling
task.cancel()
```

**DO**: Name all background tasks for debugging

```python
# Good
asyncio.create_task(stream_output(proc), name="service-logs")

# Bad
asyncio.create_task(stream_output(proc))
```

**DON'T**: Use fire-and-forget without lifecycle management

```python
# Bad - no reference, can't cancel/await
asyncio.create_task(long_running_work())

# Good - store reference and manage lifecycle
self.work_task = asyncio.create_task(long_running_work())
# Later: cancel and await in cleanup
```

### Exception Handling

**DO**: Let exceptions propagate naturally

```python
# Good - exceptions bubble up
await risky_operation()

# Bad - using sys.exit() in callbacks
def callback(task):
    try:
        task.result()
    except Exception:
        sys.exit(1)  # Don't do this!
```

**DO**: Wrap tasks that need cleanup, but still re-raise

```python
async def task_wrapper(coro, name):
    try:
        return await coro
    except asyncio.CancelledError:
        raise  # Normal cancellation
    except Exception as e:
        logger.error(f"Exception in {name}: {e}", exc_info=True)
        raise  # Still propagate!
```

## Git Commit Messages

### Critical Rule: Describe What's IN the Commit

**Commit messages must describe the actual changes in the diff, NOT the user's request or Claude's instructions.**

#### Bad Examples

```
# User said: "fix the bug"
# Commit: "Fix bug"  ❌ Too vague

# User said: "add async support"
# Commit: "Add async support"  ❌ Doesn't say what changed

# Last Claude instruction: "refactor error handling"
# Commit: "Refactor error handling"  ❌ What specifically changed?
```

#### Good Examples

```
# Commit describes actual changes:
"Replace blocking requests with async httpx

- Convert get() helper to async function
- Add httpx dependency in pyproject.toml
- Update all test files to await get() calls
- Fix integration test timeout from 60s to 120s"
✅ Specific, verifiable from diff
```

### Commit Message Checklist

Before committing, verify:

1. ✅ Run `git diff --cached` to see what's actually staged
2. ✅ Each bullet point in the message corresponds to visible changes
3. ✅ No claims about functionality that isn't in the diff
4. ✅ Mention file types changed (Dockerfiles, tests, configs, etc.)
5. ✅ If fixing a bug, mention the symptom and the fix
6. ✅ If adding a feature, list the specific new capabilities

### Pattern: Review Before Committing

```bash
# Always review the actual diff first
git diff --cached --stat
git diff --cached

# Write message based on what you SEE, not what you remember doing
git commit -m "..."
```

## Pull Requests

### Critical Rule: Read Actual Commits, Not Just File Lists

**When creating PRs that span multiple commits, you MUST read the actual commit messages and bodies, not just look at file statistics.**

#### Bad Example

```bash
# ❌ WRONG - just looking at files
git diff --stat main...HEAD
# Then writing PR description based on file names
```

This fails because:
- File names don't tell you WHY changes were made
- You'll miss important context from commit messages
- You can't properly summarize the work

#### Good Example

```bash
# ✅ CORRECT - read actual commits
git log main..HEAD --format="%h %s%n%b" --no-merges

# Review each commit message and body
# Understand the progression of work
# Group related changes into themes
# Write PR description that summarizes ALL the work
```

### PR Description Checklist

Before creating a PR:

1. ✅ Run `git log main..HEAD` to see all commits since branch point
2. ✅ Read each commit message AND body (not just subject line)
3. ✅ Identify major themes across commits
4. ✅ Group related changes together in PR description
5. ✅ Include specific examples of what was added/fixed/changed
6. ✅ Mention files changed by category (Dockerfiles, tests, docs, etc.)
7. ✅ Note any breaking changes or migration steps
8. ✅ Include test results if applicable

## General Code Quality

### Logging

- Use structured logging with levels (DEBUG, INFO, WARNING, ERROR)
- Include context in log messages (service name, request ID, etc.)
- Don't log sensitive data (passwords, tokens, PII)

### Error Messages

- Include actionable information
- Mention what failed and why
- Suggest next steps when possible

### Documentation

- Keep inline comments minimal and meaningful
- Update README.md when adding features
- Document non-obvious design decisions
- Keep CLAUDE.md project files up to date

## Testing

### Critical Rule: NEVER SKIP TESTS

**If a test fails, FIX THE ROOT CAUSE. Never skip, comment out, or ignore failing tests.**

- ❌ Do NOT use `@unittest.skip()` or `@pytest.skip()` decorators
- ❌ Do NOT comment out failing tests
- ❌ Do NOT ignore test failures
- ✅ DO fix the root cause when tests fail
- Tests exist to catch bugs - skipping them hides bugs

### Mandatory Pre-Commit Checklist

Before committing ANY code changes:

- [ ] **Run linting** - MUST pass with no errors
- [ ] **Run appropriate tests** based on what changed
- [ ] **Review test output** - Verify tests PASSED, not just exit code
- [ ] **Fix all failures** - Do NOT commit with failing tests

No shortcuts. Every item must be checked before `git commit`.

### Test Organization

- Use Makefiles for common test commands
- Name tests clearly: `test_<feature>_<scenario>`
- Run tests locally before pushing to CI
- Include both happy path and error cases

### Background Command Execution (Claude Code)

**CRITICAL: For ANY command >10 seconds, use background execution with tee:**

```python
# ✅ CORRECT - Run in background with tee to file
Bash(
    command="cargo build --release 2>&1 | tee /tmp/build.log",
    description="Build release binary",
    timeout=600000,  # 10 min
    run_in_background=True
)
# Returns immediately with shell_id

# Check progress every 10 seconds (not 5, not 30!)
BashOutput(bash_id=shell_id)
# System auto-notifies when new output available

# Do other work while command runs...

# Check again in 10 seconds
BashOutput(bash_id=shell_id)

# ✅ ALSO CORRECT - SSH commands with tee
Bash(
    command='ssh user@host "cd ~/project && cargo build 2>&1 | tee /tmp/build.log"',
    run_in_background=True
)
```

**DO NOT do this:**

```python
# ❌ WRONG - Using grep/tail to filter output (costs money/time!)
Bash("cargo build 2>&1 | grep error")  # Missing output, no file saved

# ❌ WRONG - Piping output through grep (blocks until command finishes!)
Bash("sudo cargo test ... 2>&1 | grep 'pattern'")  # No output until test ends

# ❌ WRONG - Shell loops
Bash("until grep -q DONE /tmp/log; do sleep 5; done")

# ❌ WRONG - Sleep to wait
Bash("sleep 60 && cat /tmp/log")

# ❌ WRONG - Blocking synchronous run for slow commands
Bash("cargo build --release")  # Blocks for 2+ minutes, wastes money

# ❌ WRONG - Wrong polling interval
# Check every 10 seconds, not 5, not 30!
```

**For post-hoc filtering, always log to file first:**

```bash
# ✅ CORRECT - Log to file, then grep after completion
ssh host "cargo test 2>&1 | tee /tmp/test.log"
# Later:
ssh host "grep 'pattern' /tmp/test.log"

# ❌ WRONG - Pipe directly (blocks until test finishes, no incremental output)
ssh host "cargo test 2>&1 | grep 'pattern'"
```

**Why this matters:**
- Claude's background feature lets you work while tests run
- System reminders auto-notify when tasks have output
- BashOutput gives immediate status without polling
- Timeout handled by Claude, not shell commands

### Async Tests

```python
# Pattern for async tests
class MyTest(unittest.TestCase):
    def test_async_thing(self):
        asyncio.run(self._test_async_thing())

    async def _test_async_thing(self):
        result = await async_function()
        assert result == expected
```

## Stack of PRs Workflow

### Critical Rule: Work in Phases, Test Locally, Wait for Green

**When implementing multiple related changes, break work into a stack of pull requests. Each PR must be tested locally AND pass CI before proceeding.**

### Phase-Based Development Pattern

**DO**: Break large changes into logical phases

```bash
# Example: 6 phases of changes
Phase 1: Simplify configuration
Phase 2: Replace test orchestration
Phase 3: Use native logging
Phase 4: Fix version handling (depends on Phase 3)
Phase 5: Fix error handling (depends on Phase 4)
Phase 6: Split CI tests (depends on Phase 5)
```

**Why phases matter:**
- Each phase is independently reviewable
- Easier to identify which change broke something
- Can merge phases incrementally as they pass
- Parallel CI execution across multiple PRs
- Clearer commit history and easier rollback

### Mandatory Pre-Push Checklist

**CRITICAL**: Before pushing ANY branch, you MUST complete ALL checks:

```bash
# 1. Run linting
make lint
# or: uv run python3 -m ruff check

# 2. Run format check (separate from lint!)
make format-check
# or: uv run python3 -m ruff format --check

# 3. Fix any issues found
make lint-fix   # Auto-fix linting
make format     # Auto-fix formatting

# 4. Run appropriate tests locally
make test-smoke      # Quick validation
make test-app        # If you changed app code
make test-integration # If you changed infra

# 5. Verify tests PASSED (don't just check exit code!)
# Read the output - look for "OK", "PASSED", "✓"
```

**Common mistakes:**
- ❌ Running `make lint` but forgetting `make format-check` (BOTH are required!)
- ❌ Pushing without running tests locally
- ❌ Assuming tests passed without reading output
- ❌ Testing expected behavior but not testing what changed

### Wait for CI - No Early Stops

**CRITICAL**: After pushing, you MUST wait for CI to complete and go green before moving to the next phase.

**DO**: Loop and check CI status until all jobs pass

```bash
# Check all PRs in your stack
for pr in 3 4 5 6 7 8; do
  echo "PR #$pr: $(gh pr view $pr --json statusCheckRollup --jq '...')"
done

# Keep checking until all are green
# ✅ All checks must show "SUCCESS"
# ❌ Any "FAILURE" requires immediate fix
# ⏳ "PENDING" means keep waiting
```

**DON'T**: Move to next phase while CI is pending

```bash
# ❌ WRONG - pushing Phase 5 while Phase 4 CI is still running
git checkout phase5 && git push

# ✅ CORRECT - wait for Phase 4 to go green first
# Check CI status repeatedly
# Only proceed when Phase 4 shows ✅ ALL PASSED
```

**Why waiting matters:**
- Failures often reveal issues you missed locally
- Later phases may need fixes from earlier failures
- Rebasing becomes necessary if earlier phases change
- Saves time by catching issues before they compound

### Checking Multiple GitHub Jobs in Parallel

**Use `gh` CLI to check CI status across multiple PRs:**

```bash
# Quick status summary for all PRs
echo "=== CI Status Summary ==="
for pr in 3 4 5 6 7 8; do
  echo -n "PR #$pr: "
  gh pr view $pr --json number,title,statusCheckRollup --jq \
    '.title + " - " + (.statusCheckRollup | map(.conclusion) |
     if all(. == "SUCCESS") then "✅ ALL PASSED"
     elif any(. == "FAILURE") then "❌ FAILED"
     elif any(. == "PENDING" or . == null) then "⏳ PENDING"
     else "❓ UNKNOWN" end)'
done

# Detailed check results for specific PR
gh pr checks 6

# Find which jobs failed
gh pr checks 6 | grep -E "fail|FAIL"

# Watch CI in real-time
watch -n 10 'gh pr checks 6'
```

### Rebase Chain When Earlier PRs Change

**When you fix an earlier PR in the stack, ALL later PRs must be rebased:**

```bash
# Fixed Phase 4, now rebase Phase 5 and 6
git checkout phase5 && git rebase phase4 && git push -f
git checkout phase6 && git rebase phase5 && git push -f

# This triggers new CI runs for all rebased PRs
# You must wait for ALL of them to go green again
```

### Stack of PRs Summary

**The complete workflow:**

1. **Plan phases** - Break work into logical, independent chunks
2. **Implement phase** - Write code for one phase
3. **Test locally** - Run lint, format-check, AND appropriate tests
4. **Push branch** - Create PR for the phase
5. **Wait for CI** - Do NOT proceed until green ✅
6. **Fix failures** - If CI fails, fix immediately and rebase later PRs
7. **Repeat** - Only move to next phase when current phase is GREEN
8. **Merge when ready** - Merge phases independently or as a batch

**Key principle: No shortcuts, no early stops, no assumptions. Test → Push → Wait → Green → Next.**

## File Organization

### Project-Specific Instructions

- Keep a `CLAUDE.md` in project root for project-specific context
- Update it when architecture changes
- Include recent decisions and patterns
- Reference it in commit messages when following conventions

---

## fuse-pipe Tracing and Telemetry

### Overview

fuse-pipe includes a distributed tracing system for measuring end-to-end latency of FUSE operations. Traces capture timing at multiple points in the request/response pipeline to identify bottlenecks.

### Span Structure

The `Span` struct in `protocol/wire.rs` captures timing at 8 points:

```rust
pub struct Span {
    pub t0: u64,             // Client: request created
    pub server_recv: u64,    // Server: received from socket
    pub server_deser: u64,   // Server: deserialized request
    pub server_spawn: u64,   // Server: spawned handler task
    pub server_fs_done: u64, // Server: filesystem operation complete
    pub server_resp_chan: u64, // Server: response sent to channel
    pub client_recv: u64,    // Client: received response
    pub client_done: u64,    // Client: finished processing
}
```

### Latency Phases

From the span timestamps, these latency phases are computed:

| Phase | Meaning |
|-------|---------|
| `to_server` | Network: client → server |
| `server_deser` | Request deserialization |
| `server_spawn` | Task spawn overhead |
| `server_fs` | Filesystem operation (read/write/stat) |
| `server_chan` | Response channel send |
| `to_client` | Network: server → client |
| `client_done` | Client response processing |
| **total** | End-to-end latency |

### SpanCollector API

```rust
use fuse_pipe::{SpanCollector, SpanSummary};

// Create collector
let collector = SpanCollector::new();

// Record spans (automatically done by Multiplexer)
collector.record(unique_id, span);

// Get summary with percentile statistics
if let Some(summary) = collector.summary() {
    println!("p50 total: {}µs", summary.total.p50_us);
    println!("p99 total: {}µs", summary.total.p99_us);
}

// Print formatted summary to stderr
collector.print_summary();

// Export as JSON
if let Some(json) = collector.summary_json() {
    std::fs::write("telemetry.json", &json)?;
}
```

### Enabling Tracing in Tests

#### Mount with Telemetry Collection

```rust
use fuse_pipe::{mount_with_telemetry, SpanCollector};

let collector = SpanCollector::new();
mount_with_telemetry(
    "/tmp/fuse.sock",
    "/mnt/fuse",
    256,        // num_readers
    10,         // trace_rate: trace every 10th request
    Some(collector.clone())
)?;

// After unmount, get summary
collector.print_summary();
```

#### Stress Test CLI

```bash
# Run stress test with tracing (every 10th request)
cargo test --test stress --release -- -t 10

# Client subcommand with JSON output
./stress client -s /tmp/fuse.sock -m /mnt/fuse -r 256 -t 10 \
    --telemetry-output /tmp/telemetry.json
```

### trace_rate Parameter

- `0` = Tracing disabled (no overhead)
- `1` = Trace every request (highest overhead)
- `10` = Trace every 10th request (recommended for benchmarks)
- `100` = Trace every 100th request (minimal overhead)

### Sample Output

```json
{
  "count": 1024,
  "total": {"p50_us": 85, "p90_us": 110, "p99_us": 180, "max_us": 450},
  "to_server": {"p50_us": 12, "p90_us": 18, "p99_us": 25, "max_us": 45},
  "server_deser": {"p50_us": 2, "p90_us": 3, "p99_us": 5, "max_us": 12},
  "server_spawn": {"p50_us": 3, "p90_us": 8, "p99_us": 15, "max_us": 35},
  "server_fs": {"p50_us": 55, "p90_us": 75, "p99_us": 120, "max_us": 350},
  "server_chan": {"p50_us": 1, "p90_us": 2, "p99_us": 3, "max_us": 8},
  "to_client": {"p50_us": 10, "p90_us": 15, "p99_us": 22, "max_us": 40},
  "client_done": {"p50_us": 1, "p90_us": 2, "p99_us": 3, "max_us": 10}
}
```

### Key Files

- `fuse-pipe/src/telemetry.rs` - SpanCollector, SpanSummary, LatencyStats
- `fuse-pipe/src/protocol/wire.rs` - Span struct with timing fields
- `fuse-pipe/src/client/multiplexer.rs` - Records spans when trace_rate > 0
- `fuse-pipe/src/client/mount.rs` - `mount_with_telemetry()` function

### Performance Insights

From benchmarks on c6g.metal (64 ARM cores):

- **Typical total latency**: 80-120µs for small reads
- **Main bottleneck**: `server_fs` (filesystem I/O) at 50-80µs
- **Network overhead**: ~20-30µs combined (to_server + to_client)
- **Serialization**: ~5µs combined (deser + spawn + chan)

---

**Last Updated**: 2025-12-01

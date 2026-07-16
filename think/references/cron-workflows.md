# Using Think in Automated/Cron Workflows

The think skill (alpha/beta) can be loaded into cron jobs and automated agent sessions via the `skills` parameter. This gives scheduled agents structured reasoning (Wonder → Discovery → Draft → Red Team) without manual prompting.

## Loading in cron jobs

```bash
hermes cron create "0 14,0 * * *" \
  --skills think \
  --prompt "..."
```

The think skill is loaded at session start. The agent follows the Router → sizing → dispatch flow automatically. For simple scans, think routes to think-alpha. For complex analysis with verification, think routes to think-beta.

## Key adaptations for automated use

### No slash commands
`/skill think` does not work in cron sessions. The `--skills think` parameter handles loading.

### Default to think-alpha for scans
For data-gathering tasks (reading API state, comparing snapshots, producing reports), think-alpha is the right choice — it starts at Discovery (Phase 2) and produces output without excessive cycles.

### Use think gates for compliance verification
The C1/C2/C3 gates in think-beta are ideal for automated compliance checking:
- **C1 (Evidence)**: Verify claims against actual task data before escalating
- **C2 (Delta)**: Check if the same issue was flagged in a prior scan (state file)
- **C3 (Input)**: Read full task details before concluding

### State file integration
The think Draft becomes the scan report. State file read/write happens outside the think workflow — do it before loading think (read state) and after completing the Draft (write state).

### No Red Team for routine scans
For data-gathering scans, skip Red Team. It's overkill for "read state, compare, report." Reserve it for anomaly investigation where you need to verify claims.

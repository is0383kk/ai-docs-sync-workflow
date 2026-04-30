> ## Documentation Index
> Fetch the complete documentation index at: https://docs.openclaw.ai/llms.txt
> Use this file to discover all available pages before exploring further.

# `openclaw commitments`

List and manage inferred follow-up commitments.

Commitments are opt-in, short-lived follow-up memories created from
conversation context. See [Inferred commitments](/concepts/commitments) for the
conceptual guide.

With no subcommand, `openclaw commitments` lists pending commitments.

## Usage

```bash theme={"theme":{"light":"min-light","dark":"min-dark"}}
openclaw commitments [--all] [--agent <id>] [--status <status>] [--json]
openclaw commitments list [--all] [--agent <id>] [--status <status>] [--json]
openclaw commitments dismiss <id...> [--json]
```

## Options

* `--all`: show all statuses instead of only pending commitments.
* `--agent <id>`: filter to one agent id.
* `--status <status>`: filter by status. Values: `pending`, `sent`,
  `dismissed`, `snoozed`, or `expired`.
* `--json`: output machine-readable JSON.

## Examples

List pending commitments:

```bash theme={"theme":{"light":"min-light","dark":"min-dark"}}
openclaw commitments
```

List every stored commitment:

```bash theme={"theme":{"light":"min-light","dark":"min-dark"}}
openclaw commitments --all
```

Filter to one agent:

```bash theme={"theme":{"light":"min-light","dark":"min-dark"}}
openclaw commitments --agent main
```

Find snoozed commitments:

```bash theme={"theme":{"light":"min-light","dark":"min-dark"}}
openclaw commitments --status snoozed
```

Dismiss one or more commitments:

```bash theme={"theme":{"light":"min-light","dark":"min-dark"}}
openclaw commitments dismiss cm_abc123 cm_def456
```

Export as JSON:

```bash theme={"theme":{"light":"min-light","dark":"min-dark"}}
openclaw commitments --all --json
```

## Output

Text output includes:

* commitment id
* status
* kind
* earliest due time
* scope
* suggested check-in text

JSON output also includes the commitment store path and full stored records.

## Related

* [Inferred commitments](/concepts/commitments)
* [Memory overview](/concepts/memory)
* [Heartbeat](/gateway/heartbeat)
* [Scheduled tasks](/automation/cron-jobs)

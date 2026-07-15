> ## Documentation Index
> Fetch the complete documentation index at: https://code.claude.com/docs/llms.txt
> Use this file to discover all available pages before exploring further.

# Run Claude Code behind a corporate launcher

> Route the processes Claude Code starts from its own binary, including the background service and every agent view session, through a required launcher with CLAUDE_CODE_PROCESS_WRAPPER.

Some organizations require every process on a workstation to start through a mandatory launcher. The launcher applies the sandbox, network controls, or credential injection that the company's security posture depends on, and a binary that starts without it is a policy violation.

`CLAUDE_CODE_PROCESS_WRAPPER` starts every process Claude Code launches from its own binary through your launcher: the background service, every session it hosts in [agent view](/en/agent-view), and Claude Code's relaunches after an update. Set it to your launcher's absolute path, and Claude Code runs the launcher with the Claude Code command as its arguments.

A launcher that wraps the `claude` command on your `PATH` can't reach these processes, because they start from the binary's direct path without looking up `claude`.

<Note>
  `CLAUDE_CODE_PROCESS_WRAPPER` requires Claude Code v2.1.208 or later. Earlier versions ignore the variable and start every process unwrapped.
</Note>

## What the launcher covers

With `CLAUDE_CODE_PROCESS_WRAPPER` set, Claude Code starts each of the following processes through your launcher:

* The background service that `claude agents` and background sessions start on demand.
* The terminal host and the Claude Code session inside every agent view row, including the warm standby sessions the service keeps ready.
* Sessions the service respawns after an update or a crash.
* The relaunch Claude Code performs of itself to finish installing an update, including agent view's restart-for-update action.

On Windows, the variable is ignored: the launcher contract depends on `exec`, which Windows doesn't support. A Windows machine with the variable set runs every process unwrapped and keeps working, and the only signal is a warning in the [debug log](/en/troubleshooting). If your launcher policy covers Windows, the variable doesn't satisfy it there: count Windows machines as unwrapped when you plan the rollout.

### Processes that start outside the launcher

Three processes never start through the launcher:

* An [installed background service](/en/agent-view#the-supervisor-process): `launchd` or `systemd` starts that process from its unit file. `/status` and `claude daemon status` warn when this applies, and the sessions the service spawns still start through the launcher once the service restarts with the variable in its settings.
* A session you start yourself in a terminal, which runs however you invoked it. To cover these sessions, put a script named `claude` in a directory earlier on `PATH` that runs your launcher with the real binary; don't replace the managed symlink. Self-spawns don't consult `PATH`, so the two launchers never stack.
* The first process of a `claude-cli://` deep link, which the operating system's protocol handler starts directly. Everything that session starts in the background afterward runs through the launcher. To close this path entirely, [prevent handler registration](/en/deep-links#registration-and-supported-platforms) with the `disableDeepLinkRegistration` setting.

### Helper process names in process monitors

With a launcher configured, `ps` and Activity Monitor show the versioned binary name for the background helper processes instead of Claude Code's `claude bg-pty-host` and `claude bg-spare` labels, because the launcher's `exec` rebuilds the argument list. The renaming is a side effect, not concealment: the processes are otherwise unchanged, and Claude Code identifies its own processes by binary path, never by display name.

## Set up the launcher

<Steps>
  <Step title="Write the launcher script">
    Create an executable script at an absolute path, such as `/opt/corp/launcher`. Claude Code runs it with the full Claude Code command as its arguments, and the script must end by calling `exec "$@"` so it replaces itself with Claude Code:

    ```bash theme={null}
    #!/bin/sh
    # Your organization's setup: enter the sandbox, apply
    # network controls, or inject credentials.
    exec "$@"
    ```

    Make it executable with `chmod +x`. The setup portion is whatever your launcher must do before Claude Code runs; [the launcher contract](#the-launcher-contract) below lists the rules the script has to follow.

    <Note>
      If you previously replaced the `~/.local/bin/claude` symlink with your launcher, restore the original symlink in the same change. A replaced symlink makes the first wrapped session start the background service through both launchers at once, and it puts the installation into an externally managed state: `/doctor` reports it, auto-update leaves the file in place, and cleanup of old versions stays disabled until the installer manages that path again.
    </Note>
  </Step>

  <Step title="Set CLAUDE_CODE_PROCESS_WRAPPER in settings">
    Set the variable in the `env` block of a settings file so the detached background service inherits it. A shell `export` isn't enough: the background service starts on demand, outlives your shell, and never re-reads shell profiles.

    For one machine, add it to `~/.claude/settings.json`. To deploy it to every machine in your organization, put the same block in [managed settings](/en/permissions#managed-settings):

    ```json theme={null}
    {
      "env": {
        "CLAUDE_CODE_PROCESS_WRAPPER": "/opt/corp/launcher"
      }
    }
    ```

    When more than one source sets the variable, the managed settings value overrides both `~/.claude/settings.json` and a value exported in the shell, so users can't point self-spawns at a different launcher.

    Project and local settings can't set this variable. A file committed to a repository must not be able to put a binary in front of every Claude Code process on the machine, so `CLAUDE_CODE_PROCESS_WRAPPER` in `.claude/settings.json` or `.claude/settings.local.json` is ignored, with a warning in the [debug log](/en/troubleshooting).
  </Step>

  <Step title="Restart the background service and your sessions">
    A running background service and any open `claude` sessions read the variable once at startup, so they keep launching unwrapped processes until restarted. Run `claude daemon stop --any` to stop the on-demand service; the next command that needs it, such as `claude agents`, starts a wrapped one. An [installed service](/en/agent-view#the-supervisor-process) takes `claude daemon stop` without `--any`. Then restart your open `claude` sessions.

    On machines you can't restart by hand, the first session started after the settings push retires a leftover unwrapped on-demand service automatically. A machine where no new session starts keeps its unwrapped service until one does, and an installed service always needs the restart in this step.
  </Step>

  <Step title="Verify">
    Run `/status` in a session: the Self-exec entry shows the resolved launch command and warns when the running background service doesn't match it. `claude daemon status` prints the same information from the shell, including after you unset the variable, when `/status` no longer shows the entry.
  </Step>
</Steps>

## The launcher contract

When the launcher can't run, Claude Code refuses to start the process instead of starting it unwrapped. On Windows, [the variable is ignored](#what-the-launcher-covers) and processes start unwrapped. Claude Code holds the script to these rules:

* **End with `exec "$@"`.** A launcher that forks a child and exits leaves an orphaned Claude Code process the background service can't track. Agent view marks such a session failed with a message naming the launcher, and the service reaps what the launcher left behind.
* **Don't reorder, absorb, or prepend arguments.** The first argument is the Claude Code binary and everything after it is its argv.
* **Pass every inherited environment variable through to `exec`.** Adding variables, such as injected credentials, is fine; dropping inherited ones is not.
  * The per-session authentication tokens, the model and provider selection, and `CLAUDE_CODE_PROCESS_WRAPPER` itself all travel on the inherited environment, so a launcher that rebuilds it from an allow list breaks the sessions it starts, and `/status` reports a launcher mismatch.
  * If the launcher must enter a namespace or sandbox that resets the environment, re-export the inherited environment inside it verbatim.
* **Reach `exec` within about three seconds each time the launcher runs.** A cold background dispatch runs the launcher twice in series before the first byte of output, so do slow work such as a single sign-on exchange lazily or from a cache.
  * A launcher that runs far past the budget is treated as a stalled start and restarted.
* **Tolerate being invoked from inside itself.** Claude Code applies the launcher to every nested self-spawn, so a launcher that acquires an exclusive resource must detect that it already holds it.
* **Don't write to the terminal before Claude Code starts.** Anything printed before the `exec` is reported as the crash cause if the session dies before initializing.

### Format of the `CLAUDE_CODE_PROCESS_WRAPPER` value

For most launchers, the value is just the script's absolute path, like `/opt/corp/launcher`.

To pass your launcher arguments of its own, write them after the path. Claude Code parses the value as an argument list, not a shell command:

* Whitespace separates tokens, and double quotes group a token that contains spaces.
* A value that starts with `[` is read as a JSON string array, such as `["/opt/corp/launcher", "--profile", "cc"]`.
* Shell syntax doesn't work: there is no variable expansion or globbing, and an unquoted operator such as `;`, `|`, `&`, or `$(` is rejected as a configuration error rather than reinterpreted.

When the value can't be used, Claude Code refuses to start the affected process and [reports the reason](/en/errors#claude_code_process_wrapper-launcher-errors).

## Relationship to `CLAUDE_CODE_SHELL_PREFIX`

`CLAUDE_CODE_PROCESS_WRAPPER` wraps Claude Code's own processes and passes the command through as separate argv tokens for the launcher to `exec`. [`CLAUDE_CODE_SHELL_PREFIX`](/en/env-vars) wraps the shell commands Claude runs on your behalf, such as Bash tool calls, hooks, and the commands that start stdio MCP servers, and passes each one as a single shell-quoted string in `$1` for the wrapper to re-evaluate. A launcher written for one doesn't work as the other.

## Related resources

* [Agent view](/en/agent-view): the background sessions and supervisor process the launcher covers
* [Environment variables](/en/env-vars): the `CLAUDE_CODE_PROCESS_WRAPPER` reference entry
* [Managed settings](/en/permissions#managed-settings): deliver the `env` block across a fleet
* [Launcher error reference](/en/errors#claude_code_process_wrapper-launcher-errors): the refusal messages and how to recover

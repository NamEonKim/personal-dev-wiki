# DevOps Log Commands Guide

This guide collects practical, read-only commands for inspecting operating-system, SSH, Docker, and Kubernetes logs. Log locations and service names vary by distribution, version, runtime, and configuration; verify the host's setup rather than assuming one path applies everywhere.

Replace placeholders such as `<service>`, `<container>`, `<namespace>`, `<pod>`, `<container-name>`, and `<label>` with actual values, and omit the angle brackets. Commands below inspect or follow logs; they do not clear or modify them. Some sources require administrator/root privileges or membership in a log-reading group.

## Contents

- [A reliable triage sequence](#a-reliable-triage-sequence)
- [Linux](#linux)
- [Unix, BSD, and macOS](#unix-bsd-and-macos)
- [Windows](#windows)
- [SSH](#ssh)
- [Docker](#docker)
- [Kubernetes with kubectl](#kubernetes-with-kubectl)
- [Search, filtering, and safe handling](#search-filtering-and-safe-handling)
- [Official references](#official-references)

## A reliable triage sequence

1. Establish the affected host, service or workload, namespace, and approximate time window. Check the system clock and timezone if timestamps do not line up.
2. Start with the narrowest relevant source: the service journal or system event log, the container's stdout/stderr, or the Kubernetes pod and container logs.
3. Check both the current and previous container instance after a restart or crash. Compare application errors with service restarts, node issues, and Kubernetes events.
4. Expand the time range or inspect related components only when needed. Preserve the original timestamps and context when sharing findings.

## Linux

### systemd journal

`journalctl` is the command-line reader for the systemd journal, a log service used by many Linux distributions. It applies only to systems that use systemd; it is not a universal Unix, macOS, or Windows command. Think of it as a way to ask for a particular service's logs, a recent time window, or only messages at a chosen severity.

Try these common read-only queries on a systemd Linux host:

```bash
sudo journalctl -u <service> --since "1 hour ago"
sudo journalctl -u <service> -n 200 --no-pager
sudo journalctl -u <service> -f
sudo journalctl -p err --since "1 hour ago"
sudo journalctl -k --since today
```

Replace `<service>` with a systemd unit name; for example, SSH may be `ssh` on Debian/Ubuntu or `sshd` on many RHEL-family systems. `-u` selects the service, `-n` limits output to the most recent entries, `--no-pager` prints directly in the terminal, `-f` follows new messages until stopped, `--since` sets the start of the time range, `-p err` selects error-priority messages, and `-k` selects kernel messages. Use `journalctl --list-boots` to identify available boots and `journalctl -b -1` to inspect the previous boot when retained.

If the system does not use systemd, or the journal is empty/unavailable because journal storage is not configured or retained, check the distribution- and service-specific files under `/var/log/` and the service's configured logging destination. The paths below are common examples, not guaranteed locations.

### Text-based system logs

Traditional syslog files are common on non-systemd systems and may coexist with the journal. Typical locations include:

| Log purpose | Common locations (distribution-dependent) |
| --- | --- |
| General system messages | `/var/log/syslog` (Debian/Ubuntu), `/var/log/messages` (many RHEL-family systems) |
| Authentication and SSH | `/var/log/auth.log` (Debian/Ubuntu), `/var/log/secure` (many RHEL-family systems) |
| Kernel messages | `/var/log/kern.log` on some Debian-family systems; otherwise use the journal or syslog configuration |
| Service-specific logs | A configured file under `/var/log/` or the systemd journal |

Inspect available files before choosing a path:

```bash
sudo ls -l /var/log
sudo tail -n 200 /var/log/syslog
sudo tail -f /var/log/auth.log
sudo grep -i "error" /var/log/syslog
```

Use only a path that exists on the host. Rotated logs may have suffixes such as `.1` or date strings; compressed rotations often end in `.gz` and can be read with tools such as `zless` or `zgrep`.

## Unix, BSD, and macOS

There is no single Unix log path or service manager. Check the platform and configured syslog destination first.

### BSD and other Unix-like systems

FreeBSD commonly uses `/var/log/messages` for system messages and `/var/log/auth.log` for authentication, but exact files depend on `syslog.conf` and the system version. Other Unix variants may use different facilities, file names, or centralized logging.

```sh
uname -a
ls -l /var/log
tail -n 200 /var/log/messages
grep -i "error" /var/log/messages
```

Use the relevant file only if it exists. On OpenBSD, for example, authentication-related messages are commonly written to `/var/log/authlog`; consult the host's `syslog.conf` and manual pages.

### macOS

Recent macOS versions use unified logging rather than relying on traditional text files for all system messages. The `log` utility can query recent records:

```sh
log show --last 1h --style compact
log show --last 1h --style compact --predicate 'process == "sshd"'
log stream --style compact --predicate 'process == "sshd"'
```

`log show` reads retained records; `log stream` follows new records until stopped. Some records may be private or unavailable without appropriate permissions. For third-party services, also check the application's configured log destination.

## Windows

Windows system and service events are primarily available through Event Viewer and PowerShell's `Get-WinEvent`. Typical channels include **System**, **Application**, **Security**, and service-specific channels. Access to Security events is restricted.

### Event Viewer GUI

Open **Event Viewer** (`eventvwr.msc`), then select **Windows Logs** and choose **System**, **Application**, or **Security**. Filter by time, source, event level, or event ID. Check **Applications and Services Logs** for product-specific channels.

### PowerShell

```powershell
Get-WinEvent -LogName System -MaxEvents 100
Get-WinEvent -LogName Application -MaxEvents 100 |
  Select-Object TimeCreated, ProviderName, Id, LevelDisplayName, Message
Get-WinEvent -FilterHashtable @{ LogName = 'System'; StartTime = (Get-Date).AddHours(-1) }
Get-WinEvent -ListLog * |
  Select-Object LogName, RecordCount, IsEnabled
```

For a known event channel, replace its name in `-LogName`. A channel can be disabled or inaccessible, and a query may require an elevated PowerShell session.

### Windows services and files

Use **Services** (`services.msc`) to identify a service's name and status, then look for its provider in the System/Application event channels or the product's own configured log directory. Common file locations include `%ProgramData%\<vendor>\<product>\Logs` and an application's installation or data directory, but these are conventions, not universal Windows paths.

For OpenSSH on Windows, inspect **Applications and Services Logs > OpenSSH > Operational** (the PowerShell log name is commonly `OpenSSH/Operational`):

```powershell
Get-WinEvent -LogName 'OpenSSH/Operational' -MaxEvents 100
```

## SSH

SSH troubleshooting has two distinct sides: client connection diagnostics and server-side authentication/service logs.

### Client-side connection diagnostics

Run verbose SSH from the client to see name resolution, key exchange, authentication methods, and the point at which a connection fails:

```sh
ssh -v <user>@<host>
ssh -vvv -i <private-key-path> <user>@<host>
```

Use the lowest verbosity that gives enough detail. Verbose output can reveal usernames, hostnames, file paths, and authentication metadata; review it before sharing. Do not send private keys or copy private-key contents into logs or tickets.

### Server-side logs

On Linux, look for the SSH service journal first:

```bash
sudo journalctl -u sshd --since "1 hour ago"
sudo journalctl -u ssh --since "1 hour ago"
```

Use the unit that exists. If the host writes authentication events to files, common paths are `/var/log/auth.log` on Debian/Ubuntu and `/var/log/secure` on many RHEL-family systems. BSD and macOS use their own syslog or unified logging configuration; see the platform sections above.

On Windows OpenSSH Server, inspect the `OpenSSH/Operational` event channel with Event Viewer or `Get-WinEvent`. Server-side log destinations and detail levels can be configured differently; do not assume client output proves what the server recorded.

## Docker

The Docker logging driver determines where container output is stored and which retrieval methods are available. For the default `json-file` driver, use the Docker CLI rather than reading internal log files directly.

```bash
docker ps
docker logs --tail 200 --timestamps <container>
docker logs --since 1h --tail 200 --timestamps --follow <container>
docker inspect --format '{{.HostConfig.LogConfig.Type}}' <container>
docker info --format '{{.LoggingDriver}}'
```

`docker logs` normally shows the container's stdout and stderr. Applications that write only to internal files may not show useful output there. A non-default logging driver may have different retention or may not support all log retrieval options; consult that driver's documentation.

For Docker Compose:

```bash
docker compose ps
docker compose logs --tail=200 <service>
docker compose logs --since=1h --tail=200 --follow <service>
```

Avoid depending on Docker's internal storage paths: they vary by platform, driver, and daemon configuration. Do not delete or truncate log files to free space; follow the configured rotation and retention policy.

## Kubernetes with kubectl

First confirm the active cluster context and find the target namespace, pod, and container. Use the intended context; commands can return production data.

```bash
kubectl config current-context
kubectl get namespaces
kubectl get pods -n <namespace> -o wide
kubectl get pods -n <namespace> -l <label>
kubectl describe pod <pod> -n <namespace>
```

### Read and follow pod logs

```bash
kubectl logs <pod> -n <namespace>
kubectl logs <pod> -n <namespace> -c <container-name>
kubectl logs <pod> -n <namespace> --all-containers=true --since=1h --tail=200 --timestamps
kubectl logs <pod> -n <namespace> -c <container-name> --follow
```

If a pod has multiple containers, specify `-c` or use `--all-containers=true`. For a pod managed by a Deployment, first list its pods (for example with the Deployment's label selector) and then query the selected pod. This makes it clear which replica produced the output.

### Previous container instance and events

After a container restarts, inspect its previous instance if the runtime still retains it:

```bash
kubectl logs <pod> -n <namespace> -c <container-name> --previous
kubectl get events -n <namespace> --sort-by=.metadata.creationTimestamp
kubectl describe pod <pod> -n <namespace>
```

`--previous` is useful for the previous terminated container in a pod; it is not a general archive of all historical instances. Events are short-lived cluster objects and retention depends on cluster configuration. For longer retention, use the cluster's configured logging backend.

Kubernetes commonly captures container stdout/stderr through the node's container runtime, but files on the node are implementation details and may be rotated or unavailable. Prefer `kubectl logs` and the cluster's supported logging system over manually reading node paths. The current context, RBAC permissions, and workload state determine what you can inspect.

## Search, filtering, and safe handling

- Bound output with `tail`, `-n`, or a time-range option supported by the specific tool before searching large logs.
- Search terms are case-sensitive unless you request otherwise; inspect surrounding lines before concluding that a match is an error.
- Compare timestamps across systems using the correct timezone and clock. Preserve timezone information when sharing excerpts.
- Log access may require `sudo`, an elevated PowerShell session, or membership in groups such as `systemd-journal` or `adm`. Request only the access needed.
- Logs can contain tokens, session identifiers, personal data, internal hostnames, IP addresses, and request payloads. Redact sensitive values before sharing, and follow your organization's access, retention, and incident-response policies.
- Do not clear, truncate, or delete logs during routine triage. That can destroy evidence and may violate retention requirements. Follow approved rotation, retention, and collection procedures instead.

## Official references

- [systemd journalctl manual](https://www.freedesktop.org/software/systemd/man/latest/journalctl.html)
- [Microsoft: Get-WinEvent](https://learn.microsoft.com/powershell/module/microsoft.powershell.diagnostics/get-winevent)
- [OpenSSH manual pages](https://www.openssh.com/manual.html)
- [Docker: View container logs](https://docs.docker.com/engine/logging/)
- [Kubernetes: Logging architecture](https://kubernetes.io/docs/concepts/cluster-administration/logging/)
- [Kubernetes: kubectl logs](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_logs/)

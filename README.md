# fluentbit_config

An Ansible role that installs and configures [Fluent Bit](https://fluentbit.io/) on
RHEL-family Linux and Windows Server targets, with log harvesting and
[Grafana Loki](https://grafana.com/oss/loki/) output.

## Features

- Installs Fluent Bit from the official package repository (Linux) or EXE installer (Windows)
- Configures multi-path log harvesting with per-path parser and severity filtering
- Forwards qualifying log lines to a Grafana Loki endpoint
- Filesystem buffering for Loki outage resilience
- Fully idempotent — zero changed tasks on re-run
- Molecule-tested on Rocky Linux 9

## Requirements

- Ansible 2.15+
- `ansible.posix` collection: `ansible-galaxy collection install ansible.posix`
- Target Linux: RHEL 8/9, CentOS Stream, Rocky Linux 8/9, or AlmaLinux 8/9
- Target Windows: Windows Server 2019+ with WinRM enabled

## Installation

Clone this repository alongside your playbooks:

```bash
git clone https://github.com/mgagliardi/fluentbit-config.git roles/fluentbit_config
```

Or add to `requirements.yml`:

```yaml
roles:
  - name: fluentbit_config
    src: https://github.com/mgagliardi/fluentbit-config
    version: main
```

## Role Variables

| Variable | Default | Description |
|---|---|---|
| `fluentbit_version` | `"latest"` | Fluent Bit package version |
| `fluentbit_inputs` | `[]` | **Required.** List of log input path definitions |
| `fluentbit_loki_host` | `""` | **Required.** Loki hostname or IP |
| `fluentbit_loki_port` | `3100` | Loki HTTP port |
| `fluentbit_loki_labels` | `"job=fluentbit"` | Static labels for all log streams |
| `fluentbit_loki_tenant_id` | `""` | Loki tenant ID (leave empty for single-tenant) |
| `fluentbit_buffer_path` | `/var/lib/fluent-bit/buffer` (Linux) | Filesystem buffer directory |
| `fluentbit_buffer_total_limit` | `"1G"` | Max disk space for buffer |
| `fluentbit_buffer_backlog_mem_limit` | `"5M"` | Max memory for backlog chunks |
| `fluentbit_service_log_level` | `"info"` | Fluent Bit internal log level |
| `fluentbit_flush_interval` | `5` | Seconds between flush cycles |
| `fluentbit_repo_baseurl` | `"https://packages.fluentbit.io/..."` | Override RPM repo URL |
| `fluentbit_repo_gpgkey` | `"https://packages.fluentbit.io/fluentbit.key"` | RPM repo GPG key |
| `fluentbit_windows_installer_url` | `""` | Override EXE download URL |
| `fluentbit_windows_install_dir` | `C:\fluent-bit` | Windows install directory |
| `fluentbit_custom_parsers` | `[]` | Custom parser definitions |

### Input Path Definition

Each entry in `fluentbit_inputs` accepts:

| Field | Required | Default | Description |
|---|---|---|---|
| `path` | Yes | — | File path or glob to harvest |
| `tag` | Yes | — | Fluent Bit tag for this input |
| `parser` | No | `syslog-rfc3164` | Parser name |
| `severity_field` | No | `severity` | Field name containing severity string |
| `min_severity` | No | `warn` | Minimum severity to forward (`debug`, `info`, `warn`, `error`, `critical`) |
| `multiline` | No | `false` | Enable multiline parsing |
| `multiline_parser` | No | `""` | Multiline parser name |

## Example Playbooks

### Minimal (single log path)

```yaml
- name: Deploy Fluent Bit
  hosts: log_shippers
  roles:
    - role: fluentbit_config
      vars:
        fluentbit_loki_host: loki.internal.example.com
        fluentbit_inputs:
          - path: /var/log/myapp/*.log
            tag: myapp
            parser: json
            severity_field: level
            min_severity: warn
```

### Full (multiple inputs, custom parser, tuned buffer)

```yaml
- name: Deploy Fluent Bit (production)
  hosts: all
  roles:
    - role: fluentbit_config
      vars:
        fluentbit_loki_host: loki.internal.example.com
        fluentbit_loki_port: 3100
        fluentbit_loki_labels: "job=fluentbit,env=production"
        fluentbit_flush_interval: 3
        fluentbit_buffer_path: /data/fluent-bit/buffer
        fluentbit_buffer_total_limit: "2G"
        fluentbit_inputs:
          - path: /var/log/myapp/*.log
            tag: myapp
            parser: json
            severity_field: level
            min_severity: warn
          - path: /var/log/nginx/access.log
            tag: nginx
            parser: nginx
            severity_field: status
            min_severity: error
        fluentbit_custom_parsers:
          - name: nginx
            format: regex
            regex: '^(?<remote>[^ ]*) .* \[(?<time>[^\]]*)\] "(?<method>\S+) (?<path>\S+) (?<protocol>\S+)" (?<status>[^ ]*)'
            time_key: time
            time_format: "%d/%b/%Y:%H:%M:%S %z"
```

## Testing

```bash
# Install test dependencies
pip install molecule molecule-plugins[docker] ansible-lint

# Run full test suite (converge + idempotency + verify + destroy)
molecule test

# Run only converge (faster iteration)
molecule converge

# Run lint
ansible-lint
```

## Tags

| Tag | Description |
|---|---|
| `fluentbit_install` | Package/installer tasks only |
| `fluentbit_configure` | Template rendering + directory creation |
| `fluentbit_service` | Service enable/start tasks |

## Supported Platforms

| Platform | Molecule Tested |
|---|---|
| RHEL 8/9 | ✅ Docker |
| Rocky Linux 8/9 | ✅ Docker |
| AlmaLinux 8/9 | ✅ Docker |
| CentOS Stream | ✅ Docker |
| Windows Server 2019+ | ⚠ Delegated |

## License

MIT

# Log Analyzer

[![CI](https://github.com/YOUR-USERNAME/log-analyzer/actions/workflows/ci.yml/badge.svg)](https://github.com/YOUR-USERNAME/log-analyzer/actions)
![Python](https://img.shields.io/badge/python-3.9%2B-blue)
![License](https://img.shields.io/badge/license-MIT-green)

A zero-dependency command-line tool that analyzes **Apache/Nginx access logs**, **syslog / auth logs**, and **application logs**, then produces a summary report with security alerts.

## Features

- **Auto-detects** log format (Apache/Nginx common & combined, syslog, `YYYY-MM-DD HH:MM:SS LEVEL message`)
- **Summaries**: log levels, HTTP status codes & methods, top IPs, top paths, bytes sent, activity per hour
- **Error grouping**: similar errors are collapsed (`timeout after 30s` and `timeout after 45s` count together)
- **Security alerts**: brute-force detection (repeated 401/403, SSH "Failed password") and scanner/bot detection (many 4xx)
- **Filtering** by level, time range, and regex
- **Reads `.gz`** files and multiple files at once
- **Text or JSON** output (pipe to `jq`, dashboards, etc.)
- No third-party dependencies

## Installation

```bash
git clone https://github.com/YOUR-USERNAME/log-analyzer.git
cd log-analyzer
pip install -e .
```

## Usage

```bash
loganalyzer sample_logs/access.log
loganalyzer /var/log/auth.log --threshold 3
loganalyzer app.log -l ERROR -l CRITICAL --since "2024-03-15 09:00"
loganalyzer access.log.1.gz access.log -o json > report.json
loganalyzer app.log -g "payment|timeout" -n 20
python -m loganalyzer app.log          # without installing the script
```

| Option | Description |
|---|---|
| `-f, --format` | `auto` (default), `apache`, `syslog`, `app` |
| `-l, --level` | Only include these levels (repeatable) |
| `--since`, `--until` | Time window, `YYYY-MM-DD[ HH:MM[:SS]]` (UTC) |
| `-g, --grep` | Only lines matching a regex |
| `-n, --top` | Rows in top-N tables (default 10) |
| `-t, --threshold` | Failed-auth count that triggers an alert (default 5) |
| `-o, --output` | `text` (default) or `json` |

### Example output

```
================================================================
 LOG ANALYSIS REPORT
================================================================
Lines read      : 147
Parsed          : 146  (skipped unparseable: 1)
Time range      : 2024-03-15 08:00:51  ->  2024-03-15 10:04:23
Error rate      : 8.9%

HTTP status codes
-----------------
  200        94
  401        14
  404        18
  500         4
  ...

Alerts
------
  [!] 198.51.100.23: 14 failed auth attempts (possible brute force)
  [!] 192.0.2.99: 12 client errors (possible scanner/bot)
```

## Use as a library

```python
from loganalyzer import Analyzer, Filters

a = Analyzer(Filters(levels={"ERROR"}))
a.process_file("app.log")
print(a.summary.top_errors.most_common(5))
print(a.suspicious_ips(threshold=5))
```

## Project layout

```
src/loganalyzer/
  parsers.py    # per-format line parsers + format detection
  analyzer.py   # aggregation, filters, anomaly detection
  report.py     # text / JSON rendering
  cli.py        # argparse CLI
tests/          # pytest suite
sample_logs/    # demo data (access.log, app.log, auth.log)
```

## Development

```bash
pip install -e ".[dev]"
pytest
```

## Notes & limitations

- All timestamps are normalized to UTC. Syslog lines have no year, so the current year is assumed.
- Level for syslog lines is inferred from keywords (`error`, `failed`, `warning`, ...).
- Adding a format: write a `parse_xxx(line) -> LogEntry | None` in `parsers.py` and register it in `_PARSERS` and `FORMATS`.

## License

MIT, see [LICENSE](LICENSE).

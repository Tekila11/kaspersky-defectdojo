# kaspersky-defectdojo

Imports vulnerability findings from Kaspersky Security Center (KSC) 15.x into
[DefectDojo](https://github.com/DefectDojo/django-DefectDojo), one product per managed device.

KSC already scans every managed endpoint for vulnerable applications, but that data stays inside
the Kaspersky console. This script moves it into DefectDojo, where it sits alongside findings from
every other scanner and can be triaged, deduplicated and tracked to closure in one place.

Runs unattended on a cron schedule, keeps incremental state, and is safe to re-run.

## How it works

1. Authenticates to the KSC Open API (port 13299, KSCBasic auth)
2. Fetches all managed hosts via `HostGroup.FindHosts`
3. Resolves group names via `HostGroup.GetGroupInfo`
4. Queries `v_vapm_vulnerability` per host GUID to retrieve CVE findings
5. For each host, looks up or creates a DefectDojo product (by IP first, then by name)
6. Imports findings under a "Kaspersky Vulnerability Scan" engagement

On subsequent runs `reimport-scan` is used, which closes fixed vulnerabilities and adds new ones
rather than piling up duplicates.

## DefectDojo structure

| DD object | Value |
|---|---|
| Product Type | KSC administration group name (e.g. "Servers", "IT Team") |
| Product | Host display name (`KLHST_WKS_DN`) |
| Engagement | Kaspersky Vulnerability Scan (CI/CD, permanent) |
| Scan type | Generic Findings Import |

### Product deduplication

A machine often already has a DefectDojo product created by another importer, such as a Wazuh
agent feed. Creating a second product for the same host would split its findings in two, so the
script reuses what exists:

1. Search by IP via DefectDojo endpoints, skipping products whose product type marks them as audit scope
2. Search by hostname
3. Create a new product only if neither matches

The product type is always updated to the KSC group name, except for audit products.

## Requirements

- Python 3.10 or newer
- Kaspersky Security Center 15.x with the Open API reachable (default port 13299)
- DefectDojo 2.x with an API token

### KSC service account

Role required: **Vulnerability and patch management operator**
(`Opérateur de gestion des vulnérabilités et des correctifs` on a French-language console).

The account needs a scope assignment for every group you want collected. Add one entry per group:

- Administration Server (`Serveur d'administration`)
- Managed devices (`Appareils administrés`)
- Any additional group not covered by the two above

A group with no scope assignment returns zero hosts silently rather than an error, which is the
usual reason a device is missing from DefectDojo. Run `debug_srvview.py` to list what the account
can actually see.

## Configuration

```bash
cp .env.example .env
```

| Variable | Description |
|---|---|
| `KSC_HOST` | KSC server IP or FQDN |
| `KSC_PORT` | KSC API port (default: `13299`) |
| `KSC_USERNAME` | Service account username |
| `KSC_PASSWORD` | Service account password |
| `DD_URL` | DefectDojo base URL |
| `DD_TOKEN` | DefectDojo API token |
| `DD_MIN_SEVERITY` | Minimum severity to import (default: `Low`) |
| `STATE_FILE` | Path to state file (default: `state.json`) |

## Usage

```bash
pip install -r requirements.txt
cp .env.example .env     # then fill in the values
set -a && . ./.env && set +a
python3 kaspersky_to_defectdojo.py
```

### Scheduling

```cron
0 5 * * * cd /opt/scripts/kaspersky-defectdojo && set -a && . ./.env && set +a && python3 kaspersky_to_defectdojo.py >> /var/log/kaspersky-to-dd.log 2>&1
```

## State file

`state.json` maps host GUIDs to DefectDojo test IDs:

```json
{
  "00000000-0000-0000-0000-000000000000": {
    "test_id": 42,
    "name": "WIN-SRV-01"
  }
}
```

This is what makes re-runs idempotent. Deleting it forces a full reimport from scratch.

## Debugging

`debug_srvview.py` probes the KSC API to verify connectivity, report visible host and vulnerability
counts, and identify missing group scope assignments.

```bash
set -a && . ./.env && set +a
python3 debug_srvview.py
```

| Symptom | Likely cause |
|---|---|
| Zero hosts returned | Service account has no scope assignment for that group |
| Fewer hosts than expected | A group is missing from the scope list |
| 401 from KSC | Wrong credentials, or the account lacks the vulnerability operator role |
| Findings land in a duplicate product | Host IP changed between runs; check DefectDojo endpoints |

## Related

Part of a set of integrations around DefectDojo and OpenCTI:

| Repository | Direction |
|---|---|
| [`wazuh-defectdojo`](https://github.com/Tekila11/wazuh-defectdojo) | Wazuh → DefectDojo |
| [`opencti-defectdojo`](https://github.com/Tekila11/opencti-defectdojo) | DefectDojo ↔ OpenCTI, both directions |
| [`opencti-vuln-toolkit`](https://github.com/Tekila11/opencti-vuln-toolkit) | OpenCTI connectors for CVE enrichment and scoring |

## License

[MIT](LICENSE)

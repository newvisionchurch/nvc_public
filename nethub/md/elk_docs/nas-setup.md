# NAS Setup

This project is deployed on a NAS Docker environment. Local VS Code is used for
editing, GitHub stores version history, and the NAS pulls the repo before
restarting the ELK stack.

## Active Deployment Files

- Compose file: `elk/compose.yaml`
- Logstash pipeline: `elk/logstash/pipeline/logstash.conf`
- Logstash UDP input: `51415/udp`
- Logstash file outputs: `logstash/outputs/{ap,low,noise}/YYYY/M/D/`
- Elasticsearch port: `9200`
- Kibana port: `5601`

The Compose file uses ELK `7.17.10`.

## Expected UniFi / EFG Logging Path

The EFG settings summary and PDF reference show UniFi traffic/activity logging
configured to send SIEM/syslog data to the ELK host on UDP port `51415`.

Important operational expectation:

- EFG / UniFi syslog target host: the ELK/NAS host on the management network.
- Syslog target port: `51415`.
- Logstash must continue listening on UDP `51415`.

The PDF reference also contains Traffic Logging / NetFlow screens. Some captures
show different NetFlow and Gateway DNS flow settings, so verify the live UniFi
controller before making decisions that depend on those checkboxes.

## Deployment Flow

1. Edit files locally in VS Code or Codex.
2. Review changes and avoid committing secrets or raw logs.
3. Commit and push to GitHub.
4. SSH into the NAS.
5. Pull the latest repo.
6. Restart or recreate the Docker Compose stack as needed.
7. Validate Logstash, Elasticsearch, and Kibana.

Example NAS flow:

```bash
cd /volume1/docker/nvc_network/elk   # NAS 실제 경로 확인 후 사용
git pull
docker compose up -d
docker compose ps
```

NAS의 실제 프로젝트 경로를 확인하고 위 경로를 맞게 수정하세요.

## Validation Commands

Check containers:

```bash
docker compose ps
```

Check Logstash logs:

```bash
docker logs --tail=200 logstash
```

Check Elasticsearch:

```bash
curl -u elastic:<password> http://localhost:9200/_cluster/health?pretty
```

Check recent indices:

```bash
curl -u elastic:<password> "http://localhost:9200/_cat/indices/unifi-network-*?v"
```

Do not commit real passwords into scripts or documentation.

## Logstash Config Test

When available inside the Logstash container or image:

```bash
logstash -t -f /usr/share/logstash/pipeline/logstash.conf
```

For this repo, the mounted pipeline path in Docker is:

```text
/usr/share/logstash/pipeline
```

The NAS host path is configured in `elk/compose.yaml`.

## V27 Operational Checks

After deployment, confirm in Kibana Discover:

- `event.storage_class:ap`
- `event.storage_class:low`
- `event.storage_class:noise`
- `event.problem_class:system_issue`
- `event.action:rrm_scan_trigger`
- `event.problem_class:rrm_scan_risk`
- `event.action:qos_error`
- `event.problem_class:ap_stuck`
- `event.problem_class:ap_no_service`
- `event.rca_sequence:rrm_to_ap_stuck`
- `event.rca_step:*`
- `observer.type:gateway`
- `observer.type:ap`
- `event.correlation_target:true`

For incident analysis, compare 1 minute, 5 minute, and 10 minute windows around
`rrm_scan_trigger`, `system_issue`, and AP-stuck symptom spikes.

Expected V27 index groups:

```text
unifi-network-v27-ap-*
unifi-network-v27-low-*
unifi-network-v27-noise-*
```

Expected V27 local output files inside the Logstash container:

```text
/usr/share/logstash/outputs/ap/YYYY/M/D/unifi-network-v27-ap-YYYY.MM.dd-part0001.jsonl
/usr/share/logstash/outputs/low/YYYY/M/D/unifi-network-v27-low-YYYY.MM.dd-part0001.jsonl
/usr/share/logstash/outputs/noise/YYYY/M/D/unifi-network-v27-noise-YYYY.MM.dd-part0001.jsonl
```

Example for April 28, 2026:

```text
/usr/share/logstash/outputs/ap/2026/4/28/unifi-network-v27-ap-2026.04.28-part0001.jsonl
```

On the NAS, these files appear under the host directory mounted to
`/usr/share/logstash/outputs` in `elk/compose.yaml`.
AP, low, and noise files roll to the next `partNNNN` file at approximately
50 MiB.

## Safety Notes

- Do not commit `.env`, credentials, private keys, or API tokens.
- Avoid changing Logstash input/output paths unless deployment also changes.
- If a Logstash rule changes, update `docs/logstash-rules.md`.

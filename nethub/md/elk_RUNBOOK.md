# MEMO

Copy/paste notes for NAS ELK operations.

Do not store real passwords in this file. Replace `<password>` and paths before
running commands.

## SSH / NAS Login

```bash
# Become root after SSH login.
sudo -i

# Load shell profile if needed.
source ~/.profile
```

## Docker Container Checks

```bash
# Follow Logstash logs.
sudo docker logs -f logstash

# Follow only recent Logstash logs.
sudo docker logs --tail 100 -f logstash

# Show container resource usage.
sudo docker stats

# Show running containers.
sudo docker ps
```

## Docker Compose

```bash
# Go to the ELK compose directory first.
cd /volume1/docker/elk-logstash/elk

# Restart the stack.
docker compose down
docker compose up -d

# Start or refresh running containers.
docker compose up -d

# Check compose status.
docker compose ps
```

## Logstash Config Test

```bash
# Validate the mounted Logstash pipeline inside the container.
docker exec -it logstash /usr/share/logstash/bin/logstash -t -f /usr/share/logstash/pipeline/logstash.conf
```

# Update new version of logstash.config file
docker restart -t 30 logstash
docker ps --format "table {{.Names}}\t{{.Status}}\t{{.Ports}}"
docker logs --tail 100 logstash


## Elasticsearch Checks

```bash
# List indices. Replace <password> before running.
curl -u elastic:<password> -X GET "http://localhost:9200/_cat/indices?v"

# Check cluster health.
curl -u elastic:<password> -X GET "http://localhost:9200/_cluster/health?pretty"
```

## Elasticsearch Index Cleanup

Use Kibana Dev Tools for destructive index deletes. Double-check the pattern
before running.

```text
DELETE unifi-network-v24.1*
```

V26 index groups:

```text
unifi-network-v26-ap-*
unifi-network-v26-low-*
unifi-network-v26-noise-*
```

## Logstash Debug Export

```bash
# Export focused Logstash debug logs.
cd /volume1/elk-codex-logs/logstash-debug

# Count exported files.
wc -l logstash-debug-last-*.log

# Save recent docker logs to a temp file, then count lines.
docker logs --since 5m logstash > /tmp/logstash-last-5m.log 2>&1
wc -l /tmp/logstash-last-5m.log

docker logs --since 10m logstash > /tmp/logstash-last-10m.log 2>&1
wc -l /tmp/logstash-last-10m.log

docker logs --since 30m logstash > /tmp/logstash-last-30m.log 2>&1
wc -l /tmp/logstash-last-30m.log

docker logs --since 1h logstash > /tmp/logstash-last-1h.log 2>&1
wc -l /tmp/logstash-last-1h.log
```

## Kibana Index Patterns

Current V26 patterns:

```text
unifi-network-v26-ap-*
unifi-network-v26-low-*
unifi-network-v26-noise-*
```

Older V25.2-compatible pattern:

```text
unifi-network-v25.1*
```

## Kibana Dev Tools Search

```json
GET /unifi-network-v26-ap-*/_search
{
  "track_total_hits": true,
  "size": 5000,
  "sort": [
    {
      "@timestamp": "desc"
    }
  ]
}
```

## Useful KQL

```text
event.problem_class:system_issue
event.problem_class:ap_stuck
event.problem_class.keyword:"ap_no_service"
event.action.keyword:"qos_error"
event.correlation_target.keyword:"true"
event.correlation_stage.keyword:*
event.correlation_domain.keyword:*
event.storage_class.keyword:"ap"
event.storage_class.keyword:"low"
event.storage_class.keyword:"noise"
```

## Lens Colors

```text
Red    #E74C3C
Green  #2ECC71
Blue   #3498DB
Orange #F39C12
Yellow #F2C94C
```

## Manual Date Mapping Example

Use only if a temporary test index needs an explicit `@timestamp` mapping.

```json
PUT unifi-network-test-2026.04.16
{
  "mappings": {
    "properties": {
      "@timestamp": {
        "type": "date"
      }
    }
  }
}
```

## EFG Disk / Firmware Checks

Use these only when investigating EFG system-resource issues.

```bash
# Check disk usage.
df -h

# Check rootfs mount information without using a shell pipe.
mount > /tmp/mount-list.txt
grep rootfs /tmp/mount-list.txt

# Inspect persistent storage.
ls -lh /persistent/
```

Potential remediation commands should be typed manually after confirming the
target path. Do not copy destructive commands unless the file and filesystem
state are verified.

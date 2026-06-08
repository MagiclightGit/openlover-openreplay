<p align="center">
  <a href="/README_FR.md">Français</a>
  &nbsp;|&nbsp;
  <a href="/README_ESP.md">Español</a>
  &nbsp;|&nbsp;
  <a href="/README_RU.md">Русский</a>
  &nbsp;|&nbsp;
  <a href="/README_AR.md">العربية</a>
</p>

<p align="center">
  <a href="https://openreplay.com/#gh-light-mode-only">
    <img src="static/openreplay-git-banner-light.png" width="100%">
  </a>
    <a href="https://openreplay.com/#gh-dark-mode-only">
    <img src="static/openreplay-git-banner-dark.png" width="100%">
  </a>
</p>

<h3 align="center">Session replay for developers</h3>
<p align="center">The most advanced session replay for building delightful web apps.</p>

<p align="center">
  <a href="https://docs.openreplay.com/deployment/deploy-aws">
    <img src="static/btn-deploy-aws.svg" height="40"/>
  </a>

  <a href="https://docs.openreplay.com/deployment/deploy-gcp">
    <img src="static/btn-deploy-google-cloud.svg" height="40" />
  </a>

  <a href="https://docs.openreplay.com/deployment/deploy-azure">
    <img src="static/btn-deploy-azure.svg" height="40" />
  </a>

  <a href="https://docs.openreplay.com/deployment/deploy-digitalocean">
    <img src="static/btn-deploy-digital-ocean.svg" height="40" />
  </a>
</p>

<p align="center">
  <a href="https://github.com/openreplay/openreplay">
    <img src="static/openreplay-git-hero.svg">
  </a>
</p>

OpenReplay is an open-source session replay suite you can host yourself, that lets you see what users do on your web app, helping you troubleshoot issues faster.

- **Session replay.** OpenReplay replays what users do, but not only. It also shows you what went under the hood, how your website or app behaves by capturing network activity, console logs, JS errors, store actions/state, page speed metrics, cpu/memory usage and much more.
- **Low footprint**. With a ~26KB (.br) tracker that asynchronously sends minimal data for a very limited impact on performance.
- **Self-hosted**. No more security compliance checks, 3rd-parties processing user data. Everything OpenReplay captures stays in your cloud for a complete control over your data.
- **Privacy controls**. Fine-grained security features for sanitizing user data.
- **Easy deploy**. With support of major public cloud providers (AWS, GCP, Azure, DigitalOcean).

## Features

- **Session replay:** Lets you relive your users' experience, see where they struggle and how it affects their behavior. Each session replay is automatically analyzed based on heuristics, for easy triage.
- **Spot:** A Chrome extension that lets record bugs directly from your browser — each recording includes all the technical details developers need to fix them.
- **DevTools:** It's like debugging in your own browser. OpenReplay provides you with the full context (network activity, JS errors, store actions/state and 40+ metrics) so you can instantly reproduce bugs and understand performance issues.
- **Assist:** Helps you support your users by seeing their live screen and instantly hopping on call (WebRTC) with them without requiring any 3rd-party screen sharing software.
- **Omni-search:** Search and filter by almost any user action/criteria, session attribute or technical event, so you can answer any question. No instrumentation required.
- **Analytics:** For surfacing the most impactful issues causing conversion and revenue loss.
- **Fine-grained privacy controls:** Choose what to capture, what to obscure or what to ignore so user data doesn't even reach your servers.
- **Plugins oriented:** Get to the root cause even faster by tracking application state (Redux, VueX, MobX, NgRx, Pinia and Zustand) and logging GraphQL queries (Apollo, Relay) and Fetch/Axios requests.
- **Integrations:** Sync your backend logs with your session replays and see what happened front-to-back. OpenReplay supports Sentry, Datadog, CloudWatch, Stackdriver, Elastic and more.

## Deployment Options

OpenReplay can be deployed anywhere. Follow our step-by-step guides for deploying it on major public clouds:

- [AWS](https://docs.openreplay.com/deployment/deploy-aws)
- [Google Cloud](https://docs.openreplay.com/deployment/deploy-gcp)
- [Azure](https://docs.openreplay.com/deployment/deploy-azure)
- [Digital Ocean](https://docs.openreplay.com/deployment/deploy-digitalocean)
- [Scaleway](https://docs.openreplay.com/deployment/deploy-scaleway)
- [OVHcloud](https://docs.openreplay.com/deployment/deploy-ovhcloud)
- [Kubernetes](https://docs.openreplay.com/deployment/deploy-kubernetes)

## Single-Node Maintenance

For this fork, a practical Ubuntu 24.04 single-node deploy and repair guide is available here:

- [docs/openreplay-ubuntu-single-node-deploy.md](docs/openreplay-ubuntu-single-node-deploy.md)

If you want the instance to run long-term on one machine, the most important maintenance task is preventing disk usage from growing forever.

The script below is a complete single-node maintenance setup that:

- Keeps OpenReplay data to 3 days where supported
- Runs `openreplay -c 3 --force`
- Cleans common MinIO buckets older than 3 days
- Removes succeeded and failed Kubernetes pods
- Vacuums systemd logs and apt cache
- Checks ClickHouse TTL settings every day and writes alerts if they drift

Create the maintenance script:

```bash
sudo tee /usr/local/bin/openreplay-maintenance-plus.sh >/dev/null <<'EOF'
#!/usr/bin/env bash
set -euo pipefail

LOG_FILE="/var/log/openreplay-maintenance.log"
ALERT_FILE="/var/log/openreplay-clickhouse-ttl-alert.log"
RETENTION_DAYS="3"
OR_DIR="/var/lib/openreplay"
YQ_BIN="/var/lib/openreplay/yq"

CLICKHOUSE_TABLES=(
  "experimental.sessions"
  "experimental.ios_events"
  "experimental.issues"
  "experimental.user_viewed_sessions"
  "experimental.user_viewed_errors"
  "product_analytics.events"
  "product_analytics.all_events"
  "product_analytics.event_properties"
  "product_analytics.all_properties"
  "product_analytics.autocomplete_events_grouped"
  "product_analytics.autocomplete_event_properties_grouped"
  "product_analytics.autocomplete_simple"
  "product_analytics.autocomplete_user_properties_grouped"
)

echo "==== $(date '+%F %T') maintenance start ====" >> "$LOG_FILE"

openreplay -c "$RETENTION_DAYS" --force >> "$LOG_FILE" 2>&1 || true

kubectl get pod -A --field-selector=status.phase=Succeeded -o name 2>/dev/null | xargs -r kubectl delete >> "$LOG_FILE" 2>&1 || true
kubectl get pod -A --field-selector=status.phase=Failed -o name 2>/dev/null | xargs -r kubectl delete >> "$LOG_FILE" 2>&1 || true

if [ -x "$YQ_BIN" ] && [ -f "$OR_DIR/vars.yaml" ]; then
  MINIO_HOST=$("$YQ_BIN" 'explode(.) | .global.s3.endpoint' "$OR_DIR/vars.yaml" | tr -d '"')
  MINIO_ACCESS_KEY=$("$YQ_BIN" 'explode(.) | .global.s3.accessKey' "$OR_DIR/vars.yaml" | tr -d '"')
  MINIO_SECRET_KEY=$("$YQ_BIN" 'explode(.) | .global.s3.secretKey' "$OR_DIR/vars.yaml" | tr -d '"')

  kubectl delete pod -n app minio-extra-cleanup --ignore-not-found=true >> "$LOG_FILE" 2>&1 || true

  cat <<EOF2 | kubectl apply -f - >> "$LOG_FILE" 2>&1
apiVersion: v1
kind: Pod
metadata:
  name: minio-extra-cleanup
  namespace: app
spec:
  restartPolicy: Never
  containers:
  - name: minio-extra-cleanup
    image: ghcr.io/openreplay/minio
    command: ["/bin/bash","-lc"]
    args:
      - |
        set -e
        mc alias set minio "$MINIO_HOST" "$MINIO_ACCESS_KEY" "$MINIO_SECRET_KEY"
        for bucket in mobs sessions-assets sourcemaps records spots; do
          mc rm --recursive --dangerous --force --older-than ${RETENTION_DAYS}d "minio/\${bucket}" || true
        done
    env:
    - name: MINIO_HOST
      value: "$MINIO_HOST"
    - name: MINIO_ACCESS_KEY
      value: "$MINIO_ACCESS_KEY"
    - name: MINIO_SECRET_KEY
      value: "$MINIO_SECRET_KEY"
EOF2

  kubectl wait -n app --for=jsonpath='{.status.phase}'=Succeeded pod/minio-extra-cleanup --timeout=600s >> "$LOG_FILE" 2>&1 || true
  kubectl logs -n app minio-extra-cleanup --tail=200 >> "$LOG_FILE" 2>&1 || true
  kubectl delete pod -n app minio-extra-cleanup --ignore-not-found=true >> "$LOG_FILE" 2>&1 || true
fi

CH_POD=$(kubectl -n db get pod -l app.kubernetes.io/name=clickhouse -o jsonpath='{.items[0].metadata.name}' 2>/dev/null || true)

if [ -n "${CH_POD}" ]; then
  echo "--- clickhouse ttl check ---" >> "$LOG_FILE"
  for item in "${CLICKHOUSE_TABLES[@]}"; do
    DB="${item%%.*}"
    TB="${item##*.}"

    QUERY_RESULT=$(kubectl exec -n db -c clickhouse "$CH_POD" -- clickhouse-client -q "
      SELECT create_table_query
      FROM system.tables
      WHERE database = '${DB}' AND name = '${TB}'
      FORMAT TSVRaw
    " 2>/dev/null || true)

    echo "[$(date '+%F %T')] ${item}" >> "$LOG_FILE"
    echo "$QUERY_RESULT" >> "$LOG_FILE"
    echo >> "$LOG_FILE"

    if [ -z "$QUERY_RESULT" ]; then
      echo "$(date '+%F %T') missing table definition: ${item}" >> "$ALERT_FILE"
      continue
    fi

    if ! echo "$QUERY_RESULT" | grep -Eq "INTERVAL 3 DAY|toIntervalDay\\(3\\)"; then
      echo "$(date '+%F %T') TTL mismatch for ${item}" >> "$ALERT_FILE"
    fi
  done
else
  echo "$(date '+%F %T') clickhouse pod not found" >> "$ALERT_FILE"
fi

journalctl --vacuum-time=3d >> "$LOG_FILE" 2>&1 || true
apt-get clean >> "$LOG_FILE" 2>&1 || true

echo "--- disk usage ---" >> "$LOG_FILE"
df -h / /var/lib/rancher/k3s /openreplay/storage /var/log >> "$LOG_FILE" 2>&1 || true

echo "--- top dirs ---" >> "$LOG_FILE"
du -sh /openreplay/storage /var/lib/rancher/k3s /var/log 2>/dev/null >> "$LOG_FILE" || true

echo "==== $(date '+%F %T') maintenance end ====" >> "$LOG_FILE"
EOF
```

Make it executable:

```bash
sudo chmod +x /usr/local/bin/openreplay-maintenance-plus.sh
```

Run it every day at `03:30`:

```bash
sudo tee /etc/cron.d/openreplay-maintenance >/dev/null <<'EOF'
SHELL=/bin/bash
PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin

30 3 * * * root /usr/local/bin/openreplay-maintenance-plus.sh
EOF
```

```bash
sudo chmod 644 /etc/cron.d/openreplay-maintenance
sudo systemctl restart cron
sudo systemctl status cron --no-pager
```

Rotate maintenance and pod logs:

```bash
sudo tee /etc/logrotate.d/openreplay-k3s >/dev/null <<'EOF'
/var/log/pods/*/*/*.log /var/log/openreplay-maintenance.log {
  daily
  rotate 3
  missingok
  notifempty
  compress
  delaycompress
  copytruncate
  maxsize 200M
}
EOF
```

Test once manually:

```bash
sudo /usr/local/bin/openreplay-maintenance-plus.sh
tail -100 /var/log/openreplay-maintenance.log
tail -100 /var/log/openreplay-clickhouse-ttl-alert.log
```

If ClickHouse should also keep only 3 days for the main analytical tables, use:

```bash
CH_POD=$(kubectl -n db get pod -l app.kubernetes.io/name=clickhouse -o jsonpath='{.items[0].metadata.name}')
kubectl exec -n db -c clickhouse "$CH_POD" -- clickhouse-client --multiquery <<'SQL'
ALTER TABLE experimental.sessions MODIFY TTL datetime + toIntervalDay(3);
ALTER TABLE experimental.ios_events MODIFY TTL datetime + toIntervalDay(3);
ALTER TABLE experimental.issues MODIFY TTL _timestamp + toIntervalDay(3);
ALTER TABLE experimental.user_viewed_sessions MODIFY TTL _timestamp + toIntervalDay(3);
ALTER TABLE experimental.user_viewed_errors MODIFY TTL _timestamp + toIntervalDay(3);
ALTER TABLE product_analytics.all_events MODIFY TTL _timestamp + toIntervalDay(3);
ALTER TABLE product_analytics.event_properties MODIFY TTL _timestamp + toIntervalDay(3);
ALTER TABLE product_analytics.all_properties MODIFY TTL _timestamp + toIntervalDay(3);
ALTER TABLE product_analytics.autocomplete_events_grouped MODIFY TTL _timestamp + toIntervalDay(3);
ALTER TABLE product_analytics.autocomplete_event_properties_grouped MODIFY TTL _timestamp + toIntervalDay(3);
ALTER TABLE product_analytics.autocomplete_simple MODIFY TTL _timestamp + toIntervalDay(3);
ALTER TABLE product_analytics.autocomplete_user_properties_grouped MODIFY TTL _timestamp + toIntervalDay(3);
SQL
```

`product_analytics.events` may already show as:

```sql
TTL created_at + toIntervalDay(3)
```

which is already valid for the maintenance check above.

## OpenReplay Cloud

For those who want to simply use OpenReplay as a service, [sign up](https://app.openreplay.com/signup) for a free account on our cloud offering.

## Community Support

Please refer to the [official OpenReplay documentation](https://docs.openreplay.com/). That should help you troubleshoot common issues. For additional help, you can reach out to us on one of these channels:

- [Slack](https://slack.openreplay.com) (Connect with our engineers and community)
- [GitHub](https://github.com/openreplay/openreplay/issues) (Bug and issue reports)
- [Twitter](https://twitter.com/OpenReplayHQ) (Product updates, Great content)
- [YouTube](https://www.youtube.com/channel/UCcnWlW-5wEuuPAwjTR1Ydxw) (How-to tutorials, past Community Calls)
- [Website chat](https://openreplay.com) (Talk to us)

## Contributing

We're always on the lookout for contributions to OpenReplay, and we're glad you're considering it! Not sure where to start? Look for open issues, preferably those marked as good first issues.

See our [Contributing Guide](CONTRIBUTING.md) for more details.

Also, feel free to join our [Slack](https://slack.openreplay.com) to ask questions, discuss ideas or connect with our  contributors.

## License

This monorepo uses several licenses. See [LICENSE](/LICENSE) for more details.

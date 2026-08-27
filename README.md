# To bootstrap the auto deploy service run:

## The deploy script
<code>
cat > /srv/app-containers/deploy.sh << 'EOF'
#!/usr/bin/env bash
set -euo pipefail

REPO_DIR=/srv/app-containers
APPS="app1 app2 app3"

cd "$REPO_DIR"

git fetch origin main
LOCAL=$(git rev-parse HEAD)
REMOTE=$(git rev-parse origin/main)

if [ "$LOCAL" = "$REMOTE" ]; then
  echo "no changes"
  exit 0
fi

echo "change detected: ${LOCAL:0:7} -> ${REMOTE:0:7}"
git reset --hard origin/main

failed=""
for dir in $APPS; do
  [ -f "$dir/docker-compose.yml" ] || continue
  echo "-- $dir --"
  ( cd "$dir" && docker compose pull && docker compose up -d --remove-orphans ) || failed="$failed $dir"
done

if [ -n "$failed" ]; then
  echo "FAILED:$failed" >&2
  exit 1
fi

echo "done"
EOF
chmod +x /srv/app-containers/deploy.sh
</code>

## The unit script
<code>
cat > /etc/systemd/system/app-containers-deploy.service << 'EOF'
[Unit]
Description=Pull and apply app-containers docker compose updates
Wants=network-online.target
After=network-online.target docker.service
Requires=docker.service

[Service]
Type=oneshot
WorkingDirectory=/srv/app-containers
ExecStart=/srv/app-containers/deploy.sh
User=root
Environment=HOME=/root
TimeoutStartSec=900
EOF
</code>

## The timer script
<code>
cat > /etc/systemd/system/app-containers-deploy.timer << 'EOF'
[Unit]
Description=Run app-containers deploy check every 20 minutes
[Timer]
OnBootSec=2min
OnUnitActiveSec=20min
Persistent=true
[Install]
WantedBy=timers.target
EOF

systemctl daemon-reload
systemctl enable --now app-containers-deploy.timer
systemctl start app-containers-deploy.service
journalctl -u app-containers-deploy.service -n 50 --no-pager
</code>

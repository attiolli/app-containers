# To bootstrap the auto deploy service run:

<code>
cat > /srv/app-containers/deploy.sh << 'EOF'
#!/usr/bin/env bash
set -euo pipefail
cd /srv/app-containers

APPS="hass esphome uptime"

git fetch origin main
LOCAL=$(git rev-parse HEAD)
REMOTE=$(git rev-parse origin/main)

if [ "$LOCAL" != "$REMOTE" ]; then
  echo "$(date): change detected, pulling and applying"
  git pull origin main

  for dir in $APPS; do
    [ -f "$dir/docker-compose.yml" ] || continue
    echo "-- $dir --"
    (cd "$dir" && docker compose pull && docker compose up -d --remove-orphans)
  done

  echo "$(date): done"
else
  echo "$(date): no changes"
fi
EOF

chmod +x /srv/app-containers/deploy.sh


cat > /etc/systemd/system/app-containers-deploy.service << 'EOF'
[Unit]
Description=Pull and apply app-containers docker compose updates
Wants=network-online.target
After=network-online.target docker.service

[Service]
Type=oneshot
WorkingDirectory=/srv/app-containers
ExecStart=/srv/app-containers/deploy.sh
User=root
EOF

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

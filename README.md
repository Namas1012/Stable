# Stable
# cài deps (root)
sudo apt update && sudo apt install -y wget tar jq lz4 ca-certificates

# tạo thư mục node cho user đang login
sudo -u $(logname) mkdir -p /home/$(logname)/stable
sudo chown -R $(logname):$(logname) /home/$(logname)/stable
cd /home/$(logname)/stable

# tải binary
wget -O stabled-1.1.1-linux-amd64.tar.gz "https://stable-testnet-data.s3.us-east-1.amazonaws.com/stabled-1.1.1-linux-amd64-testnet.tar.gz"
tar -xzf stabled-1.1.1-linux-amd64.tar.gz

# copy binary
sudo cp stabled /usr/bin/stabled
sudo chmod +x /usr/bin/stabled

# init config
sudo -u $(logname) /usr/bin/stabled init "NamasStable" --home /home/$(logname)/stable/.stabled --chain-id stabletestnet_2201-1

# lấy genesis
mkdir -p /home/$(logname)/downloads
curl -sS 'http://51.77.81.3:26657/genesis' -o /home/$(logname)/downloads/genesis_raw.json
jq '.result.genesis' /home/$(logname)/downloads/genesis_raw.json > /home/$(logname)/downloads/genesis.json
jq -r '.chain_id' /home/$(logname)/downloads/genesis.json
jq empty /home/$(logname)/downloads/genesis.json || (echo "genesis invalid"; exit 1)
cp /home/$(logname)/stable/.stabled/config/genesis.json /home/$(logname)/stable/.stabled/config/genesis.json.bak.$(date +%s)
cp /home/$(logname)/downloads/genesis.json /home/$(logname)/stable/.stabled/config/genesis.json
sudo chown $(logname):$(logname) /home/$(logname)/stable/.stabled/config/genesis.json
sudo chmod 644 /home/$(logname)/stable/.stabled/config/genesis.json

# Edit file app.toml & config.toml
nano /home/$(logname)/stable/.stabled/config/config.toml #copy từ file
nano /home/$(logname)/stable/.stabled/config/app.toml #copy từ file

# restore snapshot
wget -O /home/$(logname)/stable/snapshot.tar.lz4 "https://stable-snapshot.s3.eu-central-1.amazonaws.com/snapshot.tar.lz4"
sudo systemctl stop stabled || true
sudo -u $(logname) rm -rf /home/$(logname)/stable/.stabled/data
sudo -u $(logname) mkdir -p /home/$(logname)/stable/.stabled/data
sudo -u $(logname) bash -c 'lz4 -dc /home/$(logname)/stable/snapshot.tar.lz4 | tar -x -C /home/$(logname)/stable/.stabled/data'

# rất quan trọng: sửa quyền snapshot
sudo chown -R $(logname):$(logname) /home/$(logname)/stable

# mở port
ufw allow 26656/tcp

# systemd
sudo tee /etc/systemd/system/stabled.service > /dev/null <<EOF
[Unit]
Description=Stable Daemon Service
After=network-online.target

[Service]
User=$(logname)
WorkingDirectory=/home/$(logname)/stable
ExecStart=/usr/bin/stabled start --home /home/$(logname)/stable/.stabled --chain-id stabletestnet_2201-1
Restart=always
RestartSec=3
LimitNOFILE=65535
StandardOutput=journal
StandardError=journal
SyslogIdentifier=stabled

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable stabled
sudo systemctl start stabled
sudo journalctl -u stabled -f

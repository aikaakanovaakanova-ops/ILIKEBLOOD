# ILIKEBLOOD
#!/bin/bash

echo "=== Установка Anti-DDoS защиты ==="

# 1. Системные настройки
cat << EOF | sudo tee /etc/sysctl.d/99-anti-ddos.conf
net.ipv4.tcp_syncookies = 1
net.ipv4.tcp_max_syn_backlog = 8192
net.ipv4.tcp_synack_retries = 1
net.ipv4.tcp_syn_retries = 2
net.ipv4.icmp_echo_ignore_broadcasts = 1
net.ipv4.conf.all.rp_filter = 1
net.ipv4.conf.default.rp_filter = 1
net.ipv4.tcp_rfc1337 = 1
net.ipv4.tcp_fin_timeout = 10
net.ipv4.tcp_keepalive_time = 300
net.core.somaxconn = 65535
net.ipv4.ip_local_port_range = 1024 65535
net.core.netdev_max_backlog = 5000
net.ipv4.tcp_tw_reuse = 1
EOF

sudo sysctl -p /etc/sysctl.d/99-anti-ddos.conf

# 2. Основной скрипт защиты
sudo tee /usr/local/bin/anti-ddos.sh > /dev/null << 'EOF'
#!/bin/bash

THRESHOLD=70
CHECK_INTERVAL=3
BAN_TIME=7200
LOG="/var/log/anti-ddos.log"
WHITELIST="/etc/anti-ddos-whitelist.txt"

touch "$LOG"
touch "$WHITELIST"

is_whitelisted() {
    grep -qxF "$1" "$WHITELIST" 2>/dev/null
}

while true; do
    ss -Htn state established | awk '{print $4}' | cut -d: -f1 | sort | uniq -c | sort -nr | while read count ip; do
        if [[ ! "$ip" =~ ^[0-9]+\.[0-9]+\.[0-9]+\.[0-9]+$ ]] || is_whitelisted "$ip"; then
            continue
        fi

        if [ "$count" -ge "$THRESHOLD" ]; then
            if ! iptables -nL INPUT 2>/dev/null | grep -q "$ip"; then
                echo "$(date '+%Y-%m-%d %H:%M:%S') [BAN] $ip — $count соединений" >> "$LOG"
                iptables -I INPUT -s "$ip" -j DROP
                (
                    sleep "$BAN_TIME"
                    iptables -D INPUT -s "$ip" -j DROP 2>/dev/null
                    echo "$(date '+%Y-%m-%d %H:%M:%S') [UNBAN] $ip" >> "$LOG"
                ) &
            fi
        fi
    done
    sleep "$CHECK_INTERVAL"
done
EOF

sudo chmod +x /usr/local/bin/anti-ddos.sh

# 3. Белый список
sudo touch /etc/anti-ddos-whitelist.txt
echo "127.0.0.1" | sudo tee /etc/anti-ddos-whitelist.txt > /dev/null

# 4. Systemd служба
sudo tee /etc/systemd/system/anti-ddos.service > /dev/null << EOF
[Unit]
Description=Anti-DDoS Protection Service
After=network.target

[Service]
Type=simple
ExecStart=/usr/local/bin/anti-ddos.sh
Restart=always
RestartSec=5
User=root

[Install]
WantedBy=multi-user.target
EOF

# 5. Запуск
sudo systemctl daemon-reload
sudo systemctl enable --now anti-ddos.service

echo ""
echo "=== Установка завершена ==="
echo "Статус службы:"
sudo systemctl status anti-ddos.service --no-pager
echo ""
echo "Полезные команды:"
echo "  Лог:           sudo tail -f /var/log/anti-ddos.log"
echo "  Текущие баны:  sudo iptables -nL INPUT | grep DROP"
echo "  Белый список:  sudo nano /etc/anti-ddos-whitelist.txt"
echo "  Остановить:    sudo systemctl stop anti-ddos.service"
echo "  Запустить:     sudo systemctl start anti-ddos.service"
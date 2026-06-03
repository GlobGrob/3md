#!/bin/bash

# Проверка прав root
if [ "$EUID" -ne 0 ]; then
  exit 1
fi

# Определение имени хоста
HOST=$(hostname)

echo "Начало настройки для хоста: $HOST"

case "$HOST" in
    "HQ-SRV")
        echo ">>> Настройка HQ-SRV"
        
        # Шаги 2-4: Обновление и установка пакетов
        apt-get update -y
        apt-get install git docker-engine docker-compose-v2 zabbix-agent rsyslog logrotate fail2ban python3-module-systemd -y

        # Шаги 5-7: Скачивание Zabbix Docker
        cd /root || exit
        git clone https://github.com/zabbix/zabbix-docker
        systemctl enable --now docker
        cd zabbix-docker || exit

        # Шаги 8-11: Изменение порта и запуск Zabbix
        sed -i 's/ZABBIX_WEB_NGINX_HTTP_PORT=80$/ZABBIX_WEB_NGINX_HTTP_PORT=8080/' .env
        docker compose -f docker-compose_v3_alpine_mysql_latest.yaml up -d

        # Шаги 24-29: Настройка Zabbix Agent на HQ-SRV
        sed -i 's/Server=127.0.0.1/Server=0.0.0.0\/0/' /etc/zabbix/zabbix_agentd.conf
        sed -i 's/^#ListenPort=10050/ListenPort=10050/' /etc/zabbix/zabbix_agentd.conf
        sed -i 's/^#ListenIP=0.0.0.0/ListenIP=0.0.0.0/' /etc/zabbix/zabbix_agentd.conf
        systemctl enable --now zabbix_agentd

        # Шаги 30-33: Настройка DNS (добавление CNAME записи)
        echo "mon IN CNAME hq-srv" >> /etc/bind/zone/forward.db
        systemctl restart bind

        # Шаги 57-62: Настройка rsyslog сервера
        sed -i 's/^#module(load="imudp")/module(load="imudp")/' /etc/rsyslog.conf
        sed -i 's/^#input(type="imudp" port="514")/input(type="imudp" port="514")/' /etc/rsyslog.conf
        sed -i 's/^#module(load="imtcp")/module(load="imtcp")/' /etc/rsyslog.conf
        sed -i 's/^#input(type="imtcp" port="514")/input(type="imtcp" port="514")/' /etc/rsyslog.conf
        
        cat << 'EOF' >> /etc/rsyslog.conf
$template remote-incoming-logs,"/opt/%HOSTNAME%/%PROGRAMNAME%.log"
*.* action(type="omfile" dynaFile="remote-incoming-logs" dirCreateMode="0755" fileCreateMode="0644")
EOF
        systemctl enable --now rsyslog

        # Шаги 86-93: Настройка logrotate
        cat << 'EOF' > /etc/logrotate.d/logs
/opt/* {
    weekly
    size=10M
    compress
}
EOF
        echo "0 0 * * * root logrotate -f /etc/logrotate.d/logs" >> /etc/crontab

        # Шаги 95-99: Настройка fail2ban
        cat << 'EOF' > /etc/fail2ban/jail.d/ssh.conf
[sshd]
maxretry = 3
bantime = 60
port = ssh
enabled = true
backend = systemd
EOF
        systemctl enable --now fail2ban
        
        # ПРИМЕЧАНИЕ: Шаги 35-54 выполняются вручную через веб-интерфейс Zabbix (mon.au-team.irpo:8080)
        echo ">>> Настройка HQ-SRV завершена. Не забудьте выполнить шаги 35-54 в веб-интерфейсе Zabbix."
        ;;

    "BR-SRV")
        echo ">>> Настройка BR-SRV"
        
        # Шаги 13-14, 22-23: Обновление и установка Zabbix Agent
        apt-get update -y
        apt-get install zabbix-agent rsyslog-classic -y

        # Шаги 15-20: Настройка Zabbix Agent на BR-SRV
        sed -i 's/Server=127.0.0.1/Server=192.168.1.2/' /etc/zabbix/zabbix_agentd.conf
        sed -i 's/^#ListenPort=10050/ListenPort=10050/' /etc/zabbix/zabbix_agentd.conf
        sed -i 's/^#ListenIP=0.0.0.0/ListenIP=0.0.0.0/' /etc/zabbix/zabbix_agentd.conf
        systemctl enable --now zabbix_agentd

        # Шаги 78-83: Настройка отправки логов на HQ-SRV
        cat << 'EOF' > /etc/rsyslog.d/all.conf
*.* @@192.168.1.2:514
EOF
        systemctl restart rsyslog

        # Шаги 101-107: Настройка Ansible для инвентаризации
        mkdir -p /etc/ansible/PC-INFO
        cat << 'EOF' > /etc/ansible/playbook.yml
- name: PC-INFO
  hosts: all
  tasks:
    - name: hostname
      lineinfile:
        path: /etc/ansible/PC-INFO/{{ ansible_hostname }}.yml
        line: "Hostname: {{ ansible_hostname }}"
        create: true
      delegate_to: 127.0.0.1
    - name: IP-address
      lineinfile:
        path: /etc/ansible/PC-INFO/{{ ansible_hostname }}.yml
        line: "IP-address: {{ ansible_default_ipv4.address }}"
        create: true
      delegate_to: 127.0.0.1
EOF
        ansible-playbook /etc/ansible/playbook.yml
        echo ">>> Настройка BR-SRV завершена."
        ;;

    "HQ-RTR")
        echo ">>> Настройка HQ-RTR"
        
        # Шаги 64-69: Установка и настройка rsyslog-classic
        apt-get update -y
        apt-get install rsyslog-classic -y
        
        cat << 'EOF' > /etc/rsyslog.d/all.conf
*.* @@192.168.1.2:514
EOF
        systemctl restart rsyslog
        echo ">>> Настройка HQ-RTR завершена."
        ;;

    "BR-RTR")
        echo ">>> Настройка BR-RTR"
        
        # Шаги 71-76: Установка и настройка rsyslog-classic
        apt-get update -y
        apt-get install rsyslog-classic -y
        
        cat << 'EOF' > /etc/rsyslog.d/all.conf
*.* @@192.168.1.2:514
EOF
        systemctl restart rsyslog
        echo ">>> Настройка BR-RTR завершена."
        ;;

    *)
        echo "Неизвестное имя хоста: $HOST. Настройка не требуется или проверьте hostname."
        exit 1
        ;;
esac

echo "Все команды для хоста $HOST успешно выполнены!"

## Стартовая настройка сервера Ubuntu 18.04

### 1. Если нужен root, включаем, меняем пароль
```
sudo -i
sudo passwd root
```

Сделать root неактивным
```
sudo passwd -dl root
```

### 2. Добавляем нового пользователя с root-привилегиями
```
adduser <user_name>
gpasswd -a <user_name> sudo
```

### 3. Меняем порт доступа по SSH с 22 на любой другой в конфиге
```
sudo nano /etc/ssh/sshd_config
```

Отключение (logout) неактивного пользователя по таймауту
Убираем коммент со строк
```
ClientAliveInterval 120
ClientAliveCountMax 0
```
Неактивный пользователь будет разлогинен через 120 секунд бездействия

Перезапускаем SSH
```
sudo systemctl restart sshd
```

### 4. Настраиваем сообщения на почту о попытке входа на сервер под учеткой root
```
sudo nano ~/.profile .
```
Вставляем в конец строку

```
echo 'ALERT - Root Shell Access on:' `date` `who` | mail -s "Alert: Root Access from `who | awk '{print $6}'`" your@email.com
```

**Установка postfix**
```
sudo apt install mailutils
```

При установке выбираем опцию "Internet site" в меню

Если требуется повторно вызвать меню конфигурации
```
sudo dpkg-reconfigure postfix
```
Если нужна только отправка сообщений, меняем в конфиге
```
sudo nano /etc/postfix/main.cf
```
строку
```
inet_interfaces = all
```
на
```
inet_interfaces = loopback-only
```

Затем перегружаем сервис
```
sudo systemctl restart postfix
```

Проверяем работоспособность
```
echo 'Hello!' | mail -s 'Greeteings' your@email.com
```

Ошибки логируются в файле /var/log/mail.err

### 5. Настройка SSH MOTD (сообщение при логине)
```
sudo nano /etc/motd
```
Пишем приветственное сообщение для хакера. Например:

“This system is restricted to authorized access only. All activities on
this system are recorded and logged. Unauthorized access will be fully
investigated and reported to the appropriate law enforcement agencies.”

Или добавляем security login banner
```
sudo nano /etc/issue.net
```

Отключаем motd
```
sudo nano /etc/pam.d/sshd
```

Комментим строки
```
#session optional pam_motd.so motd=/run/motd.dynamic
​#session optional pam_motd.so noupdate
```

Открываем конфиг ssh
```
sudo nano etc/ssh/sshd_config
```

Добавляем строку
```
Banner /etc/issue.net
```

Перезапускаем SSH
```
sudo systemctl restart sshd
```

### 6. Устанавливаем fail2ban (защита от брутфорса)
```
sudo apt install fail2ban
```

Конфиг-файл
```
sudo nano /etc/fail2ban/jail.conf
```

Раскомментируем строки:

Не учитывать трафик с локальной машины
```
ignoreip = 127.0.0.1/8
```
Время бана клиента (по умолчанию 10 минут)
```
bantime = 600
```
Количество попыток подключения maxretry за время findtime (по умолчанию 3 раза за 10 минут)
```
findtime = 600
maxretry = 3
```
Настройки email
```
destemail = root@localhost //куда будет отправлено сообщение
sendername = Fail2Ban // имя отправителя
mta = sendmail // сервис при помощи которого будет отправлен email
```
Включаем fail2ban
```
enabled = true
```

Запускаем сервис
```
sudo service fail2ban start
```

### 7. Настраиваем фаервол

Проверить статус UFW
```
sudo ufw status verbose
```
**Пока не сделаны все настройки не включаем, чтобы не отстрелить себе ногу!**

Если требуется IPV6
```
sudo nano /etc/default/ufw
```
Выставляем
```
IPV6=yes
```

Сбрасываем все настройки (никого не впускать, всех выпускать)
```
sudo ufw default deny incoming
sudo ufw default allow outgoing
```

Разрешаем SSH
```
sudo ufw allow ssh или sudo ufw allow 22
```

Если порт был изменен (см. пункт 3), например на 2222, разрешаем коннект к нему
```
sudo ufw allow 2222
```

Включаем фаервол
```
sudo ufw enable
```

Разрешаем HTTP и HTTPS (если на сервере есть SSL-сертификат)

`sudo ufw allow http` или `sudo ufw allow 80`
`sudo ufw allow https` или `sudo ufw allow 443`



Разрешаем FTP

`sudo ufw allow ftp` или `sudo ufw allow 21/tcp`



Если нужно запретить коннект, делаем аналогично через deny

`sudo ufw deny http` или `sudo ufw deny 80`



Доступ к диапазону портов (6000-6007)
```
sudo ufw allow 6000:6007/tcp
```

Если требуется разрешить доступ с определенных IP
```
sudo ufw allow from 15.15.15.51
```

Доступ к конкретному порту
```
sudo ufw allow from 15.15.15.51 to any port 22
```

Доступ для подсети (например 15.15.15.1 - 15.15.15.254)
```
sudo ufw allow from 15.15.15.0/24
```

#### Удаление правил
По номеру
```
sudo ufw status numbered
sudo ufw delete 1
```

По названию правила
```
sudo ufw delete allow http
```

Перезагрузка UFW (отключает и удаляет все правила)
```
sudo ufw reset
```

Отключить фаервол
```
sudo ufw disable
```

### 8. Защищаем shared memory
```
sudo nano /etc/fstab
```

Добавляем в конец
```
tmpfs /run/shm tmpfs defaults,noexec,nosuid 0 0
```

Чтобы изменения вступили в силу, нужна перезагрузка сервера

```
sudo reboot
```

### 9. Сетевая безопасность
```
sudo nano /etc/sysctl.conf
```
Раскомментируем строки

```
# prevent some spoofing attacks
net.ipv4.conf.default.rp_filter=1
net.ipv4.conf.all.rp_filter=1
# Uncomment the next line to enable TCP/IP SYN cookies
net.ipv4.tcp_syncookies=1
# Do not accept ICMP redirects (prevent MITM attacks)
net.ipv4.conf.all.accept_redirects = 0
net.ipv6.conf.all.accept_redirects = 0
# Do not accept IP source route packets (we are not a router)
net.ipv4.conf.all.accept_source_route = 0
net.ipv6.conf.all.accept_source_route = 0
# Log Martian Packets
net.ipv4.conf.all.log_martians = 1
```

Перезапускаем сервис
```
sudo sysctl -p
```

### 10. Устанавливаем время на сервере
Спросить который час
```
date "+%H:%M:%S   %d/%m/%y"
```
Настраиваем свой часовой пояс
```
sudo dpkg-reconfigure tzdata
```
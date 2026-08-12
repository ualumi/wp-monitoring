## Описание

wordpress и mariadb разворачиваются в docker, nginx принимает http/https запросы, prometheus собирает метрики node exporter, grafana отображает их в дашборде OS General.

- wordpress доступен только через nginx, порт не публикуется на хосте
- mariadb доступна только внутри docker-сети
- http запросы к site.local перенаправляются на https
- доступ к wordpress админке фильтруется nginx до передачи запроса wordpress
- самоподписанный ssl сертификат допустим только для тестовой среды
- Fail2ban работает на linux хосте чтобы защищать именно хостовый ssh сервис

## Архитектура

- `nginx` reverse proxy для wordpress и grafana
- `wordpress` wordpress с Apache и PHP 8.3
- `db` mariadb 11.4
- `node_exporter` сбор метрик linux хоста
- `prometheus` хранение и запрос метрик
- `grafana` визуализация метрик
- Fail2ban защита SSH на уровне linux хоста

Конфигурации nginx, prometheus и grafana хранятся на хосте и монтируются в контейнеры в режиме read-only, постоянные данные хранятся в docker томах.

## Требования

- Debian/Ubuntu;
- Docker Engine;
- Docker Compose v2;
- OpenSSL;
- Fail2ban для защиты SSH.

## Подготовка

### 1. Переменные окружения

создать .env из примера:

```bash
cp .env.example .env
```

`в .env изменить тестовые пароли`

### 2. SSL-сертификат

cоздать сертификат для site.local:

```bash
mkdir -p ssl
openssl req -x509 -nodes -newkey rsa:2048 \
  -keyout ssl/site.local.key \
  -out ssl/site.local.crt \
  -days 365 \
  -subj "/CN=site.local" \
  -addext "subjectAltName=DNS:site.local"
```

### 3. Локальные доменные имена

Для доступа к сервисам по именам `site.local` и `metrics.local` добавить записи в `/etc/hosts`:

```bash
echo "127.0.0.1 site.local metrics.local" | sudo tee -a /etc/hosts
```

Если проект развёрнут на удалённом сервере, вместо `127.0.0.1` указать IP-адрес этого сервера.

### 4. Разрешённый IP для WordPress

Перед запуском указать в `nginx/conf.d/site.conf` IP-адрес или подсеть, с которой разрешён доступ к `/wp-admin` и `/wp-login.php` (значение зависит от среды развёртывания):

```nginx
allow 172.18.0.1;
deny all;
```

## Запуск

запустить сервисы командой:

```bash
docker compose up -d
```

после запуска доступны:

- wordpress <https://site.local>
- grafana через nginx <http://metrics.local>
- prometheus <http://localhost:9090>

Порты WordPress, MariaDB, Grafana и node exporter не публикуются на хосте. Доступ к WordPress и Grafana выполняется только через Nginx.

## Развёртывание через Ansible

`Ansible плейбук предназначен для Ubuntu` и устанавливает Docker, Docker Compose, Fail2ban и OpenSSL, копирует конфигурации, создаёт `.env` и самоподписанный сертификат, после чего запускает контейнеры.

На управляющем узле должен быть установлен Ansible. Для локального развёртывания выполнить из корня проекта:

```bash
cd ansible
ansible-playbook playbook.yml
```

Во время запуска Ansible запросит пароли пользователя MariaDB, `root` MariaDB и администратора Grafana. Для удалённого сервера необходимо заменить `localhost` в `ansible/inventory.ini` на адрес целевого хоста и настроить ssh доступ.

### Пример развёртывания на удалённом сервере

- адрес сервера 192.168.1.50
- SSH-пользователь ubuntu
- вход выполняется по SSH-ключу ~/.ssh/id_ed25519
- пользователь ubuntu имеет право выполнять команды через sudo

Указать сервер в ansible/inventory.ini:

```ini
[wordpress_servers]
192.168.1.50 ansible_user=ubuntu ansible_ssh_private_key_file=~/.ssh/id_ed25519
```

Запустить развёртывание:
```bash
ansible-playbook playbook.yml
```

Если sudo требует пароль, добавить параметр --ask-become-pass:
```bash
ansible-playbook playbook.yml --ask-become-pass
```

После завершения добавить на компьютере проверяющего записи в /etc/hosts:
```bash
192.168.1.50 site.local metrics.local
```
Проверить результат:
```bash
curl -k --resolve site.local:443:192.168.1.50 https://site.local/
curl --resolve metrics.local:80:192.168.1.50 http://metrics.local/api/health
```
## ip ограничения

c разрешённого ip ожидается 200 OK или перенаправление 301/302. С другого ip для /wp-admin и /wp-login.php ожидается 403 Forbidden, обычные страницы сайта остаются доступны.

## Мониторинг

prometheus опрашивает node exporter с интервалом 15 секунд. datasource prometheus и JSON дашборда подключаются к grafana автоматически.

Дашборд `OS General` содержит панели

- входящего сетевого трафика
- исходящего сетевого трафика
- доступного дискового пространства
- количества выделенных файловых дескрипторов

проверить targets prometheus можно на <http://localhost:9090/targets>.

node exporter предназначен для запуска на linux

## Защита SSH с Fail2ban

Fail2ban устанавливается на linux хост, поскольку ему нужен доступ к хостовому ssh, systemd journal и сетевым правилам:

```bash
sudo apt update
sudo apt install -y fail2ban
sudo cp fail2ban/jail.local /etc/fail2ban/jail.local
sudo fail2ban-client -t
sudo systemctl enable --now fail2ban
sudo fail2ban-client status sshd
```

Текущая jail блокирует ip на один час после пяти неудачных ssh аутентификаций за десять минут.

Перед проверкой на удалённом сервере следует добавить ip администратора в `ignoreip`, чтобы не потерять ssh доступ.

## Остановка

остановить и удалить контейнеры можно командой:

```bash
docker compose down
```

Удалить контейнеры вместе со всеми томами и данными можно командой:

```bash
docker compose down -v
```

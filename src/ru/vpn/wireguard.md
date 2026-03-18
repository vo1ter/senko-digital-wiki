---
title: "Руководство по установке WireGuard VPN"
description: "Пошаговая настройка и установка сервера WireGuard VPN на Debian, Ubuntu, CentOS, Fedora, AlmaLinux и других дистрибутивах"
head:
  - - meta
    - name: keywords
      content: wireguard, vpn, сервер, установка, debian, ubuntu, centos, fedora, almalinux, руководство, конфигурация
  - - meta
    - property: og:title
      content: "WireGuard VPN — полное руководство по установке"
  - - meta
    - property: og:description
      content: "Пошаговая инструкция по установке и настройке сервера WireGuard VPN на разных Linux-дистрибутивах."
---

# Руководство по установке WireGuard VPN

<img src="/images/vpn/wireguard-easy/wg-easy-logo.png" style="display: flex !important; justify-content: center !important; height: 500px;"></img>

WireGuard — это современный, быстрый и безопасный VPN‑протокол, который достаточно легко настраивается и разворачивается на практически любом сервере. Это руководство проведёт вас через процесс установки и настройки WireGuard на сервере для создания защищённого VPN‑подключения.

::: tip Подсказка
Клиентам нашего хостинга доступна полностью автоматическая установка WireGuard при первоначальном заказе услуги или с помощью скриптов в панели управления ВМ.
:::

## Требования

Перед началом убедитесь, что у вас есть:

- Сервер с поддерживаемым Linux‑дистрибутивом (Debian/Ubuntu, RHEL/CentOS/AlmaLinux/Rocky/Fedora)
- Доступ к root или sudo на сервере
- Базовые навыки работы в командной строке
- Публичный IP‑адрес сервера

## Установка

### Автоматическая установка для клиентов хостинга

#### При заказе услуги

В меню заказа через выпадающее меню выберите опцию "WireGuard", завершите заказ и дождитесь окончания обработки услуги.

Сразу после активации сервера вы получите данные для входа в ОС сервера, а также конфигурационный файл для подключения к WireGuard VPN. Достаточно будет импортировать файл конфигурации в клиент приложения WireGuard и начать использование VPN.

#### Через панель управления ВМ

Перейдите в [панель управления ВМ](https://vm.senko.digital), перейдите в управление нужным сервером, в правом верхнем углу раскройте "Меню" и выберите опцию "Запустить скрипт".

В диалоге выберите опцию "WireGuard" и активируйте опцию "Отправить письмо, которое привязано к скрипту".

По окончанию установки вы получите письмо с конфигурационным файлом для подключения к вашему WireGuard VPN серверу. Достаточно будет импортировать файл конфигурации в клиент приложения WireGuard и начать использование VPN.

#### Что устанавливается

Скрипт разворачивает сервер WireGuard VPN в Docker с автоматической генерацией клиентской конфигурации.

После установки вы получаете:

- **peer1.conf** — готовый конфигурационный файл клиента WireGuard (также отображается в виде QR-кода в логе скрипта)

#### Как подключиться

1. Установите [клиент WireGuard](https://www.wireguard.com/install/) на ваше устройство
2. Импортируйте файл `peer1.conf` (или отсканируйте QR-код на мобильном устройстве)
3. Активируйте туннель

#### Файрвол

Скрипт настраивает достаточно **ограниченный брандмауэр** — открыты только лишь порты SSH и WireGuard (`51820/udp`). Весь остальной входящий трафик по умолчанию блокируется.

#### Управление сервером

```bash
cd /opt/wireguard-docker && docker compose up -d    # запуск
cd /opt/wireguard-docker && docker compose down      # остановка
cd /opt/wireguard-docker && docker compose logs -f   # логи
```

### Стандартная установка

Процесс установки зависит от дистрибутива Linux. Используйте команды, соответствующие вашей системе.

::: code-group

```bash [Ubuntu/Debian]
# Обновите список пакетов
apt-get update -y

# Установите WireGuard и необходимые инструменты
apt-get install -y wireguard wireguard-tools iproute2 iptables
```

```bash [RHEL/CentOS/AlmaLinux/Rocky/Fedora]
# Установите репозиторий EPEL (если ещё не установлен)
yum install -y epel-release
# Или для более новых систем:
dnf install -y epel-release

# Установите WireGuard и необходимые инструменты
yum install -y wireguard-tools iproute iptables
# Или для более новых систем:
dnf install -y wireguard-tools iproute iptables
```

:::

## Настройка сервера

### 1. Создайте структуру папок

Сначала создайте необходимые папки для хранения конфигураций WireGuard:

```bash
# Установите безопасные права на создаваемые файлы
umask 077

# Создайте папки для конфигураций сервера и клиента
mkdir -p /etc/wireguard/server
mkdir -p /etc/wireguard/client
```

### 2. Сгенерируйте ключи

WireGuard использует криптографию с помощью открытых ключей. Сгенерируйте пары ключей для сервера и клиента:

```bash
# Сгенерируйте ключи для сервера
wg genkey | tee /etc/wireguard/server/privatekey | wg pubkey > /etc/wireguard/server/publickey

# Сгенерируйте ключи для клиента
wg genkey | tee /etc/wireguard/client/privatekey | wg pubkey > /etc/wireguard/client/publickey
```

### 3. Определите сетевые настройки

Определите основной сетевой интерфейс и определите настройки сети:

- **Интерфейс**: Можно узнать командой `ip route get 1.1.1.1 | awk '{print $5; exit}'` или по выводу `ip route` (чаще всего `eth0` или `ens3`)
- **Публичный IP сервера**: Проверьте `curl -4 ifconfig.io` или найдите IP сервера в личном кабинете
- **Подсеть VPN**: Выберите приватную подсеть, не пересекающуюся с вашими сетями, например `10.8.0.0/24` (при конфликте используйте `10.9.0.0/24` или иной RFC1918 диапазон)
- **Адрес сервера в VPN**: Первый доступный IP из подсети, например `10.8.0.1/24`
- **Адрес клиента в VPN**: Уникальный IP на каждого клиента, например `10.8.0.2/24`
- **Порт**: по умолчанию `51820/udp` (можно выбрать любой другой UDP‑порт)

### 4. Включите маршрутизацию (IP forwarding)

Включите пересылку пакетов, чтобы сервер мог маршрутизировать трафик между VPN и публичным интернетом:

```bash
sysctl -w net.ipv4.ip_forward=1
sysctl -w net.ipv6.conf.all.forwarding=1

# Примените изменения
sysctl --system

# Сделайте изменение постоянным
echo 'net.ipv4.ip_forward=1' > /etc/sysctl.d/99-wireguard-forwarding.conf
echo 'net.ipv6.conf.all.forwarding=1' >> /etc/sysctl.d/99-wireguard-forwarding.conf
```

### 5. Создайте конфигурацию сервера

Создайте основной конфигурационный файл WireGuard `/etc/wireguard/wg0.conf`:

```ini
[Interface]
Address = 10.8.0.1/24
ListenPort = 51820
PrivateKey = [SERVER_PRIVATE_KEY]
PostUp = iptables -t nat -A POSTROUTING -s 10.8.0.0/24 -o [INTERFACE] -j MASQUERADE; iptables -A FORWARD -i %i -o [INTERFACE] -j ACCEPT; iptables -A FORWARD -i [INTERFACE] -o %i -m state --state RELATED,ESTABLISHED -j ACCEPT
PostDown = iptables -t nat -D POSTROUTING -s 10.8.0.0/24 -o [INTERFACE] -j MASQUERADE; iptables -D FORWARD -i %i -o [INTERFACE] -j ACCEPT; iptables -D FORWARD -i [INTERFACE] -o %i -m state --state RELATED,ESTABLISHED -j ACCEPT

[Peer]
PublicKey = [CLIENT_PUBLIC_KEY]
AllowedIPs = 10.8.0.2/32
```

**Важно**: замените плейсхолдеры:

- `[SERVER_PRIVATE_KEY]`: содержимое `/etc/wireguard/server/privatekey`
- `[CLIENT_PUBLIC_KEY]`: содержимое `/etc/wireguard/client/publickey`
- `[INTERFACE]`: интерфейс вашего сервера (например, `eth0`, `ens3`)

Выставьте безопасные права на файл конфигурации:

```bash
chmod 600 /etc/wireguard/wg0.conf
```

## Настройка клиента

Создайте конфигурационный файл клиента `/etc/wireguard/client/client.conf`:

```ini
[Interface]
PrivateKey = [CLIENT_PRIVATE_KEY]
Address = 10.8.0.2/24
DNS = 1.1.1.1, 1.0.0.1

[Peer]
PublicKey = [SERVER_PUBLIC_KEY]
AllowedIPs = 0.0.0.0/0, ::/0
Endpoint = [SERVER_IP]:51820
PersistentKeepalive = 25
```

**Замените плейсхолдеры**:

- `[CLIENT_PRIVATE_KEY]`: содержимое `/etc/wireguard/client/privatekey`
- `[SERVER_PUBLIC_KEY]`: содержимое `/etc/wireguard/server/publickey`
- `[SERVER_IP]`: публичный IP‑адрес вашего сервера

Выставьте безопасные права:

```bash
chmod 600 /etc/wireguard/client/client.conf
```

## Запуск VPN

### Включение и запуск сервиса WireGuard

```bash
# Включите автозапуск при загрузке системы
systemctl enable wg-quick@wg0

# Запустите сервис WireGuard
systemctl start wg-quick@wg0
```

Альтернативный способ через `wg-quick`:

```bash
# "Поднять" интерфейс WireGuard
wg-quick up wg0

# "Опустить" интерфейс WireGuard
wg-quick down wg0
```

## Проверка

### Проверка статуса WireGuard

```bash
# Проверить, запущен ли WireGuard
systemctl status wg-quick@wg0

# Посмотреть детали интерфейса WireGuard
wg show

# Проверить сетевые интерфейсы
ip addr show wg0
```

### Тест подключения VPN

1. **Скопируйте конфигурацию клиента** на устройство
2. **Установите клиент WireGuard** на устройство
3. **Импортируйте конфигурационный файл**
4. **Подключитесь к VPN**
5. **Проверьте, что внешний IP** изменился на IP вашего сервера (например, через [check-host.net](https://check-host.net))

## Настройка брандмауэра

Если у вас включён брандмауэр (`ufw` или `firewalld`), разрешите трафик WireGuard:

### Для UFW (Ubuntu/Debian):

```bash
# Разрешите порт WireGuard
ufw allow 51820/udp

# Разрешите форвардинг (при необходимости)
ufw route allow in on wg0 out on eth0
```

### Для firewalld (RHEL/CentOS):

```bash
# Разрешите порт WireGuard
firewall-cmd --permanent --add-port=51820/udp
firewall-cmd --reload

# Включите маскарадинг
firewall-cmd --permanent --add-masquerade
firewall-cmd --reload
```

## Добавление дополнительных клиентов

Чтобы добавить новых клиентов на сервер WireGuard:

1. **Создайте папку клиента**:

   ```bash
   mkdir -p /etc/wireguard/client2
   chmod 700 /etc/wireguard/client2
   ```

2. **Сгенерируйте новую пару ключей** для клиента:

   ```bash
   wg genkey | tee /etc/wireguard/client2/privatekey | wg pubkey > /etc/wireguard/client2/publickey
   ```

3. **Добавьте новую секцию peer** в `/etc/wireguard/wg0.conf`:

   ```ini
   [Peer]
   PublicKey = [CLIENT2_PUBLIC_KEY]
   AllowedIPs = 10.8.0.3/32
   ```

4. **Перезагрузите конфигурацию WireGuard**:
   ```bash
   systemctl reload wg-quick@wg0
   ```

## Установка клиента WireGuard

### Установка на Windows

1. Перейдите на официальный сайт [wireguard.com](https://www.wireguard.com/install/) и загрузите установщик для Windows

![Загрузка клиента WireGuard для Windows](/images/vpn/wireguard-easy/windows-download-wireguard.png){data-zoomable}

2. Запустите скачанный файл и следуйте инструкциям мастера установки

![Запуск установщика WireGuard](/images/vpn/wireguard-easy/windows-open-installer.png){data-zoomable}

![Процесс установки WireGuard](/images/vpn/wireguard-easy/windows-install-wireguard.png){data-zoomable}

По завершении установки программа запустится автоматически.

3. Загрузите вашу конфигурацию в приложение

<video width="1440" autoplay controls loop muted>
  <source src="/videos/vpn/wireguard-easy/import-config-windows.mp4" type="video/mp4" />
</video>

4. Запустите VPN-соединение

Нажмите на кнопку "Активировать" (или "Activate" в английской версии).

![Активация VPN-соединения в WireGuard на Windows](/images/vpn/wireguard-easy/windows-connect.png){data-zoomable}

**Поздравляем!**

Теперь весь ваш сетевой трафик будет защищен и направлен через VPN-туннель.

### Установка на macOS

1. Найдите и установите клиент WireGuard через Mac App Store

![Страница WireGuard в Mac App Store](/images/vpn/wireguard-easy/app-store.png){data-zoomable}

2. Найдите и откройте установленное приложение

![Поиск WireGuard в Spotlight на macOS](/images/vpn/wireguard-easy/macos-wireguard-search.png){data-zoomable}

3. Добавьте вашу конфигурацию в программу

<video width="1440" autoplay controls loop muted>
  <source src="/videos/vpn/wireguard-easy/import-config-macos.mp4" type="video/mp4" />
</video>

При первом запуске система macOS запросит разрешение на создание VPN-подключений.

**Обязательно нажмите "Разрешить" ("Allow"), иначе приложение не сможет функционировать корректно**.

Активируйте VPN-соединение.

Нажмите на кнопку "Активировать" ("Activate" в английской версии).

![Включение VPN-соединения в WireGuard на macOS](/images/vpn/wireguard-easy/macos-connect.png){data-zoomable}

**Отлично!**

Теперь ваше интернет-соединение защищено VPN-туннелем.

### Мобильные устройства (Android/iOS)

1. Загрузите официальное приложение WireGuard:
   - Для Android: [Google Play Store](https://play.google.com/store/apps/details?id=com.wireguard.android&hl=en)
   - Для iOS: [App Store](https://apps.apple.com/us/app/wireguard/id1441195209)
2. Импортируйте файл конфигурации
3. Активируйте VPN-подключение в приложении

## Устранение неполадок

### Частые проблемы

1. **Нет подключения**:

   - Проверьте, открыт ли порт 51820/udp в брандмауэре
   - Убедитесь, что указан правильный публичный IP сервера
   - Проверьте, включён ли IP forwarding

2. **Нет доступа в интернет через VPN**:

   - Проверьте корректность правил iptables
   - Убедитесь, что в конфиге указан верный интерфейс
   - Проверьте, включён ли маскарадинг
   - Убедитесь, что ваш интернет-провайдер не накладывает ограничения на данный VPN-протокол

3. **Сервис WireGuard не запускается**:
   - Проверьте синтаксис конфигурации: `wg-quick up wg0`
   - Убедитесь, что права на файлы равны 600
   - Посмотрите логи: `journalctl -u wg-quick@wg0`

### Полезные команды

```bash
# Подробный статус WireGuard
wg show all

# Просмотр логов WireGuard
journalctl -u wg-quick@wg0 -f

# Тест конфигурации без запуска
wg-quick up wg0 --dry-run

# Перезапуск WireGuard
systemctl restart wg-quick@wg0
```

### Отладка сети

```bash
# Посмотреть таблицу маршрутизации
ip route show table all

# Мониторинг трафика
tcpdump -i wg0

# Проверка связности
ping -I wg0 8.8.8.8
```

## Рекомендации по безопасности

1. **Храните приватные ключи в секрете**: не делитесь ими, выставляйте права 600
2. **Периодически меняйте ключи**: регулярный перевыпуск ключей повышает безопасность
3. **Ограничивайте трафик в брандмауэре**: разрешайте только доверенные IP
4. **Мониторьте подключения**: регулярно проверяйте клиентов через `wg show`
5. **Обновляйте систему**: поддерживайте WireGuard и ОС в обновленном состоянии

## Заключение

Теперь у вас есть полностью рабочий сервер WireGuard VPN. Конфигурацию клиента можно импортировать в приложения WireGuard на Windows, macOS, iOS, Android и Linux для безопасного подключения к вашему серверу.

Не забудьте безопасно перенести конфигурационные файлы на ваши устройства, а также берегите ваши приватные ключи.

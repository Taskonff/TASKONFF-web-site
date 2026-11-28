# Как разместить TASKONFF на хостинге

Сайт и админ-панель запускаются вместе из каталога `TASKONFF/site`. Подойдёт VPS или хостинг с **Node.js 24**, долгоживущим процессом и постоянным диском. Для текущей базы SQLite используйте один экземпляр приложения. При загрузке одного HTML-файла будет доступен только предпросмотр.

Ниже пример для **Ubuntu 24.04, Nginx и HTTPS от Let's Encrypt**. В примерах замените `taskonff.example` на свой домен. Команды выполняйте через SSH под обычным пользователем с доступом к `sudo`.

## 1. Подготовьте домен и сервер

- Направьте DNS-запись `A` домена на IPv4-адрес сервера. Укажите `AAAA`, если используете IPv6. Откройте входящие порты 80 и 443 для веб-трафика.
- Установите Node.js 24 и проверьте `node --version`. Сохраните результат `command -v node` для настройки службы. [Официальная загрузка Node.js](https://nodejs.org/ru/download/)
- Загрузите `TASKONFF-site-and-CMS-hosting.zip` в домашний каталог пользователя на сервере через SFTP или панель хостинга.

```sh
sudo apt update
sudo apt install unzip nginx certbot python3-certbot-nginx
sudo install -d -m 755 -o "$(id -un)" -g "$(id -gn)" /opt/taskonff
sudo install -d -m 700 -o "$(id -un)" -g "$(id -gn)" /var/lib/taskonff
unzip "$HOME/TASKONFF-site-and-CMS-hosting.zip" -d /opt/taskonff
cd /opt/taskonff/TASKONFF/site
```

Если вы уже редактировали сайт в локальной админке, остановите локальный сайт и приватно скопируйте **содержимое** прежнего `site/data/` в `/var/lib/taskonff` перед настройкой. Перенесите также скрытый `.payload-secret` и каталог `media`. Владелец файлов должен совпадать с пользователем службы. Если раньше для подписи использовали `PAYLOAD_SECRET`, задайте то же закрытое значение на сервере.

## 2. Настройте окружение, создайте администратора и соберите сайт

В каталоге `/opt/taskonff/TASKONFF/site` создайте `.env` со следующими строками. В `SITE_URL` укажите точный HTTPS-адрес, по которому посетители и администраторы будут заходить на сайт, без путей вроде `/ru` или `/admin`.

```dotenv
SITE_URL=https://taskonff.example
CMS_DATA_DIR=/var/lib/taskonff
```

```sh
chmod 600 .env
npm ci --include=dev
NODE_ENV=production npm run admin:setup
npm run build
```

В интерактивном терминале настройка запросит ваш email и пароль администратора от 12 символов. При переносе действующей базы можно войти с существующей учётной записью. SQLite, ключ подписи и изображения сохраняются в `CMS_DATA_DIR`.

## 3. Запустите сайт как службу

Создайте `/etc/systemd/system/taskonff.service` через `sudoedit` или `sudo nano`. Замените `deploy` на пользователя из `id -un`, а `/usr/bin/node` на результат `command -v node`.

```ini
[Unit]
Description=TASKONFF Website and CMS
After=network.target

[Service]
Type=simple
User=deploy
WorkingDirectory=/opt/taskonff/TASKONFF/site
Environment=NODE_ENV=production
ExecStart=/usr/bin/node /opt/taskonff/TASKONFF/site/node_modules/next/dist/bin/next start --hostname 127.0.0.1 --port 3000
Restart=on-failure
RestartSec=3

[Install]
WantedBy=multi-user.target
```

```sh
sudo systemctl daemon-reload
sudo systemctl enable --now taskonff
curl -fsS -o /dev/null -w '%{http_code}\n' http://127.0.0.1:3000/ru
```

Ожидаемый ответ на `/ru` — `200`. Если служба не стартовала, проверьте `sudo journalctl -u taskonff -n 60 --no-pager`.

## 4. Подключите домен и HTTPS

Создайте `/etc/nginx/sites-available/taskonff`. Замените значение `server_name` на ваш домен. Лимит тела запроса оставляет место для загрузки изображений через CMS.

```nginx
server {
    listen 80;
    listen [::]:80;
    server_name taskonff.example;
    client_max_body_size 10m;

    location / {
        proxy_pass http://127.0.0.1:3000;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Forwarded-Host $host;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_buffering off;
    }
}
```

Выполните следующие команды при первом подключении сайта. Для сертификата укажите тот же домен, что и в `SITE_URL`.

```sh
sudo ln -s /etc/nginx/sites-available/taskonff /etc/nginx/sites-enabled/taskonff
sudo nginx -t
sudo systemctl enable --now nginx
sudo systemctl reload nginx
sudo certbot --nginx --redirect -d taskonff.example
```

После выдачи сертификата откройте `https://ваш-домен/ru` и `https://ваш-домен/admin`. Для другого порта приложения укажите одинаковый порт в службе и в `proxy_pass`.

## 5. Проверьте сайт перед запуском

1. Откройте `/ru`, `/en` и `/admin` по HTTPS. Войдите в админку.
2. Внесите рабочие контакты через **Настройки сайта → Публичные контакты**. Утвердите тексты политики конфиденциальности и условий использования, которые сейчас помечены как проекты, и опубликуйте их версии RU и EN.
3. Отправьте тестовую заявку и найдите её в **Заявках на проекты**. Заявки сохраняются в CMS сразу. Если нужны внешние уведомления, можно подключить HTTPS-вебхук по инструкции `site/CMS-README-RU.md`.
4. Перед обновлениями остановите службу и сделайте приватную резервную копию всего каталога `CMS_DATA_DIR`. Храните его отдельно от папки сайта и используйте при следующих развертываниях.

## Если хостинг управляет приложениями через панель

Создайте Node.js-приложение из `TASKONFF/site`, выберите Node.js 24 и укажите `SITE_URL` и `CMS_DATA_DIR` в настройках окружения. Используйте доступную в панели консоль или SSH для `npm ci --include=dev`, `npm run admin:setup` и `npm run build`. Команда запуска — `npm start`. Если платформа выдаёт приложению порт, используйте её переменную `PORT`. Домен, HTTPS и автоматический запуск настраиваются в панели хостинга.

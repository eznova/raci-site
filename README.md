# RACI: публикация через nginx

Готовая статическая версия сайта. Сборка и Node.js на сервере не нужны.

## Состав

- `site/` — актуальные HTML, CSS, JavaScript, данные матрицы и текстура шума.
- `compose.yaml` — nginx, порт хоста `8064`, порт контейнера `80`.
- `nginx/default.conf` — корень сайта `/var/www/html`.
- `systemd/raci-site-pull.service` — `git pull` от отдельного пользователя.
- `systemd/raci-site-pull.timer` — запуск каждые 30 секунд.

## Быстрый запуск без автообновления

Нужны Docker Engine и Docker Compose v2. В распакованной директории:

```bash
docker compose config
docker compose run --rm nginx nginx -t
docker compose up -d
docker compose ps
curl -f http://127.0.0.1:8064/
```

Сайт: `http://SERVER_IP:8064`. Compose публикует порт на всех интерфейсах хоста.
Встроенной авторизации Sites здесь нет: доступ определяется сетью и настройками сервера.
Для доступа извне откройте TCP 8064 в сетевом экране сервера. TLS в этот комплект не включен.

## Подготовка Git-репозитория

Поместите содержимое этого архива в корень вашего отдельного Git-репозитория:
`compose.yaml`, `nginx/`, `systemd/`, `site/`, `README.md`.
Закоммитьте и отправьте файлы в выбранную ветку. URL репозитория и ветка ниже —
заполнители: замените `YOUR_REPOSITORY_URL` и при необходимости `main`.
Архив не содержит `.git`, ключей или учетных данных.

## Установка на Linux с systemd

Нужны Git, OpenSSH client, Docker Engine и Compose v2. Команды выполняются
из учетной записи с `sudo`. Путь `/opt/raci-site` должен быть свободен.

```bash
sudo useradd --system --user-group --create-home \
  --home-dir /var/lib/raci-deploy --shell /usr/sbin/nologin raci-deploy
sudo install -d -o raci-deploy -g raci-deploy -m 0755 /opt/raci-site
```

Для приватного репозитория настройте доступ **пользователю raci-deploy**, например
read-only SSH deploy key в `/var/lib/raci-deploy/.ssh/`. Каталог `.ssh` — 0700,
приватный ключ — 0600, владелец — `raci-deploy`. Заранее добавьте проверенный ключ
Git-сервера в `known_hosts`. Сервис не запрашивает пароль и не использует интерактивный
SSH-agent. Для публичного HTTPS-репозитория ключ не нужен.

```bash
sudo -u raci-deploy -H git clone --branch main --single-branch \
  YOUR_REPOSITORY_URL /opt/raci-site

cd /opt/raci-site
sudo docker compose config
sudo docker compose run --rm nginx nginx -t
sudo docker compose up -d

# Проверка Git-доступа именно от имени сервисного пользователя
sudo -u raci-deploy -H env GIT_TERMINAL_PROMPT=0 \
  GIT_SSH_COMMAND='ssh -o BatchMode=yes -o ConnectTimeout=10' \
  git -C /opt/raci-site pull --ff-only --no-rebase --no-recurse-submodules

sudo install -m 0644 systemd/raci-site-pull.service /etc/systemd/system/
sudo install -m 0644 systemd/raci-site-pull.timer /etc/systemd/system/
sudo systemctl daemon-reload
sudo systemctl enable --now raci-site-pull.timer
```

Пользователь `raci-deploy` владеет репозиторием, но не нуждается в группе `docker`.
При другом пути или пользователе исправьте `User`, `Group`, `WorkingDirectory`
и `ReadWritePaths` в service-файле до установки.

## Как работают обновления

Таймер запускает сервис с интервалом 30 секунд и допуском 1 секунда.
Одновременных запусков одного сервиса нет; зависший pull прерывается через 25 секунд.
Первый запуск — после 30 секунд от загрузки системы; если этот момент уже прошел,
при включении таймера запуск произойдет сразу. Пропущенные запуски не накапливаются.

`git pull --ff-only --no-rebase` обновляет текущую ветку из ее настроенного upstream.
При расхождении истории обновление завершится ошибкой, без merge или сброса локальных
изменений. Серверную копию не редактируйте вручную. Ошибки доступа, сети и Git видны
в журнале; следующая попытка произойдет по таймеру.

Обновления `site/` сразу видны nginx через bind mount; перезапуск контейнера не нужен.
Браузер перепроверяет файлы при следующем запросе; уже открытая страница сама не перезагружается.
Обычный pull меняет файлы последовательно: при одновременной загрузке страницы возможна
краткая смесь версий. Это простая схема публикации без атомарного переключения релизов.

Изменения конфигурации применяются вручную:

```bash
# После изменения nginx/default.conf
sudo docker compose exec nginx nginx -t
sudo docker compose exec nginx nginx -s reload

# После изменения compose.yaml
sudo docker compose up -d

# После изменения unit-файлов: повторите install из инструкции и затем
sudo systemctl daemon-reload
sudo systemctl restart raci-site-pull.timer
```

## Диагностика и отключение

```bash
sudo systemctl list-timers raci-site-pull.timer
sudo systemctl status raci-site-pull.service
sudo journalctl -u raci-site-pull.service -n 50 --no-pager
sudo docker compose logs --tail=50 nginx
curl -f http://127.0.0.1:8064/healthz

# Остановить только автоматический pull
sudo systemctl disable --now raci-site-pull.timer
```

После успешного выполнения oneshot-сервис может иметь статус `inactive (dead)` —
это нормально: расписание обслуживает активный `.timer`.

Образ `nginx:stable-alpine` использует изменяемый тег. Для воспроизводимой инфраструктуры
можно закрепить digest после проверки образа в вашей среде.

## Проверка комплекта

Проверены состав сайта, совпадение с текущими опубликованными исходниками и синтаксис
JavaScript. Docker и nginx в среде подготовки отсутствуют: запуск контейнера и `nginx -t`
нужно выполнить на вашем сервере командами выше.

Справка: https://docs.docker.com/reference/compose-file/services/
и https://www.freedesktop.org/software/systemd/man/latest/systemd.timer.html

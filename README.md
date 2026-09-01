# RUSCAR AI Telegram Bot

Бот отвечает только в одной разрешённой группе или теме Telegram и использует
общую историю диалога для всех участников этой темы.

Локальная база знаний имеет приоритет. Основная модель Groq Qwen отвечает по
базе, а если данных недостаточно и `ENABLE_WEB_FALLBACK=true`, Groq Compound
выполняет веб-поиск и возвращает ответ со ссылками на источники.

## Безопасная настройка

1. Перевыпустите все ключи, которые ранее были опубликованы в переписке.
2. Скопируйте `.env.example` в `.env`.
3. Вставьте новые ключи только в локальный `.env`.

Не отправляйте `.env` в чат и не добавляйте его в Git.

## Первый запуск через Docker

```bash
docker compose up -d --build
docker compose logs -f ruscar-bot
```

После запуска администратор группы должен открыть тему `🤖 RUSCAR AI` и
отправить в ней:

```text
/set_ai_topic
```

Бот сохранит ID группы и ID темы в `data/ruscar_settings.json`. После этого
обычные сообщения из всех остальных чатов и тем будут молча игнорироваться.

Команда `/new_dialog` очищает общую историю разрешённой темы и доступна только
администратору группы.

## Обновление базы знаний

Файл `RUSCAR_PRO.txt` подключён в контейнер как read-only volume. После его
изменения перезапуск контейнера не требуется: бот читает актуальную версию перед
каждым запросом.

## Полезные команды сервера

```bash
docker compose ps
docker compose logs --tail=200 ruscar-bot
docker compose restart ruscar-bot
docker compose down
```

## Запуск на сервере из приватного Docker Hub

Образ содержит `bot.py` и `RUSCAR_PRO.txt`, но не содержит `.env` и ключи.
Создайте приватный репозиторий `ruscar-bot` в Docker Hub, затем локально:

```bash
docker login
docker buildx build --platform linux/amd64 \
  -t DOCKERHUB_USERNAME/ruscar-bot:1.2.3 --push .
```

На сервере скопируйте `compose.server.yaml` и `.env.server.example`, переименуйте
`.env.server.example` в `.env`, заполните секреты и точное имя образа. Затем:

```bash
docker login --username DOCKERHUB_USERNAME
docker compose -f compose.server.yaml pull
docker compose -f compose.server.yaml up -d
docker compose -f compose.server.yaml logs --tail=100 ruscar-bot
```

Для входа на сервере используйте отдельный Docker Hub access token только с
правом Read. Одновременно не запускайте локальную и серверную копии polling-бота.

## Jenkins CI/CD

Pipeline описан в `Jenkinsfile` и выполняет checkout, проверку Python, сборку и
push Docker-образа, затем обновляет production-сервер по SSH.

На Jenkins-агенте должны быть установлены Git, Docker CLI и SSH client. В
Jenkins Credentials необходимо создать:

* `dockerhub-credentials` — Username with password, где password является
  Docker Hub access token с правом Read/Write;
* `ruscar-production-ssh` — Secret file с отдельным приватным SSH-ключом
  Jenkins (`ruscar_jenkins_ed25519`);
* `ruscar-production-host` — Secret text с IP или DNS production-сервера.

Telegram- и Groq-ключи Jenkins не получает: они остаются в серверном `.env`.
Jenkins job следует настроить как Pipeline from SCM с веткой `*/main` и Script
Path `Jenkinsfile`.

### Локальный Jenkins через Docker Desktop

Docker Desktop должен работать в режиме Linux containers. Запуск Jenkins:

```powershell
docker compose -f compose.jenkins.yaml up -d --build
docker exec ruscar-jenkins cat /var/jenkins_home/secrets/initialAdminPassword
```

Откройте `http://localhost:8080`, вставьте первоначальный пароль и создайте
учётную запись администратора. Конфигурация использует отдельный Docker daemon
с TLS; порт Jenkins доступен только с локального компьютера.

После настройки credentials параметр `DEPLOY_TO_PRODUCTION` по умолчанию включён.
Jenkins проверяет ветку `main` примерно каждые пять минут и запускает pipeline,
если обнаружен новый commit. Локальный Jenkins и Docker Desktop должны быть
запущены.

При неудачном старте нового контейнера pipeline возвращает сервер на Docker-образ,
который был указан в `.env` перед deployment. Первый запуск после добавления или
изменения trigger выполните вручную, чтобы Jenkins загрузил новую конфигурацию.

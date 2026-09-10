# Запуск MCP-серверов 1С в Docker на ноутбуке с Astra Linux

Пошаговая инструкция для того, кто с Docker раньше не работал. Все команды выполняются
в терминале ноутбука с Astra Linux. Строки, начинающиеся с `#`, — пояснения, их набирать не нужно.

Идея решения: контейнер — это упакованное приложение со всеми своими зависимостями.
Ему не важно, какая ОС на хосте: одни и те же образы работают и в Windows, и в Astra Linux.
Поэтому MCP-серверы, которые не запускались напрямую в Astra, в Docker поднимутся без правок.

## 0. Что именно будет запущено

Семь сервисов из одного файла `docker-compose.yml`:

| Контейнер | Порт на ноутбуке | Зачем нужен |
|---|---|---|
| `qdrant` | 6333, 6334 | векторная база: хранит проиндексированные метаданные и справку |
| `embedding-service` | 5000 | превращает текст в векторы (модель `sergeyzh/BERTA` уже внутри образа) |
| `loader` | 8501 | веб-страница разовой загрузки справки синтакс-помощника |
| `metadata-loader` | 8502 | веб-страница загрузки и обновления описаний метаданных из XML-выгрузки |
| `mcp-metadata` | 9001 | MCP-сервер поиска по метаданным |
| `mcp-help` | 9002 (MCP), 9092 (REST + Swagger) | MCP-сервер поиска по синтакс-помощнику |
| `mcp-bsl-checker` | 9004 | MCP-сервер проверки синтаксиса через BSL Language Server |

Первые четыре — вспомогательные, последние три — те самые MCP, которые подключаются к IDE.

## 1. Проверить, что ноутбук подходит

```bash
uname -m                  # должно быть x86_64
nproc                     # 4 и больше
free -g                   # 8 ГБ ОЗУ и больше
df -h /var/lib/docker /   # 20 ГБ свободного места и больше
cat /etc/astra_version    # версия Astra Linux (1.7 или 1.8)
```

Требования издателя: x86_64 (не ARM и не Эльбрус), 4 ядра, 8 ГБ ОЗУ, 20 ГБ диска.
Если `uname -m` выдал `aarch64` или `e2k` — эти образы там не запустятся, нужен другой хост:
Astra для ARM и Эльбруса — отдельные линейки (4.7/4.8 и 8.1), образы под них издателем не собираются.

## 2. Установить Docker

В штатных репозиториях Astra Linux Docker есть, сторонние репозитории подключать не нужно.
Важно: плагин Compose v2 называется `docker-compose-v2` (не `docker-compose-plugin`, как в Debian).

```bash
# Устаревший docker-compose версии 1 удаляем, если он был установлен
sudo apt remove docker-compose

sudo apt update
sudo apt install docker.io docker-compose-v2
```

Для Astra Linux 1.7 пакет `docker-compose-v2` лежит в *базовом* репозитории (`base`) — если apt его не находит,
подключите базовый репозиторий диска/зеркала. В 1.8 он в основном (`main`), а старый `docker-compose`
с обновления 1.8.3 больше не поддерживается.

Разрешить своему пользователю работать с Docker без `sudo`:

```bash
sudo usermod -aG docker $USER
# новая группа применяется только в новом сеансе:
exec su - $USER
```

Проверка, что всё встало:

```bash
sudo systemctl status docker        # ожидаем active (running)
sudo systemctl enable docker        # автозапуск при включении ноутбука
docker --version
docker compose version              # должно быть Docker Compose version v2.x
docker run --rm hello-world         # тестовый контейнер, скачается из интернета
```

Если `hello-world` вывел приветствие — Docker работает. Если что-то упало, смотрите раздел 10.

Про режимы работы. Выше описан *привилегированный* режим — он проще и для нашего стека достаточен.
Сама Astra рекомендует *непривилегированный* (rootless): `sudo apt install rootless-helper-astra`,
`sudo systemctl start rootless-docker@<пользователь>@<метка_безопасности>`, после чего все команды из этой инструкции
выполняются через `rootlessenv`, например `rootlessenv docker compose up -d`. На hardened-ядре rootless не работает.
Если в организации есть требования по безопасности — уточните режим у администраторов до установки.

## 3. Перенести файлы поставки на ноутбук

На ноутбуке нужны всего три файла из папки `mcp/mcp_v1.6.3` этого проекта:

- `docker-compose.yml` — описание всех контейнеров;
- `.env.example` — шаблон параметров (из него сделаем `.env`);
- `bsl-ls/.bsl-language-server.json` — настройки проверок синтаксиса.

Создаём рабочий каталог и кладём в него эти файлы. Самый простой способ — забрать из репозитория проекта:

```bash
sudo apt install git
git clone https://github.com/DakCreed/kodik_project.git ~/kodik_project
mkdir -p ~/mcp-1c
cp -r ~/kodik_project/mcp/mcp_v1.6.3/. ~/mcp-1c/
cd ~/mcp-1c
ls -la            # должны быть docker-compose.yml, .env.example и каталог bsl-ls
```

В репозитории лежат только эти три файла и сама инструкция (`mcp/DEPLOY-astra-linux.md`),
поэтому лишнего не скопируется.

Альтернатива без git — копирование с Windows-машины (выполняется в PowerShell на Windows,
`user` и `192.168.1.50` замените на своё имя пользователя и IP ноутбука):

```powershell
ssh user@192.168.1.50 "mkdir -p ~/mcp-1c/bsl-ls"
scp F:\1c\kodik_project\mcp\mcp_v1.6.3\docker-compose.yml user@192.168.1.50:~/mcp-1c/
scp F:\1c\kodik_project\mcp\mcp_v1.6.3\.env.example      user@192.168.1.50:~/mcp-1c/
scp F:\1c\kodik_project\mcp\mcp_v1.6.3\bsl-ls\.bsl-language-server.json user@192.168.1.50:~/mcp-1c/bsl-ls/
```

Важно: остальные материалы поставки (PDF-документация, обработка `.epf`, HTML-описания обновлений) —
лицензионные файлы издателя. В публичный GitHub-репозиторий они не выкладываются, поэтому папка `mcp`
почти целиком добавлена в `.gitignore`; переносите их только внутри организации.

## 4. Подготовить каталоги проекта и файл .env

Нужен каталог с XML-выгрузкой исходников 1С — именно его читают проверка синтаксиса и загрузчик метаданных.
Структура обязательная: подкаталог `Configuration`, при наличии расширений — `Extensions/<ИмяРасширения>`.

```bash
mkdir -p ~/projects/proj1/1c-src/Configuration
mkdir -p ~/projects/proj1/1c-src/Extensions
```

Выгрузку делаете конфигуратором 1С: *Конфигурация → Выгрузить конфигурацию в файлы* в `~/projects/proj1/1c-src/Configuration`.

Теперь создаём файл параметров:

```bash
cd ~/mcp-1c
cp .env.example .env
nano .env       # или любой другой редактор
```

В `.env` задайте свой абсолютный путь и режим слежения за файлами:

```ini
SRC_1C_DIR=/home/user/projects/proj1/1c-src
WATCHER_POLLING=0
```

Подставьте реальное имя пользователя вместо `user` (посмотреть можно командой `echo $HOME`).
`WATCHER_POLLING=0` — нативные события файловой системы, для Linux это быстрее; значение `1`
(опрос) нужно только в Windows и для сетевых каталогов.

Именно из-за этой переменной файл `docker-compose.yml` теперь кроссплатформенный: жёстко прописанный
издателем путь `C:/you/project/1c-src` заменён на `${SRC_1C_DIR}` в трёх местах — в томах `metadata-loader`,
в томах `mcp-bsl-checker` и в его переменной `HOST_WORKSPACE_DIR`. Один и тот же compose-файл работает
и на Linux, и на Windows, меняется только `.env`.

## 5. Первый запуск

```bash
cd ~/mcp-1c
docker compose pull      # скачать образы, несколько ГБ, первый раз это долго
docker compose up -d     # запустить всё в фоне
docker compose ps        # список контейнеров и их состояние
```

В колонке `STATUS` у всех должно быть `Up` (у части сервисов — `Up (healthy)`).
Если какой-то контейнер перезапускается, смотрите его журнал:

```bash
docker compose logs -f mcp-metadata     # вместо mcp-metadata — имя нужного контейнера
docker compose logs --tail=50           # последние 50 строк по всем сразу
```

Полезно знать: `docker compose` всегда запускается из каталога, где лежит `docker-compose.yml`.

## 6. Разовая индексация данных

До индексации MCP-серверы отвечают пустыми результатами — это не ошибка, базу нужно наполнить.

**Справка синтакс-помощника** — открыть на ноутбуке в браузере http://localhost:8501

1. Имя коллекции: `1c_help_rag` (ровно это значение уже прописано в `docker-compose.yml` в переменной
   `COLLECTION_NAME_HELP` контейнера `mcp-help`).
2. Выбрать файл справки платформы. Найти его на Astra Linux можно так:

```bash
ls /opt/1cv8/x86_64/*/shcntx_ru.hbk
```

**Описания метаданных** — открыть http://localhost:8502

1. Имя коллекции: `1c_metadata_rag` (это же значение указывается в настройках IDE в заголовке `x-collection-name`).
2. Кнопка «Преобразовать в json и загрузить в Qdrant» — полная индексация.
3. Кнопка «1-Инициализировать базу» — таблица хешей файлов для отслеживания изменений.
4. Кнопка «2-Включить отслеживание» — фоновое слежение за изменениями XML.
5. В дальнейшем после выгрузки новых версий конфигурации достаточно кнопки «Актуализировать описания в Qdrant».

Время индексации: генерация JSON — десятки секунд, векторизация — единицы минут на конфигурацию
размера УТ; на ERP заметно дольше.

## 7. Проверить, что серверы отвечают

```bash
curl -s http://localhost:6333/healthz                       # Qdrant
curl -s http://localhost:5000/health                        # сервис векторизации
curl -s http://localhost:9001/health                        # MCP метаданных

# поиск по метаданным (коллекция передаётся заголовком)
curl -s --request POST http://localhost:9001/search \
  --header 'Content-Type: application/json' \
  --header 'x-collection-name: 1c_metadata_rag' \
  --data '{"query": "Продажи товаров и услуг", "object_type": "Документ"}'

# поиск по справке
curl -s --request POST http://localhost:9092/search \
  --header 'Content-Type: application/json' \
  --data '{"query": "Число прописью"}'

# проверка синтаксиса конкретного модуля
curl -s --request POST http://localhost:9004/check-file \
  --header 'Content-Type: application/json' \
  --data '{"file_path": "/home/user/projects/proj1/1c-src/Configuration/CommonModules/ОбщийМодуль/Ext/Module.bsl"}'
```

Swagger-документация MCP по справке: http://localhost:9092/docs

## 8. Подключить MCP к IDE

Адреса зависят от того, где запущена IDE.

**IDE на том же ноутбуке с Astra** — оставляем `localhost`. Файл настроек для Kodik/VS Code —
`.kodik/mcp.json` (или `.vscode/mcp.json`) в каталоге проекта 1С:

```json
{
  "servers": {
    "1c-metadata": {
      "type": "http",
      "url": "http://localhost:9001/mcp",
      "headers": { "x-collection-name": "1c_metadata_rag" }
    },
    "1c-help": {
      "type": "http",
      "url": "http://localhost:9002/mcp"
    },
    "1c-checker": {
      "type": "http",
      "url": "http://localhost:9004/bsl/mcp"
    }
  }
}
```

**IDE на другой машине (например, на этой Windows-машине)** — вместо `localhost` подставьте IP ноутбука
во всех трёх URL. Узнать IP и проверить, что порты слушаются:

```bash
hostname -I
ss -lntp | grep -E '9001|9002|9004'
```

И учтите две вещи: у MCP-серверов нет авторизации, поэтому открывать эти порты за пределы доверенной
локальной сети нельзя; а `1c-checker` проверяет файлы, лежащие *на ноутбуке* по пути из `SRC_1C_DIR`, —
пути в запросах агента должны быть путями внутри этого каталога.

Если нужен доступ только с самого ноутбука, надёжнее запретить внешние подключения — привяжите порты
к локальному интерфейсу в `docker-compose.yml`, например `"127.0.0.1:9001:9001"` вместо `"9001:9001"`.

## 9. Ежедневная эксплуатация

```bash
cd ~/mcp-1c

docker compose ps                 # что запущено
docker compose stop               # остановить (данные Qdrant сохраняются)
docker compose start              # запустить снова
docker compose restart mcp-help   # перезапустить один сервис
docker compose down               # остановить и удалить контейнеры (тома с данными остаются)
docker compose up -d              # поднять после down или после правок docker-compose.yml/.env

docker compose pull && docker compose up -d    # обновление образов на новые версии
docker stats                                   # текущее потребление CPU/памяти
```

Все сервисы описаны с `restart: unless-stopped`, поэтому после включения ноутбука они поднимутся сами —
при условии, что служба Docker в автозапуске (`sudo systemctl enable docker`).

Данные лежат в именованных томах Docker и переживают пересоздание контейнеров:
`qdrant_storage` — векторная база, `metadata_out` — сгенерированные JSON и база хешей
(этот том добавлен к поставке издателя, иначе после каждого `docker compose down` индексацию
метаданных пришлось бы начинать заново). Посмотреть тома: `docker volume ls`.
Удалить вместе с данными: `docker compose down -v` — только осознанно, потребуется повторная индексация.

## 10. Если что-то не работает

Общая последовательность:

```bash
docker compose ps                   # какой контейнер не в состоянии Up
docker compose logs --tail=100 <имя_контейнера>
sudo systemctl status docker
sudo journalctl -u docker -b -e | tail -50
```

Частые причины именно на Astra Linux:

1. **`permission denied` при обращении к `/var/run/docker.sock`.** Пользователь не в группе docker:
   `id $USER | grep docker`, при отсутствии — `sudo usermod -aG docker $USER` и заново войти в сеанс.
2. **`Error initializing network controller` в журнале Docker.** Конфликт с firewalld при включённом
   режиме изоляции Docker. Проверить состояние изоляции — `sudo astra-docker-isolation status`;
   решение описано в wiki Astra: отключить управление iptables у Docker и задать правила вручную.
3. **Контейнеры поднялись, но нет сети или DNS внутри них.** Проверить бэкенд iptables:
   `sudo update-alternatives --display iptables`. Docker совместим с `iptables-nft` и `iptables-legacy`,
   но не с правилами, созданными напрямую через `nft`; при проблемах переключиться на legacy:
   `sudo update-alternatives --set iptables /usr/sbin/iptables-legacy` и `sudo systemctl restart docker`.
   Проверка DNS: `docker exec -it qdrant getent hosts docker.io`.
4. **`DIGSIG: [ERROR] NOT SIGNED` в `dmesg`.** Включена замкнутая программная среда (ЗПС), она блокирует
   неподписанные исполняемые файлы: `sudo dmesg | grep -i digsig`, `cat /sys/digsig/elf_mode`
   (1 — строгий режим). Настройка Docker под включённую ЗПС в открытых источниках не описана —
   это вопрос к техподдержке Astra.
5. **Не запускается rootless-режим.** На hardened-ядре он невозможен (отключён `CONFIG_USER_NS`):
   `zcat /proc/config.gz | grep CONFIG_USER_NS`. Используйте обычный (привилегированный) режим, как в этой инструкции.
6. **`docker compose` не найден, есть только `docker-compose`.** Установлен старый Compose v1;
   поставьте `docker-compose-v2` (раздел 2). Синтаксис `${SRC_1C_DIR:?...}` в compose-файле требует именно v2.
7. **Ошибка вида `ne zadan SRC_1C_DIR`.** Не создан файл `.env` рядом с `docker-compose.yml`
   или в нём не заполнена переменная `SRC_1C_DIR` (раздел 4).
8. **`mcp-bsl-checker` не находит файл модуля.** Путь в запросе должен начинаться со значения `SRC_1C_DIR`,
   а сам каталог — быть доступен для чтения: `ls -l ~/projects/proj1/1c-src`.
9. **Не хватает памяти, контейнеры убиваются (`OOMKilled` в `docker compose ps`).** Тяжёлая модель
   векторизации на 8 ГБ ОЗУ не поместится; оставьте модель `sergeyzh/BERTA` по умолчанию и уменьшите
   `EMBEDDING_BATCH_SIZE`/`FILES_BATCH_SIZE` в `docker-compose.yml`.

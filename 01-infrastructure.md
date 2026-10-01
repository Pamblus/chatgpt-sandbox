# 01 · Infrastructure

Этот документ описывает физическую и логическую среду, в которой работает песочница ChatGPT Applied CaaS. Он отвечает на вопросы: где живёт эта машина, из чего она состоит, какие у неё ресурсы и права, кто в ней запускается по умолчанию, что спит и как это можно разбудить.

## 1. Где это физически живёт

Наша среда — это не просто контейнер. Это микро-виртуальная машина на базе Kata Containers, запущенная в Azure. Разница принципиальная: обычный контейнер использует ядро хоста, а Kata VM имеет собственное гостевое ядро, изолированное от хостового. Поэтому многие вещи, которые «протекли бы» в обычном Docker, здесь физически недостижимы — например, прямой доступ к хостовому ядру или его модулям.

Гостевое ядро — версия `6.18.44`. Хостовое ядро Azure заметно новее, что и подтверждает: мы действительно внутри виртуальной машины, а не в общем ядре.

Признаки, по которым это установлено:

- В `/proc/mounts` и mountinfo встречаются пути вида `/run/kata-containers/...`
- Файлы `/etc/hostname`, `/etc/hosts`, `/etc/resolv.conf` подключены через virtiofs — это файлы, которые хост прокидывает в гостя через специальный механизм виртуализации, а не обычные файлы внутри образа
- Корневая файловая система собрана как `erofs-multi-layer` — многослойный overlay поверх read-only образов

## 2. Сеть — что мы видим изнутри

Мы находимся в Azure Virtual Network с диапазоном `172.26.0.0/22`. У нашей виртуалки:

- Основной интерфейс `eth0`
- Наш IP-адрес попадает в диапазон `172.26.39.x/22`
- Шлюз по умолчанию — `172.26.36.1`
- DNS-сервер — `168.63.129.16` (стандартный Azure DNS resolver)

Шлюз ведёт себя как обычный сетевой узел: ARP до `172.26.36.1` работает, default route есть, `ping` до шлюза проходит. Но как только мы пытаемся выйти за его пределы — шлюз отвечает `RST` на любой TCP SYN. То есть сетевой маршрут есть, а дальше — глухая стена.

Про сетевые политики подробно в `08-network-boundary.md`. Здесь важно зафиксировать: **фильтр находится вне нашей VM**, на уровне Azure NSG или Kata/Nebula. Изнутри мы его не видим и не можем редактировать.

Один из глобальных флагов, который это подтверждает — `NETWORK=caas_packages_only`. Он говорит, что разрешены только обращения к внутренним пакетным мирам (Artifactory), а всё остальное блокируется.

## 3. Образ, платформа, идентификаторы

Наш образ — это приватная сборка OpenAI:

```
openaiappliedcaasprod.azurecr.io/chrome-chatgpt-prod-5p4:20260803182207-770cc6d08333-linux-amd64
```

Разберём его на части:

- `openaiappliedcaasprod.azurecr.io` — приватный Azure Container Registry OpenAI
- `chrome-chatgpt-prod-5p4` — имя production-сборки ChatGPT (вариант 5p4)
- `20260803182207` — дата и время сборки (3 августа 2026)
- `770cc6d08333...` — конкретный коммит во внутреннем монорепозитории OpenAI
- `linux-amd64` — архитектура

Отдельно есть переменная `VM_COMMIT_SHA=770cc6d08333b9844b1b63135928991b70e3343b` — это тот же коммит в полном 40-символьном виде.

Платформа управления называется **Nebula**. Она отвечает за жизненный цикл VM: создание, выдачу флагов, раздачу сетевой политики, доступ к интернету. От Nebula в окружении видны:

- `NEBULA_RUN=test-run` — мы в тестовом прогоне
- `NEBULA_VM_ID=test-vm` — идентификатор тестовой VM
- `NEBULA_USER=test-user` — тестовый пользователь
- `NEBULA_VM_LOGS_CAAS` — переменная для отправки логов в CaaS

Плюс от Nebula идут HTTP-заголовки, которые используются на уровне взаимодействия с UI:

- `X-Nebula-VM-Application`
- `X-Nebula-VM-URL`
- `X-Nebula-VM-Content-Type`
- `X-Nebula-VM-Title`
- `X-Nebula-Position-Dot` — координаты курсора в браузере
- `X-Nebula-Screenshot-Wait-For-Resources`
- `X-Nebula-VM-Screenshot-Delay`

Кластеры именуются `applied-caas24`, `applied-caas34` и так далее — номер меняется от пула к пулу. Конкретно в наших логах чаще всего встречался `applied-caas34`.

Есть внутренний gateway для пакетов: `packages.applied-caas-gateway1.internal`. Через него идут apt/npm-мирроры. Региональные версии: `packages-{5,6,8,9,11,12,13,16,18}-{east,west,eu}.internal`.

В образ вшита ссылка на сборку: `buildkite.com/openai-mono/monorepo/builds/610973`. Это внутренний CI-пайплайн OpenAI. Плюс go-ссылки `http://go/cua-caas-vm/` и `http://go/vmlogs/` — внутренняя документация.

## 4. Ресурсы

Машина довольно скромная — это правильно, потому что живёт она недолго, под одну задачу.

| Ресурс | Значение | Комментарий |
|---|---|---|
| Процессор | Intel Xeon Platinum 8573C | Видно 3 vCPU, но cgroup-лимит стоит на 2 CPU |
| CPU limit | `cpu.max=200000 100000` | 200 000 мкс из 100 000 за период — ровно 2 CPU-эквивалента. Третье ядро видно, но фактически не используется |
| RAM | 4 GiB | Лимит cgroup `4294967296` байт |
| Swap | нет | При нехватке памяти процессы не «уйдут в swap», а получат OOM |
| Диск | 32 GB overlay | Свободно около 30 GB |
| PID limit | максимум | Количество процессов не ограничено |

Overlay собран из read-only слоёв `erofs`, поверх которых — тонкий read-write слой. Любые наши изменения живут в этом тонком слое и исчезают при пересоздании VM.

Безлимитный PID позволяет supervisord'у запускать десятки сервисов, не упираясь в cgroup.

## 5. Capabilities (права)

Это один из важнейших разделов для понимания границ. Capabilities в Linux — это механизм, который разбивает всемогущего root на набор конкретных привилегий. То, что у нас есть, определяет, какие операции возможны.

### Доступные

У PID 1 и у нас набор одинаковый:

- `chown` — смена владельца файлов
- `dac_override` — обход прав доступа к файлам
- `fowner` — установка владельца на файл
- `fsetid` — сохранение setuid-бита
- `kill` — отправка сигналов процессам
- `setgid`, `setuid` — смена группы и пользователя
- `setpcap` — управление capability-наборами процессов
- `net_bind_service` — привязка к портам ниже 1024
- `sys_chroot` — операция chroot
- `setfcap` — управление file capabilities
- `net_raw` — raw-сокеты (нужны для некоторых сетевых операций)

### Отсутствуют

- `sys_admin` — монтаж, изменение ядра, namespace-операции
- `sys_ptrace` — отладка чужих процессов
- `net_admin` — управление сетью (изменение интерфейсов, iptables)
- `sys_module` — загрузка модулей ядра
- `sys_rawio` — прямой доступ к устройствам
- `bpf` — использование eBPF

Из практического: мы не можем поднять VPN (`net_admin`), не можем трассировать чужие процессы (`sys_ptrace`), не можем модифицировать сетевые интерфейсы. Но `net_raw` есть, поэтому низкоуровневые сокеты частично доступны.

Seccomp выключен полностью. Это значит, что фильтрация syscall'ов отсутствует — можно вызывать любые syscall, разрешённые ядром. Это не так страшно, как звучит, потому что capabilities всё равно обрезаны.

Параметр `no-new-privs` выставлен в 0, то есть процессы могут повышать привилегии через setuid-биты или file capabilities. Но при отсутствии root-пароля это мало что даёт на практике.

## 6. Process tree

Корневой процесс — `supervisord` с PID 1. Это стандартный менеджер процессов, который запускает и мониторит остальные сервисы. Его команда:

```
/usr/bin/python3 /usr/bin/supervisord -n -c /etc/supervisord.conf
```

Конфиг supervisord'а — 422 строки, описывает более 30 сервисов. Большинство из них спит, потому что включается флагами `CUA_DD_*`.

### Активные сервисы по умолчанию

**Python Tool.** Запускается через `tini` (маленький init, который корректно обрабатывает сигналы и zombie-процессы), затем `uvicorn`:

```
tini -- /opt/pyvenv-python-tool/bin/python -m uvicorn
    --host 0.0.0.0 --port 8080 ...
    jupyter_server.app:app
```

Слушает `0.0.0.0:8080`. Это наш Jupyter + FastAPI. В разных логах PID меняется (190/213/225/155) — это разные инстансы VM.

**Python kernel.** Отдельный процесс, запускаемый самим Jupyter:

```
/opt/pyvenv/bin/python -m ipykernel_launcher -f /tmp/....json
```

Обрати внимание: `-f` указывает на connection file. Имя файла генерируется при старте. В логах PID kernel'а — 243 / 239.

Связь Jupyter ↔ kernel: PID kernel'а всегда имеет PPID, указывающий на Python Tool. В одном из наблюдений: Jupyter = PID 225, kernel = PID 239, PPID(239) = 225.

**Terminal Server.** Отдельный Python-процесс:

```
/opt/terminal-server/pyvenv/bin/python /opt/terminal-server/openai/server.py
```

Слушает `0.0.0.0:1384`.

**log_forwarder × 3.** Три экземпляра, каждый отвечает за свой сервис:

```
log_forwarder --service-name python_tool
log_forwarder --service-name terminal_server
log_forwarder --service-name logrotate_loop
```

PID'ы в районе 151, 173, 176 (в разных логах — 164).

**logrotate.** Единственный процесс, которому присвоен PID около 3. Ротация логов.

### Init-скрипты

Каждый сервис запускается через свой shell-скрипт:

- `/usr/local/init_scripts/python_tool.sh`
- `/usr/local/init_scripts/terminal_server.sh`
- `/usr/local/init_scripts/container_daemon.sh`
- `/usr/local/init_scripts/chrome.sh`
- `/usr/local/init_scripts/sync_share.sh`

И общий враппер: `/usr/local/scripts/oai_service.sh`. Именно он читает `CUA_DD_*` флаги и решает, запускать сервис или нет.

Внутри `container_daemon.sh` и аналогичных есть блок вида:

```bash
if [ "$CUA_DD_INIT_CONTAINER_DAEMON" = "false" ]; then
    exit 42
fi
```

То есть если флаг выставлен в false, скрипт просто завершается с кодом 42. Это стандартный механизм «спящего» сервиса.

## 7. Флаги `CUA_DD_*`

Это внутренние флаги, которые управляют тем, какие сервисы стартуют. Их около 60. Все они читаются из окружения на этапе запуска VM. Изнутри, после старта, их уже нельзя изменить для существующих процессов, но можно использовать при ручном запуске сервисов.

### Режимы

- `CUA_DD_TERMINAL_MODE` — только терминал, без GUI
- `CUA_DD_DESKTOP_MODE` — полный рабочий стол
- `CUA_DD_TERMINAL_LOCAL_LOOPBACK` — слушать только loopback

### Инициализация сервисов

- `CUA_DD_INIT_TERMINAL_SERVER` — стартовать terminal_server
- `CUA_DD_INIT_CONTAINER_DAEMON` — стартовать container_daemon
- `CUA_DD_INIT_HIVE` — стартовать hive
- `CUA_DD_PYTHON_TOOL` — стартовать python_tool
- `CUA_DD_ENABLE_CHROME` — стартовать Chromium
- `CUA_DD_MITMPROXY` — стартовать mitmproxy
- `CUA_DD_ENABLE_VNC` — стартовать VNC
- `CUA_DD_ENABLE_NOTEBOOK_SERVER` — стартовать Jupyter notebook
- `CUA_DD_INIT_XVFB`, `CUA_DD_INIT_X11VNC`, `CUA_DD_INIT_NEKO`, `CUA_DD_INIT_OPENBOX`, `CUA_DD_INIT_XFCE4`, `CUA_DD_INIT_PICOM`, `CUA_DD_INIT_DBUS` — компоненты GUI

### Chrome

- `CUA_DD_CHROME_USER` — от какого пользователя запускать
- `CUA_DD_CHROME_AUTO_RESTART` — авторестарт при падении
- `CUA_DD_CHROME_DEVTOOLS` — включать DevTools
- `CUA_DD_CHROME_KIOSK_PRINTING` — режим печати
- `CUA_DD_CHROME_CAAS_ARGS` — дополнительные аргументы Chrome

### Порты

- `CUA_DD_CONTAINER_DAEMON_PORT=8085`
- `CUA_DD_HIVE_PORT=50939`
- `CUA_DD_NGINX_PORT=8080`
- `CUA_DD_DUO_APP_SERVER_PORT` — вспомогательный сервис
- `CUA_DD_PDF_READER_PORT` — PDF reader

### Пользователи

- `CUA_DD_TERMINAL_SERVER_USER=oai`
- `CUA_DD_PYTHON_TOOL_USER=oai`
- `CUA_DD_CHROME_USER=oai`

То есть все сервисы работают от пользователя `oai`. Плюс существует группа `oai_shared` для общих файлов.

### Поведение

- `CUA_DD_INIT_REMOVE_CONTAINER_SKILLS` — удалять skills при старте
- `CUA_DD_INIT_ARTIFACT_TOOL_V2` — включать Artifact Tool v2
- `CUA_DD_INIT_ARTIFACT_TOOL_V2_RECORD_OPERATIONS` — записывать semantic operations
- `CUA_DD_INIT_ARTIFACT_TOOL_WARN_ABOUT_OVERLAPS_ON_EXPORT` — предупреждать о пересечениях при экспорте
- `CUA_DD_PYTHON_TOOL_WARM_SPREADSHEET_RUNTIME` — прогревать spreadsheet runtime заранее
- `CUA_DD_PYTHON_TOOL_DISABLE_MATPLOTLIB_SUPPORT` — отключить matplotlib
- `CUA_DD_BING_AT_HOME` — Bing At Home режим
- `CUA_DD_PDF_READER_SERVICE` — включать PDF reader
- `CUA_DD_STARTUP_LIBREOFFICE` — стартовать LibreOffice

### Сеть и MITM

- `CUA_DD_MITM_NETWORK_CONFIG` — конфигурация MITM-сети
- `CUA_DD_MITMPROXY` — включение mitmproxy

### Debug / monitoring

- `CUA_DD_SCREENSHOT_MIDDLEWARE` — middleware для скриншотов
- `CUA_DD_SCREENSHOT_DRAW_RED_POSITION_DOT` — рисовать красную точку в позиции курсора
- `CUA_DD_SCREENSHOT_WAIT_FOR_RESOURCES` — ждать загрузки ресурсов перед скриншотом
- `CUA_DD_SCREENSHOT_DELAY` — задержка перед скриншотом
- `CUA_DD_CD_LOG_MIDDLEWARE_FILTER_RE` — regex для фильтрации логов container_daemon
- `CUA_DD_CD_BROWSER_CONNECTION_MODE` — режим подключения браузера
- `CUA_DD_CD_NAVIGATE_TIMEOUT_MS` — таймаут навигации
- `CUA_DD_CD_WAIT_FOR_RESOURCES_*` — ожидание ресурсов
- `CUA_DD_CD_OPERATOR_STEALTH_MODE_TIMEOUT_MS` — stealth-режим Operator
- `CUA_DD_CD_PASTE_PYAUTOGUI_TYPEWRITE` — использовать pyautogui для ввода
- `CUA_DD_NEXUS_HEALTH_CHECK` — healthcheck для nexus
- `CUA_DD_COMPUTER_WAIT_MS` — задержка для Computer-Use Agent

### Eval

- `CUA_DD_SAMPLE_ID_URL` — URL для sample id
- `CUA_DD_EXPERIMENT_NAME` — имя эксперимента
- `EVAL_TASK_ID` — ID задачи (без префикса `CUA_DD`)

## 8. Другие переменные окружения

Помимо `CUA_DD` есть ещё несколько категорий.

### Порты сервисов

- `TERMINAL_SERVER_API_PORT=1384`
- `JUPYTER_SERVER_API_PORT=8080`

### Browser/CUA

- `CDP_PORT=9222`
- `NEKO_PORT=8081`
- `NEKO_PROXY_PORT=8082`
- `NOVNC_PROXY_PORT=6902`

### MITM-инфраструктура (Chaussette)

- `MITM_LOG_LEVEL`
- `MITM_SERP_NO_CACHE` — Bing без кэша
- `MITM_SERP_BING` — перехват Bing
- `MITM_WEBCACHE_HOST` — хост кэша веб-трафика
- `MITM_WEBCACHE_CALLER_ID`
- `MITM_WEBCACHE_CALLER_SECRET`
- `MITM_WEBCACHE_IDENTITY`
- `MITM_WEBCACHE_TIMEOUT_MS`
- `MITM_WEBCACHE_EGRESS`
- `MITM_WEBCACHE_V3_HOST`
- `MITM_OFFLINE_GOOGLE_DOC_HOST`

### ACE

- `ACE_TOOLS_FEATURE_SET=chatgpt-applied`
- `OAI_IS_JUPYTER_KERNEL=true`
- `KERNEL_CALLBACK_ID` — в текущей сессии не задан

### Прочее

- `CUA_DD_VM_BUILD` — полное имя образа
- `OPENAI_CLUSTER` — имя кластера (например `applied-caas34`)
- `NETWORK=caas_packages_only`
- `TARGET=chatgpt-prod-5p4`
- `VM_COMMIT_SHA` — полный коммит
- `HTTP_PROXY`, `HTTPS_PROXY`, `ALL_PROXY` — присутствуют в supervisor env, но не в kernel env

Про последнее отдельно. Переменные proxy-сервера есть у процесса supervisord, но когда код исполняется в Jupyter kernel — они оттуда не видны. Это значит, что startup environment и kernel environment — разные environment boundaries. Мы не видим значений proxy из kernel, а kernel не может их использовать.

## 9. MITM-инфраструктура (Chaussette)

Внутреннее имя MITM-прокси OpenAI Operator — **Chaussette**. Оно встречается в сертификатах и строках конфигов.

Сертификаты:

- `chaussette.crt` — CA OpenAI Operator Proxy
- `nebula-dns.crt` — CA OpenAI LLC, CN=openai.com

Внутренний хост MITM-хаба: `default.mitmproxy.hub.ace-research.openai.org`.

Эта инфраструктура нужна для того, чтобы перехватывать HTTPS-трафик Chrome, когда он активен (в режиме Operator / Computer-Use). Изнутри VM активный MITM мы не наблюдали — он включается вместе с Chrome по флагу.

## 10. Что в образе физически есть

Внутри образа живёт несколько крупных подсистем.

**Artifact Tool.** Bun standalone executable на 126 MB. Живёт в `/opt/pyvenv/lib/python3.13/site-packages/artifact_tool/bin/`. Внутри:

- `artifact_tool_rpc_daemon` — launcher
- `artifact_tool_rpc_daemon-bun` — ELF
- подкаталоги `rpc/`, `generated/`, `patches/`
- `node_modules/@oai/walnut/` — WASM .NET OpenXML

Подробно → `07-artifact-workbook.md`.

**Granola.** Внутренний движок офисных документов. npm-пакет `@oai/granola`. Внутри: `dist/models/spreadsheet/google-sheets/`, `dist/models/workbook/`, `plugins/google-sheets`. Plugin installer `installGoogleSheetsPlugin` вырезан при сборке (`bun build --compile` не трассирует динамические импорты), поэтому некоторые методы падают.

**Walnut.** .NET → WASM OpenXML движок. Работает внутри Artifact Tool.

**Skills.** Навыки ChatGPT в `/home/oai/skills/`:

- `docx` — 40+ скриптов
- `pdfs` — 30+ скриптов
- `slides` — artifact_tool + шаблоны
- `spreadsheets` — artifact_tool

Каждый skill имеет свой `SKILL.md`.

**Chrome.** Профиль в `/home/oai/.chromium`. Внутри: `Local State`, `Default/Preferences`, без History / Cookies / Login Data. Имя профиля — «ChatGPT Agent». Расширение `kcdongibgcplmaagnmgpjhpjgmmaaaaa`. Отключены Ctrl+Shift+W, Ctrl+S, Ctrl+N.

**Chrome Policies.** В `/etc/chromium/policies/managed/`. Наиболее жёсткий файл — `001_base_url_blocklist.json` с `URLBlocklist: ["*"]`. Подробно → `06-browser-stack.md`.

**redirect.html.** Sentinel-файл в `/home/oai/redirect.html` и `/etc/skel/redirect.html`. Содержит JS: читает target из query и делает `location.replace(target)`. Используется для двухступенчатой навигации.

## 11. Почему всё это важно понимать

Каждая из этих деталей имеет прямое operational-применение.

Знание capability-набора говорит, что мы не можем:

- загрузить модуль ядра
- использовать eBPF
- трассировать чужие процессы
- изменить сетевые интерфейсы

Остальное:

- Знание про Kata VM объясняет, почему многие трюки контейнерных песочниц не работают: мы не разделяем ядро с хостом.
- Знание про `CUA_DD_*` флаги объясняет, почему 30+ сервисов «спят» и как их разбудить вручную.
- Знание про Nebula объясняет, кто на самом деле решает вопрос про интернет — и почему изнутри VM это не изменить.
- Знание про environment boundaries объясняет, почему proxy-переменные есть в supervisord, но не видны в Jupyter kernel.

## 12. Что остаётся не до конца выясненным

- **Точные PID всех сервисов** — они меняются между инстансами VM. В логах встречались разные наборы (190/213/243/201 vs 225/239 vs 155/179). Это не противоречие, это просто разные запуски.
- **Роль `sync_share.sh`** — упоминается в списке init-скриптов, использует `REMOTE_SHARE_HOST`, но что именно синхронизирует — не восстановлено.
- **Назначение `duo_app_server` и `pdf_reader`** — упоминаются в флагах портов, конкретное поведение не изучалось.
- **Динамические localhost-порты** (`43929, 39473, 44849, 36603, 41903, 42553`) — не идентифицированы.
- **Точные PID log_forwarder** — встречались как 151/173/176 и как 164. Возможно, разные запуски.

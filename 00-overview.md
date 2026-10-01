# 00 · Overview — ChatGPT Linux Sandbox

**Статус документа:** рабочий, собран из runtime-наблюдений и статического анализа.
**Область:** изолированная микро-VM ChatGPT Applied CaaS (Code Interpreter / Advanced Data Analysis).
**Дата фиксации:** сентябрь 2026.

## Маркеры статуса

| Маркер | Значение |
|---|---|
| ✅ runtime | проверено в живой VM |
| 🟢 static | найдено в бинаре/исходниках |
| 🟡 hypothesis | требует верификации |
| ⭕ disabled | предусмотрено, но выключено по умолчанию |
| ❌ disproved | опровергнуто экспериментом |

Дальше по тексту маркеры ставятся у каждого нетривиального утверждения. Если маркера нет — это следствие из уже помеченных фактов.

## TL;DR

Изолированная микро-VM (Kata Containers в Azure), в которой модель выполняет код и команды от имени пользователя. Внутри — Python-инструмент (Jupyter), terminal-сервер (shell), Artifact Tool (офисные документы через RPC), container_daemon (Rust-сервис с файловым API и HTTP-прокси), hive (WebSocket-шина), плюс спящий браузерный стек (Chromium + CDP + Xvfb).

Это **не** «сервер ChatGPT». Это «руки» модели — отдельная микро-ВМ, живущая ровно на время задачи, полностью изолированная от интернета.

Главный ответ по сети: **интернета нет, обход невозможен**, потому что блокировка двухслойная — Chrome Managed Policy внутри VM + Azure NSG/sidecar egress снаружи. Решение о включении интернета принимает оркестратор Nebula при создании VM, не сама машина и не модель.

## 1. Полная карта портов и сервисов

| Порт | Сервис | Тип | Default | Триггер включения |
|---|---|---|---|---|
| `8080` | Python Tool (Jupyter) | Python / FastAPI / Uvicorn | ✅ ACTIVE | — |
| `1384` | Terminal Server | Python / FastAPI + PTY | ✅ ACTIVE | — |
| `8085` | container_daemon | Rust / Axum / Tokio | ⭕ OFF | `CUA_DD_INIT_CONTAINER_DAEMON=true` |
| `50939` | hive | Rust / WebSocket | ⭕ OFF | `CUA_DD_INIT_HIVE=true` |
| `8888` | Jupyter notebook | Python | ⭕ OFF | `CUA_DD_ENABLE_NOTEBOOK_SERVER=true` |
| `9222` | Chromium CDP | Chrome DevTools Protocol | ⭕ OFF | `CUA_DD_ENABLE_CHROME=true` |
| `4444` | mitmproxy | MITM-прокси | ⭕ OFF | `CUA_DD_MITMPROXY=true` |
| `5901` | x11vnc | VNC | ⭕ OFF | `CUA_DD_ENABLE_VNC=true` |
| `6902` | novnc | VNC-HTTP | ⭕ OFF | `CUA_DD_ENABLE_VNC=true` |
| `8081` | neko | browser/UI инфраструктура | ⭕ OFF | `CUA_DD_INIT_NEKO=true` |
| `8082` | neko proxy | то же | ⭕ OFF | — |
| `9999` | offline_proxy | forward HTTP proxy | ⭕ OFF | `START_PROXY=true PROXY_PORT=9999` |
| unix sock | artifact_tool_rpc_daemon | JSON-RPC | lazy | при `import artifact_tool` |

**Ingress снаружи** (проброшено за пределы VM): только `:8080` и `:1384`. ✅ *(из runtime-наблюдений; соответствует отсутствию маршрутов к другим портам)*

Динамические localhost-порты (наблюдались, не идентифицированы): `43929, 39473, 44849, 36603, 41903, 42553`. 🟡

Из ранних логов (не перепроверено на текущей VM): `40231` (предположительно shell channel Jupyter), `59331` (предположительно iopub). 🟡

## 2. Общая архитектура

```
┌─────────────────────────────────────────────────────────┐
│ ChatGPT Platform (вне VM)                               │
│  - модель (GPU)                                         │
│  - Nebula orchestrator                                  │
│  - sidecar egress proxy                                 │
│  - host-side consumer                                   │
└──────────────────────┬──────────────────────────────────┘
                       │  unknown host-side transport
                       │  (ingress → :8080, :1384)
                       ▼
┌─────────────────────────────────────────────────────────┐
│ Kata Containers микро-VM                                │
│                                                         │
│  supervisord (PID 1)                                    │
│    ├── python_tool        :8080  (Jupyter + kernel)     │
│    ├── terminal_server    :1384  (shell через PTY)      │
│    ├── log_forwarder × 3     (stdout/stderr вверх)      │
│    └── [спящие]                                         │
│         ├── container_daemon :8085                      │
│         ├── hive             :50939                     │
│         ├── Chromium + CDP   :9222                      │
│         ├── mitmproxy        :4444                      │
│         └── VNC/Neko/…                                  │
│                                                         │
│  Artifact Tool (Bun RPC daemon, lazy)                   │
│    - unix socket /tmp/artifact_tool_rpc_*.sock          │
│    - 194 класса (Workbook, Slide, Range, …)             │
│    - Yjs-совместимый CRDT                               │
│                                                         │
│  Сеть:                                                  │
│    eth0: 172.26.39.x/22, gw 172.26.36.1                 │
│    NETWORK=caas_packages_only                           │
│    egress: закрыт (RST + DNS fail)                      │
│    ingress: только :8080, :1384                         │
└─────────────────────────────────────────────────────────┘
```

Подробности по инфраструктуре → **`01-infrastructure.md`**.

## 3. Шесть «зон» — карта документации

| Файл | Зона | Что покрывает |
|---|---|---|
| `01-infrastructure.md` | Среда | Kata/Azure/Nebula, ресурсы, capabilities, process tree, `CUA_DD_*` флаги |
| `02-container-daemon.md` | `:8085` | Управление UI/FS/exec/сетью, offline_proxy, browser subsystem |
| `03-jupyter-python-tool.md` | `:8080` | Python execution, command/event, callbacks, long-poll |
| `04-terminal-server.md` | `:1384` | Shell через PTY |
| `05-filesystem.md` | Files | `/file`, `/files`, path guard, `/mnt/data` publish |
| `06-browser-stack.md` | Browser | Xvfb + Chromium + CDP + Chrome policies |
| `07-artifact-workbook.md` | Artifacts | RPC, CRDT, Google Sheets/Slides bridges |
| `08-network-boundary.md` | Egress | Двухслойная блокировка, ACE `allow_internet` |
| `09-hive.md` | Hive | WebSocket broadcast, протокол |
| `99-reference.md` | Справочники | Все env, endpoints, идентификаторы, глоссарий |

## 4. Что работает / что выключено

### ✅ Активно по умолчанию

- Python execution через Jupyter (`:8080`)
- Shell через terminal_server (`:1384`)
- Логирование (log_forwarder ×3)
- Файловый publish через `/mnt/data/` → платформенный storage → UI

### ⭕ Выключено, но можно включить флагом `CUA_DD_*` / `START_PROXY`

- container_daemon `:8085` (управление UI/FS/exec/offline_proxy)
- hive `:50939`
- Chromium + CDP `:9222`
- mitmproxy `:4444`
- VNC/Neko
- Jupyter notebook server `:8888`
- nginx `:8080` (в других сборках)

### 🟢 Static (в бинаре/коде, runtime не проверялось)

- `get_ip` при startup container_daemon → `api.ipify.org` → `set_ip`
- `offline_proxy` pipeline
- Chrome policies (`URLBlocklist: ["*"]`)
- ACE types (`EnsureUserMachineRequest` и др.)

### 🟡 Требует верификации

- Роль `40231` / `59331` как Jupyter shell/iopub channels
- Формулировка «Hive — мост к Chrome Extensions» (в логах №8/№9 есть, XREF не подтверждён)
- Точное соответствие PID / snapshot-инстансов разных VM (мы видели разные PID'ы в разных логах)

### ❌ Опровергнуто

- «container_daemon имеет эксклюзивный сетевой путь наружу» — опровергнуто runtime: daemon упирается в тот же DNS-fail (`getaddrinfo → gai rc=-3`)
- «`127.0.0.1:8765` в UI может дотянуться до runtime» — negative result, `TypeError: Failed to fetch`
- «Rust/Tokio на :8080» в этой сборке — здесь чистый Python; возможно в других сборках (полиморфизм)

## 5. Ключевые концепции

- **ACE** (Applied Computing Engine) — внутренний RPC-протокол между моделью и User Machine. Типы есть в установленном runtime, транспорт — вне VM.
- **Nebula** — платформа управления VM. Решает `allow_internet` при создании.
- **CaaS** (Containers as a Service) — модель развёртывания.
- **CUA** (Computer-Use Agent), **CUA DD** (CUA Data Daemon) — префикс флагов.
- **Kata Containers** — микро-VM на базе лёгкой виртуализации.
- **Chaussette** — внутреннее имя MITM-прокси OpenAI Operator.
- **Granola** — внутренний движок офисных документов.
- **Walnut** — .NET → WASM OpenXML движок внутри Artifact Tool.
- **Hive** — локальная WebSocket broadcast-шина.
- **Offline Proxy** — forward HTTP proxy внутри container_daemon для локальных upstream.
- **Artifact Tool** — RPC-движок для Workbook/Slides/Docs.
- **Skills** — навыки ChatGPT (`/home/oai/skills/*`).

Полный глоссарий → **`99-reference.md`**.

## 6. Что можно делать (краткая сводка)

Полные рецепты — в файлах по зонам. Здесь только верхний уровень.

### Через `:8080` (Python Tool)

- выполнять произвольный Python в persistent kernel
- получать IOPub events (stream, display_data, status, error)
- регистрировать 4 allowlisted callback'а
- **rich output → UI**: HTML, `<button>`, `<input>`, PNG ✅

### Через `:1384` (Terminal Server)

- запускать shell-команды через PTY
- читать stdout/stderr, писать в stdin, убивать процесс

### Через Artifact Tool RPC

- создавать/редактировать Workbook, Slides, Docs
- писать values, formulas, formatting, merge
- сериализовать состояние (Proto + CRDT)
- восстанавливать Workbook из snapshot
- инкрементальная CRDT-репликация A → B
- конвертировать Artifact ops в Google Sheets/Slides API requests (с подменой `baseUrl`)

### Через `:8085` (container_daemon, если поднять)

- файловый API в `/home/oai/**` (read/write)
- UI control (click/type/scroll) — если запущен Chromium
- screenshot через работающий Xvfb
- offline_proxy — только для локальных upstream
- `/offline_sites` — таблица Host → forward address

### Machine → UI

- HTML-блоки, кнопки, inputs из Python → отображаются в UI ✅
- PNG → attachment ✅
- файлы в `/mnt/data/` → платформа подхватывает и показывает в UI ✅
- UI → runtime через JS `fetch('127.0.0.1:...')` — ❌ не работает

## 7. Чего делать нельзя

- **Интернета нет** — двухслойная блокировка, обход невозможен изнутри VM.
- **Файлы вне `/home/oai/**`** — path guard, `/tmp`, `/etc`, `/root` → `400 blocked path`.
- **Ingress снаружи на `:8085`/`:50939`/`:9222`** — не проброшены.
- **Персистентность** — всё в overlay, исчезает при пересоздании VM. Единственный способ сохранить — сериализация Workbook (Proto + CRDT) до конца сессии.
- **Capabilities**: нет `sys_admin`, `sys_ptrace`, `net_admin`, `sys_module`, `bpf`, `sys_rawio`.
- **Raw sockets**: нет (ping = `Operation not permitted`).

Подробный разбор egress → **`08-network-boundary.md`**.

## 8. Главный открытый вопрос

Единственный крупный незакрытый участок — **точный host-side consumer `:8080` извне VM**.

Известно:

- ingress снаружи только `:8080` и `:1384`
- `:8080` — единственный кандидат на роль «пути наружу» для machine → UI
- machine → UI реально работает (HTML, PNG, файлы через `/mnt/data/`)
- последний шаг (publish в UI) выполняет **платформа**, не sandbox

Неизвестно:

- каким именно транспортом host-side consumer обращается к `:8080/pull_message`
- как платформа отличает «обычный» output от attachment/file

Это вынесено как открытый вопрос в **`03-jupyter-python-tool.md`** и **`08-network-boundary.md`**.

## 9. Минимальный smoke-test

После запуска VM:

```bash
# 1. Jupyter / Python Tool
curl -i http://127.0.0.1:8080/status

# 2. Shell
curl -i http://127.0.0.1:1384/healthcheck

# 3. Проверить активные сервисы
ss -ltnp
ps auxf

# 4. Env PID 1
tr '\0' '\n' < /proc/1/environ | sort

# 5. Проверить egress (ожидаем RST)
python3 -c "
import socket
for h,p in [('1.1.1.1',443),('8.8.8.8',53)]:
    try:
        s=socket.create_connection((h,p),2); print('OPEN',h,p); s.close()
    except Exception as e:
        print('FAIL',h,p,type(e).__name__)
"
```

Развёрнутые smoke-tests для каждой зоны → `99-reference.md` (раздел «Минимальные проверки»).

## 10. Как читать эту документацию

- Хочешь быстро понять, где что живёт → раздел 1 (карта портов) + раздел 3 (карта файлов).
- Работаешь с Python/kernel → `03-jupyter-python-tool.md`.
- Работаешь с офисными документами → `07-artifact-workbook.md`.
- Разбираешься с сетевыми границами → `08-network-boundary.md`.
- Ищешь конкретный env / идентификатор / endpoint → `99-reference.md`.
- Разбираешься с Chromium/UI → `06-browser-stack.md`.

Каждый файл самодостаточен, но при необходимости ссылается на соседей.

# ChatGPT Linux Sandbox — Reverse Engineering Documentation

Документация по внутренней архитектуре изолированной микро-VM ChatGPT Applied CaaS (Code Interpreter / Advanced Data Analysis). Собрана из runtime-наблюдений, статического анализа бинарников и экспериментов внутри песочницы.

**Область:** Kata Containers в Azure, `:8080` (Jupyter), `:1384` (Terminal), `:8085` (container_daemon), `:50939` (hive), Artifact Tool RPC, CRDT, Chrome stack, network boundary.

**Ключевой вывод:** Интернета нет. Обход невозможен. Двухслойная блокировка — Chrome Managed Policy внутри VM + Azure NSG/sidecar egress снаружи.

**Маркеры статуса во всех файлах:**

| Маркер | Значение |
|---|---|
| ✅ runtime | проверено в живой VM |
| 🟢 static | найдено в бинаре/исходниках |
| 🟡 hypothesis | требует верификации |
| ⭕ disabled | предусмотрено, но выключено |
| ❌ disproved | опровергнуто экспериментом |

## Файлы

```
00-overview.md
01-infrastructure.md
02-container-daemon.md
03-jupyter-python-tool.md
04-terminal-server.md
05-filesystem.md
06-browser-stack.md
07-artifact-workbook.md
08-network-boundary.md
09-hive.md
10-chrome-policies.md
11-ethics-and-limits.md
12-cua-dd-flags.md
13-method.md
99-reference.md
```
## Содержание

| № | Файл | Описание |
|---|---|---|
| 00 | [**Overview**](00-overview.md) | Точка входа. Карта портов, сервисов и компонентов. TL;DR по всей системе. Ответ на главный вопрос про интернет. Глоссарий и связи между зонами. |
| 01 | [**Infrastructure**](01-infrastructure.md) | Где физически живёт VM. Kata Containers, Azure VNet, Nebula. Ресурсы (CPU/RAM/disk), capabilities, seccomp, process tree, полный список `CUA_DD_*` флагов. |
| 02 | [**container_daemon** (`:8085`)](02-container-daemon.md) | Rust/Axum/Tokio сервис. Управление UI, файлами, exec, browser subsystem, offline_proxy. Полная карта routes. `get_ip` startup chain. Отдельная подсистема offline_proxy. |
| 03 | [**Jupyter / Python Tool** (`:8080`)](03-jupyter-python-tool.md) | Основной execution layer. Command path (`/execute`) vs Event path (`/pull_message`). Callback subsystem, allowlist, deque(maxlen=1000). Long-poll доказательство. Machine → UI. |
| 04 | [**Terminal Server** (`:1384`)](04-terminal-server.md) | Shell execution через PTY. `/open`, `/read/{pid}`, `/write/{pid}`, `/kill/{pid}`. Отличия от Jupyter. Работа с интерактивными программами. |
| 05 | [**Filesystem**](05-filesystem.md) | Три пути работы с файлами: Python напрямую, container_daemon API, `/mnt/data/`. Path guard `/home/oai/**`. Disasm-подтверждение guard'а. Публикация файлов в UI. |
| 06 | [**Browser Stack**](06-browser-stack.md) | Xvfb `:0` + Chromium + CDP `:9222` + chromiumoxide. `/browser/*` endpoints, GUI actions, `/screenshot`. redirect.html sentinel. Chrome profile "ChatGPT Agent". |
| 07 | [**Artifact Tool / Workbook**](07-artifact-workbook.md) | RPC-движок для офисных документов. Bun standalone 126 MB. 7 базовых методов. Recorder pipeline. CRDT (Proto + Y.Doc). 194 класса. Google Sheets/Slides bridges. |
| 08 | [**Network Boundary**](08-network-boundary.md) | Двухслойная блокировка egress. Chrome Managed Policy + Azure NSG. DNS fail, RST, `getaddrinfo`. Ingress только `:8080`/`:1384`. ACE `EnsureUserMachineRequest.allow_internet`. |
| 09 | [**Hive** (`:50939`)](09-hive.md) | WebSocket broadcast-шина. Отдельный ELF-бинарник. Протокол IN/OUT. `kind` — просто строка, не discriminator. Sender исключается из broadcast. UUID injection. |
| 10 | [**Chrome Policies**](10-chrome-policies.md) | Полная документация Enterprise Managed Policies. `URLBlocklist: ["*"]`, `ERR_BLOCKED_BY_ADMINISTRATOR`. Коды значений. Матрица файлов `000`/`001`/`010`/`020`/`040`/`100`. |
| 11 | [**Ethics and Limits**](11-ethics-and-limits.md) | Этические и юридические границы. Что законно, что серая зона, что нарушает ToS. Почему не стоит обходить egress. Ответственное раскрытие. Рекомендации для разных ролей. |
| 12 | [**CUA_DD Flags**](12-cua-dd-flags.md) | Полный справочник `CUA_DD_*` флагов (~60 штук). Категории: режимы, инициализация сервисов, Chrome, порты, пользователи, behavior, network, screenshot, CD, eval. |
| 13 | [**Method**](13-method.md) | Методика исследования. Пять уровней доказательства. Как исследовать новый endpoint / сервис / бинарник / протокол. Чек-лист. Формат записи фактов. Работа с противоречиями. |
| 99 | [**Reference**](99-reference.md) | Справочник. Сводные таблицы портов и endpoints. Все env-переменные. Внутренние идентификаторы. Типы и модели. Публичное / непубличное. Глоссарий. Debugging matrix. Smoke-tests. |

## Быстрая навигация

**По сервисам:**

- Python execution → [03](03-jupyter-python-tool.md)
- Shell → [04](04-terminal-server.md)
- Управление VM (UI, файлы, exec, browser) → [02](02-container-daemon.md)
- WebSocket broadcast → [09](09-hive.md)
- Офисные документы / CRDT → [07](07-artifact-workbook.md)

**По вопросам:**

- «Есть ли интернет?» → [08](08-network-boundary.md)
- «Как устроена VM?» → [01](01-infrastructure.md)
- «Что за флаги `CUA_DD_*`?» → [12](12-cua-dd-flags.md)
- «Как исследовать что-то новое?» → [13](13-method.md)
- «Где найти конкретный факт?» → [99](99-reference.md)

**Ограничения:**

- Двухслойная блокировка egress → [08](08-network-boundary.md) + [10](10-chrome-policies.md)
- Этические границы → [11](11-ethics-and-limits.md)

## Как читать

- **Начало** — начать с [00-overview](00-overview.md), потом по зонам.
- **По конкретной задаче** — сразу в нужный файл через навигацию выше.
- **Ищешь справочную информацию** — [99-reference](99-reference.md).
- **Хочешь продолжить исследование** — [13-method](13-method.md).

Каждый файл самодостаточен, но ссылается на соседей по необходимости.

## Статус

Документация собрана из 14 источников (runtime-логи, дизассемблер, анализ бинарников, эксперименты с API). Все утверждения промаркированы уровнями доказательства.

**Что покрыто:**

- ✅ Архитектура VM (Kata, Azure, Nebula)
- ✅ Python execution (`:8080`)
- ✅ Shell execution (`:1384`)
- ✅ container_daemon API (`:8085`)
- ✅ Artifact Tool RPC + CRDT + Google bridges
- ✅ Chrome stack + Xvfb + CDP
- ✅ Network boundary (egress закрыт)
- ✅ Hive protocol
- ✅ Chrome policies
- ✅ Все флаги `CUA_DD_*`

**Что остаётся открытым:**

- Точный host-side consumer `/pull_message`
- Реальные клиенты Hive
- Полные схемы некоторых GUI endpoints container_daemon
- Точная protobuf schema и Yjs-структура Artifact Tool

Документация составлена в исследовательских целях.

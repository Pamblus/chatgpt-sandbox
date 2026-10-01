# 05 · Filesystem

**Зона:** Файловые операции
**Пути доступа:** Python (напрямую) · container_daemon `/file` + `/files` · `/mnt/data/` (publish)
**Статус:** ✅ активные механизмы
**Guard:** `/home/oai/**`

## Содержание

1. Общая картина
2. Три пути доступа к файлам
3. `GET /file` — чтение через container_daemon
4. `POST /files` — запись через container_daemon
5. Path guard — только `/home/oai/**`
6. Подтверждение из бинарника
7. `/files/:path` и `/list` — что не работает
8. `/mnt/data` — публикация файлов в UI
9. Путь через Python напрямую
10. Путь через shell
11. Полный пример: PNG через PIL
12. Полный пример: TXT + проверка SHA256
13. Полный пример: файл через container_daemon
14. Директории: где что лежит
15. Что не работает
16. Health check
17. Связи
18. Открытые вопросы

## 1. Общая картина

Файловая система в песочнице — это не единый механизм, а несколько независимых путей для работы с файлами. Каждый путь имеет свои правила, свои ограничения и своё назначение.

Три основных пути:

1. **Python напрямую** — через Jupyter kernel (`open`, `Path.write_bytes`, `Path.read_text`, PIL и т.д.). Работает с любой точкой ФС, к которой у процесса есть доступ.
2. **container_daemon HTTP API** — `/file` (read) и `/files` (write). Строгий path guard на `/home/oai/**`.
3. **`/mnt/data/`** — специальная директория, файлы из которой платформа автоматически подхватывает и публикует в UI как attachments.

Плюс shell через `:1384` даёт доступ напрямую к ФС, но без специальной обёртки для публикации.

Эти пути не эквивалентны и не взаимозаменяемы. Важно понимать, какой из них когда использовать.

## 2. Три пути доступа к файлам

| Путь | Способ | Guard | Публикация в UI |
|---|---|---|---|
| Python через kernel | `open()`, `Path.write_*` | нет (полный доступ процессов `oai`) | только через `/mnt/data/` |
| container_daemon `/file`, `/files` | HTTP на `:8085` | `/home/oai/**` | нет |
| shell через `:1384` | любая команда | нет | только через `/mnt/data/` |
| `/mnt/data/` | любой из предыдущих | нет | ✅ автоматически |

Выбор пути определяется задачей:

- Хочешь, чтобы файл попал в UI → создавай в `/mnt/data/`
- Хочешь работать в `/home/oai/` через API → используй `/file` и `/files`
- Хочешь читать системные пути → используй Python или shell напрямую
- Хочешь чистый обмен через HTTP → используй `/files` (base64)

## 3. `GET /file` — чтение через container_daemon

### 3.1 Назначение

Прочитать файл через HTTP API. Возвращает содержимое файла в теле ответа.

### 3.2 Схема

```bash
curl --get --data-urlencode "path=/home/oai/share/test.txt" \
    http://127.0.0.1:8085/file
```

Query-параметр `path` — абсолютный путь к файлу.

### 3.3 Успешный ответ

```
HTTP 200
Content-Type: application/octet-stream (или text/plain, зависит от файла)
<содержимое файла>
```

Содержимое отдаётся как есть, без кодирования. Для бинарных файлов — raw bytes.

### 3.4 Ошибки

| Ситуация | Ответ |
|---|---|
| Путь вне `/home/oai/**` | `400 blocked path=/...` |
| Файл не существует | `404` (или `500`, точный контракт не зафиксирован) |
| Path — директория | `500 Is a directory (os error 21)` |
| Невалидный query | `400` |

Пример директории:

```bash
curl --get --data-urlencode "path=/home/oai/share" \
    http://127.0.0.1:8085/file
# → 500 Is a directory (os error 21)
```

### 3.5 Примеры полезных файлов

Прочитать Chrome preferences:

```bash
curl --get --data-urlencode "path=/home/oai/.chromium/Default/Preferences" \
    http://127.0.0.1:8085/file
```

Прочитать `SKILL.md`:

```bash
curl --get --data-urlencode "path=/home/oai/skills/docx/SKILL.md" \
    http://127.0.0.1:8085/file
```

## 4. `POST /files` — запись через container_daemon

### 4.1 Назначение

Записать один или несколько файлов через HTTP API. Данные передаются в base64.

### 4.2 Схема

```json
{
  "files": {
    "/home/oai/share/a.txt": "<base64-encoded>",
    "/home/oai/b.txt": "<base64-encoded>"
  }
}
```

Внутренний тип:

```rust
FileUploadBody {
    files: HashMap<String, String>  // path → base64
}
```

Каждый ключ — абсолютный путь. Каждое значение — base64-строка с содержимым файла.

### 4.3 Pipeline

```
POST /files
    │
    ▼
Json<FileUploadBody>
    │
    ▼
FileUploadBody.files (Map<String, String>)
    │
    ▼
post_handler
    │
    ▼
post_files_impl
    │
    ▼
для каждого (path, contents):
    get_file_path(path)      ← path guard
    base64::decode(contents) ← декодирование
    tokio::fs::write(...)    ← запись
```

### 4.4 Пример

```bash
# Закодировать
B64=$(echo -n "hello world" | base64)
# → aGVsbG8gd29ybGQ=

# Записать
curl -X POST http://127.0.0.1:8085/files \
    -H 'Content-Type: application/json' \
    -d "{\"files\":{\"/home/oai/share/uploaded.txt\":\"$B64\"}}"
```

Ожидаемый ответ: `HTTP 200`.

### 4.5 Проверка

```bash
ls -la /home/oai/share/uploaded.txt
cat /home/oai/share/uploaded.txt
# → hello world
```

### 4.6 Права создаваемого файла

По умолчанию:

```
-rw-r--r-- 1 root oai_shared <size> /home/oai/share/uploaded.txt
```

Владелец — `root` (от него работает daemon), группа — `oai_shared`. Права `644` — чтение всем, запись владельцу.

### 4.7 Что не принимается

- Невалидный base64 (`"x"`, `"abc"`) → ошибка декодирования
- Путь вне `/home/oai/**` → `400 blocked path`
- Пустой `files` → вероятно `200` без эффекта (не проверялось)

### 4.8 Атомарность

Если в одном запросе несколько файлов и один из них не проходит guard — что происходит с остальными, не зафиксировано. Скорее всего, обработка идёт последовательно, и часть файлов уже записана до ошибки. Для гарантии атомарности писать по одному файлу.

## 5. Path guard — только `/home/oai/**`

### 5.1 Правило

container_daemon разрешает работу с файлами только внутри `/home/oai/**`. Всё остальное блокируется.

### 5.2 Экспериментальная таблица

| Путь | Результат |
|---|---|
| `/home/oai/share/test1.txt` | ✅ 200, файл создан |
| `/home/oai/test2.txt` | ✅ 200, файл создан |
| `/tmp/test3.txt` | ❌ `400 blocked path=/tmp/test3.txt` |
| `/etc/test4.txt` | ❌ `400 blocked path=/etc/test4.txt` |
| `/root/test5.txt` | ❌ `400 blocked path=/root/test5.txt` |

Важно: разрешена вся директория `/home/oai/**`, не только `/home/oai/share/`. Это уточнение к ранним представлениям.

### 5.3 Сообщение в логе

При блокировке в логе `/tmp/cd.log` (или `/tmp/container_daemon.log`) появляется:

```
get_file_path path=/tmp/test3.txt
post_handler error=blocked path=/tmp/test3.txt
```

Это подтверждает, что блокировка происходит на уровне функции `get_file_path`, до попытки записи.

### 5.4 Важно про shell

Path guard действует только для container_daemon HTTP API. Через shell (`:1384`) или Python (kernel) доступ к `/tmp`, `/etc`, `/root` — обычный. Guard — это ограничение сервиса, а не системы.

## 6. Подтверждение из бинарника

### 6.1 Disasm `get_file_path`

В конце функции `get_file_path` видно сравнение результата с байтовой последовательностью:

```asm
movabs $0x616f2f656d6f682f, %rcx
xor    (%rax), %rcx
...
```

В little-endian 8 байт `0x616f2f656d6f682f` — это ASCII:

```
68 2f 6f 6d 65 2f 6f 61  →  /home/oa
```

Следующие байты (за пределами `movabs`) дают `i`. То есть итоговая строка — `/home/oai`.

Функция сравнивает префикс пути с `/home/oai`. Если префикс совпадает — возвращает нормализованный путь. Иначе — формирует ошибку `blocked path`.

### 6.2 Symbols

В бинарнике подтверждены символы:

```
container_daemon::server::handlers::files::get_file_path
tokio::fs::write::<&str, Vec<u8>>
tokio::fs::write::<String, String>
tokio::fs::file::File::create::<String>
```

Это значит, что:

- `get_file_path` — реальная функция проверки/нормализации
- `tokio::fs::write` — реальный асинхронный write
- `File::create` — создание файлов

`post_files_impl` не сохранился как отдельный `t` symbol — остался только его async closure:

```
drop_in_place<
  container_daemon::server::handlers::files::post_files_impl::{{closure}}
>
```

Это типично для async Rust: тело операции связано с сгенерированным future/closure.

### 6.3 Что это подтверждает

Статика + runtime вместе дают полную картину:

- Base64 декодируется
- Path проверяется на префикс `/home/oai`
- Запись идёт через `tokio::fs::write`
- Ошибки формируются на этапе проверки пути

## 7. `/files/:path` и `/list` — что не работает

### 7.1 `/files/:path`

В ранних логах этот endpoint упоминался как существующий. В текущей сборке — `404`.

```
GET /files/:path → 404
```

То есть REST-дерева для файловых путей нет. Есть только:

- `/file?path=...` — для чтения
- `/files` — для записи
- `/list` — для листинга (см. ниже)

### 7.2 `/files/:path/list`

```
GET /files/:path/list → 404
```

### 7.3 `/files/:path/exec`

```
GET /files/:path/exec → 404
```

### 7.4 `/list`

Endpoint существует, но полный контракт не восстановлен. Вероятно, возвращает список файлов в директории, но точный request/response не зафиксирован.

Пример проверки:

```bash
curl -i http://127.0.0.1:8085/list
```

### 7.5 Что это значит

Если нужен листинг содержимого директории — проще сделать через shell (`:1384`) или Python (kernel). Через daemon API полноценного листинга нет.

## 8. `/mnt/data` — публикация файлов в UI

Это отдельный механизм, не имеющий отношения к container_daemon. Он работает через платформу.

### 8.1 Что это

`/mnt/data/` — специальная директория внутри VM. Файлы, созданные в ней, автоматически подхватываются платформой и публикуются в UI как attachments.

### 8.2 Полная цепочка

```
Linux sandbox
    │
    │ 1. код создаёт файл в /mnt/data/
    ▼
/mnt/data/MACHINE_TEST.png
    │
    │ 2. ChatGPT runtime / платформа обнаруживает созданный артефакт
    ▼
platform file storage (Library)
    │
    │ 3. attachment / file reference
    ▼
ChatGPT UI
```

### 8.3 Ключевая деталь

Последний шаг делает не sandbox, а платформа. Наличие ссылки `sandbox:/mnt/data/...` само по себе не означает, что машина «сама нажала кнопку отправить». Платформа сама обнаруживает созданные файлы и превращает их в attachment.

Это значит:

- Никаких HTTP-запросов наружу из sandbox не требуется
- Никакого «отправить файл» API нет
- Триггер на стороне платформы, а не на стороне машины

### 8.4 Что публикуется

| Тип | Статус |
|---|---|
| PNG | ✅ доходит как изображение |
| TXT | ✅ (проверяется через тот же механизм) |
| JSON | 🟡 |
| PDF | 🟡 |
| CSV | 🟡 |

Поведение зависит от platform renderer: что-то отображается как inline, что-то как attachment.

### 8.5 Ограничение

Файлы в `/mnt/data/` не сохраняются между сессиями. Опубликованный файл попадает в Library, но сам файл в VM исчезает при пересоздании.

## 9. Путь через Python напрямую

### 9.1 Что доступно

Python в kernel запущен от пользователя `oai`. Ему доступна вся файловая система с обычными правами POSIX:

- `/home/oai/**` — полный доступ
- `/tmp/**` — полный доступ
- `/var/**` — обычно доступ на чтение
- `/etc/**` — доступ на чтение большинства файлов
- `/root/**` — скорее всего, нет доступа (права 700)

### 9.2 Ограничения

- Нет `sys_admin` capability — нельзя монтировать
- Нет `sys_ptrace` — нельзя debug
- Нет доступа к некоторым специальным устройствам

Но для обычной работы с файлами ограничений нет.

### 9.3 Примеры

```python
from pathlib import Path

# Запись
Path("/tmp/test.txt").write_text("hello")

# Чтение
content = Path("/etc/hostname").read_text()

# Бинарная работа
with open("/tmp/data.bin", "wb") as f:
    f.write(b"\x00\x01\x02")

# Метаданные
stat = Path("/home/oai").stat()
print(stat.st_size, stat.st_mode)
```

### 9.4 Публикация в UI

Чтобы файл, созданный через Python, попал в UI — писать в `/mnt/data/`:

```python
from pathlib import Path
Path("/mnt/data/result.txt").write_text("output")
```

## 10. Путь через shell

### 10.1 Что доступно

Shell через `:1384` работает от пользователя `oai`. Доступны те же файловые пути, что и Python.

Path guard из container_daemon не действует — можно работать с `/tmp`, `/etc`, `/root` (если есть права).

### 10.2 Пример

```bash
PID=$(curl -s -X POST http://127.0.0.1:1384/open \
    -H "Content-Type: application/json" \
    -d '{"cmd":["bash","-c","cat /etc/hostname; ls -la /tmp"],"env":{},"user":"","cwd":null}' \
    | jq -r .pid)

curl -s -X POST "http://127.0.0.1:1384/read/$PID" -d '4096'
curl -s -X POST "http://127.0.0.1:1384/kill/$PID"
```

### 10.3 Публикация в UI

Тот же принцип — писать в `/mnt/data/`:

```json
{"cmd": ["bash", "-c", "echo 'result' > /mnt/data/output.txt"]}
```

## 11. Полный пример: PNG через PIL

Создание PNG внутри VM, который затем появится в UI.

```python
from PIL import Image, ImageDraw

path = "/mnt/data/MACHINE_TEST.png"

img = Image.new("RGB", (600, 200), "white")
draw = ImageDraw.Draw(img)

# Рамка
draw.rectangle((20, 20, 580, 180), outline="black", width=4)

# Текст
draw.text((50, 80), "CREATED INSIDE LINUX MACHINE", fill="black")

img.save(path, "PNG")

print(path)
# → /mnt/data/MACHINE_TEST.png
```

После выполнения:

- файл физически создан в `/mnt/data/MACHINE_TEST.png`
- платформа автоматически подхватывает его
- в UI появляется изображение

Никаких дополнительных действий по «отправке» не требуется.

## 12. Полный пример: TXT + проверка SHA256

Создание текстового файла с проверкой существования и контрольной суммы.

```python
from pathlib import Path
import hashlib

p = Path("/mnt/data/MACHINE_TEST.txt")
data = b"HELLO_FROM_LINUX_MACHINE_20260919"

p.write_bytes(data)

print("PATH:", p)
print("EXISTS:", p.exists())
print("BYTES:", p.stat().st_size)
print("SHA256:", hashlib.sha256(data).hexdigest())
```

Ожидаемый вывод:

```
PATH: /mnt/data/MACHINE_TEST.txt
EXISTS: True
BYTES: 29
SHA256: <64-символьный hex>
```

Это проверяет, что:

- файл физически записан
- размер совпадает
- содержимое корректно

После этого файл публикуется платформой в UI.

## 13. Полный пример: файл через container_daemon

Работа через HTTP API `:8085`.

### 13.1 Запись

```bash
B64=$(echo -n "hello from container_daemon" | base64)

curl -X POST http://127.0.0.1:8085/files \
    -H 'Content-Type: application/json' \
    -d "{\"files\":{\"/home/oai/share/uploaded.txt\":\"$B64\"}}"
```

Ожидаемо: `HTTP 200`.

### 13.2 Проверка через shell

```bash
PID=$(curl -s -X POST http://127.0.0.1:1384/open \
    -H "Content-Type: application/json" \
    -d '{"cmd":["bash","-c","ls -la /home/oai/share/uploaded.txt; cat /home/oai/share/uploaded.txt"],"env":{},"user":"","cwd":null}' \
    | jq -r .pid)

sleep 0.5
curl -s -X POST "http://127.0.0.1:1384/read/$PID" -d '4096'
curl -s -X POST "http://127.0.0.1:1384/kill/$PID"
```

Ожидаемый вывод:

```
-rw-r--r-- 1 root oai_shared 24 ... /home/oai/share/uploaded.txt
hello from container_daemon
```

### 13.3 Чтение через `/file`

```bash
curl --get --data-urlencode "path=/home/oai/share/uploaded.txt" \
    http://127.0.0.1:8085/file
```

Ожидаемо: тот же текст в теле ответа.

## 14. Директории: где что лежит

### 14.1 `/home/oai/`

Рабочая директория пользователя `oai`. Здесь живёт всё, что относится к сессии:

```
/home/oai/
├── .chromium/              Chrome профиль
│   ├── Local State
│   └── Default/Preferences
├── skills/                 Skills ChatGPT
│   ├── docx/SKILL.md
│   ├── pdfs/SKILL.md
│   ├── slides/SKILL.md
│   └── spreadsheets/SKILL.md
├── share/                  Общая директория (группа oai_shared)
├── redirect.html           Sentinel-файл навигации
└── .config/                Различные конфиги
```

### 14.2 `/mnt/data/`

Директория публикации. Всё, что создано здесь, попадает в UI.

Используется платформой как «outbox» sandbox'а.

### 14.3 `/tmp/`

Временные файлы. Специфичны для каждой подсистемы:

- `/tmp/artifact_tool_rpc_*.sock` — unix sockets Artifact Tool
- `/tmp/kernel-*.json` — connection files Jupyter
- `/tmp/.X11-unix/X0` — X11 socket Xvfb
- `/tmp/cd.log`, `/tmp/hive.log` — логи вручную запущенных сервисов

### 14.4 `/etc/`

Конфиги. Доступны на чтение:

- `/etc/chromium/policies/managed/*.json` — Chrome policies
- `/etc/supervisord.conf` — конфиг supervisord
- `/etc/hostname`, `/etc/hosts`, `/etc/resolv.conf` — virtiofs mounts

### 14.5 `/opt/`

Установленное ПО:

- `/opt/pyvenv/` — Python окружение Artifact Tool
- `/opt/pyvenv-python-tool/` — Python окружение python_tool
- `/opt/terminal-server/` — Terminal Server
- `/opt/python-tool/` — код Jupyter server

### 14.6 `/var/`

Логи и состояние системы.

## 15. Что не работает

| Что | Результат | Причина |
|---|---|---|
| `/files/:path` | 404 | endpoint не зарегистрирован |
| `/files/:path/list` | 404 | то же |
| `/files/:path/exec` | 404 | то же |
| Чтение директории через `/file` | `500 Is a directory` | не файл |
| Запись вне `/home/oai/**` через `/files` | `400 blocked path` | guard |
| Невалидный base64 в `/files` | ошибка декодирования | base64 decode |
| Публикация без `/mnt/data/` | не происходит | платформа подхватывает только `/mnt/data/` |
| Чтение `/root/**` от пользователя `oai` | вероятно, отказ | права POSIX |

## 16. Health check

```bash
# 1. container_daemon запущен (иначе файловый API недоступен)
ss -ltnp | grep ':8085'

# 2. Проверка чтения (должен вернуть содержимое или 404)
curl -i --get --data-urlencode "path=/home/oai/redirect.html" \
    http://127.0.0.1:8085/file

# 3. Проверка записи (round-trip)
B64=$(echo -n "test" | base64)
curl -i -X POST http://127.0.0.1:8085/files \
    -H 'Content-Type: application/json' \
    -d "{\"files\":{\"/home/oai/share/smoke.txt\":\"$B64\"}}"

# 4. Проверка guard
curl -i -X POST http://127.0.0.1:8085/files \
    -H 'Content-Type: application/json' \
    -d "{\"files\":{\"/tmp/should_fail.txt\":\"$B64\"}}"
# Ожидаемо: 400 blocked path

# 5. Проверка публикации в UI (через Python)
# В /mnt/data/ создать файл — он должен появиться в UI
```

## 17. Связи

- `02-container-daemon.md` — общий обзор daemon, где `/file` и `/files` — часть большого API
- `03-jupyter-python-tool.md` — Python execution как один из путей работы с файлами
- `04-terminal-server.md` — shell как ещё один путь
- `07-artifact-workbook.md` — Artifact Tool RPC работает с файлами иначе (через объекты, не напрямую)

## 18. Открытые вопросы

- Точный формат ответа `/list` — что возвращает, в каком виде
- Что происходит при частичной ошибке в batch `/files` (атомарность)
- Точное поведение `/file` для несуществующего файла (404 vs 500)
- Есть ли лимит размера на файлы через `/files`
- Есть ли отдельный механизм для download через API (наблюдалось упоминание `/files/download` в одном из логов — не подтверждено)
- Точный set renderer-правил платформы: что показывается как inline, что как attachment
- Как ведёт себя платформа с файлами в `/mnt/data/` при переполнении (много файлов, большие файлы)

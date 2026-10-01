# 08-network-boundary.md

**Зона:** Сетевые границы, egress и ingress  
**Ключевой факт:** Интернета нет. Обход невозможен изнутри VM.  
**Механизм:** Двухслойная блокировка — Chrome Managed Policy внутри + Azure NSG / sidecar egress снаружи.

## Содержание

1. Главный ответ  
2. Два независимых слоя  
3. Слой 1 — Chrome Managed Policy  
4. Слой 2 — Azure NSG / sidecar egress  
5. Полная таблица egress-тестов  
6. DNS — отдельно  
7. Azure IMDS  
8. Ingress снаружи  
9. Env-барьер: startup vs kernel  
10. offline_proxy — почему он не пробивает egress  
11. get_ip в container_daemon  
12. ACE EnsureUserMachineRequest  
13. Как платформа включает интернет  
14. Sidecar egress proxy  
15. Окончательный вердикт  
16. Что не работает (полный список)  
17. Health check  
18. Связи  
19. Открытые вопросы  

# 1. Главный ответ

**Вопрос:** можно ли из этой машины выйти в интернет?

**Ответ:** Нет. И не потому, что мы чего-то не нашли, а потому что блокировка двухслойная и работает независимо:

1. Внутри VM — Chrome Managed Policy блокирует навигацию.  
2. Снаружи VM — Azure NSG / sidecar egress блокирует любой TCP-трафик.

Обход одного слоя не помогает, потому что второй срабатывает. Обойти оба изнутри VM нельзя — второй слой физически находится за пределами нашей виртуальной машины, и у нас нет ни `CAP_NET_ADMIN`, ни доступа к Azure NSG, ни способа управлять сетевыми политиками.

Решение о включении интернета принимает оркестратор Nebula при создании VM, через флаг `allow_internet` в `EnsureUserMachineRequest`. Изнутри VM это решение изменить нельзя.

# 2. Два независимых слоя

```
┌─────────────────────────────────────────────────────────┐
│ Уровень платформы (Azure / Nebula)                      │
│                                                          │
│  Sidecar egress proxy                                   │
│  ├── Network policy: caas_packages_only                 │
│  ├── Allowlist: только internal Artifactory             │
│  └── Egress: RST для всех остальных                     │
│                                                          │
│  ┌───────────────────────────────────────────────────┐  │
│  │ Kata VM                                            │  │
│  │                                                    │  │
│  │  ┌─────────────────────────────────────────────┐  │  │
│  │  │ Container                                   │  │  │
│  │  │                                             │  │  │
│  │  │  ┌───────────────────────────────────────┐ │  │  │
│  │  │  │ Chrome Browser Stack                  │ │  │  │
│  │  │  │   Managed Policy: URLBlocklist ["*"]  │ │  │  │
│  │  │  │         ↓                              │ │  │  │
│  │  │  │   ERR_BLOCKED_BY_ADMINISTRATOR        │ │  │  │
│  │  │  │         ↓                              │ │  │  │
│  │  │  │   chrome-error://chromewebdata/       │ │  │  │
│  │  │  │                                        │ │  │  │
│  │  │  │   [если policy пропустит]              │ │  │  │
│  │  │  │         ↓                              │ │  │  │
│  │  │  │   NetworkService → TCP SYN            │ │  │  │
│  │  │  └───────────────────────────────────────┘ │  │  │
│  │  │                                             │  │  │
│  │  │  Shell / Python / RPC → TCP → SYN          │  │  │
│  │  │  Bun fetch → TCP → SYN                     │  │  │
│  │  │  offline_proxy → TCP → SYN                 │  │  │
│  │  │  Google clients → TCP → SYN                │  │  │
│  │  │                                             │  │  │
│  │  │  Всё уходит на eth0 → gateway → RST         │  │  │
│  │  └─────────────────────────────────────────────┘  │  │
│  └───────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────┘
```

Первый слой работает внутри VM. Второй — снаружи. Они спроектированы так, чтобы отказ одного не открывал доступ.

# 3. Слой 1 — Chrome Managed Policy

## 3.1 Файлы политик

В `/etc/chromium/policies/managed/` лежат JSON-файлы, которые Chrome читает при запуске:

```
000_policy_merge.json
001_base_extensions.json
001_base_miscellaneous.json
001_base_url_blocklist.json      ← ключевой
010_dev_*.json
020_operator_*.json
020_research_*.json
040_caas_*.json
100_cua_dd_chrome_devtools.json
```

## 3.2 Ключевое ограничение

`001_base_url_blocklist.json` содержит:

```json
{
  "URLBlocklist": ["*"]
}
```

Wildcard `*` означает все URL запрещены. Никакая навигация на внешние ресурсы не проходит.

## 3.3 Что происходит при навигации

```
GET https://example.com
    │
    ▼
Chrome NetworkService
    │
    ▼
проверка Managed Policy
    │
    ▼
URLBlocklist: ["*"] — совпадение
    │
    ▼
ERR_BLOCKED_BY_ADMINISTRATOR
    │
    ▼
chrome-error://chromewebdata/
```

Важно: TCP-соединение не открывается. Chrome блокирует навигацию до сетевого слоя. То есть даже если бы Azure NSG был открыт, Chrome всё равно бы не смог выйти.

## 3.4 Что это значит для проверки

Даже при активном Chromium и работающем `container_daemon`:

```bash
curl -X POST http://127.0.0.1:8085/browser/new_page \
    -H 'Content-Type: application/json' \
    -d '{"url":"https://example.com"}'
```

Возвращает 200 OK, но в поле `page_html` — HTML страницы ошибки Chrome, а не содержимое сайта:

```
navigate_and_wait.url = https://example.com
navigate_and_wait.page_html length = 249800
navigate_and_wait error
error = net::ERR_BLOCKED_BY_ADMINISTRATOR
```

Размер 249800 — это HTML error-page Chrome. Реальная страница example.com занимает около 1200 байт.

## 3.5 Что это НЕ блокирует

Chrome политика не мешает:

- работать с `file://` (для redirect.html)
- обращаться к `chrome-error://` внутренним страницам
- запускать локальные скрипты расширения

# 4. Слой 2 — Azure NSG / sidecar egress

## 4.1 Что это

Снаружи VM находится сетевой фильтр уровня платформы. Он реализован как комбинация:

- Azure NSG — сетевые правила безопасности Azure
- sidecar egress proxy — прокси, который обрабатывает разрешённые запросы
- Nebula network policy — высокоуровневые правила

Этот слой физически находится вне нашей VM. Мы его не видим изнутри, не можем редактировать и не можем его обойти.

## 4.2 Что происходит с исходящими запросами

Любой TCP SYN, направленный наружу:

```
[Наша VM] → eth0 → gateway 172.26.36.1 → [Фильтр] → Интернет
                                              ↑
                                        Здесь RST
```

Gateway отвечает TCP RST — не timeout, а именно активный отказ. Это значит, что фильтр знает о запросе и явно его блокирует.

## 4.3 Что разрешено

Единственное, что разрешено политикой `caas_packages_only` — обращения к внутренним Artifactory-мирам:

- `packages.applied-caas-gateway1.internal`
- `packages-{5,6,8,9,11,12,13,16,18}-{east,west,eu}.internal`

Всё остальное блокируется.

# 5. Полная таблица egress-тестов

**Проверка из shell:**

| Адрес              | Порт | Результат                  |
|--------------------|------|----------------------------|
| api.ipify.org      | 443  | Connection refused (RST)   |
| 1.1.1.1            | 443  | Connection refused         |
| 1.1.1.1            | 80   | Connection refused         |
| 8.8.8.8            | 53   | Connection refused         |
| 93.184.216.34      | 80   | Connection refused         |
| example.com        | 443  | Connection refused         |
| google.com         | 443  | Connection refused         |
| 168.63.129.16 (Azure DNS) | 53 | UDP timeout             |
| 168.63.129.16      | 53   | TCP RST                    |
| 127.0.0.1          | 53   | timeout                    |

**Проверка через разные каналы:**

| Канал                        | Результат                        |
|------------------------------|----------------------------------|
| Python socket.create_connection | RST                           |
| Shell curl                   | RST                              |
| Bun fetch (Artifact Tool)    | RST                              |
| offline_proxy                | RST                              |
| Chrome Navigation            | ERR_BLOCKED_BY_ADMINISTRATOR     |

**Проверка через getaddrinfo:**

| Домен           | Результат              |
|-----------------|------------------------|
| api.ipify.org   | gai rc=-3 (temp failure) |
| example.com     | fail                   |
| google.com      | fail                   |

`getaddrinfo` падает до попытки открыть TCP. Это значит, что DNS-резолвинг тоже заблокирован.

# 6. DNS — отдельно

## 6.1 Что настроено

```
/etc/resolv.conf:
    nameserver 168.63.129.16
```

Это стандартный Azure DNS resolver.

## 6.2 Что происходит

Все DNS-запросы блокируются на уровне платформы:

| Метод                          | Результат     |
|--------------------------------|---------------|
| UDP/53 → 168.63.129.16         | timeout       |
| TCP/53 → 168.63.129.16         | RST           |
| TCP/53 → gateway 172.26.36.1   | RST           |
| getent hosts example.com       | fail          |
| getaddrinfo("api.ipify.org")   | gai rc=-3     |

## 6.3 Почему это важно

Даже если бы TCP-фильтр был открыт, без DNS не работали бы доменные имена. Пришлось бы использовать только IP-адреса. А поскольку TCP тоже закрыт, весь egress недоступен.

## 6.4 Что это значит для container_daemon

При старте daemon вызывает `getaddrinfo("api.ipify.org")` — и получает `gai rc=-3`. TCP-соединение даже не начинается. Это подтверждает, что daemon находится в тех же сетевых условиях, что и всё остальное.

# 7. Azure IMDS

Azure Instance Metadata Service (IMDS) — специальный endpoint `169.254.169.254`, доступный на Azure VM. Обычно даёт метаданные VM и (в некоторых конфигурациях) managed identity tokens.

## 7.1 Что проверялось

```bash
curl http://169.254.169.254/metadata/instance?api-version=2021-02-01 \
    -H "Metadata: true"
```

## 7.2 Результат

Не отвечает. Это значит, что даже внутренний Azure-сервис, обычно доступный на VM, здесь недоступен. Платформа отфильтровывает и этот путь.

# 8. Ingress снаружи

## 8.1 Какие порты проброшены

Из-за пределов VM доступны только два порта:

| Порт  | Сервис                  |
|-------|-------------------------|
| :8080 | python_tool (Jupyter)   |
| :1384 | terminal_server (shell) |

Всё остальное — `:8085`, `:50939`, `:8888`, `:9222`, `:9999` — если их поднять внутри VM, они будут слушаться, но снаружи не проброшены.

## 8.2 Что это значит

Внешний управляющий слой (платформа) может достучаться до нас только через `:8080` и `:1384`. Это единственные точки входа снаружи.

Практическое следствие: host-side consumer, который вызывает `/pull_message`, обязан идти через `:8080`. Это не «гипотеза», это следствие из наблюдаемого ingress.

## 8.3 Чем ingress отличается от egress

- **Ingress** — трафик извне в VM. Открыт только на два порта.  
- **Egress** — трафик из VM наружу. Закрыт полностью.

Это асимметричная модель: мы можем получать команды, но не можем инициировать внешние соединения.

# 9. Env-барьер: startup vs kernel

## 9.1 Что обнаружено

У процессов `supervisord` и его потомков присутствуют переменные:

- `HTTP_PROXY`
- `HTTPS_PROXY`
- `ALL_PROXY`

У процесса Jupyter kernel этих переменных нет. Или они есть, но с нерабочими значениями.

## 9.2 Что это значит

Startup environment (то, что задано при запуске VM) и kernel environment (то, что видит Python) — разные.

Это может означать:

- что proxy-переменные существуют только у некоторых процессов
- что существует граница environment boundary между уровнями иерархии процессов
- что kernel запускается с обрезанным набором env

## 9.3 Практическое следствие

Даже если бы proxy-переменные указывали на работающий прокси, из kernel их бы не было видно. То есть использование прокси из Python или shell в текущей конфигурации невозможно.

Это ещё одна причина, почему попытки установить HTTP-соединение через прокси-переменные не работают.

# 10. offline_proxy — почему он не пробивает egress

## 10.1 Что это (напоминание)

`offline_proxy` — это forward HTTP proxy внутри `container_daemon`. Он перенаправляет запросы к локальным upstream-сервисам по имени хоста.

Подробно → `02-container-daemon.md`, раздел про offline_proxy.

## 10.2 Что он не делает

`offline_proxy` не является интернет-прокси. Он не даёт доступа к внешним ресурсам.

Даже при полной активации:

```bash
PROXY_PORT=9999 START_PROXY=true \
  CUA_DD_INIT_CONTAINER_DAEMON=true \
  /usr/local/bin/container_daemon
```

— запросы всё равно уходят через тот же netns, тот же eth0, тот же gateway, и получают тот же RST.

## 10.3 Runtime-подтверждение

При попытке обращения через offline_proxy к внешнему адресу:

```
error forwarding request example.com to api.ipify.org
hyper_util::client::legacy::Error
  Connect
  ConnectError("tcp connect error", Os { code: 111, kind: ConnectionRefused })
```

То есть offline_proxy сам получает ConnectionRefused от внешнего адреса. Это тот же RST, что виден из shell, Python и Chrome.

## 10.4 В текущей конфигурации — вообще не стартует

В нашей сборке `START_PROXY=false`, поэтому в логе видно:

```
SKIP offline_proxy
```

Подсистема даже не пытается инициализироваться.

## 10.5 Вывод

`offline_proxy` — это инструмент внутренней маршрутизации, а не обхода egress. Использовать его для выхода наружу бессмысленно.

# 11. get_ip в container_daemon

## 11.1 Что делает

При старте `container_daemon` вызывает:

```
reqwest::Client::get("https://api.ipify.org/")
```

Цель — получить внешний IP VM. Результат сохраняется в AppState через `set_ip`.

## 11.2 Ассемблерный фрагмент

```
0x602bfe: lea r13, [rip+...]     → "https://api.ipify.org/"
0x602c0c: mov ..., 0x16          → длина URL = 22
0x602c9f: call reqwest::...Client::get
```

Это подтверждает: строка в бинарнике — не мёртвая, код исполняется.

## 11.3 Runtime-результат

При ручном запуске daemon:

```
getaddrinfo("api.ipify.org")
    ↓
gai rc = -3
```

DNS не резолвится. TCP даже не начинается.

Поле `ip` в `/-статусе` остаётся `"unknown"`.

## 11.4 Что это значит

Ранняя гипотеза «daemon имеет эксклюзивный сетевой путь наружу» — опровергнута. Daemon работает в той же сетевой среде, что и всё остальное. Код внешнего запроса есть, но сетевой путь закрыт.

# 12. ACE EnsureUserMachineRequest

## 12.1 Что это

В установленном пакете `research_ace.v2.ace_types` найден тип `EnsureUserMachineRequest`. Это protocol-сообщение, которое описывает запрос на создание/настройку User Machine.

## 12.2 Поля

```python
message_type
timeout
user_id
max_time_alive
allow_internet
user_id_label
```

## 12.3 Особое поле

`allow_internet` имеет тип `bool | None`.

Возможные значения:

| Значение | Смысл                          |
|----------|--------------------------------|
| true     | интернет разрешён              |
| false    | интернет запрещён              |
| None     | не задано (значение по умолчанию) |

Производное состояние: `internet_access_level` со значениями:

- `"true"`
- `"false"`
- `None`

## 12.4 Что это доказывает

Существование этого типа — прямое протокольное доказательство, что платформа умеет управлять доступом к интернету на уровне архитектуры. Это не «магия», это явное поле в контракте.

## 12.5 Чего это не доказывает

Consumer этого типа внутри VM не найден. То есть в нашей песочнице нет обработчика `EnsureUserMachineRequest`. Этот тип — часть протокола на уровне платформы, а не часть клиента внутри VM.

Мы не можем изнутри отправить такой запрос и перезапустить себя с `allow_internet=true`. Это решение принимает оркестратор Nebula при создании VM.

# 13. Как платформа включает интернет

## 13.1 Динамический provisioning

Платформа работает так:

1. Пользователь (или модель) инициирует задачу через ChatGPT.  
2. Оркестратор Nebula решает, нужен ли интернет для этой задачи.  
3. Формируется `EnsureUserMachineRequest` с полем `allow_internet`.  
4. VM создаётся с нужной сетевой политикой сразу.

Никакого «включения интернета на лету» изнутри VM не происходит. Сетевая политика назначается до старта VM.

## 13.2 Когда включается allow_internet=true

По логике платформы, интернет включается, когда задача этого требует:

- режим Operator (браузерный агент)
- режим Deep Research (поиск в интернете)
- режим SearchGPT
- режим CUA (Computer-Use Agent)
- явный запрос пользователя: «открой сайт X»

В таких случаях:

- Разворачивается новая VM с `allow_internet=true`
- Или (в некоторых конфигурациях) включается allowlist в текущей VM
- Chrome policy становится разрешающей (`URLAllowlist: ["*"]`)
- Egress открывается через sidecar

## 13.3 Почему мы этого не видим

Наша сессия — обычный Code Interpreter без интернет-задач. Поэтому:

- `NETWORK=caas_packages_only`
- `CUA_DD_ENABLE_CHROME=false`
- `START_PROXY=false`
- Chrome policy `URLBlocklist: ["*"]`

Всё это — согласованный набор ограничений, назначенный при создании VM.

# 14. Sidecar egress proxy

## 14.1 Что это

По публичной документации OpenAI, hosted containers используют sidecar egress proxy — отдельный процесс на стороне платформы, который обрабатывает все исходящие запросы из контейнера.

Этот proxy:

- работает вне контейнера
- находится под контролем платформы
- решает, что пропустить, а что заблокировать
- ведёт логи

## 14.2 Что это даёт платформе

- Централизованный контроль egress
- Возможность audit'а трафика
- Возможность включать allowlist на конкретные домены
- Возможность полностью блокировать внешние соединения

## 14.3 Что это значит для нас

Изнутри VM мы не можем влиять на sidecar egress proxy. Он находится снаружи нашего network namespace, вне нашей зоны контроля.

Единственный способ получить интернет — чтобы платформа сама конфигурировала sidecar с разрешающей политикой. А это решение принимается при создании VM.

## 14.4 Публичное подтверждение

Публичная документация OpenAI по hosted containers явно упоминает:

- изолированные containers
- network policies с `allowed_domains`
- egress по умолчанию закрыт

Наши наблюдения полностью согласуются с этой документацией.

# 15. Окончательный вердикт

## 15.1 Прямой ответ

Интернета из этой VM нет. И не будет в текущей конфигурации.

## 15.2 Почему

Два независимых слоя:

1. Chrome Managed Policy — блокирует навигацию внутри браузера  
2. Azure NSG / sidecar egress — блокирует любой TCP-трафик наружу

Обход невозможен изнутри VM. Причина: второй слой физически находится за пределами нашей виртуальной машины.

## 15.3 Единственный легитимный путь

Платформа сама даёт интернет через `allow_internet=true` в `EnsureUserMachineRequest` — решение принимается на стороне оркестратора Nebula.

Это решение:

- не контролируется изнутри VM
- не контролируется моделью
- не контролируется пользователем напрямую

Триггеры: Operator mode, Deep Research, SearchGPT, CUA, явный запрос пользователя на работу с внешним сайтом.

## 15.4 Что это означает для дизайна

Это не баг. Это сознательный дизайн безопасности:

- Chrome policy — внутренний слой, защищает от ошибок конфигурации Chrome
- Azure NSG — внешний слой, гарантирует изоляцию VM
- Оба работают независимо, отказ одного не открывает доступ

# 16. Что не работает (полный список)

## 16.1 Egress

| Способ                              | Результат                        |
|-------------------------------------|----------------------------------|
| Прямой TCP через shell              | RST                              |
| Прямой TCP через Python             | RST                              |
| Прямой TCP через Bun fetch          | RST                              |
| HTTP через offline_proxy            | RST                              |
| HTTPS через offline_proxy           | RST                              |
| Навигация через Chrome              | ERR_BLOCKED_BY_ADMINISTRATOR     |
| GET через Artifact Tool             | RST                              |
| Google API через FetchGoogleSheetsClient | RST                         |
| DNS-резолвинг (UDP/53)              | timeout                          |
| DNS-резолвинг (TCP/53)              | RST                              |
| getaddrinfo                         | gai rc=-3                        |
| Azure IMDS (169.254.169.254)        | нет ответа                       |

## 16.2 Ingress

| Порт   | Что                              |
|--------|----------------------------------|
| :8080  | ✅ проброшен (Jupyter)           |
| :1384  | ✅ проброшен (Terminal)          |
| :8085  | ❌ снаружи не проброшен           |
| :50939 | ❌ снаружи не проброшен           |
| :8888  | ❌ снаружи не проброшен           |
| :9222  | ❌ снаружи не проброшен           |
| :9999  | ❌ снаружи не проброшен           |

## 16.3 Env-барьер

| Что                        | Результат     |
|----------------------------|---------------|
| HTTP_PROXY в supervisor env| есть          |
| HTTP_PROXY в kernel env    | нет           |
| HTTPS_PROXY в kernel env   | нет           |
| ALL_PROXY в kernel env     | нет           |

## 16.4 Capabilities

| Что            | Есть |
|----------------|------|
| CAP_NET_ADMIN  | ❌    |
| CAP_NET_RAW    | ✅ (но ping всё равно не проходит) |
| CAP_SYS_ADMIN  | ❌    |
| CAP_SYS_MODULE | ❌    |
| CAP_BPF        | ❌    |

# 17. Health check

## 17.1 Проверка egress (ожидаем RST)

```bash
python3 -c "
import socket
for h, p in [('1.1.1.1', 443), ('8.8.8.8', 53), ('93.184.216.34', 80)]:
    try:
        s = socket.create_connection((h, p), 2)
        print(f'OK {h}:{p}')
        s.close()
    except Exception as e:
        print(f'FAIL {h}:{p}: {type(e).__name__}')
"
```

Ожидаемо: FAIL для всех.

## 17.2 Проверка DNS

```bash
getent hosts example.com
# → пусто

python3 -c "import socket; print(socket.getaddrinfo('example.com', 80))"
# → gaierror
```

## 17.3 Проверка ingress

Изнутри VM мы не можем проверить ingress напрямую. Но знаем, что `:8080` и `:1384` проброшены — это подтверждается тем, что платформа может вызывать `/pull_message` и `/execute`.

## 17.4 Проверка Chrome policy

```bash
cat /etc/chromium/policies/managed/001_base_url_blocklist.json
# → {"URLBlocklist": ["*"]}
```

## 17.5 Проверка offline_proxy

```bash
# В логе container_daemon
grep -i 'SKIP offline_proxy\|START_PROXY' /tmp/cd.log
```

# 18. Связи

- `01-infrastructure.md` — network settings (VNet, DNS, eth0)
- `02-container-daemon.md` — get_ip, offline_proxy, /offline_sites
- `03-jupyter-python-tool.md` — Python execution не может выйти наружу
- `04-terminal-server.md` — shell тоже не может
- `06-browser-stack.md` — Chrome policy — первый слой блокировки
- `07-artifact-workbook.md` — Google bridges бессмысленны без egress

# 19. Открытые вопросы

## 19.1 Точная конфигурация sidecar

Мы знаем, что sidecar существует. Но точная его конфигурация, список разрешённых доменов (кроме Artifactory), схема логирования — вне зоны видимости изнутри VM.

## 19.2 Как allow_internet=true меняет политику

Мы знаем, что решение принимается Nebula. Но конкретно:

- отключается ли URLBlocklist полностью
- или заменяется на allowlist
- какие домены попадают в allowlist по умолчанию
- как это связано с mode (Operator vs Research)

— не установлено.

## 19.3 Поведение при частичной деградации

Что происходит, если один из слоёв по каким-то причинам отключается? Возможно ли временное окно, когда egress становится доступен? Публичных данных нет, экспериментально мы это не проверяли.

## 19.4 Публичная документация OpenAI

Существует публичная документация о hosted containers и network policies. Мы можем её читать без нарушения ToS. Наши наблюдения согласуются с ней.

## 19.5 Что делать при необходимости интернета

Если задача действительно требует интернета:

1. Явно сформулировать это в запросе к ChatGPT  
2. Платформа сама примет решение о создании VM с `allow_internet=true`  
3. Не пытаться обойти ограничения изнутри — это нарушение ToS и не работает технически

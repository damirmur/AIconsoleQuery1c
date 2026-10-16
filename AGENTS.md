# AGENTS.md — чистый стенд ИИ-агента для работы с кодом 1С

> Цель: воспроизвести на **чистом Windows-ПК** систему работы с BSL-кодом для ИИ-агента **без** веса конфигураций/обработок-конфигуратора. Только код, расширения, линтинг и сборка.

---

## 1. Что ставить на чистый ПК

### 1.1 Обязательное (ядро)

| № | Компонент | Зачем агенту | Версия | Источник |
|---|---|---|---|---|
| 1 | **Python 3.9+** (рекоменд. 3.11–3.14) | Рантайм 79 навыков `cc-1c-skills` | 3.11+ | https://www.python.org/downloads/ — Windows installer (64-bit), галочка *Add to PATH* |
| 2 | **cc-1c-skills (Python-ветка)** | Создание/правка/валидация метаданных, форм, ролей, MXL/СКД, выгрузка/загрузка XML, сборка EPF | ветка `python` / `opencode-py` | https://github.com/Nikolay-Shirokov/cc-1c-skills — `git clone -b python https://github.com/Nikolay-Shirokov/cc-1c-skills.git` и скопировать `.opencode/skills/` в корень проекта |
| 3 | **Python-зависимости** `lxml Pillow psutil` | Без них навыки падают | `lxml>=4.9.0`, `Pillow>=10.0.0`, `psutil>=5.9.0` | `pip install -r requirements.txt` (файл в комплекте `cc-1c-skills`) |
| 4 | **BSL Language Server 1.0.7 (JPackage)** | Офлайн-линтинг BSL (`analyze -r json/sarif/console`) | 1.0.7 JRE17 встроена | https://github.com/1c-syntax/bsl-language-server/releases/tag/v1.0.7 — `bsl-language-server-1.0.7-jpackage.zip` или `bsl-language-server-1.0.7-exec.jar` + JRE17 |
| 5 | **Платформа 1С:Предприятие 8.3** (`1cv8.exe`, `ibcmd.exe`) | Сборка EPF/ERF, `DumpConfigToFiles/LoadConfigFromFiles`, `UpdateDBCfg` | 8.3.24+ (рекоменд. 8.3.27) | https://releases.1c.ru — дистрибутив «Технологическая платформа 8.3» (требует ИТС-аккаунт) или https://v8.1c.ru/predpriyatie-podderzhka/; учебная версия — https://v8.1c.ru/predpriyatie/uchebnaya-versiya/ |
| 6 | **Node.js 18+** | Только если нужен `web-*` (публикация Apache) | 18+ | https://nodejs.org/ |
| 7 | **OpenCode** | Раннер навыков (`opencode` CLI) | latest | https://opencode.ai/docs — `npm i -g opencode` |

### 1.2 Опциональное (только под задачу)

| Компонент | Когда нужен | Источник | Примечание |
|---|---|---|---|
| `v8unpack 1.2.11` (`pip install v8unpack`) | Быстрая правка только `.bsl` без пересборки формы; legacy-проекты с бинарным `Form.json` | https://github.com/saby-integration/v8unpack / PyPI `pip install v8unpack` | На чистом стенде не обязателен — `epf-build/dump` из `cc-1c-skills` делает то же через платформу, но точнее хранит ссылочные типы |
| `1c-buddy` (MCP `ask_1c_ai/check_1c_code`) | ИИ-подсказки по синтаксису BSL, поиск по ИТС | https://code.1c.ai + https://github.com/1c-syntax/1c-buddy | Требует токен `ONEC_AI_TOKEN`, см. §9 |
| `V8Reader.epf` | Визуальный аудит форм/ролей, сравнение `*.cf/*.epf` | http://infostart.ru/public/106310/ | Интерактивная обработка, агенту не нужна |

---

## 2. Что дублируется и что актуально (разбор)

На практике 3 канала делают одно и то же. Оставлен актуальный, остальные — legacy.

| Операция | Канал A: `cc-1c-skills` (актуально) | Канал B: кастомный MCP 1С | Канал C: `v8unpack` (legacy) | Вердикт |
|---|---|---|---|---|
| **Выгрузить конфигурацию/расширение в XML** | `db-dump-xml -Mode Full/Changes/Partial -Objects "..."` + `-Extension` | `dump_config/dump_extension` → `1cv8 DESIGNER /DumpConfigToFiles` без `Changes/Partial` | — | **A**. B не умеет `Changes/Partial/UpdateInfo`, не пишет `ConfigDumpInfo.xml` |
| **Загрузить XML в базу (+расширение)** | `db-load-xml -Mode Full/Partial -Files ... -Extension Имя -UpdateDB` | `load_config/load_extension` | — | **A**. Поддерживает частичную загрузку файлами, B — только полный каталог |
| **Собрать EPF/ERF из XML** | `epf-build -SourceFile src/Обработка.xml -OutputFile build/Обработка.epf` — без базы создаёт temp-базу с заглушками | `build_epf` → `/LoadExternalDataProcessorOrReportFromFiles` | `python -m v8unpack -B build_tmp out.epf` | **A** — основной. C — только для быстрой правки `.bsl` когда форма уже настроена и трогать `Form.json` нельзя |
| **Разобрать EPF/ERF в XML** | `epf-dump -InputFile build/Обработка.epf -OutputDir src` — требует базу, иначе ссылочные типы потеряются | `dump_epf` | `python -m v8unpack -E in.epf build_tmp` | **A**. C теряет типы в пустой базе |
| **Создать объект метаданных** | `meta-compile -JsonPath def.json -OutputDir src/cf` — JSON DSL, авто-UUID, регистрация в `Configuration.xml` | только чтение метаданных | — | **A**, альтернативы нет |
| **Править объект** | `meta-edit -Operation add-attribute ...` + `form-edit`/`cfe-borrow` | — | правка `Form.json` вручную (ломает UUID) | **A** |
| **Валидация** | `cf-validate`/`cfe-validate -ConfigPath src/cf` + `meta-validate`/`form-validate`/`role-validate`/… | нет | нет | **A** |
| **Линтинг BSL** | `tools/lint.ps1` → `bsl-language-server analyze -s src -c .bsl-language-server.json -r json,sarif,console` | `1c-buddy check_1c_code` (онлайн) | — | **BSL LS** — офлайн must-have; `1c-buddy` — опциональный AI-доп |
| **Выполнить запрос / метаданные** | — | `execute_query`/`get_metadata` via COM `V83.COMConnector` | — | **B** — единственный COM-канал, ставь только если нужен доступ к живой базе |
| **Создать форму** | `form-compile` JSON DSL или `-FromObject` | — | — | **A** |

**Итого на чистом ПК:** ядро = `Python + cc-1c-skills + BSL LS + 1cv8`.

---

## 3. Источники — где скачать (точные URL)

| Что | URL | Что качать |
|---|---|---|
| **Python** | https://www.python.org/downloads/ | `Windows installer (64-bit)` 3.11+ |
| **cc-1c-skills (Python-ветка)** | https://github.com/Nikolay-Shirokov/cc-1c-skills — ветка `python` | `Code → Download ZIP` ветки `python` → распаковать `.opencode/skills/` в корень проекта. Альтернатива: `git clone -b python https://github.com/Nikolay-Shirokov/cc-1c-skills.git` |
| **Зависимости Python** | PyPI | `pip install -r requirements.txt` → `lxml>=4.9.0`, `Pillow>=10.0.0`, `psutil>=5.9.0` |
| **BSL Language Server** | https://github.com/1c-syntax/bsl-language-server/releases | `bsl-language-server-1.0.7-jpackage.zip` (JPackage+JRE17) или `bsl-language-server-1.0.7-exec.jar` + JRE17 |
| **Платформа 1С 8.3** | https://releases.1c.ru (ИТС) или https://v8.1c.ru/predpriyatie/ | «Технологическая платформа 8.3» (x64) — нужен `1cv8.exe` и `ibcmd.exe` в `C:\Program Files\1cv8\...` Без ИТС — учебная версия https://v8.1c.ru/predpriyatie/uchebnaya-versiya/ |
| **Node.js** (опционально) | https://nodejs.org/ | LTS 18+ |
| **OpenCode** | https://opencode.ai/docs | `npm i -g opencode` |
| **v8unpack** (опционально) | https://github.com/saby-integration/v8unpack / PyPI | `pip install v8unpack` |
| **1c-buddy** (опционально) | https://code.1c.ai (токен) + https://github.com/1c-syntax/1c-buddy | `ONEC_AI_TOKEN` — см. §9 |
| **V8Reader** (опционально) | http://infostart.ru/public/106310/ | `V8Reader.epf` |

Проверка: `python --version` (≥3.9), `python -c "import lxml"`, `bsl-language-server.exe analyze --help`, `1cv8.exe /?`.

---

## 4. Пошаговая установка на чистом Windows 10/11

### Шаг 0 — подготовка каталога
```powershell
mkdir C:\1C-Agent; cd C:\1C-Agent
git init
```

### Шаг 1 — Python + зависимости
```powershell
winget install Python.Python.3.11   # или скачай с python.org, галочка Add to PATH
python --version                    # ≥3.9
pip install -r requirements.txt     # из комплекта cc-1c-skills
# вручную: pip install "lxml>=4.9.0" "Pillow>=10.0.0" "psutil>=5.9.0"
python -c "import lxml, PIL, psutil; print('ok')"
```

### Шаг 2 — cc-1c-skills
```powershell
git clone -b python https://github.com/Nikolay-Shirokov/cc-1c-skills.git .opencode_tmp
xcopy /E /I .opencode_tmp\.opencode .opencode
# Проверка
dir .opencode\skills | Measure-Object
python ".opencode/skills/cf-validate/scripts/cf-validate.py" --help
python ".opencode/skills/epf-build/scripts/epf-build.py" --help
```

### Шаг 3 — BSL Language Server
```powershell
# Скачай bsl-language-server-1.0.7-jpackage.zip с GitHub Releases, распакуй в C:\tools\bsl-language-server
C:\tools\bsl-language-server\bsl-language-server.exe analyze --help
# Создай .bsl-language-server.json и tools/lint.ps1 (шаблон — из репозитория cc-1c-skills или сгенерируй по §6.6)
.\tools\lint.ps1  # должен выдать No issues found на пустом src/
```

### Шаг 4 — Платформа 1С
```powershell
# Установи Технологическую платформу 8.3 x64 с releases.1c.ru
"C:\Program Files\1cv8\common\1cestart.exe"
"C:\Program Files\1cv8\8.3.27.2342\bin\1cv8.exe" /?
# Создай пустую файловую базу для сборки EPF (или полагайся на temp-базу epf-build):
& "C:\Program Files\1cv8\8.3.27.2342\bin\1cv8.exe" CREATEINFOBASE File="C:\Bases\TempAgent"
# Для COM-доступа (опционально, от Администратора):
regsvr32 "C:\Program Files\1cv8\8.3.27.2342\bin\comcntr.dll"
```

### Шаг 5 — OpenCode
```powershell
npm i -g opencode
@"
{
  "`$schema": "https://opencode.ai/config.json"
}
"@ | Set-Content opencode.json -Encoding utf8
```

### Шаг 6 — проверка стенда
```powershell
python ".opencode/skills/cf-init/scripts/cf-init.py" --help
python ".opencode/skills/meta-compile/scripts/meta-compile.py" --help
python ".opencode/skills/form-compile/scripts/form-compile.py" --help
python ".opencode/skills/epf-build/scripts/epf-build.py" --help
.\tools\lint.ps1 -Severity Warning
# Тест: создать пустую конфигурацию и провалидировать
python ".opencode/skills/cf-init/scripts/cf-init.py" -Name TestCF -OutputDir src\cf\test
python ".opencode/skills/cf-validate/scripts/cf-validate.py" -ConfigPath src\cf\test
```

### Шаг 7 — (опционально) v8unpack / 1c-buddy
```powershell
pip install v8unpack
python -m v8unpack --help
```

---

## 5. Структура чистого проекта после установки

```
C:\1C-Agent\
  .bsl-language-server.json   # конфиг линтера
  .opencode/skills/           # 79 папок (cf-*, cfe-*, meta-*, form-*, epf-*, db-*, role-*, mxl-*, skd-*, xdto-*...)
  tools/lint.ps1              # обёртка BSL LS
  tools/reports/<timestamp>/  # json/sarif/console отчёты
  src/cf/                     # выгрузки конфигураций (читаемый XML: Configuration.xml + Catalogs/...)
  src/cfe/<Имя>/              # расширения (Configuration.xml + заимствованные объекты)
  src/epf/                    # исходники обработок ( *.xml + Ext/*.bsl )
  build/                      # собранные *.epf/*.erf
  requirements.txt
  opencode.json
```

`.v8-project.json` (опционально) резолвит базы для всех `db-*`/`epf-*`: `v8path`, `databases[].{id,alias,name,connect,configSrc,branches}`, `extensions[].src`.

---

## 6. Как работать агенту (без конфигураций — только код)

> Ниже — команды Python-навыков напрямую (без слэш-команд). Все пути — PowerShell.

### 6.1 Объекты метаданных

```powershell
# Создать справочник/документ/регистр из JSON
@'
{ "type": "Catalog", "name": "Организации", "attributes": ["ИНН: String(12)", "Директор: CatalogRef.ФизическиеЛица"] }
'@ | Set-Content def.json -Encoding utf8
python ".opencode/skills/meta-compile/scripts/meta-compile.py" -JsonPath def.json -OutputDir src/cf/my

# Добавить реквизит/ТЧ
python ".opencode/skills/meta-edit/scripts/meta-edit.py" -ObjectPath src/cf/my/Catalogs/Организации.xml -Operation add-attribute -Value "КПП: String(9) | index"

# Удалить/переименовать
python ".opencode/skills/meta-edit/scripts/meta-edit.py" -ObjectPath src/cf/my/Catalogs/Организации.xml -Operation remove-attribute -Value "УстаревшийРекв"
python ".opencode/skills/meta-edit/scripts/meta-edit.py" -ObjectPath src/cf/my/Catalogs/Организации.xml -Operation modify-attribute -Value "СтароеИмя: name=НовоеИмя, type=Строка(500)"

# Валидация
python ".opencode/skills/meta-validate/scripts/meta-validate.py" -ObjectPath src/cf/my/Catalogs/Организации.xml
python ".opencode/skills/cf-validate/scripts/cf-validate.py" -ConfigPath src/cf/my
```

Шорткат реквизитов: `"Имя: Тип | req,index,nonneg,master"`; составной `CatalogRef.A + String(25)`; ТЧ `tabularSections: {"Товары": ["Номенклатура: CatalogRef.Номенклатура"]}`.

### 6.2 Расширения

```powershell
python ".opencode/skills/cfe-init/scripts/cfe-init.py" -Name МоёРасширение -ConfigPath src/cf/base -OutputDir src/cfe/МоёРасширение
python ".opencode/skills/meta-compile/scripts/meta-compile.py" -JsonPath def.json -OutputDir src/cfe/МоёРасширение
python ".opencode/skills/cfe-validate/scripts/cfe-validate.py" -ExtensionPath src/cfe/МоёРасширение -ConfigPath src/cf/base
```

Нюанс 8.3.27: у собственных объектов расширения не ставить `<ObjectBelonging>`, типы — `<v8:TypeId>UUID</v8:TypeId>` из базовой конфигурации, не `cfg:CatalogRef`.

### 6.3 Формы

```powershell
python ".opencode/skills/form-compile/scripts/form-compile.py" -JsonPath form.json -OutputPath src/cf/my/Catalogs/Номенклатура/Forms/ФормаЭлемента/Ext/Form.xml
python ".opencode/skills/form-validate/scripts/form-validate.py" -FormPath src/cf/my/Catalogs/Номенклатура/Forms/ФормаЭлемента/Ext/Form.xml
python ".opencode/skills/form-info/scripts/form-info.py" -FormPath ...
```

DSL: `group/input/check/table/pages/button`, `path: "Объект.Реквизит"`, `events: {OnCreateAtServer: "ПриСозданииНаСервере"}`. Для внешних обработок `DataPath` — реквизиты формы, не `Объект.*`.

### 6.4 Обработки/отчёты (EPF/ERF)

```powershell
# Разобрать (требует базу!)
python ".opencode/skills/epf-dump/scripts/epf-dump.py" -InfoBasePath C:\Bases\TempAgent -InputFile build/МояОбработка.epf -OutputDir src/epf
# Собрать (без базы — temp-база с заглушками)
python ".opencode/skills/epf-build/scripts/epf-build.py" -InfoBasePath C:\Bases\TempAgent -SourceFile src/epf/МояОбработка.xml -OutputFile build/МояОбработка.epf
# v8unpack-альтернатива только для правки .bsl:
python -m v8unpack -E build/МояОбработка.epf src/epf/build_tmp
# ...правь Module.bsl...
python -m v8unpack -B src/epf/build_tmp build/МояОбработка.epf
```

### 6.5 БД (выгрузка/загрузка XML)

```powershell
python ".opencode/skills/db-dump-xml/scripts/db-dump-xml.py" -InfoBasePath C:\Bases\TempAgent -ConfigDir src/cf/my -Mode Full
python ".opencode/skills/db-dump-xml/scripts/db-dump-xml.py" -InfoBasePath C:\Bases\TempAgent -ConfigDir src/cf/my -Mode Partial -Objects "Справочник.Номенклатура,Документ.Заказ"
python ".opencode/skills/db-load-xml/scripts/db-load-xml.py" -InfoBasePath C:\Bases\TempAgent -ConfigDir src/cf/my -Mode Full -UpdateDB
python ".opencode/skills/db-load-xml/scripts/db-load-xml.py" -InfoBasePath C:\Bases\TempAgent -ConfigDir src/cfe/МоёРасширение -Extension МоёРасширение -UpdateDB
python ".opencode/skills/db-update/scripts/db-update.py" -InfoBasePath C:\Bases\TempAgent
```

Режимы `db-dump-xml`: `Full/Changes/Partial/UpdateInfo`; `db-load-xml Full` перезатирает всё — спрашивать подтверждение.

### 6.6 Линтинг

```powershell
.\tools\lint.ps1                  # Warning (default)
.\tools\lint.ps1 -Severity Error
.\tools\lint.ps1 -SourceDir src/cf/my -Severity Information
# Результат: tools/reports/<timestamp>/bsl-json.json + bsl-ls.sarif
```

Правило стенда: перед коммитом `Severity>=Warning` → `No issues found`. Конфиг подавляет `UnusedLocalMethod/Variable` (события форм), `MagicNumber`, `LineLength` — не править без нужды.

### 6.7 Роли/прочее

```powershell
python ".opencode/skills/role-compile/scripts/role-compile.py" -JsonPath role.json -OutputDir src/cf/my
# role.json: {"name":"ЧтениеНоменклатуры","objects":["Catalog.Номенклатура: @view"]}
python ".opencode/skills/role-validate/scripts/role-validate.py" -RightsPath src/cf/my/Roles/ЧтениеНоменклатуры/Ext/Rights.xml
```

MXL/СКД/XDTO — `mxl-compile/decompile/validate`, `skd-compile/edit/validate`, `xdto-compile/edit/validate` аналогично.

---

## 7. Подводные камни

- `Form.json/elem.json` (v8unpack) — бинарная структура с UUID; правка ломает форму. Для новых полей/кнопок — `form-compile` или пересборка в Конфигураторе, не ручная правка JSON.
- Расширение: собственные объекты — без `<ObjectBelonging>`; заимствованные — `Adopted + ExtendedConfigurationObject`. Ссылочные типы — `UUID` из `InternalInfo/GeneratedType category="Ref"` базовой конфигурации.
- `ОпределяемыйТип` из того же расширения в 8.3.27 конфликтует с `ObjectBelonging` — вместо него составной `<v8:TypeId>` напрямую.
- `Соединение.Метаданные()` через COM — медленно; кэшируй `Тип→МетаОбъект` в `Соответствие`.
- `ПолеФормы.СписокВыбора` на сервере — только чтение, заполняй на клиенте через `Элементы.Поле.СписокВыбора.Добавить(...)`.
- `build_tmp/` — копия `.bsl`, игнор линтера по `\build_tmp\`.

---

## 8. Контрольный чек-лист чистого стенда

- [ ] `python --version` ≥3.9, `pip install -r requirements.txt` без ошибок
- [ ] `.opencode/skills/` — 79 папок, `cf-validate --help` отрабатывает
- [ ] `bsl-language-server.exe analyze -s src -c .bsl-language-server.json -o out -r console --silent` — `No issues found`
- [ ] `C:\Program Files\1cv8\8.3.*\bin\1cv8.exe` запускается, `epf-build` собирает тестовый `build/Тест.epf`
- [ ] `.\tools\lint.ps1 -Severity Warning` — зелёный
- [ ] (опционально) `v8unpack -E/-B` на тестовом `.epf` — без ошибок

---

## 9. API 1c-buddy (1С:Напарник) на сервере — кратко

**Что это:** код `1c-buddy-go` встроен Go-модулем в сервис `llm-1c` (`:8086`), отдельного процесса нет. Все запросы проксируются в апстрим `https://code.1c.ai` по токену `ONEC_AI_TOKEN` из `/home/dmyr/llm-1c/llm.env`.

**Внешние префиксы (алиасы 1:1, префикс срезается):**
```
/api/1c-buddy/* — основной
/api/onec/*, /api/1c/* — алиасы
```
Живьём: `https://astro3d.ru/api/1c-buddy/...` → `127.0.0.1:8086`.

**Авторизация:** единый ключ `LLM_PROXY_KEY` (фолбэк `NOTIFY_KEY`). Способы: заголовки `X-API-Key` / `X-LLM-Proxy-Key` / `Authorization: Bearer`, или query `?api_key=` (также `key`, `llm_proxy_key`, `balancer_key`, `llm_key`). Без ключа — `403`. Исключение: `GET /chat/static/*` (CSS/JS) — без ключа. Внутри OpenAI-часть проверяет только `Bearer` / `?api_key=`, шлюз сам прокидывает ключ внутрь. STDIO не проксируется (для IDE — прямой `1c-buddy mcp-stdio`).

### 9.1 Браузерный чат
- `GET /chat` — HTML (требует ключ), статика грузится сама
- `GET /chat/api/config` — конфиг UI (версия, лимиты)
- `POST /chat/api/mcp/list-tools` — инструменты внешних MCP для чата
- `POST /chat/api/stream` (алиас `/chat/http-stream`) — **основной транспорт**. Тело: `{message, conversation_id, create_new_session, programming_language, parent_uuid, workspace_instructions, mcp_config}`. Ответ: NDJSON `application/x-ndjson`, строки `{"type": meta|delta|reasoning|tool_call|tool_result|tokens|done|error}`.
- `WS /chat/ws` — фолбэк (тот же JSON первым фреймом). Внимание: nginx для `/api/` не прокидывает `Upgrade`, чистый WS даёт `400` — использовать HTTP-стрим.

### 9.2 OpenAI-совместимый API
- `GET /v1/models` — только discovery: `{"data":[{"id":"1c-buddy",...}]}`, без апстрима
- `POST /v1/chat/completions` — **рабочий вход**: `{model:"1c-buddy", messages:[{role,content}], stream:false|true, metadata:{conversation_id}}`. Без `stream` — JSON `chat.completion`; `stream:true` — SSE `text/event-stream`. Заголовок `X-1C-Conversation-Id` прокидывается.
- `WS /v1/chat/completions/ws` — то же по сокету.

### 9.3 MCP (для агентов/IDE)
- `GET /mcp` — визитка `{"name":"1C.ai Gateway MCP","version":"1.4.1","endpoint":"/mcp"}`
- `POST /mcp` — Streamable HTTP, JSON-RPC: `initialize` → из заголовка ответа взять `Mcp-Session-Id` → `tools/list` → `tools/call {name, arguments}` с header `Mcp-Session-Id`. Каждый call открывает новый диалог в `1C.ai`.
- `WS /mcp/ws` — JSON-RPC по сокету, сессия привязана к соединению.
- Инструменты (8): `ask_1c_ai` (общий вопрос), `explain_1c_syntax`, `check_1c_code` (`syntax|review`), `modify_1c_code`, `search_1c_documentation`, `search_its`, `fetch_its` (по `id`, `root` — корень), `diff_1c_documentation_versions`. Алиасы: `Search_1C_Documentation`, `Search_ITS`, `Fetch_ITS`, `Diff_1C_Documentation_Versions`.

**Проверка (`KEY=$LLM_PROXY_KEY` из `llm.env`):**
```bash
curl "http://127.0.0.1:8086/api/1c-buddy/chat/api/config?api_key=$KEY"
curl "http://127.0.0.1:8086/api/1c-buddy/v1/models?api_key=$KEY"
curl -X POST "http://127.0.0.1:8086/api/1c-buddy/v1/chat/completions?api_key=$KEY" \
  -H "Content-Type: application/json" \
  -d '{"model":"1c-buddy","messages":[{"role":"user","content":"Ответь одним словом: тест"}]}'
curl -N -X POST "http://127.0.0.1:8086/api/1c-buddy/chat/api/stream?api_key=$KEY" \
  -H "Content-Type: application/json" -d '{"message":"Привет","create_new_session":true}'
# Ошибки: 403 — ключ; 500 на /chat/api/stream — нет ONEC_AI_TOKEN; 502 на /v1/* или /mcp — апстрим code.1c.ai недоступен
```

**Код:** `onec_gateway.go`, `internal/buddy/{chat,openai,mcp,onec,config}`. Доки: `html/docs/api-1c-buddy-balancer.md`, `api.md` (раздел `ANY /api/1c-buddy`).

# Руководство по запуску OCLint

## 1. Назначение и роль в проекте

OCLint — это инструмент статического анализа для C, C++ и Objective-C, который фокусируется на улучшении качества и поддерживаемости кода. Он помогает выявлять проблемы, такие как высокая сложность, нарушения стандартов кодирования, "запахи кода" (code smells) и потенциальные ошибки.

**Разделение обязанностей:**
- **Качество и поддерживаемость кода:** OCLint в первую очередь нацелен на выявление участков кода, которые трудно читать, понимать и поддерживать. Это включает проверки на цикломатическую сложность, длину методов, количество параметров и т.д.
- **Обнаружение "запахов кода":** Ищет паттерны в коде, которые не являются ошибками, но могут указывать на проблемы в дизайне или реализации.
- **Дополнение к другим анализаторам:**
    - В отличие от Clang-Tidy, который также проверяет стиль и паттерны, OCLint может иметь другой набор правил и фокус, особенно на метриках кода.
    - Дополняет Cppcheck и Clang Static Analyzer, которые больше сосредоточены на поиске конкретных ошибок и уязвимостей, а не на общих аспектах качества кода.
- **Настраиваемые правила:** Позволяет определять собственные правила или настраивать существующие для соответствия специфическим требованиям проекта.

## 2. Конфигурация

OCLint настраивается в основном через параметры командной строки. Он также поддерживает конфигурацию пороговых значений для различных метрик и выбор правил.

**Основные опции командной строки:**
- `-R <path_to_rules>`: Загружает пользовательские правила из указанной директории (динамические библиотеки `.dylib` или `.so`).
- `-disable-rule <rule_name>`: Отключает определенное правило.
- `-rc <parameter>=<value>`: Изменяет параметр правила. Например, `-rc CYCLOMATIC_COMPLEXITY=15` устанавливает порог цикломатической сложности.
- `-report-type <type>`: Тип отчета. Возможные значения:
    - `text` (по умолчанию)
    - `html`
    - `xml`
    - `json`
    - `pmd`
- `-o <path>`: Путь для сохранения отчета.
- `-max-priority-1 <threshold>`: Максимальное количество нарушений с приоритетом 1 перед тем, как OCLint вернет ненулевой код выхода.
- `-max-priority-2 <threshold>`
- `-max-priority-3 <threshold>`
- `-- <compiler_options>`: Опции, передаваемые компилятору (Clang) для корректного парсинга кода. Это очень важно, так как OCLint использует Clang для анализа.

**Пример конфигурации порогов через командную строку:**
```bash
oclint -rc LONG_LINE=120 -rc NCSS_METHOD=50 ...
```
Полный список правил и их параметров можно найти в документации OCLint или с помощью команды `oclint -list-rules` (если такая опция поддерживается вашей версией).

**Файл `.oclint` (если поддерживается):**
Некоторые версии или интеграции OCLint могут поддерживать конфигурационный файл `.oclint` в формате YAML для определения правил и порогов, но основной способ — командная строка.

## 3. Запуск OCLint

OCLint, как и Clang-Tidy, требует информации о компиляции проекта, обычно через `compile_commands.json`.

**А. Вручную (используя `oclint-json-compilation-database`):**
OCLint поставляется с инструментом `oclint-json-compilation-database`, который читает `compile_commands.json` и запускает `oclint` для каждого файла.

**Шаг 1: Генерация `compile_commands.json`**
Убедитесь, что CMake генерирует `compile_commands.json`:
```cmake
set(CMAKE_EXPORT_COMPILE_COMMANDS ON)
```

**Шаг 2: Запуск анализа:**
```bash
# Предполагается, что compile_commands.json находится в директории 'build'
oclint-json-compilation-database -p build -- -report-type html -o oclint_report.html -max-priority-1 0 -max-priority-2 10
```
- `-p build`: Указывает путь к директории сборки, где находится `compile_commands.json`.
- Опции после `--` передаются `oclint`.
- `-max-priority-1 0`: Завершить с ошибкой, если есть хотя бы одно нарушение с приоритетом 1.

**Б. Запуск для отдельных файлов (требует указания всех опций компилятора):**
Это менее удобно для проектов, но возможно:
```bash
oclint path/to/source.cpp -report-type text -- -Iinclude -std=c++17 <другие_опции_компилятора>
```

**В. Интеграция с CMake:**
Можно добавить пользовательскую цель для запуска OCLint.
```cmake
# CMakeLists.txt (пример)
# Убедитесь, что compile_commands.json генерируется
set(CMAKE_EXPORT_COMPILE_COMMANDS ON)

# Найти oclint-json-compilation-database
find_program(OCLINT_JSON_DB_EXECUTABLE oclint-json-compilation-database)

if(OCLINT_JSON_DB_EXECUTABLE)
    add_custom_target(oclint
        COMMAND ${OCLINT_JSON_DB_EXECUTABLE}
            -p ${CMAKE_BINARY_DIR} # Путь к compile_commands.json
            --
            -report-type html -o ${CMAKE_BINARY_DIR}/oclint_report.html
            -max-priority-1 0 # Завершить с ошибкой при нарушениях P1
            # Добавьте другие опции oclint здесь, например, -rc или -disable-rule
        WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}
        COMMENT "Running OCLint static analysis..."
        VERBATIM
    )
else()
    message(WARNING "oclint-json-compilation-database not found. OCLint analysis will be skipped.")
endif()
```
Запуск: `make oclint`.

**Г. Интеграция с PyDoit (пример задачи в `dodo.py`):**
```python
# dodo.py (пример)
import os
from doit.tools import CmdAction

BUILD_DIR = "build" # Где находится compile_commands.json
OCLINT_JSON_DB_CMD = "oclint-json-compilation-database"
REPORT_FILE = "oclint_report.html"

# Исходные файлы проекта (для file_dep)
# PROJECT_FILES = [...] # Определите список файлов

def task_ensure_compile_commands():
    """Ensure compile_commands.json exists."""
    # ... (аналогично другим задачам, проверяющим compile_commands.json)
    pass

def task_oclint():
    """Run OCLint static analysis."""
    # Опции для oclint (после --)
    oclint_options = (
        f"-report-type html -o {REPORT_FILE} "
        f"-max-priority-1 0 " # Завершить с ошибкой при P1 нарушениях
        f"-max-priority-2 10 "
        f"-rc LONG_LINE=120 "
        # Добавьте другие правила и пороги: -rc CYCLOMATIC_COMPLEXITY=15 -disable-rule ShortVariableName
    )
    
    cmd = f"{OCLINT_JSON_DB_CMD} -p {BUILD_DIR} -- {oclint_options}"
    
    return {
        "actions": [CmdAction(cmd)],
        "file_dep": [os.path.join(BUILD_DIR, "compile_commands.json")], # + PROJECT_FILES
        "task_dep": ["ensure_compile_commands"],
        "targets": [REPORT_FILE],
        "verbosity": 2,
        "doc": "Run OCLint for code quality and maintainability checks."
    }
```
**Примечание по PyDoit:**
- Убедитесь, что `oclint` и `oclint-json-compilation-database` установлены и доступны в Docker-контейнере.
- `compile_commands.json` должен быть сгенерирован перед запуском этой задачи.
- Настройте `oclint_options` в соответствии с требованиями вашего проекта.

## 4. Интерпретация результатов

OCLint генерирует отчеты в выбранном формате (`text`, `html`, `xml`, `json`, `pmd`).
- **HTML-отчет:** Наиболее наглядный, показывает список нарушений с указанием файла, строки, правила и описания.
- **XML/JSON/PMD:** Удобны для интеграции с CI-системами (Jenkins, SonarQube, GitLab CI) и другими инструментами анализа.
- **Текстовый вывод:** Подходит для быстрой проверки в консоли.

OCLint присваивает каждому нарушению приоритет (1, 2 или 3, где 1 — самый высокий). Опции `-max-priority-X` позволяют настроить код возврата OCLint, что важно для CI: если количество нарушений определенного приоритета превышает порог, OCLint завершится с ошибкой, и CI-пайплайн будет помечен как неудачный.

## 5. Советы по использованию

- **Начните с набора правил по умолчанию:** Затем постепенно настраивайте, отключая ненужные правила или изменяя пороги (`-rc`).
- **Фокусируйтесь на приоритетах:** Сначала исправляйте нарушения с высоким приоритетом (P1).
- **Интегрируйте с CI:** Это ключевой момент для поддержания качества кода. Используйте `-max-priority-X` для автоматического фейла сборки при критических нарушениях.
- **Используйте `compile_commands.json`:** Это самый надежный способ предоставить OCLint информацию о сборке.
- **Настройте опции компилятора:** Убедитесь, что опции, передаваемые OCLint после `--`, соответствуют тем, с которыми собирается ваш проект (стандарт C++, пути включения, макросы).
- **Регулярно обновляйте OCLint:** Новые версии могут содержать улучшения, новые правила и исправления.
- **Создавайте собственные правила:** Если стандартных правил недостаточно, OCLint позволяет разрабатывать свои правила на C++ (требует сборки OCLint с вашими правилами).
- **Не стремитесь к нулю предупреждений сразу:** Особенно в легаси-коде. Определите реалистичные пороги и постепенно улучшайте код.
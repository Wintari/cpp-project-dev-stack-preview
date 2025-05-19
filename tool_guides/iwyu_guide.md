# Руководство по запуску Include-what-you-use (IWYU)

## 1. Назначение и роль в проекте

Include-what-you-use (IWYU) — это инструмент для анализа и оптимизации директив `#include` в C и C++ коде. Его основная цель — помочь разработчикам включать только те заголовочные файлы, которые действительно необходимы для компиляции каждого файла исходного кода. Это приводит к уменьшению времени компиляции, улучшению понимания зависимостей и снижению вероятности ненужных пересборок при изменениях в заголовочных файлах.

**Разделение обязанностей:**
- **Оптимизация `#include`:** IWYU фокусируется исключительно на анализе директив препроцессора `#include`.
- **Уменьшение зависимостей:** Помогает минимизировать зависимости между файлами, предлагая заменить полные включения на опережающие объявления (forward declarations), где это возможно.
- **Дополнение к другим анализаторам:** IWYU не ищет ошибки в коде или проблемы стиля, как Clang-Tidy или Cppcheck. Его задача — чисто техническая оптимизация процесса сборки и структуры зависимостей.

## 2. Конфигурация

IWYU в основном управляется через параметры командной строки. Он также использует информацию из `compile_commands.json` для понимания того, как компилируется код. Дополнительно, поведение IWYU можно настраивать с помощью специальных комментариев (pragmas) в исходном коде.

**Основные опции командной строки IWYU:**
- `-p <build_path>`: Указывает путь к директории сборки, содержащей `compile_commands.json`.
- `--verbose=<level>` (или `-v<level>`): Уровень детализации вывода (0-7).
- `--mapping_file=<file>`: Указывает файл маппинга символов к заголовочным файлам. IWYU поставляется с некоторыми стандартными маппингами (например, для STL, libc).
- `--check_also=<glob>`: Проверяет также файлы, соответствующие glob-паттерну (например, заголовочные файлы проекта).
- `--transitive_includes_only`: Сообщает только о ненужных транзитивных включениях.
- `--no_default_mappings`: Не использовать стандартные маппинги.
- `--max_line_length=<len>`: Максимальная длина строки для форматирования (используется `fix_includes.py`).
- Опции, передаваемые Clang, должны идти после `--`.

**IWYU Pragmas (комментарии в коде):**
IWYU позволяет управлять своим поведением для конкретных строк или файлов с помощью комментариев:
- `// IWYU pragma: keep`: Указывает IWYU сохранить данную директиву `#include`, даже если она кажется ненужной.
- `// IWYU pragma: no_include "foo.h"`: Запрещает IWYU предлагать включение `foo.h`.
- `// IWYU pragma: export`: Если этот комментарий находится в `foo.h`, и `bar.h` включает `foo.h`, то пользователи `bar.h` также получают доступ к символам из `foo.h` (полезно для umbrella headers).
- `// IWYU pragma: private, include "foo/bar.h"`: Указывает, что текущий файл является приватной частью `foo/bar.h`. Пользователи должны включать `foo/bar.h`, а не этот файл напрямую.

**Файлы маппинга (`.imp`):**
Эти файлы помогают IWYU понять, какой заголовочный файл предоставляет какой символ. Формат файла маппинга:
```
[
  { "symbol": ["MyClass", "private"], "header": "\"my_class.h\"", "map_to": ["MyClass", "public"] },
  { "symbol": ["ns::MyFunc", "public"], "header": "<my_utils/my_func.h>", "map_to": ["ns::MyFunc", "public"] }
]
```

## 3. Запуск IWYU

IWYU требует точных опций компиляции для каждого файла, поэтому его обычно запускают с использованием `compile_commands.json`.

**А. Использование `iwyu_tool.py` (рекомендуемый способ):**
IWYU поставляется со скриптом `iwyu_tool.py`, который читает `compile_commands.json` и запускает `include-what-you-use` для каждого файла.

**Шаг 1: Генерация `compile_commands.json`** (см. руководства по Clang-Tidy/Cppcheck).

**Шаг 2: Запуск `iwyu_tool.py`:**
```bash
# Путь к iwyu_tool.py и include-what-you-use может потребовать уточнения
# Предполагается, что compile_commands.json в 'build'
iwyu_tool.py -p build -j $(nproc) -- 
    --transitive_includes_only \
    --mapping_file=my_project.imp # Ваш файл маппинга, если есть
    # Другие опции IWYU
```
- `-j N`: Количество параллельных задач.
- Опции после `--` передаются непосредственно `include-what-you-use`.

**Б. Применение исправлений с `fix_includes.py`:**
Вывод `iwyu_tool.py` (или `include-what-you-use`) содержит рекомендации. Скрипт `fix_includes.py` (также поставляется с IWYU) может автоматически применить эти рекомендации.
```bash
# Запуск iwyu_tool.py и передача его вывода в fix_includes.py
iwyu_tool.py -p build --verbose=1 | fix_includes.py --nosafe_headers --reorder

# Или сначала сохранить вывод в файл:
iwyu_tool.py -p build > iwyu_suggestions.txt
fix_includes.py --nosafe_headers < iwyu_suggestions.txt
```
- `--nosafe_headers`: Позволяет `fix_includes.py` изменять любые заголовочные файлы (используйте с осторожностью).
- `--reorder`: Переупорядочивает `#include` в соответствии со стандартным стилем (например, Google C++ Style Guide).
- `--dry_run`: Показать изменения без их применения.

**В. Интеграция с CMake:**
Можно создать пользовательскую цель для запуска IWYU.
```cmake
# CMakeLists.txt (пример)
set(CMAKE_EXPORT_COMPILE_COMMANDS ON)

find_program(IWYU_TOOL_PY iwyu_tool.py)
find_program(FIX_INCLUDES_PY fix_includes.py)
find_program(IWYU_EXECUTABLE include-what-you-use)

if(IWYU_TOOL_PY AND FIX_INCLUDES_PY AND IWYU_EXECUTABLE)
    add_custom_target(iwyu
        COMMAND ${CMAKE_COMMAND} -E env IWYU_BINARY=${IWYU_EXECUTABLE} 
                ${IWYU_TOOL_PY} -p ${CMAKE_BINARY_DIR} -j $<NUMBER_OF_PROCESSORS> -- \
                --mapping_file=${CMAKE_CURRENT_SOURCE_DIR}/project.imp # Пример файла маппинга
                # Другие опции IWYU
        WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}
        COMMENT "Running Include-What-You-Use analysis..."
        VERBATIM
    )

    # Отдельная цель для применения исправлений (запускать вручную после iwyu)
    add_custom_target(iwyu-fix
        COMMAND ${CMAKE_COMMAND} -E env IWYU_BINARY=${IWYU_EXECUTABLE} 
                ${IWYU_TOOL_PY} -p ${CMAKE_BINARY_DIR} -j $<NUMBER_OF_PROCESSORS> -- \
                --mapping_file=${CMAKE_CURRENT_SOURCE_DIR}/project.imp \
                | ${FIX_INCLUDES_PY} --nosafe_headers --reorder
        WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}
        COMMENT "Applying IWYU suggestions... (Review changes carefully!)"
        VERBATIM
    )
    # add_dependencies(iwyu-fix iwyu) # Если нужно запускать последовательно
else()
    message(WARNING "IWYU tools not found. IWYU analysis will be skipped.")
endif()
```
Запуск: `make iwyu` и затем `make iwyu-fix` (или аналогичные команды).
**Примечание:** Передача `IWYU_BINARY` через `env` может быть необходима, если `iwyu_tool.py` не находит `include-what-you-use` автоматически.

**Г. Интеграция с PyDoit (пример задачи в `dodo.py`):**
```python
# dodo.py (пример)
import os
from doit.tools import CmdAction

BUILD_DIR = "build" # Где compile_commands.json
IWYU_TOOL_PY = "iwyu_tool.py" # Путь к скрипту
FIX_INCLUDES_PY = "fix_includes.py" # Путь к скрипту
IWYU_EXECUTABLE = "include-what-you-use" # Путь к исполняемому файлу

# PROJECT_FILES = [...] # Список исходных файлов для file_dep

def task_iwyu_analyze():
    """Run IWYU analysis and suggest changes."""
    # Опции для include-what-you-use (после --)
    iwyu_opts = (
        "--transitive_includes_only "
        # "--mapping_file=project.imp "
        "--verbose=1 "
    )
    
    # iwyu_tool.py может потребовать указания пути к include-what-you-use
    # через переменную окружения IWYU_BINARY или параметр командной строки, если он есть.
    # Здесь мы предполагаем, что iwyu_tool.py найдет его сам или IWYU_EXECUTABLE в PATH.
    cmd = f"IWYU_BINARY={IWYU_EXECUTABLE} {IWYU_TOOL_PY} -p {BUILD_DIR} -j $(nproc) -- {iwyu_opts}"
    
    return {
        "actions": [CmdAction(cmd, save_out='iwyu_suggestions')], # Сохраняем вывод в переменную
        "file_dep": [os.path.join(BUILD_DIR, "compile_commands.json")], # + PROJECT_FILES
        "uptodate": [False], # Всегда запускать для анализа
        "verbosity": 2,
        "doc": "Analyze #include directives with IWYU."
    }

def task_iwyu_fix():
    """Apply IWYU suggestions using fix_includes.py."""
    # Используем сохраненный вывод из task_iwyu_analyze
    # ВАЖНО: Это упрощенный пример. В doit передача вывода между задачами сложнее.
    # Лучше записать вывод iwyu_analyze в файл и читать его здесь.
    # Или объединить анализ и исправление в одну команду с пайпом.
    
    # Пример с пайпом (если doit поддерживает это напрямую или через shell):
    # cmd_analyze = f"IWYU_BINARY={IWYU_EXECUTABLE} {IWYU_TOOL_PY} -p {BUILD_DIR} -j $(nproc) -- {iwyu_opts}"
    # cmd_fix = f"{FIX_INCLUDES_PY} --nosafe_headers --reorder"
    # full_cmd = f"{cmd_analyze} | {cmd_fix}"

    # Пример чтения из файла (предполагается, что iwyu_analyze пишет в iwyu_suggestions.txt)
    suggestions_file = "iwyu_suggestions.txt"
    cmd_analyze_to_file = f"IWYU_BINARY={IWYU_EXECUTABLE} {IWYU_TOOL_PY} -p {BUILD_DIR} -j $(nproc) -- --verbose=1 > {suggestions_file}"
    cmd_fix_from_file = f"{FIX_INCLUDES_PY} --nosafe_headers --reorder < {suggestions_file}"

    return {
        "actions": [
            CmdAction(cmd_analyze_to_file), # Сначала генерируем файл
            CmdAction(cmd_fix_from_file)    # Затем применяем исправления
        ],
        "file_dep": [os.path.join(BUILD_DIR, "compile_commands.json")], # + PROJECT_FILES
        "task_dep": ["ensure_compile_commands"], # Убедиться, что compile_commands.json есть
        "verbosity": 2,
        "doc": "Apply IWYU suggestions. REVIEW CHANGES CAREFULLY!"
    }

```
**Примечание по PyDoit:** Убедитесь, что все инструменты IWYU (`iwyu_tool.py`, `fix_includes.py`, `include-what-you-use`) установлены и доступны в Docker-контейнере. Применение исправлений — деструктивная операция, поэтому ее следует выполнять с осторожностью и проверять изменения.

## 4. Интерпретация результатов

IWYU выводит список рекомендаций для каждого проанализированного файла. Он указывает:
- Какие `#include` следует удалить.
- Какие `#include` следует добавить.
- Где можно использовать опережающее объявление вместо полного `#include`.

Пример вывода:
```
path/to/file.cpp should add these lines:
#include "needed_header.h"  // for NeededClass

path/to/file.cpp should remove these lines:
- #include "unneeded_header.h"  // lines 10-10

The full include-list for path/to/file.cpp:
#include "needed_header.h"  // for NeededClass
#include "another_header.h" // for AnotherSymbol
--- PCH --- 
// (если используется PCH)
```
Скрипт `fix_includes.py` читает этот вывод и модифицирует файлы.

## 5. Советы по использованию

- **Начните с малого:** Применяйте IWYU сначала к небольшому подмножеству файлов или с опцией `--transitive_includes_only`, чтобы не быть перегруженным изменениями.
- **Используйте файлы маппинга:** Для библиотек STL, Boost и других стандартных библиотек IWYU обычно имеет встроенные маппинги. Для вашего собственного кода или сторонних библиотек без маппингов может потребоваться создать `.imp` файлы.
- **Проверяйте изменения:** Всегда тщательно проверяйте изменения, вносимые `fix_includes.py`, особенно если используется `--nosafe_headers`. Автоматические изменения могут иногда приводить к ошибкам компиляции.
- **Используйте прагмы IWYU:** Для сложных случаев или для сохранения специфических `#include` используйте `// IWYU pragma: keep` и другие прагмы.
- **Интегрируйте в CI (для анализа):** Можно добавить шаг анализа IWYU в CI, чтобы отслеживать состояние `#include` директив. Автоматическое применение исправлений в CI обычно не рекомендуется без тщательного тестирования.
- **Регулярно запускайте:** Поддерживайте чистоту `#include` директив, запуская IWYU периодически.
- **Понимайте ограничения:** IWYU не идеален и может давать ложные срабатывания или пропускать некоторые оптимизации, особенно в сложном коде с макросами или шаблонами.
- **`--no_default_mappings` и `--check_also`:** Эти опции могут быть полезны для точной настройки анализа вашего проекта, особенно если вы хотите анализировать только свои заголовочные файлы и не полагаться на стандартные маппинги.
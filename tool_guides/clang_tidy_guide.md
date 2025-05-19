# Руководство по запуску Clang-Tidy

## 1. Назначение и роль в проекте

Clang-Tidy - это инструмент статического анализа на основе LibTooling для C++. Его основная задача в данном проекте — выявление и исправление типичных ошибок программирования, проверка соответствия стандартам кодирования и обнаружение потенциальных проблем, связанных со стилем кода и распространенными паттернами ошибок.

**Разделение обязанностей:**
В соответствии с общей стратегией анализа кода, Clang-Tidy фокусируется на:
- Обеспечении единообразия стиля кодирования (если Clang-Format не используется для этого с той же строгостью или для специфичных проверок стиля, которые Clang-Format не покрывает).
- Обнаружении ошибок, которые могут быть не замечены компилятором, но связаны с логикой, использованием API и современными практиками C++.
- Автоматическом исправлении некоторых найденных проблем.

Он дополняет другие статические анализаторы, такие как Cppcheck (который больше сосредоточен на ошибках, которые компиляторы обычно не находят) и Clang Static Analyzer (для более глубокого анализа).

## 2. Конфигурация

Clang-Tidy управляется через конфигурационный файл `.clang-tidy`. Этот файл должен находиться в корневой директории проекта или в одной из родительских директорий анализируемых файлов. Если файл `.clang-tidy` не найден, Clang-Tidy будет использовать свои проверки по умолчанию.

**Пример `.clang-tidy` файла:**
```yaml
Checks: >
  -*,
  bugprone-*,
  cert-*,
  cppcoreguidelines-*,
  modernize-*,
  performance-*,
  portability-*,
  readability-*

# Пример включения конкретных проверок из группы, если она была отключена выше, или для большей ясности:
# modernize-use-nullptr,
# readability-identifier-naming

WarningsAsErrors: "*" # Трактовать все предупреждения как ошибки (рекомендуется для CI)

HeaderFilterRegex: '.*' # Анализировать все заголовочные файлы. Рекомендуется сузить до заголовочных файлов вашего проекта, например: '^(путь/к/вашим/исходникам|путь/к/вашим/включениям)/.*'

AnalyzeTemporaryDtors: true

# Настройки для конкретных проверок, например, для именования:
CheckOptions:
  - key:             readability-identifier-naming.ClassCase
    value:           CamelCase
  - key:             readability-identifier-naming.StructCase
    value:           camelBack
  - key:             readability-identifier-naming.VariableCase
    value:           camelBack
  # ... и другие стили
```
**Примечание:** Список проверок (`Checks`) и их параметры (`CheckOptions`) следует тщательно адаптировать под нужды и стандарты кодирования вашего проекта. Полный список доступных проверок можно найти в официальной документации Clang-Tidy.

## 3. Запуск Clang-Tidy

Clang-Tidy требует информации о том, как компилируется ваш проект. Обычно это предоставляется через базу данных компиляции (`compile_commands.json`), которую генерирует CMake.

**Шаг 1: Генерация `compile_commands.json`**
Убедитесь, что в вашей конфигурации CMake (в `CMakeLists.txt`) установлен флаг для генерации `compile_commands.json`:
```cmake
set(CMAKE_EXPORT_COMPILE_COMMANDS ON)
```
После этого запустите CMake для генерации файлов сборки. Файл `compile_commands.json` появится в вашей директории сборки (например, `build/`).

**Шаг 2: Запуск Clang-Tidy**

**А. Вручную (для одного файла):**
```bash
# Предполагается, что вы находитесь в директории, где лежит compile_commands.json
# или указываете путь к ней через -p
# Например, если compile_commands.json в директории 'build':
clang-tidy -p build path/to/your/sourcefile.cpp
```

**Б. Для всего проекта (используя `run-clang-tidy.py` или аналогичный скрипт):**
Clang поставляется со скриптом `run-clang-tidy.py` (или `run-clang-tidy`), который может запускать `clang-tidy` для всех файлов, указанных в `compile_commands.json`.
```bash
# Убедитесь, что run-clang-tidy.py доступен в вашем PATH
# или укажите полный путь к нему. Путь к директории сборки также важен.
run-clang-tidy.py -p build -header-filter='.*' -fix -j $(nproc) # Для Linux
# Для Windows можно использовать -j %NUMBER_OF_PROCESSORS% или указать число ядер
# run-clang-tidy.py -p build -header-filter='.*' -fix -j 4
```
- `-p build`: Указывает на директорию сборки с `compile_commands.json`.
- `-header-filter='.*'`: Применяет проверки ко всем заголовочным файлам. Настройте для анализа только ваших заголовочных файлов (например, `-header-filter='^(src|include)/.*'`).
- `-fix`: Автоматически применяет исправления (используйте с осторожностью, сначала просмотрите изменения или используйте `-export-fixes=fixes.yaml` для сохранения исправлений в файл).
- `-j N`: Запускает анализ в `N` потоков.

**В. Интеграция с PyDoit (пример задачи в `dodo.py`):**
```python
# dodo.py (пример)
import os
from doit.tools import LongRunning

# Путь к директории сборки, где находится compile_commands.json
BUILD_DIR = "build"
# Путь к исполняемому файлу run-clang-tidy (может отличаться в вашей системе)
# Возможно, потребуется указать полный путь или убедиться, что он в PATH Docker-контейнера
RUN_CLANG_TIDY_PY = "run-clang-tidy" # или run-clang-tidy.py

# Исходные файлы и заголовочные файлы проекта (примерный способ получения)
# В реальном проекте это лучше получать динамически или из CMake
SOURCE_FILES = ["src/main.cpp"] # Добавьте ваши файлы
HEADER_FILES = ["include/my_header.h"] # Добавьте ваши файлы
PROJECT_FILES = SOURCE_FILES + HEADER_FILES

def task_ensure_compile_commands():
    """Ensure compile_commands.json exists."""
    compile_commands_path = os.path.join(BUILD_DIR, "compile_commands.json")
    return {
        "actions": [
            # Эта команда должна запускать CMake, если compile_commands.json отсутствует
            # Пример: f"cmake -B {BUILD_DIR} -S . -DCMAKE_EXPORT_COMPILE_COMMANDS=ON"
            # Здесь мы просто выводим сообщение, если файл не найден
            (lambda: print(f"Please generate {compile_commands_path} using CMake first.") if not os.path.exists(compile_commands_path) else None)
        ],
        "targets": [compile_commands_path],
        "uptodate": [os.path.exists(compile_commands_path)],
        "verbosity": 2,
    }

def task_clang_tidy():
    """Run clang-tidy static analysis."""
    compile_commands_path = os.path.join(BUILD_DIR, "compile_commands.json")
    # Фильтр для заголовочных файлов вашего проекта
    header_filter = "'-header-filter=^(src|include)/.*'" # Одинарные кавычки важны для shell
    
    # Команда для CI может включать -export-fixes=clang_tidy_fixes.yaml
    # и --warnings-as-errors='*'
    cmd = f"{RUN_CLANG_TIDY_PY} -p {BUILD_DIR} {header_filter} --quiet"
    
    return {
        "actions": [LongRunning(cmd)],
        "file_dep": PROJECT_FILES + [compile_commands_path, ".clang-tidy"], # Зависимости
        "task_dep": ["ensure_compile_commands"], # Зависит от наличия compile_commands.json
        "verbosity": 2,
    }
```
**Примечание по PyDoit:** Приведенный пример является упрощенным. Для полноценной интеграции потребуется более точное определение зависимостей (`file_dep`), особенно списка исходных и заголовочных файлов, и корректная обработка путей в Docker-контейнере.

## 4. Интерпретация результатов

Clang-Tidy выводит предупреждения и ошибки с указанием файла, строки и описанием проблемы. Если используется флаг `-fix`, он может применить исправления напрямую или создать YAML-файл с исправлениями (`-export-fixes=fixes.yaml`), который затем можно применить с помощью `clang-apply-replacements`.

В CI/CD пайплайне вывод Clang-Tidy (обычно с опцией `--warnings-as-errors='*'`) анализируется для принятия решения о прохождении этапа контроля качества. Ненулевой код возврата от `run-clang-tidy.py` обычно означает наличие проблем.

## 5. Советы по использованию

- **Начните с малого:** Если вы интегрируете Clang-Tidy в существующий проект, начните с небольшого набора проверок (например, `readability-*`, `bugprone-*`) и постепенно расширяйте его, исправляя найденные проблемы.
- **Конфигурационный файл:** Всегда используйте файл `.clang-tidy` для управления проверками. Это обеспечивает согласованность и версионирование конфигурации.
- **Интеграция с CI/CD:** Обязательно интегрируйте Clang-Tidy в ваш CI/CD процесс для автоматического контроля качества кода.
- **Регулярные обновления:** Clang-Tidy активно развивается, регулярно обновляйте его и конфигурацию, так как появляются новые проверки и улучшаются существующие.
- **Используйте `-explain-config`:** Эта опция помогает понять, какие проверки активны для данного файла и откуда они берутся (из командной строки, файла `.clang-tidy` и т.д.).
  ```bash
  clang-tidy --explain-config -p build path/to/your/sourcefile.cpp
  ```
- **Автоматические исправления:** Используйте `-fix` или `-export-fixes` с осторожностью, особенно на больших кодовых базах. Всегда проверяйте изменения перед их коммитом.
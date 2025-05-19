# Руководство по запуску Cppcheck

## 1. Назначение и роль в проекте

Cppcheck — это инструмент статического анализа для C/C++. Его основная цель — обнаружение типов ошибок, которые компиляторы обычно не выявляют. В данном проекте Cppcheck используется для раннего обнаружения потенциальных проблем, таких как утечки памяти (на статическом уровне), переполнения буфера и использование неинициализированных переменных.

**Разделение обязанностей:**
- **Фокус на ошибках, пропускаемых компилятором:** Cppcheck специализируется на поиске ошибок, которые не являются синтаксическими, но могут привести к неопределенному поведению или уязвимостям.
- **Дополнение к другим анализаторам:**
    - **Потенциальные утечки памяти:** Cppcheck выполняет статический анализ на предмет утечек, который дополняется более точным динамическим анализом Valgrind (Memcheck).
    - **Использование неинициализированных переменных:** Дополняет динамический анализ MemorySanitizer (MSan).
    - **Переполнение буфера:** Дополняет динамический анализ AddressSanitizer (ASan).
- Cppcheck не фокусируется на стиле кода так, как Clang-Tidy или Clang-Format.

## 2. Конфигурация

Cppcheck можно настраивать с помощью различных флагов командной строки. Для более сложных конфигураций или для интеграции в систему сборки можно использовать файлы проекта Cppcheck (`.cppcheck`) или GUI-инструмент (`cppcheck-gui`).

**Основные опции командной строки:**
- `--enable=<check>`: Включает определенные типы проверок. Например, `all` (все проверки), `warning`, `style`, `performance`, `portability`, `information`, `unusedFunction`.
- `--std=<standard>`: Устанавливает стандарт C++. Например, `c++11`, `c++14`, `c++17`, `c++20`.
- `--platform=<platform>`: Указывает целевую платформу (например, `unix64`, `win64`). Это помогает Cppcheck лучше понимать размеры типов данных и специфичные для платформы API.
- `--suppress=<id>`: Подавляет определенное предупреждение по его ID или категории.
- `--suppressions-list=<file>`: Загружает список подавлений из файла.
- `--inline-suppr`: Разрешает подавление предупреждений непосредственно в коде с помощью комментариев вида `// cppcheck-suppress anId`.
- `--library=<file.cfg>`: Загружает конфигурационный файл для специфичной библиотеки (например, Qt, Boost). Cppcheck поставляется с набором таких файлов.
- `-I <dir>`: Добавляет директорию для поиска заголовочных файлов.
- `-D<symbol>`: Определяет препроцессорный символ.
- `--xml`: Вывод результатов в формате XML (полезно для CI).
- `--output-file=<file>`: Записывает вывод в файл.

**Пример конфигурационного файла для подавлений (`suppressions.txt`):**
```
// Подавить конкретное предупреждение по ID для всех файлов
*:unmatchedSuppression

// Подавить предупреждение о неиспользуемой функции в конкретном файле
src/legacy_code.cpp:unusedFunction

// Подавить все предупреждения о стиле в тестовых файлах
tests/*:style
```

## 3. Запуск Cppcheck

**А. Вручную (для одного файла или директории):**
```bash
# Анализ одного файла с включением всех проверок и указанием стандарта C++17
cppcheck --enable=all --std=c++17 path/to/your/sourcefile.cpp

# Анализ всех файлов в директории src, включая поддиректории, с выводом в XML
cppcheck --enable=all --std=c++17 --xml --output-file=cppcheck_report.xml src/

# Анализ с использованием файла подавлений и конфигурации для Qt
cppcheck --enable=warning,performance --std=c++17 --suppressions-list=suppressions.txt --library=qt.cfg src/
```

**Б. Интеграция с CMake:**
Cppcheck можно интегрировать в процесс сборки CMake. Обычно это делается путем добавления пользовательской цели (custom target).

Пример добавления цели в `CMakeLists.txt`:
```cmake
find_package(Cppcheck)

if(CPPCHECK_FOUND)
    add_custom_target(cppcheck
        COMMAND ${CPPCHECK_EXECUTABLE}
            --enable=all
            --std=c++17 # Укажите ваш стандарт
            # --platform=unix64 # или win64, в зависимости от вашей среды Docker
            --inline-suppr
            # --suppressions-list=${CMAKE_CURRENT_SOURCE_DIR}/cppcheck_suppressions.txt # Если есть файл подавлений
            --xml
            --output-file=${CMAKE_BINARY_DIR}/cppcheck_report.xml
            ${CMAKE_SOURCE_DIR} # Анализировать все исходники проекта
        WORKING_DIRECTORY ${CMAKE_BINARY_DIR}
        COMMENT "Running Cppcheck static analysis..."
        VERBATIM
    )
    # Можно добавить зависимость, чтобы cppcheck запускался, например, после сборки
    # add_dependencies(my_target cppcheck)
else()
    message(WARNING "Cppcheck not found. Static analysis with Cppcheck will be skipped.")
endif()
```
После этого вы можете запустить анализ командой `make cppcheck` (или аналогичной для вашей системы сборки).

**В. Интеграция с PyDoit (пример задачи в `dodo.py`):**
```python
# dodo.py (пример)
import os
from doit.tools import CmdAction

# Путь к исполняемому файлу Cppcheck (может отличаться в вашей системе)
CPPCHECK_CMD = "cppcheck"

# Директории с исходным кодом
SOURCE_DIRS = ["src", "include"] # Добавьте ваши директории

# Файлы проекта (примерный способ получения)
# В реальном проекте это лучше получать динамически
PROJECT_FILES = []
for src_dir in SOURCE_DIRS:
    for root, _, files in os.walk(src_dir):
        for file in files:
            if file.endswith(('.cpp', '.h', '.hpp')):
                PROJECT_FILES.append(os.path.join(root, file))

def task_cppcheck():
    """Run Cppcheck static analysis."""
    # Формируем строку с путями к исходным файлам/директориям
    paths_to_check = " ".join(SOURCE_DIRS)
    output_file = "cppcheck_report.xml"
    
    # Пример команды. Настройте флаги под ваш проект.
    # --error-exitcode=1 важен для CI, чтобы задача завершалась с ошибкой при наличии проблем
    cmd = (
        f"{CPPCHECK_CMD} "
        f"--enable=warning,performance,portability " # Выберите нужные проверки
        f"--std=c++17 " # Укажите ваш стандарт
        f"--platform=unix64 " # или win64
        f"--inline-suppr "
        # f"--suppressions-list=cppcheck_suppressions.txt " # Если есть файл подавлений
        f"--xml --xml-version=2 " # XML версии 2 для лучшей интеграции с CI инструментами
        f"--output-file={output_file} "
        f"--error-exitcode=1 " # Завершить с ошибкой, если найдены проблемы
        f"{paths_to_check}"
    )
    
    return {
        "actions": [CmdAction(cmd)],
        "file_dep": PROJECT_FILES, # Зависит от всех исходных файлов
        # "targets": [output_file], # Файл отчета
        "verbosity": 2,
        "doc": "Run Cppcheck to find potential bugs."
    }

```
**Примечание по PyDoit:** Убедитесь, что Cppcheck установлен в вашем Docker-контейнере и доступен по пути, указанному в `CPPCHECK_CMD`. Флаг `--error-exitcode=1` важен для CI, чтобы задача завершалась с ошибкой, если Cppcheck находит проблемы.

## 4. Интерпретация результатов

Cppcheck выводит информацию о найденных проблемах, указывая файл, строку и описание. Если используется опция `--xml`, результаты сохраняются в XML-файл, который может быть использован для генерации отчетов или интеграции с CI-системами (например, Jenkins, GitLab CI).

Пример вывода в консоль:
```
[src/main.cpp:10]: (error) Array 'arr[5]' accessed at index 10, which is out of bounds.
[src/utils.cpp:25]: (warning) Variable 'x' is assigned a value that is never used.
[include/my_class.h:5]: (style) struct 'MyClass' has a constructor 'MyClass()' but no destructor
```

В CI/CD пайплайне ненулевой код возврата от Cppcheck (если используется `--error-exitcode`) сигнализирует о наличии проблем и может приводить к остановке сборки/развертывания.

## 5. Советы по использованию

- **Начните с основных проверок:** Включите сначала `warning` и `performance`. Постепенно добавляйте `portability` и `style`, если это соответствует вашим целям.
- **Используйте `--platform`:** Это помогает Cppcheck давать более точные результаты, особенно для кросс-платформенных проектов.
- **Настройте подавления:** Для легаси-кода или специфических ситуаций используйте файлы подавлений или inline-подавления, чтобы скрыть ложные срабатывания или некритичные предупреждения. Но делайте это осознанно.
- **Интегрируйте с CI:** Автоматический запуск Cppcheck в CI-пайплайне — ключ к поддержанию качества кода.
- **Регулярно обновляйте Cppcheck:** Новые версии содержат улучшения и новые проверки.
- **Используйте `--check-config`:** Эта опция проверяет конфигурацию Cppcheck и может помочь выявить проблемы в настройках или нераспознанные файлы.
- **Комбинируйте с другими инструментами:** Помните, что Cppcheck — это один из инструментов в вашем арсенале. Его сила в дополнении к проверкам компилятора и другим анализаторам, таким как Clang-Tidy и динамические анализаторы.
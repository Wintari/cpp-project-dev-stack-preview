# Руководство по запуску Valgrind

## 1. Назначение и роль в проекте

Valgrind — это набор инструментов для динамического анализа программ. Наиболее известным и часто используемым инструментом из этого набора является **Memcheck**, который предназначен для обнаружения ошибок, связанных с управлением памятью (утечки памяти, использование неинициализированной памяти, некорректное освобождение памяти, выход за границы массива и т.д.). Другие инструменты Valgrind включают Helgrind/DRD (обнаружение гонок данных), Callgrind/Cachegrind (профилирование производительности).

**Разделение обязанностей:**
- **Точное обнаружение утечек памяти (Memcheck):** Это основная роль Valgrind/Memcheck в проекте. Он предоставляет детальную информацию об утечках, что дополняет статический анализ Cppcheck (который может находить лишь *потенциальные* утечки).
- **Профилирование производительности (Callgrind/Cachegrind):** Используется для выявления узких мест в производительности приложения.
- **Обнаружение гонок данных (Helgrind/DRD):** Альтернатива или дополнение к ThreadSanitizer (TSan) для поиска проблем в многопоточном коде.
- **Динамический анализ:** Valgrind выполняет анализ во время выполнения программы, что позволяет находить ошибки, которые трудно или невозможно обнаружить статически.

**Важно:** Valgrind значительно замедляет выполнение программы (в 10-50 раз), поэтому его обычно используют на этапах тестирования и отладки, а не в продакшене.

## 2. Конфигурация (Инструмент Memcheck)

Memcheck, как и другие инструменты Valgrind, управляется через параметры командной строки. Специальных конфигурационных файлов обычно не требуется, но поведение можно тонко настроить.

**Основные опции командной строки для Memcheck:**
- `valgrind --tool=memcheck <valgrind_options> ./your_program <program_options>`: Базовый синтаксис.
- `--leak-check=yes|summary|full`: Определяет, как проверять утечки памяти.
    - `yes` или `summary`: Краткий отчет об утечках.
    - `full`: Детальный отчет с трассировками стека для каждого блока утечки (рекомендуется).
- `--show-leak-kinds=all|definite|indirect|possible|reachable`: Какие типы утечек показывать.
    - `all`: Показывает все типы.
    - `definite,indirect`: Обычно наиболее интересные.
- `--track-origins=yes`: Помогает отслеживать источники неинициализированных значений (замедляет еще больше).
- `--log-file=<filename>`: Записывает вывод Valgrind в файл.
- `--xml=yes --xml-file=<filename.xml>`: Вывод в формате XML (полезно для CI и автоматической обработки).
- `--gen-suppressions=yes|all|no`: Генерирует подавления для ошибок, которые вы считаете ложными срабатываниями или приемлемыми. Эти подавления можно сохранить в файл и использовать с опцией `--suppressions=<file>`.
- `--suppressions=<file>`: Использует файл подавлений для игнорирования определенных ошибок.
- `--error-exitcode=<number>`: Устанавливает код выхода, если Valgrind обнаружит ошибки (по умолчанию 0, установите ненулевое значение для CI).
- `--keep-debuginfo=yes`: Помогает Valgrind лучше работать с отладочной информацией.

**Пример файла подавлений (`valgrind.supp`):**
```
{
   <a_suppression_name>
   Memcheck:Leak
   fun:some_function_in_library_causing_issues
   ...
}
{
   <another_suppression>
   Memcheck:Value8
   obj:/path/to/library.so
   fun:another_problematic_function
}
```

## 3. Запуск Valgrind (Memcheck)

Valgrind запускает вашу программу в своей среде эмуляции.

**А. Вручную:**
```bash
# Убедитесь, что ваша программа скомпилирована с отладочной информацией (-g)

# Базовый запуск с проверкой утечек
valgrind --tool=memcheck --leak-check=full ./my_application arg1 arg2

# Запуск с отслеживанием источников неинициализированных значений и записью в лог
valgrind --tool=memcheck --leak-check=full --track-origins=yes --log-file=valgrind_report.txt ./my_application

# Запуск с генерацией XML-отчета и ненулевым кодом выхода при ошибках
valgrind --tool=memcheck --leak-check=full --xml=yes --xml-file=valgrind_report.xml --error-exitcode=1 ./my_application

# Использование файла подавлений
valgrind --tool=memcheck --leak-check=full --suppressions=my_valgrind.supp ./my_application
```

**Б. Интеграция с CMake (через CTest):**
Valgrind можно интегрировать с системой тестирования CTest в CMake. Это позволяет автоматически запускать тесты под Valgrind.

В `CMakeLists.txt`:
```cmake
find_program(MEMORYCHECK_COMMAND valgrind)
if(MEMORYCHECK_COMMAND)
    # Установка команды для проверки памяти по умолчанию для CTest
    # Эта команда будет добавлена перед каждой командой теста
    set(CTEST_MEMORYCHECK_COMMAND "${MEMORYCHECK_COMMAND}")
    # Опции для Valgrind/Memcheck
    set(CTEST_MEMORYCHECK_COMMAND_OPTIONS
        "--tool=memcheck;--leak-check=full;--show-leak-kinds=all;--error-exitcode=1;--xml=yes;--xml-file=\"${CMAKE_BINARY_DIR}/CTestMemcheck-\${CTEST_TEST_NAME}.xml\""
    )
    # Тип проверки памяти (например, Valgrind)
    set(CTEST_MEMORYCHECK_TYPE "Valgrind")
    # Включить проверку памяти по умолчанию
    # set(CTEST_MEMORYCHECK_SUPPRESSIONS_FILE "${CMAKE_CURRENT_SOURCE_DIR}/valgrind.supp") # Если есть файл подавлений
    
    # Включить возможность запуска тестов с проверкой памяти
    # Это добавит опцию -T memcheck к ctest
    include(CTest)
    enable_testing()
    
    # Пример добавления теста
    add_executable(my_test_program test_main.cpp)
    add_test(NAME MyTestProgram COMMAND my_test_program)
else()
    message(WARNING "Valgrind not found. Memory checking will be disabled.")
endif()
```
Запуск тестов под Valgrind:
```bash
cd build
ctest -T memcheck
# или
# make Experimental # (или test, memcheck, в зависимости от генератора и версии CMake)
```
Результаты (включая XML-отчеты) будут в директории сборки (например, `build/Testing/Temporary/`).

**В. Интеграция с PyDoit (пример задачи в `dodo.py`):**
```python
# dodo.py (пример)
from doit.tools import CmdAction

VALGRIND_CMD = "valgrind"
# Путь к вашей исполняемой программе (предполагается, что она собирается другой задачей)
EXECUTABLE_PATH = "./build/my_application" 
# Аргументы для вашей программы
PROGRAM_ARGS = "arg1 arg2"

# Зависимости: исполняемый файл и, возможно, тестовые данные
# FILE_DEPS = [EXECUTABLE_PATH, "test_data.txt"]

def task_valgrind_memcheck():
    """Run the application under Valgrind Memcheck."""
    log_file = "valgrind_memcheck_report.txt"
    xml_file = "valgrind_memcheck_report.xml"
    
    # Опции для Valgrind Memcheck
    # --error-exitcode=1 важен для CI
    valgrind_options = (
        f"--tool=memcheck "
        f"--leak-check=full "
        f"--show-leak-kinds=all "
        f"--track-origins=yes " # Может замедлить, но дает больше информации
        f"--log-file={log_file} "
        f"--xml=yes --xml-file={xml_file} "
        f"--error-exitcode=1 "
        # f"--suppressions=valgrind.supp " # Если есть файл подавлений
    )
    
    cmd = f"{VALGRIND_CMD} {valgrind_options} {EXECUTABLE_PATH} {PROGRAM_ARGS}"
    
    return {
        "actions": [CmdAction(cmd)],
        "file_dep": [EXECUTABLE_PATH], # Зависит от сборки исполняемого файла
        "targets": [log_file, xml_file], # Файлы отчетов
        "task_dep": ["build_my_application"], # Зависит от задачи сборки
        "verbosity": 2,
        "doc": "Run Memcheck for memory error detection and leak checking."
    }

```
**Примечание по PyDoit:**
- Убедитесь, что Valgrind установлен в вашем Docker-контейнере.
- Программа должна быть скомпилирована с отладочной информацией (`-g`).
- `EXECUTABLE_PATH` и `PROGRAM_ARGS` должны быть настроены для вашего приложения.
- Задача `build_my_application` должна существовать и собирать исполняемый файл.

## 4. Интерпретация результатов (Memcheck)

Valgrind выводит подробную информацию об ошибках:
- **Тип ошибки:** (например, `Invalid read`, `Invalid write`, `Conditional jump or move depends on uninitialised value(s)`, `Definitely lost`, `Possibly lost`).
- **Трассировка стека:** Показывает, где в коде произошла ошибка или где был выделен утекший блок памяти.
- **Размер и адрес блока памяти:** Для ошибок, связанных с памятью.

**Пример вывода (утечка памяти):**
```
==12345== HEAP SUMMARY:
==12345==     in use at exit: 72,704 bytes in 1 blocks
==12345==   total heap usage: 2 allocs, 1 frees, 73,728 bytes allocated
==12345== 
==12345== 72,704 bytes in 1 blocks are definitely lost in loss record 1 of 1
==12345==    at 0x483DD99: malloc (vg_replace_malloc.c:380)
==12345==    by 0x109233: allocate_buffer (main.c:25)
==12345==    by 0x1092A8: main (main.c:40)
```
Этот вывод показывает, что 72,704 байта были "определенно потеряны" (definitely lost), выделены функцией `malloc` (вызванной из `allocate_buffer` в `main.c:25`) и не освобождены.

- **XML-отчеты:** Могут быть проанализированы инструментами CI (например, Jenkins с Valgrind плагином) или другими скриптами для автоматической оценки.
- **Код возврата:** Если используется `--error-exitcode` с ненулевым значением, Valgrind вернет этот код при обнаружении ошибок, что позволяет CI-системам определить неудачное выполнение.

## 5. Советы по использованию (Memcheck)

- **Компилируйте с `-g`:** Всегда используйте отладочную информацию для получения точных номеров строк и имен функций в отчетах Valgrind.
- **Тестируйте небольшие участки:** Из-за замедления, лучше тестировать отдельные компоненты или сценарии, а не все приложение целиком за один раз (если оно большое).
- **Используйте `--leak-check=full`:** Для наиболее детальной информации об утечках.
- **Используйте `--track-origins=yes`:** Для отладки ошибок, связанных с неинициализированной памятью, хотя это и замедляет выполнение.
- **Создавайте файлы подавлений:** Для системных библиотек или стороннего кода, который вы не можете исправить, используйте подавления, чтобы сосредоточиться на проблемах в вашем коде. Генерируйте их с помощью `--gen-suppressions=yes`.
- **Интегрируйте с CI:** Запускайте Valgrind для ваших тестов в CI-пайплайне, используя `--error-exitcode` и, возможно, анализ XML-отчетов.
- **Не используйте оптимизации, скрывающие ошибки:** Иногда агрессивные оптимизации компилятора (`-O2`, `-O3`) могут изменять код так, что Valgrind не увидит некоторые ошибки. Для отладки с Valgrind иногда полезно снизить уровень оптимизации (например, до `-O1` или `-O0`).
- **Помните о других инструментах Valgrind:** Если вам нужно профилировать производительность, используйте `valgrind --tool=callgrind` (и `kcachegrind` для визуализации). Для проблем с потоками — `valgrind --tool=helgrind` или `valgrind --tool=drd`.
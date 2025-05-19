# Руководство по запуску Sanitizers (ASan, TSan, MSan, UBSan)

## 1. Назначение и роль в проекте

Sanitizers — это набор высокоэффективных инструментов для динамического анализа, встроенных в компиляторы Clang и GCC. Они предназначены для обнаружения различных типов ошибок во время выполнения программы с относительно низкими накладными расходами по сравнению с Valgrind. Каждый Sanitizer специализируется на определенном классе ошибок.

**Основные типы Sanitizers и их роли:**
- **AddressSanitizer (ASan):**
    - **Назначение:** Обнаружение ошибок доступа к памяти: использование после освобождения (use-after-free), переполнение буфера (heap, stack, global), использование после возврата (use-after-return), использование после выхода из области видимости (use-after-scope), двойное освобождение (double-free), некорректное освобождение (invalid-free).
    - **Роль в проекте:** Основной инструмент для динамического обнаружения ошибок памяти. Дополняет статический анализ Cppcheck и Clang Static Analyzer, а также может быть быстрее и менее ресурсоемок, чем Valgrind/Memcheck для некоторых сценариев.
- **ThreadSanitizer (TSan):**
    - **Назначение:** Обнаружение гонок данных (data races) и других проблем синхронизации в многопоточных приложениях (например, deadlock).
    - **Роль в проекте:** Основной инструмент для динамического обнаружения гонок данных. Дополняет Valgrind (Helgrind/DRD).
- **MemorySanitizer (MSan):**
    - **Назначение:** Обнаружение чтений из неинициализированной памяти.
    - **Роль в проекте:** Основной инструмент для динамического обнаружения использования неинициализированной памяти. Дополняет статический анализ Cppcheck и Valgrind/Memcheck (который также может находить такие проблемы, но MSan часто делает это эффективнее).
- **UndefinedBehaviorSanitizer (UBSan):**
    - **Назначение:** Обнаружение различных видов неопределенного поведения C++, которые не связаны напрямую с памятью или потоками. Примеры: целочисленное переполнение, сдвиги некорректной ширины, разыменование нулевого или невыровненного указателя, достижение конца функции, возвращающей значение, без оператора return, и т.д.
    - **Роль в проекте:** Общий инструмент для выявления широкого спектра проблем, связанных с неопределенным поведением.

**Разделение обязанностей:**
Sanitizers являются ключевыми инструментами динамического анализа, предоставляя быструю обратную связь о проблемах во время выполнения. Они дополняют статический анализ (который находит потенциальные проблемы до запуска) и Valgrind (который может быть более тщательным для некоторых проверок памяти, но значительно медленнее).

## 2. Конфигурация

Sanitizers включаются с помощью специальных флагов компилятора (и линковщика) при сборке программы. Обычно это флаг `-fsanitize=<sanitizer_name>`.

**Основные флаги компилятора (Clang/GCC):**
- **AddressSanitizer (ASan):**
    - `-fsanitize=address`
    - `-fno-omit-frame-pointer`: Рекомендуется для получения более качественных стектрейсов.
    - `-g`: Всегда компилируйте с отладочной информацией для лучших отчетов.
- **ThreadSanitizer (TSan):**
    - `-fsanitize=thread`
    - `-fPIE -pie` (для исполняемых файлов) или `-fPIC` (для библиотек): Часто требуется для TSan.
    - `-g`
- **MemorySanitizer (MSan):**
    - `-fsanitize=memory`
    - `-fno-omit-frame-pointer`
    - `-fsanitize-memory-track-origins=2`: Для отслеживания источников неинициализированных значений (увеличивает накладные расходы).
    - `-g`
    - **Важно:** Для MSan необходимо, чтобы все части программы (включая все библиотеки, с которыми она линкуется) были скомпилированы с MSan. Это может быть сложно для системных библиотек. Clang предоставляет специальные инструментированные версии стандартных библиотек.
- **UndefinedBehaviorSanitizer (UBSan):**
    - `-fsanitize=undefined`: Включает набор проверок UBSan по умолчанию.
    - Можно включать специфические проверки: `-fsanitize=integer,nullability,bounds,...`
    - `-fno-sanitize-recover=all` (или для конкретных проверок): Прекращает выполнение программы при первой ошибке UBSan. По умолчанию UBSan пытается продолжить выполнение.
    - `-fsanitize=float-divide-by-zero,float-cast-overflow` и т.д. для проверок с плавающей точкой.
    - `-g`

**Важно:**
- Обычно нельзя комбинировать ASan, TSan и MSan в одной сборке (например, нельзя одновременно использовать `-fsanitize=address` и `-fsanitize=thread`).
- UBSan можно комбинировать с ASan или TSan (но не всегда с MSan).
  Пример: `-fsanitize=address,undefined`

**Переменные окружения для Runtime-конфигурации:**
Поведение Sanitizers во время выполнения можно настраивать через переменные окружения. Общая переменная — `ASAN_OPTIONS`, `TSAN_OPTIONS`, `MSAN_OPTIONS`, `UBSAN_OPTIONS`.

- **ASAN_OPTIONS:**
    - `detect_leaks=1`: Включить детектор утечек (LeakSanitizer, LSan), который часто интегрирован с ASan. По умолчанию может быть включен.
    - `halt_on_error=1`: Останавливать программу при первой ошибке.
    - `log_path=/path/to/asan.log`: Путь для лог-файла.
    - `print_stacktrace=1`: Печатать стектрейс.
- **TSAN_OPTIONS:**
    - `report_bugs_to_file=1 log_path=/path/to/tsan.log`
    - `halt_on_error=1`
- **MSAN_OPTIONS:**
    - `halt_on_error=1`
- **UBSAN_OPTIONS:**
    - `print_stacktrace=1`
    - `halt_on_error=1` (если не используется `-fno-sanitize-recover`)
    - `log_path=/path/to/ubsan.log`

Пример: `ASAN_OPTIONS=detect_leaks=1:log_path=asan_log.txt ./my_program`

## 3. Запуск Sanitizers

**Шаг 1: Компиляция с флагами Sanitizer**
Необходимо добавить соответствующие флаги `-fsanitize=...` к командам компиляции и линковки.

**А. Интеграция с CMake:**
В `CMakeLists.txt`:
```cmake
# Общие флаги
set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -g -fno-omit-frame-pointer")
set(CMAKE_EXE_LINKER_FLAGS "${CMAKE_EXE_LINKER_FLAGS}") # Для линковщика

# Выбор Sanitizer (например, через опцию CMake)
# cmake -DENABLE_ASAN=ON ..
option(ENABLE_ASAN "Enable AddressSanitizer" OFF)
option(ENABLE_TSAN "Enable ThreadSanitizer" OFF)
option(ENABLE_MSAN "Enable MemorySanitizer" OFF)
option(ENABLE_UBSAN "Enable UndefinedBehaviorSanitizer" OFF)

if(ENABLE_ASAN)
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -fsanitize=address")
    set(CMAKE_EXE_LINKER_FLAGS "${CMAKE_EXE_LINKER_FLAGS} -fsanitize=address")
    # Для статической линковки библиотеки ASan (если нужно и доступно)
    # set(CMAKE_EXE_LINKER_FLAGS "${CMAKE_EXE_LINKER_FLAGS} -static-libasan") 
endif()

if(ENABLE_TSAN)
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -fsanitize=thread")
    set(CMAKE_EXE_LINKER_FLAGS "${CMAKE_EXE_LINKER_FLAGS} -fsanitize=thread")
    # TSan часто требует PIE для исполняемых файлов
    set(CMAKE_POSITION_INDEPENDENT_CODE ON) 
endif()

if(ENABLE_MSAN)
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -fsanitize=memory -fsanitize-memory-track-origins=2")
    set(CMAKE_EXE_LINKER_FLAGS "${CMAKE_EXE_LINKER_FLAGS} -fsanitize=memory")
    # MSan требует, чтобы все было скомпилировано с ним. Это может потребовать
    # использования специальной инструментированной стандартной библиотеки.
    # message(WARNING "MSan requires all linked code, including system libraries, to be instrumented.")
endif()

if(ENABLE_UBSAN)
    # Можно выбрать конкретные проверки или использовать 'undefined' для набора по умолчанию
    set(UBSAN_CHECKS "undefined") # Например: "integer,nullability,bounds"
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -fsanitize=${UBSAN_CHECKS} -fno-sanitize-recover=all")
    set(CMAKE_EXE_LINKER_FLAGS "${CMAKE_EXE_LINKER_FLAGS} -fsanitize=${UBSAN_CHECKS}")
endif()

# ... ваш код для добавления исполняемых файлов и тестов ...
add_executable(my_program main.cpp)
```
После этого соберите проект как обычно. CMake передаст флаги компилятору.

**Шаг 2: Запуск исполняемого файла**
Запустите скомпилированный исполняемый файл. Если Sanitizer обнаружит ошибку, он выведет отчет в stderr (или в лог-файл, если настроено) и, возможно, завершит программу.
```bash
# После сборки с CMake -DENABLE_ASAN=ON
./my_program <program_arguments>

# С runtime опциями
ASAN_OPTIONS=log_path=asan.log UBSAN_OPTIONS=print_stacktrace=1 ./my_program
```

**Б. Интеграция с PyDoit (пример задачи в `dodo.py`):**
PyDoit будет управлять процессом сборки с нужными флагами.
```python
# dodo.py (пример)

# Предполагается, что у вас есть задача сборки, например, 'compile_program'
# Эта задача должна принимать параметры для флагов Sanitizer

def task_compile_with_asan():
    """Compile the program with AddressSanitizer."""
    # Команда сборки (например, вызов CMake или g++)
    # Добавьте -fsanitize=address и -fno-omit-frame-pointer
    compile_cmd = "g++ -std=c++17 -g -fno-omit-frame-pointer -fsanitize=address src/main.cpp -o build/my_program_asan"
    return {
        "actions": [compile_cmd],
        "targets": ["build/my_program_asan"],
        "file_dep": ["src/main.cpp"],
        "doc": "Compile with ASan"
    }

def task_run_with_asan():
    """Run the ASan-instrumented program."""
    # Установка ASAN_OPTIONS перед запуском
    # В doit это можно сделать через CmdAction с env параметром или обернув в shell-скрипт
    run_cmd = "ASAN_OPTIONS='detect_leaks=1:halt_on_error=1' ./build/my_program_asan arg1"
    return {
        "actions": [run_cmd],
        "file_dep": ["build/my_program_asan"],
        "task_dep": ["compile_with_asan"],
        "doc": "Run with ASan"
    }

# Аналогичные задачи для TSan, MSan, UBSan
```
Это упрощенный пример. В реальном проекте вы, скорее всего, будете использовать CMake для сборки, а PyDoit будет вызывать CMake с соответствующими `-DENABLE_SANITIZER=ON` опциями.

## 4. Интерпретация результатов

Sanitizers предоставляют подробные отчеты об ошибках, обычно включающие:
- **Тип ошибки:** (например, `heap-buffer-overflow`, `stack-use-after-return`, `data race`, `use of uninitialized value`, `integer overflow`).
- **Трассировка стека:** Для потока, где произошла ошибка, а также для связанных событий (например, выделение/освобождение памяти для ASan, доступ к данным из других потоков для TSan).
- **Адреса памяти и смещения:** Для ошибок доступа к памяти.
- **Имена файлов и номера строк:** Если код скомпилирован с отладочной информацией (`-g`).

**Пример отчета ASan (переполнение буфера на куче):**
```
==12345==ERROR: AddressSanitizer: heap-buffer-overflow on address 0x602000000010 at pc 0x000000400a0c bp 0x7ffc00000f00 sp 0x7ffc00000ef8
WRITE of size 4 at 0x602000000010 thread T0
    #0 0x400a0b in main /path/to/project/main.cpp:10
    #1 0x7f0000000b97 in __libc_start_main (/lib/x86_64-linux-gnu/libc.so.6+0x21b97)
    #2 0x4008e9 in _start (/path/to/project/my_program+0x4008e9)

0x602000000010 is located 0 bytes to the right of 16-byte region [0x602000000000,0x602000000010)
allocated by thread T0 here:
    #0 0x400d0d in operator new(unsigned long) /path/to/asan_interceptors.cpp:402
    #1 0x4009a1 in main /path/to/project/main.cpp:8
    #2 0x7f0000000b97 in __libc_start_main (/lib/x86_64-linux-gnu/libc.so.6+0x21b97)
```
Отчет показывает тип ошибки, адрес, стек вызовов для операции записи, а также информацию о том, где был выделен блок памяти, к которому произошел доступ за его пределами.

Для CI/CD важно, чтобы Sanitizers завершали программу с ненулевым кодом выхода при обнаружении ошибки (обычно это поведение по умолчанию, или его можно настроить с `halt_on_error=1` или `-fno-sanitize-recover`).

## 5. Советы по использованию

- **Компилируйте с `-g`:** Всегда используйте отладочную информацию для получения максимально полезных отчетов.
- **Используйте `-fno-omit-frame-pointer`:** Для ASan и MSan это улучшает качество стектрейсов.
- **Тестируйте регулярно:** Включайте запуск тестов с Sanitizers в ваш регулярный процесс разработки и CI.
- **Один Sanitizer за раз (кроме UBSan):** Не пытайтесь скомбинировать ASan, TSan и MSan. UBSan обычно можно комбинировать с ASan или TSan.
- **LSan (LeakSanitizer):** Часто поставляется вместе с ASan и включается по умолчанию или через `ASAN_OPTIONS=detect_leaks=1`. Он обнаруживает утечки памяти в конце выполнения программы.
- **MSan требует полной инструментированности:** Это самый сложный в настройке Sanitizer, так как все зависимости, включая системные библиотеки, должны быть скомпилированы с MSan. Рассмотрите использование специальных сборок Clang с инструментированными библиотеками или Docker-образов, где это уже настроено.
- **Подавление ошибок:** Как и другие инструменты, Sanitizers могут давать ложные срабатывания или сообщать о проблемах в стороннем коде. Можно использовать файлы подавлений (например, `ASAN_OPTIONS=suppressions=my_asan.supp`) или специальные комментарии/атрибуты в коде для подавления ошибок.
    - `__attribute__((no_sanitize("address")))` для функций.
- **Символизация стектрейсов:** Если стектрейсы не символизированы (показывают только адреса), убедитесь, что отладочная информация присутствует и что у вас есть `llvm-symbolizer` (для Clang) в `PATH`.
- **Накладные расходы:** Sanitizers добавляют накладные расходы на производительность (ASan ~2x, TSan ~5-15x, MSan ~3x) и потребление памяти. Они предназначены для этапов разработки и тестирования.
- **Выбирайте правильный Sanitizer:**
    - Для ошибок памяти: ASan.
    - Для гонок данных: TSan.
    - Для неинициализированной памяти: MSan (если можете настроить) или ASan (некоторые проверки) / Valgrind.
    - Для прочего неопределенного поведения: UBSan.
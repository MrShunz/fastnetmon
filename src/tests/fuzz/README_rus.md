# Fuzzing

В этом разделе описан процесс фаззинг-тестирования, использованные подходы и методы, включая как успешные, так и неудачные попытки.

## Навигация
--------------------------------

- [Docker Image](#docker-image)
- [CMake](#cmake)
- [Файловая структура](#файловая-структура)
- [Пример запуска фаззинга](#пример-запуска-фаззинга-вручную-для-отдельных-фаззинг-оберток)
- [Другие техники фаззинга](#другие-техники-фаззинга)
- [Запуск фаззинга в контейнере](#запуск-фаззинга-в-docker-contianer)

## Docker Image
--------------------------------

Скрипт базируется на `tests/Dockerfile.ubuntu-24.04-afl++`.
В этом образе собирается и устанавливается все необходимое для дальнейшего тестирования:

|    Модуль                                   |  Описание                                                   |
|---------------------------------------------|-------------------------------------------------------------|
|   [`AFL++`](https://github.com/AFLplusplus) | Фаззер                                                      |
|   [`casr`](https://github.com/ispras/casr)  | Утилита для проверки, минимазации и кластеризации крашей. Нужен rust |
|   [`desock`](https://github.com/zardus/preeny/) | Утилита для подмены системных вызовов. Используется в проекте для замены вызова функции `socket`, которая принимает данные с интерфейса, на функцию, которая принимает данные из консоли |

Так же в проекте используются санитайзеры (ASAN, UBSAN и др.) для поиска ошибок в динамике.

### Build Docker Image
--------------------------------

Сборка Docker image для тестирования происходит из коренной директории `fastnetmon` и запускается командой: 
```bash
docker  build -f tests/Dockerfile.ubuntu-24.04-afl++ . -t fuzz
```
По завершению сборки буден получен `image` с названием `fuzz`

## Cmake
--------------------------------
В исходный файл `CmakeLists.txt` был добавлен ряд опций, которые позволяют собирать отдельные фаззинг-обертки используя различные опции cmake.
*В таблице опции будут указываться с приставкой `-D`, которая позволяет задать опцию, как аргумент утилиты cmake при запуске из командой строки*


|    Опция                           |  Описание                                                   |
|------------------------------------|-------------------------------------------------------------|
|  `-DENABLE_FUZZ_TEST`              | Собирает две фаззинг обертки под `AFL++`. Использовать **только с копилятором `afl-c++`** или его вариациями                                                      |
|  `-DENABLE_FUZZ_TEST_DESOCK`       | Данная опция позволяет реализоват изменение поведения стандартной функции `socket`. Теперь данные будут браться не с сетевого сокета, из потока ввода. **Инструментирует оригинальный исполняемый файл `fastnetmon`** |
|   `-DCMAKE_BUILD_TYPE=Debug`     | Отладочная опция, необходимая для корректной работы отладчиков. Внимание! **Не использовать на релизе и при тестах - будут ложные срабатывания санитайзера на таких функциях, как `assert()`



## Файловая структура
--------------------------------
```
fuzz/
├── README.md
├── README_rus.md        
├── fastnetmon.conf                              
├── parse_sflow_v5_packet_fuzz.cpp               
├── process_netflow_packet_v5_fuzz.cpp           
└──  scripts/                                    
│   ├── minimize_out.sh
│   ├── afl_pers_mod_instr.sh   
│   ├── start_fuzz_conf_file.sh                  
│   └── start_fuzz_harness.sh                    
```

### Описание файловов
--------------------------------

| Файл                                    | Описание                                                                                  |
|-----------------------------------------|------------------------------------------------------------------------------------------|
| `README.md`                             | Документация на **английском языке** о фаззинг-тестировании проекта.                                      |
| `README_rus.md`                         | Документация на **русском языке** о фаззинг-тестировании проекта.                                      |
| `fastnetmon.conf`                       | Конфигурационный файл для FastNetMon. Оставлены для работы только протоколы netflow и sflow             |
| `parse_sflow_v5_packet_fuzz.cpp`        | Обертка для фаззинга функции `parse_sflow_v5_packet_fuzz` через `AFL++`.                                   |
| `process_netflow_packet_v5_fuzz.cpp`    | Обертка для фаззинга функции `process_netflow_packet_v5_fuzz` через `AFL++`.                                                                       |


| Файл/Директория                         | Описание                                                                                         | Запуск                                                                                              |
|-----------------------------------------|--------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------|
| `/scripts/`                             | Директория со скриптами для автоматизации фаззинга.                                              |                                                                                                     |
| `/scripts/minimize_out.sh`              | Скрипт для минимизации, верификации и кластеризации выходных данных (падений).                   | `./minimize_out.sh <path_to_out_directory> <./binary>`                                              |
| `/scripts/start_fuzz_conf_file.sh`      | Скрипт для запуска фаззинга конфигурационных файлов. Запускает несколько сессий tmux.   Использует опции `./fastnetmon --configuration_check --configuration_file`.         | Запускать из текущей директории без дополнительных опций. Окружение создаётся автоматически.         |
| `/scripts/start_fuzz_harness.sh`        | Скрипт для тестирования бинарных файлов, скомпилированных из обёрток в отдельные исполняемые файлы. Предназначен для обёрток, скомпилированных под `AFL++`.    При запуске настраивает окружение, создаёт папки, запускает две сессии tmux с экземплярами фаззера. После окончания фаззинга запускает скрипт `minimize_out.sh` для кластеризации крашей.                                        | `./start_fuzz_harness.sh <path/to/bin>`    Скрипт завершится, если не будет найдено новых путей в течение определённого времени.    По умолчанию время равно 1 часу. Для изменения изменить переменную `TIME` (в секундах) внутри скрипта.                                                          |
| `/scripts/afl_pers_mod_instr.sh`    | Скрипт, который добавляет инструментацию `AFL++` для фаззинга в `persistant mode` **Важно! Сейчас используется только с `netflow_collector.cpp и `sflow_collector.cpp` | `./afl_pers_mod_instr.sh <netflow_plugin/netflow_collector.cpp>` |



## Пример запуска фаззинга вручную для отдельных фаззинг-оберток
--------------------------------
Запускаем контейнер:
```bash
docker run --privileged -it fuzz  /bin/bash
```

Для запуска фаззинга AFL++ разрешим многопточную работу:
```bash
echo core | tee /proc/sys/kernel/core_pattern
```

**Не забудьте собрать корпус данных в папку in**

Для тестового запуска будем использовать один sed с единичкой:
```bash
mkdir in
echo "1" >> in/1
```

При стандратной сборке `docker image` у нас будет папка `build_fuzz_harness`, внутри которой будт скомпилированы для фаззинг-обертки:
- `parse_sflow_v5_packet_fuzz`
- `process_netflow_packet_v5_fuzz`

**Запустим фаззинг:**
```bash
afl-fuzz -i in -o out -- ./parse_sflow_v5_packet_fuzz
```
Или

```bash
afl-fuzz -i in -o out -- ./process_netflow_packet_v5_fuzz
```

- В папке `build_netflow_pers_mod` будет находится код, сделанный для фаззинга функции `process_netflow_packet_v5` через AFL++ `persistant mode`.
- В папке `build_sflow_pers_mod` будет находится код, сделанный для фаззинга функции `parse_sflow_v5_packet` через AFL++ `persistant mode`.

Если сборка делается вручную, то используйте скрипт `afl_pers_mod_instr.sh` для инструментации файлов.

Запуск фаззинга этих функций одинаков, поскольку инструментируется финальный исполняемый файл `fastnetmon`
```bash
afl-fuzz -i in -o out -- ./fastnetmon
```
**ВАЖНО!**
Для фаззинга в многопотоке файла `fastnetmon` необходимо для каждого инстанса фаззера для `fastnetmon` подавать свой конфигурационный файл, где будут указаны разные порты для протоколов, иначе инстансы будут конфликтовать и несколько потоков не смогу запуститься. 


## Пример запуска фаззинга через скрипт автоматизации
--------------------------------
*Все действия происходят внутри контейнера, где рабочая директория `src`, поэтому пути строятся относительно данной папки*

Для фаззинга их используем скрипт `start_fuzz_harness`.

Для оберток скомпилированных в бинарные файлы:

```bash
/src/tests/fuzz/scripts/start_fuzz_harness.sh ./build_fuzz_harness/process_netflow_packet_v5_fuzz
```
Или 
```bash
/src/tests/fuzz/scripts/start_fuzz_harness.sh ./build_fuzz_harness/parse_sflow_v5_packet_fuzz
```

Для инструментации `fastnetmon`:
```bash
/src/tests/fuzz/scripts/start_fuzz_harness.sh ./build_netflow_pers_mod/fastnetmon
```
Или 

```bash
/src/tests/fuzz/scripts/start_fuzz_harness.sh ./build_sflow_pers_mod/fastnetmon
```


После запуска будет создана директория `<bin_name>_fuzz_dir`, внутри которой будут созданы папка `input`.
В корне системы будет создана папка `/output`, в которую будут отправлены выходные файлы фаззинга и файлы кластеризации. 
Так нужно, чтобы было удобно получить данные после фаззинга в контейнере (Смотрите ниже).
Будет запущена сессия `tmux` с инстансом фаззера `AFL++`.
Фаззинг будет продолжаться, пока не будет прироста новых путей в течении часа (можно поменять значение внутри скрипта).
Далее сессия tmux будет завершена, начнется кластеризация и проверка падений с помощью скрипта `minimize_out.sh`

## Другие техники фаззинга

### Грубое вмешательство в код с помощью Persistant Mode AFL++
--------------------------------
Фаззер `AFL++` позволяет переписать часть кода по фаззинг, попутно кратно увеличив скорость фаззинга (програма не заавершается после подачи одного набора даннх, а перезапускает цикл с функцией целью несколько раз).

Таким способом можно инструментировать две разных цели:
- `src/netflow_plugin/netflow_collector.cpp : start_netflow_collector(...)`
- `src/sflow_plugin/sflow_collector.cpp : start_sflow_collector(...)`

Как выглядит инструментация:
1. Перед целевой функций добавить конструкцию `__AFL_FUZZ_INIT();`
2. Цикл `while (true)` заменить на `while (__AFL_LOOP(10000))`
3. `char udp_buffer[udp_buffer_size];` заменить на `unsigned char * udp_buffer =__AFL_FUZZ_TESTCASE_BUF;`
4. `int received_bytes = recvfrom(sockfd, udp_buffer, udp_buffer_size, 0, (struct sockaddr*)&client_addr, &address_len);` заменить на `int received_bytes = __AFL_FUZZ_TESTCASE_LEN;`
5. Собрать с компилятором `AFL++` и санитайзерами. Собирать какие-либо обертки не нужно.
6. Запустить фаззинг `afl-fuzz -i in -o out -- ./fastnetmon`

Для этих двух целей сделан скрипт автоматизации внедрения инструментации кода. 
Смотрите подрбобнее скрипт  `/scripts/afl_pers_mod_instr.sh`.


### Иные технинки фаззинга
--------------------------------
| Название            | Описание                                                                                  |
|---------------------|------------------------------------------------------------------------------------------|
| `AFLNet`            | Особенности протокола (отсутствие обратной связи) не дают использовать данный фаззер                                      |
| `desock`            | Инструментация кода успешна, но фаззер не запускается - так же не может собрать обратную связь. Считаю данный способ **перспективным**, необходима корректировка фаззера. |
| `libfuzzer`         | Были написаны обертки под `libfuzzer` и внедрены в cmake, но из-за особенностей сборки и ориентации проекта на фаззинг через `AFL++` были убраны из проекта. Коммит с работающим [`libfuzzer`](https://github.com/pavel-odintsov/fastnetmon/commit/c3b72c18f0bc7f43b535a5da015c3954d716be22)


## Запуск фаззинга в docker contianer

### Пара слов для контекста

На каждый поток фаззера нужен один системный поток.

В скрипте `start_fuzz_harness.sh` есть время ограничения фаззинга. 
Переменная TIME отвечает за параметр "вермя нахождения последнего пути". 
Если этот параметр перестает обнуляться, значит фаззер заходит в тупик и нет смысла продолжать фаззинг.
Из эмпирического опыта такой параметр нужно ставить на 2 часа. В проекте стоит на 1 час. 
Если необходимо выполнить поверхностное тестирование, то можно сократить этот параметр до 10 - 15 минут, тогда общее время фаззинга будет занимать несколько часов. 

### Сборка и запуск

Сборка

```bash
cd fastnetmon
docker build -f tests/Dockerfile.ubuntu-24.04-afl++ -t fuzz .
```

Запуск контейнера
```
mkdir work
docker run -v $(pwd)/work:/output --privileged -it fuzz /bin/bash -c "/src/tests/fuzz/scripts/start_fuzz_harness.sh  ./build_netflow_pers_mod/fastnetmon"
```
Таким способом можно запустить любую обертку / бинарный файл, просто подав команду из пункта *Пример запуска фаззинга через скрипт автоматизации* в кавычках после аргумента `-c`

После окончая фаззинга результаты будут помещены на хостовую систему в папку `work` - там будет как папка с результатами, так и папка для кластеризации.

Контейнер будет иметь `status exit`, его можно перезапустить вручную и проверить краши.


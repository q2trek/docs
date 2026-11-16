# Первый запуск: сервер Quake 2 на 60 Гц

## 1. Что получится

Вы соберёте из исходников форков q2trek движок Q2PRO, моды OpenFFA и OpenTDM
и запустите выделенный сервер на 60 и на 30 Гц. Затем вы проверите, что частота
настоящая: подключите три вида клиентов и посчитаете, сколько пакетов в секунду
получает каждый. В конце вы прогоните часть автоматических тестов и узнаете,
что поменять на живом сервере.

Ниже «частота сервера» означает `sv_fps`, число шагов мира в секунду. «Снимком»
называется пакет с состоянием мира, который сервер отправляет клиенту после шага.
Под «upstream» понимается исходный проект, от которого отходит форк: Q2PRO, OpenFFA или
OpenTDM без правок q2trek.

Всё в этом документе выполнено 2026-11-16 на x86_64 Manjaro Linux, на ревизиях
из [REVISIONS.md](REVISIONS.md). На других системах команды не проверялись.

## 2. Что нужно

| Что | Зачем |
| --- | --- |
| Linux x86_64, bash | Команды проверены в bash; в других оболочках не проверялись |
| gcc, make, meson, ninja, pkg-config, git, python3 | Сборка и тесты |
| zlib (заголовки) | Движок |
| SDL2 (заголовки) | Клиент: окно и ввод |
| strace | Подсчёт пакетов в разделе 6 |
| Данные Quake 2 | Каталог вашей копии игры с `baseq2/pak0.pak` и `pak1.pak` (Steam, GOG, диск). Карта `q2dm1` из этой инструкции лежит в `pak1.pak`, без него разделы 5–7 не пройти |
| Свободный UDP-порт | Для сервера, по умолчанию 27910 |

На Arch и Manjaro всё ставится первой командой. Имена пакетов в ней
проверены через `pacman -Sp`; на машине проверки всё уже было установлено.
Вторая команда, для Debian и Ubuntu, не проверялась.

```
sudo pacman -S --needed base-devel meson ninja git python zlib sdl2-compat strace   # Arch, Manjaro
sudo apt install build-essential meson ninja-build pkg-config git python3 \
    zlib1g-dev libsdl2-dev strace                                             # Debian, Ubuntu (не проверялось)
```

Дальше все пути строятся от трёх переменных. Задайте их в каждом новом окне
терминала:

```bash
WORK=$HOME/q2trek-first            # пустой каталог для исходников, сборок и логов
Q2DIR="$HOME/quake2"               # каталог, в котором лежит baseq2/pak0.pak
PORT=27910                         # UDP-порт сервера
```

Проверьте, что данные на месте и порт свободен. Первая команда должна
перечислить файлы `pak*.pak`, а вторая не должна ничего вывести:

```bash
ls "$Q2DIR"/baseq2/pak*.pak
ss -uln | grep ":$PORT " || true
```

## 3. Взять код

Нужны четыре репозитория. У трёх форков основная ветка называется `master`: это upstream
плюс изменения частоты и исправления, она же стоит по умолчанию на GitHub. Ветка `tickrate`
рядом с ней хранит замороженную историю первого набора правок. В командах ниже `master`
указана явно, чтобы не перепутать. У тестов основная ветка называется `main`, и она же стоит
по умолчанию.

```bash
mkdir -p "$WORK" && cd "$WORK"
git clone -b master https://github.com/q2trek/q2pro.git
git clone -b master https://github.com/q2trek/openffa.git
git clone -b master https://github.com/q2trek/opentdm.git
git clone https://github.com/q2trek/tickrate-tests.git
```

Тесты сравнивают форк с исходным проектом и берут его из `origin/master`
(`run.sh:6-8@trekTests`). После `git clone` имя `origin`
занято самим форком. Поэтому форк переименовывается в `q2trek`, а под именем
`origin` добавляется upstream. README тестов описывает ту же последовательность:
переименовать, затем добавить upstream (`README.md:77-79@trekTests`).

```bash
cd "$WORK"
git -C q2pro   remote rename origin q2trek
git -C openffa remote rename origin q2trek
git -C opentdm remote rename origin q2trek
git -C q2pro   remote add origin https://github.com/q2pro/q2pro.git
git -C openffa remote add origin https://github.com/q2pro/openffa.git
git -C opentdm remote add origin https://github.com/packetflinger/opentdm.git
for r in q2pro openffa opentdm; do git -C $r fetch -q origin; done
for r in q2pro openffa opentdm; do git -C $r log -1 --format="$r %h %s" origin/master; done
```

Последняя команда показывает, на каком коммите сейчас стоит upstream каждого проекта. Базы, от которых
отходят форки, перечислены в [REVISIONS.md](REVISIONS.md). Upstream со временем
уходит вперёд, и это нормально.

## 4. Собрать

### 4.1. Движок и клиент форка

Частоту выше 10 Гц включает опция сборки `variable-fps`
(`meson_options.txt:149@trekQ2`). По умолчанию она выключена. Опубликованные
ночные сборки Q2PRO тоже собраны без неё (`README.md:12-13@trekQ2`).

Набор опций ниже совпадает с тем, что использует `run.sh` тестов
(`run.sh:97-99@trekTests`) с двумя отличиями: клиенту включён SDL2, а x11 здесь выключен,
тогда как `run.sh` задаёт его переменной. Звук и графические библиотеки отключены, чтобы сборка
зависела от минимума пакетов. Такой клиент годится для проверки, но играет без звука. Сборка
предупредит об этом: `WARNING: Neither software sound nor OpenAL enabled` у обеих целей
и `WARNING: No video drivers enabled, client will not be built` у серверной. Это ожидаемо.

```bash
cd "$WORK"
OPTS=(-Dwrap_mode=nofallback -Dsystem-wide=false -Dvariable-fps=true
      -Dx11=disabled -Dwayland=disabled -Dopenal=disabled -Dsoftware-sound=disabled
      -Dlibcurl=disabled -Dlibjpeg=disabled -Dlibpng=disabled -Davcodec=disabled
      -Dgame-new-api=false)
meson setup build-ded q2pro "${OPTS[@]}" -Dsdl2=disabled
ninja -C build-ded q2proded
meson setup build-cl q2pro "${OPTS[@]}" -Dsdl2=enabled
ninja -C build-cl q2pro
grep 'define USE_FPS' build-ded/config.h build-cl/config.h
```

Последняя строка должна дважды показать `#define USE_FPS 1`. Если там `0`,
опция не применилась (`meson.build:477@trekQ2`). Тогда удалите каталог сборки
и повторите `meson setup`. В конце `meson setup` в списке `Features` стоит
строка `variable-fps : YES`.

Сервер собирается секунд за шесть, а клиент секунд за восемь (16 ядер).

### 4.2. Клиент upstream для сравнения

Чтобы увидеть, что получит игрок с обычным Q2PRO, соберите клиент из upstream
с той же опцией. В каталог `q2pro-upstream` попадает копия `origin/master` без
истории git, поэтому версия такого клиента выглядит как `r…~unknown`.

Массив `OPTS` задан в 4.1. В новом терминале задайте его снова, иначе meson соберёт клиент
с настройками по умолчанию, без `variable-fps`.

```bash
cd "$WORK"
mkdir -p q2pro-upstream
git -C q2pro archive origin/master | tar -x -C q2pro-upstream
meson setup build-cl-up q2pro-upstream "${OPTS[@]}" -Dsdl2=enabled
ninja -C build-cl-up q2pro
```

### 4.3. Моды

OpenFFA объявляет поддержку переменной частоты только при сборке с флагом
`CONFIG_VARIABLE_SERVER_FPS=y` (`Makefile:51@trekFFA`,
`README.asciidoc:21@trekFFA`). OpenTDM объявляет её всегда
(`g_save.c:374@trekTDM`, `README.md:6@trekTDM`).

```bash
cd "$WORK"
make -C openffa -j"$(nproc)" CONFIG_VARIABLE_SERVER_FPS=y
make -C opentdm -j"$(nproc)"
```

OpenTDM собирается со своими статическими curl, OpenSSL и zlib из каталога
`deps/` репозитория. В имени готовой библиотеки есть версия, например
`gamex86_64-opentdm-r419~c64cf55.so`.

### 4.4. Куда положить библиотеки игры

Сервер ищет `gamex86_64.so` в каталоге игры: сначала в `homedir`, потом
в `libdir`, а затем так же в `baseq2` (`src/server/game.c:946-1002@trekQ2`).
Имя файла должно быть ровно `gamex86_64.so`, поэтому библиотеку OpenTDM надо
переименовать при копировании.

Каталогом данных сервера ниже служит `$WORK/srv`. Паки Quake 2 подключаются
ссылками.

```bash
cd "$WORK"
mkdir -p srv/baseq2 srv/opentdm srv/openffa run logs
ln -sf "$Q2DIR"/baseq2/pak*.pak srv/baseq2/
cp opentdm/gamex86_64-opentdm-*.so srv/opentdm/gamex86_64.so
cp openffa/gamex86_64.so srv/openffa/gamex86_64.so
```

Маска `pak*.pak` подключит все паки из каталога, в том числе сторонние. Если в вашей копии
лежат лишние паки, подключите только `pak0.pak`, `pak1.pak` и `pak2.pak`.

### 4.5. Если забыть флаг

| Что забыто | Что увидите | Почему |
| --- | --- | --- |
| `-Dvariable-fps=true` у движка | `sv_fps is read-only.`, в serverinfo `sv_fps 10` | Без опции `sv_fps` только для чтения и равен 10 (`src/server/main.c:2277-2280@trekQ2`) |
| `CONFIG_VARIABLE_SERVER_FPS=y` у OpenFFA | Ошибки нет, сервер молча работает на 10 Гц | Мод не объявил поддержку, и движок записывает в `sv_fps` настоящую частоту 10 (`src/server/init.c:44-51@trekQ2`) |
| Флаг при повторной сборке OpenFFA | `make: Nothing to be done for 'all'.`, библиотека прежняя | Объектные файлы не пересобираются от смены флага. Сначала `make -C openffa clean` |

## 5. Запустить

### 5.1. OpenTDM на 60 Гц

Минимальный конфиг. `g_gamemode 2` задаёт режим дуэли из README OpenTDM; для
проверки частоты режим не важен. Пароль rcon здесь нужен только для команд
проверки. На живом сервере поставьте свой.

```bash
cat > "$WORK/srv/opentdm/tdm60.cfg" <<'EOF'
set hostname "q2trek opentdm 60 Hz"
set sv_fps 60
set deathmatch 1
set maxclients 4
set g_gamemode 2
set rcon_password "change-me"
map q2dm1
EOF
```

Запустите сервер. Он слушает только `127.0.0.1` и работает в фоне. Номер процесса
сохраняется в файл, по нему сервер потом и останавливается.

```bash
cd "$WORK"
./build-ded/q2proded +set basedir "$WORK/srv" +set libdir "$WORK/srv" \
    +set homedir "$WORK/run/home" +set game opentdm \
    +set net_ip 127.0.0.1 +set net_port "$PORT" +exec tdm60.cfg \
    < /dev/null > logs/server.log 2>&1 &
echo $! > run/server.pid
sleep 3
cat logs/server.log
```

`basedir` и `libdir` заданы явно: в сборке с `-Dsystem-wide=false` оба по
умолчанию равны текущему каталогу. В логе должны быть такие строки (пути
сокращены):

```
Execing tdm60.cfg
Loaded game library from …/srv/opentdm/gamex86_64.so
==== InitGame ====
libcurl/7.88.1 OpenSSL/3.1.0 zlib/1.3.2 initialized.
Resolving API server opentdm.org -> …
SpawnServer: q2dm1
Map has 10 spawn points.
====== Q2PRO initialized ======
q2proded r…~dd240d1b, …
```

Число после `r` и хэш после `~` зависят от того, что лежит в `master` в день сборки;
на 2026-11-16 это `dd240d1b`. Строки `Can't access …/gamex86_64.so` перед `Loaded game library`
появляются при поиске по каталогам из 4.4, это не ошибка. Так же безвредны
`Can't load random maps file opentdm/smartmap.cfg` (списка случайных карт нет) и, у OpenFFA
в 6.4, `Couldn't load …/openffa/highscores/q2dm1.txt` (таблица рекордов ещё не создана). Частоту сервер при старте не печатает:
как её увидеть, сказано в разделе 6.

OpenTDM при старте обращается к opentdm.org за конфигами и статистикой. Это
включено по умолчанию (`g_http_enabled 1`, `g_save.c:220@trekTDM`,
`README.md:380@trekTDM`). Без сети строки про curl и opentdm.org будут другими;
на частоту это не влияет (по коду, не проверялось).

### 5.2. OpenFFA

От OpenTDM отличаются только `+set game openffa` и конфиг без `g_gamemode`. Запуск OpenFFA
показан в 6.4.

## 6. Убедиться, что частота настоящая

### 6.1. Что сервер объявляет

Для запросов к серверу понадобится маленький скрипт. Он отправляет серверу
одну команду без подключения к игре, как это делают браузеры серверов
(`status`, `rcon …`), и печатает ответ.

```bash
cat > "$WORK/q2query.py" <<'EOF'
import socket, sys
s = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)
s.settimeout(2)
s.sendto(b"\xff\xff\xff\xff" + " ".join(sys.argv[2:]).encode() + b"\n", ("127.0.0.1", int(sys.argv[1])))
print(s.recv(65535)[4:].decode(errors="replace"))
EOF
python3 "$WORK/q2query.py" "$PORT" status | head -2 | tr '\\' '\n' | grep -x -A1 sv_fps
python3 "$WORK/q2query.py" "$PORT" rcon change-me sv_fps
```

Ожидаемый ответ: `sv_fps`, затем `60`, затем строка `print` (так сервер начинает каждый
ответ на rcon) и `"sv_fps" is "60"  default: "10"`. Сервер форка публикует частоту в serverinfo,
поэтому её видно до подключения (`README.md:39-40@trekQ2`).

### 6.2. Три клиента и счёт пакетов

Теперь подключим три клиента:

- `fork`: клиент форка;
- `up`: клиент upstream из 4.2;
- `p34`: клиент форка на старом протоколе 34 (`cl_protocol 34`,
  `doc/client.asciidoc:33@trekQ2`).

Клиенты работают без окна и без звука 30 секунд и выходят сами. Они стартуют
с промежутком в 2 секунды: когда все три подключались в одну и ту же секунду,
сервер принимал только одного. Каждый клиент запускается под `strace`,
который записывает все приёмы UDP. Скрипт берёт отрезок
в 10 секунд, с 8-й по 18-ю секунду после первого ответа сервера: к этому времени клиент
уже загрузил карту. Полученные снимки скрипт считает по числу принятых датаграмм.
Пока клиенты подключены, скрипт спрашивает у сервера `status s` и `status n`
и замеряет, сколько процессорного времени ушло у сервера примерно за те же
10 секунд.

```bash
cat > "$WORK/count.py" <<'EOF'
import re, sys
for path in sys.argv[1:]:
    ts, sizes = [], []
    for line in open(path):
        m = re.match(r"(\d+\.\d+) recvfrom\(.*\) = (\d+)$", line.rstrip())
        if m and int(m.group(2)) > 0:
            ts.append(float(m.group(1)))
            sizes.append(int(m.group(2)))
    t0 = ts[0] + 8
    win = [n for t, n in zip(ts, sizes) if t0 <= t < t0 + 10]
    print(f"{path.rsplit('/', 1)[-1]}: {len(win) / 10:.1f} датаграмм/с, {sum(win) / 10:.0f} байт/с")
EOF
cat > "$WORK/measure.sh" <<'EOF'
#!/usr/bin/env bash
# measure.sh ПОРТ МЕТКА: три клиента на 30 с, status s и status n, счёт датаграмм
set -u
W=$(dirname "$(realpath "$0")") PORT=$1 TAG=$2
client() {
    env -u DISPLAY -u WAYLAND_DISPLAY SDL_VIDEODRIVER=offscreen SDL_AUDIODRIVER=dummy \
        timeout -k 5 30 strace -ttt -e trace=recvfrom -o "$W/logs/$TAG-$1.trace" \
        "$2" +set basedir "$W/srv" +set homedir "$W/run/cl-$1" +set s_enable 0 \
        +set name "$1" "${@:3}" +connect "127.0.0.1:$PORT" > "$W/logs/$TAG-$1.log" 2>&1 &
    sleep 2
}
client fork "$W/build-cl/q2pro"
client up   "$W/build-cl-up/q2pro"
client p34  "$W/build-cl/q2pro" +set cl_protocol 34
SV=$(cat "$W/run/server.pid")
sleep 6;  a=$(awk '{print $14 + $15}' "/proc/$SV/stat")
sleep 10; b=$(awk '{print $14 + $15}' "/proc/$SV/stat")
python3 -c "print(f'сервер: {($b - $a) / $(getconf CLK_TCK) / 10 * 100:.1f} % одного ядра')"
python3 "$W/q2query.py" "$PORT" rcon change-me status s
python3 "$W/q2query.py" "$PORT" rcon change-me status n
wait
python3 "$W/count.py" "$W/logs/$TAG"-{fork,up,p34}.trace
EOF
bash "$WORK/measure.sh" "$PORT" tdm60
```

Вывод на сервере 60 Гц (сокращён):

```
num name            proto options upd fps
  0 fork               36           0  60
  1 up                 36           0  20
  2 p34                34           0  10

num name              pps  cmds   dup   rej held%
  0 fork             62.5  62.5   0.0   0.0    0%
  1 up               30.3  62.5   0.0   0.0    0%
  2 p34              62.5  62.5   0.0   0.0    0%

recommended client packet rate: >= 60

tdm60-fork.trace: 60.0 датаграмм/с, 1200 байт/с
tdm60-up.trace: 20.0 датаграмм/с, 400 байт/с
tdm60-p34.trace: 10.0 датаграмм/с, 300 байт/с
```

Как это читать:

- `status s` показывает согласованные настройки клиентов. В колонке `fps` стоит частота
  снимков, которую сервер назначил клиенту при подключении
  (`src/server/commands.c:685@trekQ2`, `src/server/user.c:1547@trekQ2`).
  В колонке `proto` стоит протокол: 36 у Q2PRO, 34 у старого.
- `status n` показывает, что клиенты присылают сами за последние полные 10 секунд
  (`src/server/commands.c:525-541@trekQ2`). Колонка `pps` считает пакеты в секунду от
  клиента, `cmds` считает команды движения в секунду. `held%` показывает долю кадров сервера,
  в которых игрок имеет скорость, но не сдвинулся: другие видят такие кадры
  как остановки (`README.md:40-43@trekQ2`).
- Строки `count.py` показывают, сколько снимков клиент получил на самом деле. Они должны
  совпасть с колонкой `fps`.
- Если `count.py` упал на пустом файле `.trace`, клиент не подключился или не получал снимков.
  Лог клиента `logs/<метка>-<клиент>.log` при этом бывает пуст и у подключившегося клиента,
  по нему судить нельзя. Смотрите лог сервера (строка `<имя> connected`) и `status s`.

Почему частоты разные. Клиент форка и сервер форка договариваются о точной
шкале времени: клиент шлёт настройку `CLS_EXACTFPS` перед запросом частоты,
и сервер подтверждает её (`src/server/user.c:1566-1569@trekQ2`,
`README.md:32-34@trekQ2`). Клиент без такой договорённости считает период
снимка целым числом миллисекунд. Такому клиенту сервер выбирает наибольшую частоту,
у которой период целый и которая делит частоту сервера
(`src/server/user.c:1531-1535@trekQ2`). Из 60 Гц это 20 (50 мс), из 30 Гц это 10.
Старый протокол получает 10 Гц, как раньше (`README.md:28@trekQ2`).

Мир при этом идёт на 60 Гц для всех. Разница только в том, каждый ли кадр
сервер отправляет клиенту: клиенту назначается делитель частоты
(`src/server/user.c:1542@trekQ2`).

### 6.3. Переход на 30 Гц без перезапуска

`sv_fps` относится к отложенным настройкам: новое значение запоминается и применяется
при следующей загрузке карты (`src/server/main.c:2277@trekQ2`).

```bash
python3 "$WORK/q2query.py" "$PORT" rcon change-me set sv_fps 30
python3 "$WORK/q2query.py" "$PORT" rcon change-me sv_fps
python3 "$WORK/q2query.py" "$PORT" rcon change-me map q2dm1
sleep 3
python3 "$WORK/q2query.py" "$PORT" rcon change-me sv_fps
bash "$WORK/measure.sh" "$PORT" tdm30
kill "$(cat "$WORK/run/server.pid")"
```

После `set` сервер отвечает `sv_fps will be changed for next game.`,
а `sv_fps` показывает `"60" … latched: "30"`. После `map` он показывает `"30"`. Последние
три строки замера: 30, 10 и 10 датаграмм в секунду.

### 6.4. OpenFFA на 60 Гц

```bash
cd "$WORK"
grep -v g_gamemode srv/opentdm/tdm60.cfg | sed 's/opentdm/openffa/' > srv/openffa/ffa60.cfg
./build-ded/q2proded +set basedir "$WORK/srv" +set libdir "$WORK/srv" \
    +set homedir "$WORK/run/home" +set game openffa \
    +set net_ip 127.0.0.1 +set net_port "$PORT" +exec ffa60.cfg \
    < /dev/null > logs/server-ffa.log 2>&1 &
echo $! > run/server.pid
sleep 3
bash "$WORK/measure.sh" "$PORT" ffa60
kill "$(cat "$WORK/run/server.pid")"
```

Ожидается то же, что у OpenTDM: 60, 20 и 10.

### 6.5. Итог проверки

| Частота сервера | Клиент форка | Q2PRO upstream, собранный с `variable-fps` | Протокол 34 |
| --- | --- | --- | --- |
| 60 | 60 | 20 | 10 |
| 30 | 30 | 10 | 10 |
| 10 | 10 | 10 | 10 |

Строка 10 Гц снята так же, с конфигом `sv_fps 10`. Отдельно проверен клиент
upstream, собранный без `variable-fps`, как ночные сборки: на сервере 60 Гц он
получил 10 снимков в секунду, в `status s` у него `fps 10`. Так описано
и в README тестов (`README.md:56@trekTests`). Ночные сборки Q2PRO собраны без
`variable-fps` (`README.md:12-13@trekQ2`), поэтому их пользователи получат 10.

## 7. Прогнать тесты

`run.sh` собирает всё сам в каталог `OUT` и запускает сценарии на частотах от
10 до 60 Гц. Ему нужны три клона форков с upstream в `origin` (раздел 3)
и данные Quake 2 с картой `q2dm1` (`run.sh:9-12@trekTests`).

Каждый этап заканчивается словом PASS, FAIL или NOT_RUN. NOT_RUN значит, что
не хватило входных данных; причина записана в первой строке лога этапа
в `$OUT/log` (`README.md:87-89@trekTests`). Код выхода: 0 (всё прошло), 1
(что-то упало) или 3 (неполный прогон, в котором всё прошло)
(`run.sh:18-20@trekTests`). Тесты сами проверяют, что не изменили клоны:
если `git status` до и после прогона различается, это FAIL
(`run.sh:325-326@trekTests`). Поэтому не правьте файлы в клонах во время
прогона; сборка модов в разделе 4 следов в `git status` не оставляет.

Короткий прогон. Этапы называются `ИМЯ@ЧАСТОТА`, а `--only` выбирает их по
регулярному выражению. Сборки выполняются всегда.
В выражение входят и клиентские этапы: `fleet@60` берёт клиентов, собранных этими этапами
(`run.sh:191-205@trekTests`).

```bash
cd "$WORK/tickrate-tests"
Q2_DATA="$Q2DIR" OUT="$WORK/tt-out" ./run.sh \
    --only 'client-fixed@10|client@60|client-legacy@60|status-n@60|fleet@60' \
    ../q2pro ../openffa ../opentdm > "$WORK/logs/tests-short.log" 2>&1
rc=$?
tail -18 "$WORK/logs/tests-short.log"
echo "rc=$rc"
```

Он занимает меньше минуты вместе со сборками. В конце должно быть так:

```
PASS     build-ded
…
PASS     fleet@60
PASS     build-upstream-ded
partial run, not green: --only '…' left out 78 stage(s)
rc=3
```

Что проверяют эти этапы:

| Этап | Что проверяет |
| --- | --- |
| `client-fixed@10`, `client@60` | Клиент форка на сервере 10 и 60 Гц |
| `client-legacy@60` | Клиент с файлами клиента из upstream на сервере 60 Гц |
| `status-n@60` | Сервер публикует `sv_fps` в serverinfo и отвечает на `status n` |
| `fleet@60` | Четыре вида клиентов одновременно получают 60, 20, 10 и 10 снимков в секунду (`run.sh:372@trekTests`) |

В логе `fleet@60` (`$WORK/tt-out/log/fleet@60.log`) видно, сколько кадров
в секунду принял каждый, например `ok   19 frames/s accepted (expected 20)`.
Здесь вышло 60, 19, 9 и 9 при ожидаемых 60, 20, 10 и 10. Расхождение на один
кадр укладывается в допуск теста, этап проходит.

Полный прогон запускается той же командой без `--only`:

```
cd "$WORK/tickrate-tests"
Q2_DATA="$Q2DIR" OUT="$WORK/tt-out" ./run.sh ../q2pro ../openffa ../opentdm
```

Он прогоняет все частоты и должен закончиться кодом 0: 92 этапа PASS. При готовых сборках
это заняло 4 минуты 11 секунд. `RATES="30 60"` сокращает список частот, но такой
прогон тоже считается неполным и заканчивается кодом 3 в лучшем случае.

## 8. Поставить на живой сервер

### 8.1. Что меняется относительно 10 Гц

| Что | На 10 Гц | На 60 Гц |
| --- | --- | --- |
| Движок | Любой Q2PRO | Q2PRO из q2trek/q2pro с `variable-fps`. Upstream с той же опцией тоже пустит 60 Гц, но шаг будет 16 мс и мир пойдёт на 4 % быстрее (`README.md:15-18@trekQ2`) |
| OpenFFA | Обычная сборка | `make CONFIG_VARIABLE_SERVER_FPS=y` |
| OpenTDM | Обычная сборка | Та же сборка |
| Конфиг | нет | `set sv_fps 60` до команды `map` |
| Адрес | нет | Задать в `net_ip` адрес сетевого интерфейса сервера вместо `127.0.0.1` или не задавать `net_ip` вовсе (`doc/server.asciidoc:28@trekQ2`); не проверялось |

Допустимы значения 10, 20, 30, 40, 50 и 60. Другие значения оба мода
отвергают и останавливают сервер (`g_tdm_core.c:162@trekTDM`,
`g_main.c:1437@trekFFA`, `doc/server.asciidoc:258-263@trekQ2`).

На работающем сервере частота меняется так, как в 6.3: `set sv_fps 60`, затем
`map`.

### 8.2. Какие игроки что получат

| Клиент игрока | Снимков в секунду с сервера 60 Гц | Откуда известно |
| --- | --- | --- |
| Клиент q2trek/q2pro | 60 | Замер 6.2 |
| Q2PRO upstream, собранный с `variable-fps` | 20 | Замер 6.2 |
| Ночная сборка Q2PRO и любой Q2PRO без `variable-fps` | 10 | Замер 6.5 |
| Протоколы 34 и 35 (исходный Quake 2, R1Q2 и другие) | 10 | Замер 6.2 для 34; для 35 по описанию (`doc/server.asciidoc:267-268@trekQ2`) |
| Q2PRO-X 1.6 с `cl_serverfps_adapt 1` | 60 | По описанию клиента, не по его исходнику; не проверялось |
| Q2PRO-X 1.6 с `cl_serverfps_adapt 0` (по умолчанию) | 10, как раньше | По описанию клиента, не по его исходнику; не проверялось |

Q2PRO-X 1.6 при подготовке документа не запускался.

Чтобы проверить конкретного игрока, выполните `rcon status s` и посмотрите колонку `fps`
в его строке. Там стоит та частота, которую сервер ему назначил.
`rcon status n` показывает, сколько пакетов и команд движения он присылает.

### 8.3. Пакеты от игрока

`status n` советует присылать не меньше одного пакета на кадр сервера
(`recommended client packet rate: >= 60`). Клиент форка по умолчанию шлёт
пакет на каждый свой кадр: `cl_maxpackets 0` (`src/client/input.c:695@trekQ2`,
`doc/client.asciidoc:42@trekQ2`). У upstream по умолчанию 30
(`README.md:60-70@trekQ2`). В замере 6.2 клиент upstream слал около 30 пакетов
в секунду. `held%` у всех был 0, но это ничего не говорит: зрители не
двигались. Сколько пакетов даёт лимит 30, зависит от `cl_async`: при `cl_async 1`
(по умолчанию) клиент проверяет лимит на каждом проходе главного цикла и шлёт
около 30 пакетов в секунду, при `cl_async 0` проверяет раз в кадр `cl_maxfps 62`
и шлёт около 21 (`README.md:60-70@trekQ2`, `src/client/input.c:821-847@trekQ2`).
Замер 6.2 совпадает с первым случаем. При 21 пакете сервер 60 Гц проводит два
кадра из трёх без новой команды от этого игрока, и другие видят его движение
ступеньками. Как выглядит бегущий игрок с таким клиентом, здесь не мерилось.

### 8.4. Трафик и процессор

Что измерено. В разделе 6 три клиента стояли зрителями на пустой q2dm1.

| Частота сервера | Байт в секунду: форк / upstream / протокол 34 | Процессор сервера OpenTDM |
| --- | --- | --- |
| 60 | 1200 / 400 / 300 | 0,6–0,8 % одного ядра |
| 30 | 600 / 200 / 300 | 0,5–0,6 % |
| 10 | 200 / 200 / 300 | 0,4 % |

Загрузка процессора снята в двух прогонах (на 10 Гц в одном), отсюда разброс. В третьем прогоне на той же
машине под другой нагрузкой вышло 1,2 % на 60 Гц и 0,7 % на 30 Гц: цифры зависят от того, что
ещё делает машина. Цифры для
настоящей игры с движением и стрельбой отсюда не следуют.

Что известно по коду, но не мерилось. Сервер сравнивает объём последних 10
сообщений клиенту, пересчитанный на секунду, с его `rate`. Если объём больше,
сервер пропускает кадр этому клиенту (`src/server/send.c:58-83@trekQ2`,
`src/server/server.h:237@trekQ2`). По умолчанию `rate` клиента 15000 байт
в секунду (`src/client/main.c:2828@trekQ2`). Сервер ограничивает его значениями
`sv_min_rate` 15000 и `sv_max_rate` 60000 (`src/server/main.c:2300-2301@trekQ2`,
`doc/server.asciidoc:151-158@trekQ2`). На 60 Гц 15000 байт в секунду дают
в среднем 250 байт на снимок. Вывод, не проверенный замером: в людной игре
игрокам с `rate 15000` сервер будет пропускать снимки. Возможная мера, тоже
не проверенная: игрокам поставить `rate` повыше, серверу поднять
`sv_min_rate`.

Ограничения нет только у клиента в том же процессе, что и сервер (игра на
своём локальном сервере). Подключение по UDP к `127.0.0.1`, как в разделе 6,
ограничивается так же, как внешнее
(`src/server/main.c:2120-2121@trekQ2`, `inc/common/net/net.h:105@trekQ2`).

### 8.5. Демо и MVD

Запись демо и MVD остаётся на 10 Гц. Сервер пишет в MVD только кадры,
кратные 100 мс (`src/server/mvd.c:1017@trekQ2`,
`src/server/server.h:141@trekQ2`). Формат демо не меняется
(`README.md:35-37@trekQ2`).

## 9. Если что-то не так

Все случаи ниже воспроизведены при подготовке документа.

| Что видно | Причина | Что сделать |
| --- | --- | --- |
| `sv_fps is read-only.`, в serverinfo `sv_fps 10` | Движок собран без `variable-fps` | Пересобрать с `-Dvariable-fps=true` и проверить `USE_FPS 1` в `config.h` (4.1) |
| OpenFFA запускается без ошибок, но `rcon sv_fps` показывает 10 | Мод собран без `CONFIG_VARIABLE_SERVER_FPS=y` | `make -C openffa clean`, затем сборка с флагом (4.5) |
| `Game Error: Invalid server FPS` (OpenTDM) или `Invalid value '45' for 'sv_fps' cvar` (OpenFFA), сервер выходит | `sv_fps` не кратно 10 или больше 60 | Поставить 10, 20, 30, 40, 50 или 60 (8.1) |
| `Can't access …/gamex86_64.so` во всех каталогах и `ERROR: Failed to load game library` | Библиотека OpenTDM скопирована под своим длинным именем или не в тот каталог | Скопировать как `srv/opentdm/gamex86_64.so` (4.4) |
| `set sv_fps 30` ответил `sv_fps will be changed for next game.`, а частота прежняя | Отложенная настройка | Выполнить `map` (6.3) |
| В `status s` у игрока `fps 20` или `10` при сервере 60 Гц | Клиент без договорённости о точной шкале или без `variable-fps`; либо игрок сам занизил `cl_updaterate` | Так и должно быть (8.2). Полную частоту даёт клиент форка с `cl_updaterate 0` или 60 |
| `git remote add origin` в клоне форка: `remote origin already exists` | `origin` после клонирования указывает на сам форк | Переименовать его, как в разделе 3 |
| Тесты: `NOT_RUN  upstream-q2pro: missing input: Q2PRO has no origin/master` | В клоне нет upstream под именем `origin` | Раздел 3. Воспроизведено с несуществующей веткой `UPSTREAM_REF=nosuch/master` |

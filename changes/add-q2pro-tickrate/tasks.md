*Перенесено в q2trek/docs 2026-11; адаптация описана в ADAPTATION.md.*
*Что убрано и заменено при переносе: [ADAPTATION.md](../../ADAPTATION.md).*

# Tasks

Задачи выполнены в исходном проекте; результат отражён в форках q2pro, openffa, opentdm и в tickrate-tests.

Change: `add-q2pro-tickrate`, редакция для 60 Гц. Все задачи относятся к **предстоящей реализации**; создание OpenSpec не закрывает ни одну из них.

Обязательные входы: `proposal.md`, `design.md`, все delta specs, `testing.md`, `sources.md`. Идентификаторы T01–T22 определены в `testing.md`.

**Порядок:** 1 → 2. Затем группы 3 → 4 (движок); группы 6–9 идут параллельно движку на харнессе из группы 1. Группа 14 (находки ручной сессии и внешнего ревью 2026-10-28, design D12–D15): 14.1 — первая, 14.2 — после 14.1, 14.3–14.5 независимы. В каждой задаче, меняющей поведение: baseline 10 Гц этого поведения (если ещё нет) → RED на 60 Гц → исправление → GREEN и baseline 10 Гц. После каждой группы прогоняется полный локальный набор `tickrate-tests`. Тестовые швы и критерии из других артефактов в задачах не переопределяются.

## 1. Локальный набор и несущие baseline

- [x] 1.2 Baseline scheduler при 10 Гц: T01/T04/T05 — ранний вызов, остаток, sleep, pause, uninitialized, hitch 250→100, map/error, `usercmd.msec` и allowance 1,6 с, команда из RunFrame. Проверка: проходят на исходном алгоритме; многошаговый catch-up и граница пакетных команд не выдаются за реализованные.
- [x] 1.3 API3-харнесс обоих модов (отдельно собранный настоящий модуль: GetGameAPI, SpawnEntities, ClientThink, RunFrame) и baseline T09–T12 при 10 Гц: выстрелы, ammo, held cadence, граната, hurt, классы crush, включая `rotating_touch` с двумя ClientThink в одном world tick. Проверка: измеряются игровые последствия, а не внутренние поля; известный high-FPS дефект не закреплён.

## 2. Тестируемая variable-FPS сборка

- [x] 2.2 Подключить режим 60 Гц к fixtures и подтвердить воспроизведение известных high-FPS регрессий перед их исправлением. Проверка: причины RED для clocks, weapon, damage, mover, HUD и grenade сохранены там, где дефект воспроизводится; уже правильные пути отмечены GREEN-before без искусственного ломания.

## 3. Точные server/client clocks

- [x] 3.1 Добавить общий рациональный timing contract и T01/T02 для 10/20/30/40/50/60 Гц: широкие произведения, floor/ceil, обратное преобразование на границе. Проверка: виртуальная минута даёт точный счёт; scheduler не использует постоянные 16/33 мс; rebase для T(1)=16 без обратного off-by-one.
- [x] 3.2 Перевести production server scheduling и native sleep на контракт, сохранив paused/uninitialized/hitch и разделение реальных часов; описать clocks/units для разработчиков. Проверка T01/T04: точный счёт при 60 Гц, residual/sleep, allowance 1,6 с, нет повторного учёта external time или input; все потребители `SV_FRAMETIME`/`CL_FRAMETIME`/reciprocal классифицированы; baseline 10 Гц проходит.
- [x] 3.3 Перевести client frame-time/interpolation/animation consumers и rebase на согласованные anchors; baseline парсера и шкалы 10 Гц (T02/T07) — до правки. Проверка T02: 30/60 Гц, dropped frames, no-lerp cases, непрерывное presentation time при смене rate, фаза 100-миллисекундных keyframes; map/demo reset корректен.
- [x] 3.4 Обновить инициализацию и сброс clocks: map/serverdata, disconnect, MVD waiting room. Проверка T01/T02/T08: прежние debt/anchor не переживают эпоху; capability state сбрасывается (после 4.1).
- [x] 3.5 Тест demo/MVD: baseline 10 Гц, затем record/playback/seek из согласованных 60 Гц; код записи не меняется. Проверка T08: длительность и формат прежние, рациональная шкала не навязана старой записи.

## 4. Capability и updaterate

- [x] 4.1 Capability по D3: новый `CLS_` индекс в том же reliable flush до `CLS_FPS` и `SVS_` ответ сервера; номера — высокие индексы вне enum (design D3). Capability шлётся из `CL_UpdateRateSetting` перед `CLS_FPS` при каждом `CL_Begin` и смене `cl_updaterate`; клиент включает рациональную шкалу только после ответа; сброс при serverdata. Проверка T03: IDs не конфликтуют; старый сервер игнорирует индекс и новый клиент остаётся в старой шкале; старый клиент не получает ответа.
- [x] 4.2 Rate selection и legacy fallback по таблице D3; re-anchor поздней настройки по D1 с сохранением `cl.servertime`. Проверка T03: весь набор requests даёт указанные rates; T07 fault fixture: потерянный (в том числе фрагментированный) carrier — кадры приходят, настройка опаздывает, delta не повреждена, presentation time непрерывно; на сервере без `USE_FPS` запись demo работает.
- [x] 4.3 Integration одновременно подключённых клиентов разных версий; описать совместимость. Проверка T03/T07: новый на 60 Гц, legacy на 20 Гц и клиент без FPS на 10 Гц одновременно имеют правильную шкалу; переходы 20↔30↔60 через `cl_updaterate` непрерывны; нового формата `svc_frame` нет. Совместимость: новый↔новый — точная шкала после `SVS_EXACTFPS`=1; новый↔старый сервер — старая шкала; старый/без-FPS клиент↔новый сервер — 20/10 Гц.

## 5. Catch-up и локальный transport

- [x] 5.2 Граница команд по D4: снимок `cmd_buffer.cursize` на входе в `SV_Frame` до `NET_GetPackets`, рост перед шагом прерывает пачку. Проверка T05: packet-queued config/kick/map не опережаются шагом, game-queued map останавливает пачку, прежний bind не блокирует её, существующий `wait` плюс packet-queued `map` не дают шага до `map`, долг списан один раз.
- [x] 5.3 Loopback capacity/drain для выбранных targets с кандидатом 8 и проверкой фактической пачки. Проверка T06: 5 world sends плюс async/control доставлены без потерь, FIFO/wrap и оба направления; fragmentation/packetdup учтены, намеренный overflow наблюдаем.

## 6. Оружие и граната

- [x] 6.1 Исправить OpenTDM pending attack до реального weapon callback, включая MG/HB, сохранив cadence. Проверка T09: короткий tap во всех 6 фазах даёт допустимый выстрел и расход ammo, cadence удержанного огня 10 Гц сохранён, один intent не превращается в очередь атак.
- [x] 6.2 Проверить OpenFFA second-tap/weapon-thunk путь и исправить только воспроизведённую потерю. Проверка T09: ClientThink→RunFrame проходит все 6 фаз и случаи 10 Гц; при GREEN-before зафиксировано, что исправление не требуется.
- [x] 6.3 Очистка и потребление pending state на death/respawn/spectator/map/switch/no-ammo и TDM countdown/timeout/resume. Проверка T09: прежний tap не стреляет после восстановления контекста, TDM hand-grenade release/held/death/switch проверены по baseline мода; API3 prefix layouts сохранены.
- [x] 6.4 Исправить реальные секунды fuse и charge curve ручной гранаты OpenFFA. Проверка T10: 3,2 с от arming, при remaining 1 с forward speed 660±1, немедленный взрыв при remaining≤0, quick/held/death/switch, один расход ammo; baseline 10 Гц не изменён.

## 7. Урон окружения

- [x] 7.1 Нормализовать OpenTDM hurt damage interval и фазонезависимый звук обоих модов. Проверка T11: normal/SLOW, все 6 фаз, независимый sound cooldown, SILENT/NO_PROTECTION и прежняя shared-source политика; baseline 10 Гц проходит.
- [x] 7.2 Исправить только world-driven continuous crush sources обоих модов (door/plat/`rotating_blocked`/crusher, `func_object` над движущимся потерпевшим — его `info` в харнессе становится обязательным assertion) с немедленным первым hit и ограничителем source/victim; `rotating_touch`, train/secret cooldown и one-shot paths не трогать. Проверка T12: same-tick duplicates, неудобные фазы, 2 потерпевших/2 источника, object/debris, blocked/reversal; `rotating_touch` даёт удар на каждый допустимый контакт из ClientThink, как при 10 Гц, путь `SV_Impact` не меняется; общего MOD_CRUSH gate нет.
- [x] 7.3 Покрыть reset/reuse нового cooldown state. Проверка T11/T12: удалённая entity не передаёт throttle новой, существующий shared cooldown не меняет баланс.

## 8. Движущиеся объекты

- [x] 8.1 Engine collision fixture и baseline T13/T14 при 10 Гц до правки (T12 повторно прогоняется на этой fixture); затем устранить quantization drift в затронутых push paths, сохраняя сетку 1/8 и среднюю скорость. Проверка T13 и quantization cases T14: 100 единиц при 100 единицах/с, оба направления, диагональ, короткий путь, ограниченная ошибка пути и endpoint, нет большого последнего скачка; carry пригоден для linear и accelerated callers.
- [x] 8.2 Включить carry в team push transaction и lifecycle. Проверка T13/T14: блокировка второго компонента после перемещения первого и пассажира откатывает positions и carry; unblock/reverse/new path/reset/reuse без рывка; game-save не добавляется.
- [x] 8.3 Исправить accelerated final segment OpenTDM с явным 100-миллисекундным controller interval, не меняя linear callers. Проверка T14: ненулевой и нулевой остаток, конечная скорость и длительность от первого движения, endpoint≤1/8, нет drift внутри control segments, однократный endfunc, baseline 10 Гц.
- [x] 8.4 Проверить accelerated FFA physics и связанные collision scenarios. Проверка T13/T14: единицы, пределы скорости и endpoint сохранены, dispatch delay не сравнивается как прежние 100 мс; baseline 10 Гц обоих модов проходит.

## 9. Таймеры, HUD и status

- [x] 9.1 Baseline T15 при 10 Гц, затем исправить TDM integer countdown в обоих consumers match/timeout, `players`, достижимый scoreboard и задержку screenshot 0,5 с. Проверка T15: 180/300 кадров при 60 Гц, соседние, нулевые и отрицательные значения, фактический текст и момент screenshot.

- [x] 9.3 Ограниченный аудит временных констант и lifecycle обоих модов, тесты только для обнаруженных непокрытых ветвей. Проверка T09–T16: limits/ready/vote/intermission/respawn и 100-миллисекундные weapon/animation периоды; raw `+1/+2` классифицированы, не заменены механически; список решений сохранён.

## 11. Наблюдение и полный локальный набор

- [x] 11.1 Добавить недостающие observables для world, debt, valid live frames, epochs, suppression, fragment и loopback через существующие diagnostic APIs. Проверка T19: счётчики отражают native state, demo semantics не меняются, длинное окно не обрезано trace на 512 тиков.
- [x] 11.2 Negative controls и lifecycle измерений. Проверка T07/T19: низкий effective rate, transport loss, fragmentation, invalid base, queue pressure и устойчивая задержка у границы history дают различимые результаты; map/reconnect/playback не образуют ложных окон.
- [x] 11.3 Полный локальный набор запускается одной командой, описан в README `tickrate-tests`. Проверка: набор проходит; намеренная регрессия ломает соответствующий тест; нет tickrate skip/xfail и опасных release testing flags; в `git diff --cached` нет тестовых файлов.

## 14. Находки сессии и внешнего ревью (D12–D15)

- [x] 14.1 Отделить в API3-харнессе расписание usercmd от тиков мира: каждый тик, 30 пакетов/с (по умолчанию Q2PRO), 10 пакетов/с и одно чтение на пачку до 5 шагов; `msec` команд копит 16/17 мс по `floor`-разностям. Существующие сценарии переводятся без изменения смысла. Проверка: секунда команд — 1000 мс при 60 Гц; до продуктовых правок 14.2–14.6 полный набор на неизменённом кандидате даёт тот же список PASS (сверка списков, не только rc), а намеренно удалённая проверка опоры/урона в T13/T14 даёт FAIL.
- [x] 14.2 Опора пассажира по D12 в `G_RunFrame` обоих модов. Сначала RED на production-логике: T14 «опускающаяся платформа с пассажиром» при 10/60 Гц в расписаниях 14.1 (FFA `func_plat` и TDM team mover), T13 «откат team push с пассажиром без свежего ввода» и многоцикловый T14 «подъём/спуск ×5». Проверка T13/T14: на каждом тике `groundentity` = платформа, относительная высота ≤ 1/8, нет EV_FALL*/потери HP, по одному start/end звуку на цикл, `s.sound` = 0 после остановки, длительность циклов не растёт; у не-клиентов (`func_object`, предметы) поведение прежнее — существующие T12 проходят без изменений; 10 Гц меняется только в случае отката с пассажиром без ввода.
- [x] 14.3 Граната при pending tap по D14 в обоих модах. Проверка T10: tap во время взвода не сдвигает бросок — gunframe 12 на том же weapon update, где потреблён pending, как на 10 Гц (RED: на один update позже; на 60 Гц при расписании ревьюера 216 вместо 210), обычные tap/held/release гранаты прежние, pending для остального оружия (T09) не меняется.
- [x] 14.4 Строгий `CLS_EXACTFPS` по D14. Проверка T03: значения 2, −1 и protocol ≠ 36 — нет ответа, `exactfps` не включается; те же значения после согласования не выключают его; значение 1 на protocol 36 (и его повтор) — прежний ответ.
- [x] 14.5 Переполнение `MAX_CRUSH` по D14 в обоих модах, комментарий `ponytail:` уходит. Проверка T12: при 65+ активных парах удар не чаще раза в 0,1 с на потерпевшего (RED: 60/с), первый удар немедленный, переход fallback→таблица не даёт двух ударов быстрее 0,1 с, новая карта и переиспользованный edict начинают с немедленного удара; при неполной таблице результаты T12 прежние.

## 15. Внешнее ревью 2026-11-02 (D16)

Порядок: сначала публичные форки и их tickrate-tests (сценарий RED на текущем коде → правка → GREEN → негативный контроль).

- [x] 15.1 Сценарии в tickrate-tests; тестовая сборка модов `-ftrivial-auto-var-init=pattern` вместо `zero` (`build.py`). Сценарии: удар с нулевым `dir` — `velocity` до и после совпадает (OpenFFA); press→release→press→release одной пачкой до одного RunFrame, оба мода, несколько фаз оружия; timeout с release во время паузы (MG, граната) и удержанием гранаты, resume без нового usercmd, окно после resume ≥ 2 шагов оружия, несколько фаз паузы; BFG короткий полёт в стену — удары лазера и момент вторичного взрыва по мировому времени 10 Гц (OpenTDM), урон вторичного взрыва цели у стены одинаков на 10 и 60 Гц (оба мода), OpenFFA по фазам `h_div` (лишний удар — записать, не чинить); поворот viper bomb за секунду; packetdup на loopback в отдельном варианте сборки с `packetdup-hack`, который гоняет только этот сценарий (T06 требует `!USE_PACKETDUP`). Где RED на текущем коде: knockback — под pattern; второй tap — OpenTDM на всех частотах, OpenFFA страховочный (уже исправлен); release в timeout — все частоты; удержание гранаты — страховочный; лазер BFG — 50/60 Гц, вторичный взрыв и урон — от 20 Гц; viper — выше 10 Гц; packetdup — только в своём варианте сборки.
- [x] 15.2 Правки D16.1–D16.5 и D16.7 в форках новыми коммитами (в текстах — что меняется на 10 Гц). Проверка: 15.1 GREEN; матрица 10/20/30/40/50/60 на pattern проходит целиком (новые чтения стека — находка); прежние T09/T10 проходят; на каждую правку негативный контроль со своим якорем `sed` (новая строка `attack_pending = true;` не должна задевать старый `neg-opentdm-pending`), пойман своим CHECK; `run.sh --negative` rc=0.

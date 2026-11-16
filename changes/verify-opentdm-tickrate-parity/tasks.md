*Перенесено в q2trek/docs 2026-11; адаптация описана в ADAPTATION.md.*
*Что убрано и заменено при переносе: [ADAPTATION.md](../../ADAPTATION.md).*

# Tasks

Исправления долиты в форк OpenTDM.

Change: `verify-opentdm-tickrate-parity`. Входы: `proposal.md`, `design.md`, delta spec
`q2pro-opentdm-runtime` (OT-T06, OT-T07), архив `add-q2pro-tickrate` (стиль тестов, T01–T22).
Идентификатор новых сценариев — T23 (`t23_*` в `tickrate-tests/mods/api3/scenarios.h`; тест в публичных тестах отсутствует).

**Порядок:** 1 → 2.

## 1. Таблица временных мест

- [ ] 1.1 Построить `SITES.md` по design D3: все вхождения
  `framenum|realframenum|FRAMETIME|SERVER_FPS|SECS_TO_FRAMES|FRAMES_TO_SECS|FRAMESYNC` в
  `opentdm/*.c *.h`, по строке на место, столбцы файл:строка / выражение / класс / тест или
  «контракт (ссылка)» / «нет». Проверка: число строк таблицы равно числу вхождений `grep -c`;
  у каждого места ровно один класс.
- [ ] 1.2 Отметить контракты из design Non-Goals (`rotating_touch`, `Think_Weapon`, endpoint, мёртвый
  `ScoreboardMessage`) со ссылкой на OT-T02/решение аудита и больше не трогать. Проверка: в таблице
  нет строки «контракт» без ссылки.
- [ ] 1.3 Список непокрытых мест класса «секунды» и «каденция» → перечень сценариев T23 с именем,
  наблюдаемой величиной и допуском (design D1), в `SITES.md` внизу. Проверка: каждое непокрытое
  место ссылается на сценарий; сценариев без места нет.

## 2. Дифференциальные тесты T23 (api3)

- [ ] 2.1 Общий helper сравнения 10/60 в `tickrate-tests/mods/api3` (запуск сценария при `HARNESS_RATE=10` и `60`,
  запись событий с временем в секундах, сравнение с допуском 0,1 с для моментов и точным равенством
  счётчиков). Переиспользовать `h_real_phase`, `h_sounds`, `h_stat`; новых файлов не плодить.
  Проверка: `smoke` при 10 и 60 PASS; helper использован хотя бы в одном t23.
- [ ] 2.2 T23 матчевые часы и фазы: countdown/ready, match/overtime/sudden death, timeout и его лимит,
  резюм, intermission, HUD `time_remaining`, `players`, scoreboard handicap, ghost. Проверка: зелёные
  при 10 и 60.
- [ ] 2.3 T23 предметы и голоса: respawn предметов, powerup длительности и предупреждение (OT-T06,
  включая таймаут), vote window, flood protection (`flood_msgs/waves` в секундах). Проверка: как 2.2;
  `t18_powerup_warn_timeout` остаётся, новый тест добавляет случай «таймаут на том же кадре» как
  допуск по спеке.
- [ ] 2.4 T23 среда и presentation (OT-T06): flash/blend суммы за 100 мс, gun offset/history, kick
  hold, bob в воздухе, лава/слизь удары и звук, fall/drown debounce. Проверка: зелёные при 10 и 60.
- [ ] 2.5 Движущиеся объекты и debounce: двери/платформы/`rotating_blocked` в OpenTDM, debounce
  касаний и blocked, если `SITES.md` показал непокрытые места. Проверка: зелёные на обеих ревизиях
  (баг не ожидается); при красном — fix по D6.
- [ ] 2.6 Каждое красное место: минимальная правка `opentdm`, тест красный до и
  зелёный после. Проверка: красный лог до правки
  сохранён; полный набор стадий PASS после.

*Перенесено в q2trek/docs 2026-11; адаптация описана в ADAPTATION.md.*
*Что убрано и заменено при переносе: [ADAPTATION.md](../../ADAPTATION.md).*

# q2pro-opentdm-runtime Specification

## Purpose
Run the existing OpenTDM module, including its distinct TDM and Duel modes, on the accepted Q2PRO foundation while preserving game rules, configuration.

## Requirements

### Requirement: OT-03 Teams readiness and spectators

OpenTDM MUST preserve its existing team admission, team limits, locks/invitations, captain controls, spectators/chase, ready/notready, warmup, and countdown behavior. TDM MUST retain `g_gamemode=0`; Duel MUST retain `g_gamemode=2` and one playing seat per team. Server connection capacity, occupied playing seats, and spectators MUST remain distinct.

A TDM countdown with one player on each team MUST retain the existing pseudo-1v1 behavior and effective Duel semantics.

#### Scenario: All playing clients control countdown

- **WHEN** both nonempty teams are in warmup and every playing client becomes ready, or a playing client becomes notready during countdown
- **THEN** the existing countdown starts or aborts according to OpenTDM rules, ready counts describe those actual players, and spectators do not satisfy a team's readiness
- **AND** team changes during countdown or an active match remain subject to the existing module restrictions.

#### Scenario: Duel and pseudo-1v1 remain distinct from TDM 2v2

- **WHEN** explicit Duel uses one player per team, TDM starts with one player per team, or TDM starts with two players per team
- **THEN** explicit Duel retains its two-seat rule, the one-per-team TDM case retains pseudo-1v1 and effective Duel presentation, and the four-player TDM case retains team-score semantics

#### Scenario: Team admission and spectator guards remain effective

- **WHEN** a participant joins a full or locked team, uses an authorized team invitation, enters spectator/chase, or attempts to ready as a spectator
- **THEN** the existing admission, permission, and spectator rules determine the outcome, without adding a playing seat, replacing a human with a bot, or bypassing the Duel limit
- **AND** normal movement, collisions, weapons, damage, death, and respawn remain the existing OpenTDM gameplay for admitted players.

### Requirement: OT-04 Match clocks ties and timeouts

OpenTDM MUST preserve its current match duration, tie policy, overtime, sudden death, timeout/resume, and disconnect recovery semantics. Existing module defaults MUST remain `g_match_time=600`, `g_max_players_per_team=4`, `g_match_countdown=15`, `g_tie_mode=1`, `g_overtime=60`, `g_timeout_limit=2`, `g_timeout_captain=0`, `g_max_timeout=300`, and `g_1v1_timeout=60` unless explicitly overridden by the accepted startup/configuration contract.

A zero `g_match_time` MUST remain a numeric zero-duration match limit, not an unlimited FFA limit. OpenTDM MUST NOT gain an active player/team frag limit from another target. During `MM_TIMEOUT`, playing clients and game time MUST remain frozen while the module's real-frame timeout and resume countdown advance. The accepted guard against match-end processing cancelling a timeout MUST remain effective without a new engine scheduler bypass.

#### Scenario: Match duration and tie policy use OpenTDM units

- **WHEN** an OpenTDM match reaches its configured duration with a lead or a tie, including a configured duration of zero
- **THEN** the current finish, overtime, or sudden-death rule follows the configured tie mode, the duration retains its correct units, and zero is not reported or treated as unlimited

#### Scenario: Timeout freezes play while resume time advances

- **WHEN** an eligible player calls the existing timeout command, requests resume, or reaches the configured timeout deadline
- **THEN** current caller/captain/admin and timeout-count rules are enforced, player simulation and match time remain frozen during the timeout, and real-frame timeout/resume processing continues to the existing resume outcome

#### Scenario: Sudden death and disconnect timeout retain their guards

- **WHEN** a timeout overlaps an expired sudden-death deadline or a Duel participant disconnects during a match
- **THEN** match-end processing does not cancel the active timeout, and the existing Duel reconnect/forfeit/automatic-resume rules determine the subsequent outcome

### Requirement: OT-05 Map decisions and effective game-directory files

OpenTDM MUST preserve its existing map-vote/admin validation and permissions, explicit-map priority, automatic rotation ordering, actual map transitions, and separate match-score intermission.

Configured maplist, config-vote, and smartmap files MUST be read from the effective game directory, including when native runtime CWD differs from the package data directory. Correct filesystem resolution MUST preserve the short game identity and legacy fallback. `g_maplistfile` MUST remain empty by default; absent optional config/smartmap files MUST retain their existing optional behavior. The change MUST NOT introduce these files as new mandatory defaults or change map-policy failure semantics as a side effect of resolving valid supplied paths. The config scan MUST terminate and skip entries that are not `.cfg` or have an invalid name.

#### Scenario: Explicit map choice wins over rotation

- **WHEN** a validated map vote or authorized explicit admin map command selects a map different from the automatic rotation choice
- **THEN** the selected map takes precedence over rotation and `DF_SAME_LEVEL`, while an automatic transition without an explicit choice retains the existing ordering and authored next-map behavior
- **AND** clients follow the actual new map/epoch and become playable again.

#### Scenario: Match intermission repeats the current map

- **WHEN** a normal match finishes and its score intermission ends without an explicit map action
- **THEN** OpenTDM returns to its existing warmup/reset behavior on the current map, preserving the final-score observation until the proper reset
- **AND** this match transition is not reported as automatic map rotation or a newly loaded map.

#### Scenario: Configured loose files resolve in the effective game directory

- **WHEN** valid optional maplist/config/smartmap files are supplied in the effective OpenTDM game directory and native runtime CWD is elsewhere
- **THEN** the same files are found and used by the relevant native module consumers, including allowed versus unlisted map behavior and config-vote discovery
- **AND** an existing correctly supplied file is not silently ignored because it was resolved relative to runtime CWD.

#### Scenario: Config scan skips invalid entries and terminates

- **WHEN** the effective `configs/` directory contains an entry with an invalid name or a non-`.cfg` name followed by a valid `.cfg`, or only invalid entries
- **THEN** the scan finishes, lists exactly the valid `.cfg` files (none in the second case), and the server keeps running
- **AND** an empty or missing `configs/` directory keeps its existing optional behavior.

### Requirement: OT-06 Truthful status and ordinary OpenTDM HUD

OpenTDM MUST preserve its current ordinary team/time/vote/scoreboard HUD and spectator views. Protocol 36 MUST NOT be treated as proof of OpenFFA extended-layout behavior or as a request to add that behavior to OpenTDM.

#### Scenario: Ordinary scoreboard remains usable

- **WHEN** a player uses the current score binding, opens team/score/time/vote displays, or enters spectator/chase
- **THEN** the ordinary OpenTDM HUD reflects the real game state and remains readable
- **AND** compatibility is checked through the actual OpenTDM scoreboard behavior without requiring the unused extended-layout userflag or another module's HUD branch.

### Requirement: OT-T01 Pending attack reaches the actual weapon action

Допустимое нажатие между разрешёнными оружейными обновлениями MUST сохраняться до обработки реального выстрела. Его потребление MUST учитывать оружейный callback, включая Machinegun и Hyperblaster. Одно pending-нажатие MUST NOT давать серию выстрелов, обходить cooldown или подменять реальное удержание гранаты.

#### Scenario: Short tap in all six phases

- **GIVEN** игрок может стрелять выбранным оружием и имеет ammo
- **WHEN** tap/release проходит настоящую обработку пользовательского ввода в каждой из шести фаз между 100-миллисекундными оружейными обновлениями
- **THEN** MG/HB и контрольное дискретное оружие выполняют один допустимый выстрел с соответствующим расходом ammo
- **AND** тест не ограничивается состоянием latch или вызовом generic weapon function; held fire сохраняет прежнюю скорострельность.

#### Scenario: Timeout and invalidated input

- **WHEN** pending attack пересекает timeout→resume, countdown, death/respawn, spectator, weapon switch, no-ammo либо map transition
- **THEN** старое недопустимое событие не становится выстрелом после восстановления контекста
- **AND** ручная граната сохраняет реальный release между weapon updates, hold до взрыва, death/switch, одну гранату и расход ammo по собственному исходному балансу OpenTDM.

#### Scenario: Pending tap while the grenade is cooking

- **WHEN** во время взвода ручной гранаты поступает короткое нажатие, сохранённое как pending attack
- **THEN** момент броска определяется только реальными hold/release игрока и не сдвигается
- **AND** pending attack другого оружия работает как прежде.

### Requirement: OT-T02 Time-based hurt and source-aware crush

Hurt и crush MUST сохранять реальные интервалы и существующую семантику источника независимо от world tickrate. Первый допустимый контакт MUST NOT теряться из-за глобальной фазы. Новое ограничение повторного crush MUST различать независимые пары источник/потерпевший и MUST NOT отменять физический blocked/reversal. `rotating_touch` вызывается из обработки касаний каждого ClientThink (input-driven) и MUST сохранять исходное поведение: новый world-tick limiter на него не распространяется.

#### Scenario: Normal and slow hurt with independent sound

- **WHEN** непрерывный контакт начинается в любой из шести фаз при 60 Гц
- **THEN** normal hurt допускает 10 ударов в полуоткрытой первой секунде, SLOW сохраняет секундный интервал, звук имеет отдельный секундный cooldown
- **AND** прежняя shared-source политика, SILENT и NO_PROTECTION сохранены; урон не умножается на частоту мира, звук не исчезает из-за modulo-фазы.

#### Scenario: Classify continuous, throttled and one-shot sources

- **WHEN** проверяются door/plat/`rotating_blocked`, `rotating_touch`, train/secret door и падающий объект (неподвижный и над движущимся игроком)
- **THEN** непрерывные воздействия, включая повторные удары падающего объекта, сохраняют 10-герцевый damage budget без повторов одного тика; train/secret сохраняют свой 0,5-секундный shared-source cooldown
- **AND** разовый удар, удаление debris и немедленная реакция blocked проходят независимо от ограничителя повторного damage
- **AND** `rotating_touch` наносит урон на каждый ClientThink, как при 10 Гц (две команды в одном world tick — два удара).

#### Scenario: Independent contacts and reused entities

- **WHEN** новый continuous-source limiter обслуживает две жертвы, два источника либо переиспользованную entity
- **THEN** независимые контакты не подавляют друг друга, устаревшее состояние не переносится на новую entity
- **AND** сохранение прежнего shared-source cooldown train/hurt не ошибочно заменяется новым per-victim правилом.

#### Scenario: Limiter capacity is exhausted

- **WHEN** число одновременно активных пар источник/потерпевший превышает ёмкость таблицы ограничителя
- **THEN** каждый потерпевший получает непрерывный crush не чаще ограниченного интервала, первый удар немедленный
- **AND** нет пути, где непрерывный crush наносится без ограничения на каждом world tick, а состояние не переходит к переиспользованной entity.

### Requirement: OT-T03 Accelerated movers keep their control interval

Ускоренный mover MUST сохранять существующий 100-миллисекундный профиль контроллера, включая остаточный финальный сегмент. Финальная скорость MUST NOT вычисляться как прохождение остатка control interval за один более короткий world tick. Длительность движения MUST отделяться от задержки запуска на следующий тик.

#### Scenario: Nonzero final remainder

- **WHEN** ускоренный механизм проходит маршрут с ненулевым остатком при 10 и 60 Гц
- **THEN** остаток проходит с предусмотренными скоростью и control interval, callback завершения вызывается один раз
- **AND** endpoint на координатной сетке достигается с ошибкой не более 1/8 по оси, а внутри 100-миллисекундных участков не накапливается ошибка квантования; общий carry согласован с rollback
- **AND** сравнение начинается с первого фактического движения; допустимое сокращение dispatch delay не маскирует ускоренный финальный участок.

#### Scenario: Shared completion helper

- **WHEN** тот же модуль выполняет линейный маршрут, reversal или blocked movement
- **THEN** исправление accelerated tail не навязывает всем движениям 100-миллисекундный последний шаг
- **AND** профиль, пассажир и физическая реакция соответствуют типу механизма.

### Requirement: OT-T04 Quantized movement does not accumulate drift

Перевод линейного движения на координатную сетку MUST сохранять среднюю скорость без накапливающейся ошибки и достигать endpoint. Исправление MUST NOT состоять из большого финального скачка. Новое дробное состояние MUST участвовать в откате составного push и очищаться при смене жизненного цикла механизма. Игрок, стоящий на движущемся механизме, MUST сохранять опору независимо от частоты прихода usercmd.

#### Scenario: One hundred units at one hundred units per second

- **WHEN** линейный mover проходит 100 единиц со скоростью 100 единиц/с при 60 Гц без препятствия
- **THEN** после предусмотренной длительности достигнута точка 100 с ошибкой не более 1/8 единицы по оси
- **AND** движение не заканчивается на 97,5, не исправляется последним скачком 2,5 единицы и вызывает завершение ровно один раз.

#### Scenario: Team rollback and reinitialization

- **WHEN** второй компонент team mover блокируется после перемещения первого и пассажира, затем движение возобновляется или разворачивается
- **THEN** откат сохраняет согласованность позиций и нового дробного остатка всех затронутых частей
- **AND** новый маршрут, teleport/reset и повторное использование entity не наследуют чужое накопление.

#### Scenario: Rider on a descending platform without fresh input

- **WHEN** игрок стоит на опускающейся платформе при 10 и 60 Гц, а usercmd приходят каждый тик, 30 или 10 раз в секунду либо одним чтением на пачку
- **THEN** на каждом world tick игрок сохраняет платформу опорой и остаётся на ней с относительной высотой не более 1/8 единицы
- **AND** нет событий падения и потери здоровья, звуки старта и остановки звучат по одному разу на цикл, циклический звук выключается после остановки, длительность повторных циклов не меняется
- **AND** у entity без клиента прежнее правило сброса опоры сохраняется.

#### Scenario: Rollback with an idle rider

- **WHEN** составной механизм откатывается из-за блокировки, а его пассажир в этом тике не прислал usercmd
- **THEN** пассажир сохраняет опору на механизме и не падает после отката.

### Requirement: OT-T05 Match timers and displays retain their units

Match/countdown/timeout, vote, intermission, `players` и достижимые scoreboard MUST сохранять реальные секунды. Отображаемый положительный countdown MUST округляться вверх без float-ошибки, неположительный — к нулю. Автоматический screenshot MUST сохранять исходную задержку 0,5 с после начала соответствующего intermission.

#### Scenario: Integer countdown boundaries

- **WHEN** остаётся 180 или 300 кадров при 60 Гц, один кадр до/после целой секунды либо неположительное число кадров
- **THEN** HUD показывает соответственно 3 или 5 секунд, корректные соседние значения либо ноль
- **AND** проверены и match, и timeout строки, а не только отдельный числовой helper.

#### Scenario: Player time and screenshot delay

- **WHEN** игрок находится в матче 60 секунд, открывает `players`/достижимый scoreboard либо начинается intermission с auto-screenshot
- **THEN** человекочитаемое время равно одной минуте, screenshot ставится через прежние 0,5 секунды
- **AND** интервалы, которые означают порядок следующих шагов, не преобразуются без разбора в 100-миллисекундные задержки.

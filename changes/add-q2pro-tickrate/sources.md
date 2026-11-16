*Перенесено в q2trek/docs 2026-11; адаптация описана в ADAPTATION.md.*
*Что убрано и заменено при переносе: [ADAPTATION.md](../../ADAPTATION.md).*

# Основания и пределы исследования

## S2. Часы server/client и lifecycle

- `inc/common/utils.h`: `frametime_t`, `Com_ComputeFrametime` используют целый `100/div`; при 60 Гц получается 16 мс, что не равно 1/60 с. Рациональный credit и правила проекции в design — предлагаемое исправление, а не уже имеющееся поведение.
- `src/server/main.c`: `SV_Frame`, `SV_RunGameFrame`, `SV_GiveMsec`; один world step за вызов, внешний realtime, sleep/residual и hitch clamp, период command allowance.
- `src/server/server.h`, `src/server/init.c`: frame-rate macros, `set_frame_time`, initialization; variable FPS применяется при соответствующем game feature.
- `src/common/common.c`: внешний native loop, `Qcommon_Frame` error boundary, существующие преобразования elapsed. `com_eventTime` и `svs.realtime` не являются одной идентичной wall-clock переменной.
- `src/client/entities.c`: `CL_DeltaFrame`, keyframes, animation time и active state; `src/client/main.c`: `CL_SetClientTime` и input/send cadence; `src/client/parse.c`: `set_server_fps`, serverdata/map и valid-frame boundary; `src/client/client.h`: frame/time state.
- `src/client/demo.c`, `src/server/mvd.c`, `src/server/mvd/client.c`: sampling 10 Гц, renumbering, playback/seek, dummy input и waiting room. Изменение этих форматов для 60 не требуется.

Исходный арифметический дефект: планирование 60 шагов по 16 мс занимает 960 мс. Виртуальные примеры с целочисленным credit и floor/ceil подтверждают выбранную математическую модель; они не заменяют будущий production scheduler test.

## S3. Настройки соединения, loss и сетевые очереди

- `src/server/user.c`: `set_client_fps`, `SV_AlignKeyFrames`, `SV_ParseClientSetting`, packet/stringcmd handling. Частоты выбираются через GCD; `clc_setting` читает short index/value, неизвестный index игнорируется.
- `src/client/parse.c`: `CL_ParseSetting` читает long index/value, неизвестные settings не меняют timing. Это место для явного согласования режима; protocol 36 сам по себе новый смысл не доказывает.
- `src/common/net/chan.c`: `NetchanNew_Transmit` не повторяет outstanding reliable в каждом последующем пакете. Следующий unreliable snapshot может прийти после потери settings carrier; до применения настройки (обычно около RTT, без гарантированной границы) `set_server_fps` (`src/client/parse.c`) затем перепривязывает шкалу (TR-04, поздняя настройка).
- `src/server/send.c`: `SV_SendAsyncPackets` не шлёт новые reliable активным клиентам, но фрагменты уходят и оттуда, и из `SV_SendClientMessages`. `SV_RateDrop` и fragment backlog — разные причины недоставки нового frame.
- `src/common/net/net.c`: `MAX_LOOPBACK=4`, FIFO с масочной индексацией и потеря старых сообщений при переполнении. 8 — кандидат нового размера, требующий проверки фактической пачки, не доказанная универсальная граница.
- `src/server/entities.c`: frame/packed-entity history и delta encoding. Нельзя объявить исправлением произвольное увеличение history без проверки wire limits.
- `src/server/game.c`, `opentdm/g_tdm_vote.c`: команды могут ставиться при обработке пакетов, до `RunFrame`; `cmd_buffer` после шага не покрывает эту границу.

## S4. Моды: сохранение боевой семантики

- `opentdm/p_client.c`, `p_weapon.c`: ограничитель `next_weapon_think`, очистка latch и проверка live buttons MG/HB; real callbacks нужны для доказательства выстрела. Timeout возвращает ClientThink до обычного обновления ввода, поэтому resume требует отдельного lifecycle-теста.
- `openffa/p_client.c`: `weapon_thunk`/`FRAMESYNC` и очистка `latched_buttons`; второй tap между weapon updates требует проверки, но его дефект не объявлен воспроизведённым без настоящего API3-сценария.
- `openffa/p_weapon.c`: `GRENADE_TIMER=3×HZ`, добавка `+2` при взведении и целочисленное деление в charge curve. Одно и то же реальное remaining 1 с даёт исходные 660 при 10 Гц против 640 при 60 Гц. Значение `+2` соответствует 0,2 с только при 10 Гц. `openffa/g_weapon.c::fire_grenade2` добавляет up/jitter к forward velocity, а при timer ≤ 0 немедленно взрывает entity: это учтено в oracles.
- `{openffa,opentdm}/g_trigger.c`: hurt damage/sound, SILENT/SLOW/NO_PROTECTION. FFA уже нормализует normal damage с интервалом 0,1 с; звук может фазово исчезать. В TDM normal damage ещё завязан на один world tick.
- `{openffa,opentdm}/g_func.c`, `g_misc.c`, `g_phys.c`, `g_combat.c`: source-specific damage/blocked, существующие 0,5 с train/secret cooldown, falling object (первый удар сразу; над движущимся игроком повторяется каждый world tick через сброс `groundentity` по `linkcount` — найдено в группе 1) и rotating touch. Общего предложенного фильтра `MOD_CRUSH` в коде нет; спецификация отвергает его как ошибочное будущее решение, не требует «откатить уже сделанный gate».
- `opentdm/g_func.c`: `Think_AccelMove` имеет control interval 0,1 с, общий `Move_Final` использует FRAMETIME. `SV_Push` в `g_phys.c` округляет translation до 1/8; идеальный remaining не компенсирует накопленное отклонение фактической позиции. Маршрут TDM длиной 100 единиц со скоростью 100 единиц/с при 60 Гц ранее воспроизведён с endpoint 97,5 в изолированной модели настоящих helpers.
- `openffa/g_func.c`: пересчёт фактического endpoint и ускорение с dt/dt². Такой же endpoint-дефект здесь не объявляется доказанным; тест нужен для drift/финального скачка и сохранения физики.
- `{openffa,opentdm}/g_main.c`: API3 initialization и save callbacks. В этих срезах OpenFFA write/read callbacks пусты, OpenTDM назначает DummyWrite/DummyRead. Полноценное сохранение игр ради carry не добавляется.

Ранее выполненные изолированные C-proofs подтвердили отдельные weapon/damage/mover/HUD случаи. Они используют извлечённые функции/ограниченную среду: это исследовательские свидетельства, не покрытие production после будущих изменений. Спека требует заменить такую роль проверками настоящих модулей.

## S5. Потребители времени и диагностика

- `opentdm/g_tdm_core.c`: match/timeout `ceil((float)remaining×FRAMETIME)`, screenshot через 5 world frames, clock transitions; `g_cmds.c` и `p_hud.c`: `/600` в player/scoreboard time.

- `src/client/parse.c`: `cls.demo.frames_read` увеличивается после принятого валидного frame; `cl_shownet` печать сама по себе не означает принятие.
- `src/server/server.h`, `src/server/send.c`, `src/server/main.c`: debug conditions `SV_DPrintf`/`sv_debug`; `developer 1` не доказывает отсутствие suppression.

## S8. Ручная сессия и внешнее ревью (2026-10-28)

- Сверено по коду: `{openffa,opentdm}/g_main.c` `G_RunFrame` (сброс опоры), `*/p_client.c` `ClientThink` (единственное обновление `groundentity_linkcount`), `src/server/user.c` (вызов `ClientThink` только при разборе usercmd), `src/client/input.c` (`cl_maxpackets` 30 по умолчанию), `src/common/pmove/template.c` (проверка земли на 0,25), `openffa/p_view.c` (пороги fall events), `*/g_utils.c` `G_CrushAllowed` (`return true` при переполнении), `tickrate-tests/mods/api3/harness.c` `h_tick` (ClientThink каждый тик, msec 16).
- Справочно: Quake II Rerelease `rerelease/g_main.cpp` — та же перепроверка опоры трассой на 1 единицу для не-монстров (design D12, отклонённый вариант).

*Перенесено в q2trek/docs 2026-11; адаптация описана в ADAPTATION.md.*
*Что убрано и заменено при переносе: [ADAPTATION.md](../../../../ADAPTATION.md).*

# Spec Delta

## ADDED Requirements

### Requirement: OT-T06 Presentation and environment damage keep the 100-millisecond cadence on the real clock

Обратная связь по урону, смещение оружия, view kick, bob, урон лавы/слизи и предупреждения о
powerup MUST при любом `sv_fps` из диапазона 10…60 вести себя как при 10 Гц: пачками по 100 мс
реального времени, с теми же суммами и теми же звуками. Каденция MUST идти по реальным часам, а не
по матчевым, поэтому во время таймаута матча presentation MUST продолжать обновляться с той же
частотой, а предупреждение о скором окончании powerup MUST NOT повторяться.

#### Scenario: Damage flash and blend are batched per 100 ms

- **WHEN** игрок получает урон несколькими ударами в разных кадрах одного 100-миллисекундного окна
  при 10 и 60 Гц
- **THEN** flash длится 100 мс реального времени, суммы урона и blend за окно совпадают с 10 Гц
- **AND** число звуков боли за окно равно одному, как при 10 Гц.

#### Scenario: Gun offset, view angle history and view kick advance once per 100 ms

- **WHEN** игрок поворачивает камеру и получает view kick при 60 Гц
- **THEN** смещение оружия и история углов меняются ровно один раз за 100 мс, а kick применяется и
  удерживается столько же, сколько при 10 Гц
- **AND** в воздухе bob равен нулю при обоих тикрейтах.

#### Scenario: Lava and slime damage and burn sound at 60 Hz

- **WHEN** игрок стоит в лаве или слизи одну секунду при 10 и 60 Гц
- **THEN** число ударов и суммарный урон совпадают, звук горения звучит с прежним debounce после
  любого из двух звуков
- **AND** звук не исчезает и не учащается из-за фазы кадра.

#### Scenario: Powerup fade warning once and never during timeout

- **WHEN** у powerup остаётся 3 секунды при 60 Гц, в том числе когда матч в таймауте
- **THEN** предупреждение выводится не более одного раза за powerup, а в таймауте не выводится вовсе
- **AND** допускается потеря единственного предупреждения, если таймаут начался в тот же кадр, когда
  оно должно было прозвучать.

#### Scenario: Fall and drown pain debounce covers the batched frame

- **WHEN** игрок получает урон от падения или утопления при 60 Гц
- **THEN** штатный звук боли не звучит в кадре пачки поверх звука падения/утопления, как при 10 Гц
- **AND** debounce длится те же 100 мс матчевого времени, что и при 10 Гц.

### Requirement: OT-T07 Rate-limited commands and ping handicap scale with the tickrate

Ограничители частоты команд (`ghost` и другие команды с TDM rate limit) и jitter ping-handicap
MUST выражаться в реальных секундах и давать при 60 Гц ту же частоту событий, что при 10 Гц.

#### Scenario: Ghost command rate limit is 0.2 s at any sv_fps

- **WHEN** игрок повторяет `ghost` каждый кадр при 10 и 60 Гц
- **THEN** команда принимается не чаще одного раза в 0,2 с реального времени при обоих тикрейтах.

#### Scenario: Ping handicap jitter pattern on real seconds

- **WHEN** scoreboard с ping-handicap строится в течение 2 секунд при 10 и 60 Гц
- **THEN** последовательность поправок (+3 / −2) и моменты их смены по реальному времени совпадают с
  10 Гц.

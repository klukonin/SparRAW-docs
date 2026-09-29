# Рандомизатор отсрочки маяка 4.1: опыты на железе

Опыты с переписанными на C++ блоками ucode 4.1.0.1000, дополняющие документацию
прошивки. Механизм рандомизатора (гейт `bi_mode == 2`, LFSR r47, `uniform × 1024 мкс`,
накопитель 0x801450, компаратор TSF 0x886d30/34/38), причины, по которым он не сдвигает
маяк, и сводные замеры —
[BEACONING.md §4](https://github.com/klukonin/SparRAW-firmware/blob/main/docs/BEACONING.md#4-рандомизатор-отсрочки-маяка-обе)
и [§9.2](https://github.com/klukonin/SparRAW-firmware/blob/main/docs/BEACONING.md#92-рандомизатор-маяк-не-сдвигает-41-железо--код);
однобайтовый патч гейта и его проверка —
[4.1/docs/DISTBCN-PATCH.md](https://github.com/klukonin/SparRAW-firmware/blob/main/4.1/docs/DISTBCN-PATCH.md);
переключатели сборки — [REWRITING.md](https://github.com/klukonin/SparRAW-firmware/blob/main/docs/REWRITING.md#переключатели-сборки);
биты событий и примитивы ожидания — [UCODE-EVENT-BITS.md](UCODE-EVENT-BITS.md).

Во всех опытах память узла (`blob_uc_code`) сверена со сборкой побайтово; лог ucode
читается сборщиком трассы ([TRACING-TOOLKIT.md](TRACING-TOOLKIT.md)).

## Развёртка вызывается на пути TBTT **[код + железо]**

`l1_task__entry` на аппаратном TBTT (r42 бит 4 `bi1_event`) зовёт
`bi_ap_mon_if__trigger` со входом 0 и аргументом 0 (аргумент задаётся в слоте задержки):

```
9298c4  mov_s r0,0        ; code = 0
9298c6  bl.d  0x9304c8
9298ca  mov_s r1,r0       ; слот задержки: arg = 0
```

Вход 0 диспетчера (тело 0x9304e4) передаёт аргумент насквозь (`mov r2,r13` 0x930508 →
`basic_sm__dispatch_loop`), поэтому `ctx->kind = 0` и `brne r0,0x1` 0x93124c уводит в
развёртку 0x923de0. Путь `kind = 1 → bti_transmitter_bi2_flow` приходит из
`bi_manager__run_rx_flow_by_kind` по биту 19 `busy_event`; `bti_transmitter_bi2_flow`
(каталог `bf_gpc_service`, ассерт `bf_gpc_service.h:134`) — не передатчик: он обслуживает
окна GP0/GP1/GP2 для сбора BF-метрики и не кладёт в кольцо TX-команд. Из восьми вызывающих
`bi_ap_mon_if__trigger` только один жёстко передаёт `mov r2,0x1`.

Замер на AP: `ctx->kind = 0` (контекст 0x801e94), `bcon_kind` (0x801438) = 1, r41
принимает ровно два значения — 0 и 0x40000000 (бит 30 взведён). Развёртка вызывается, но
рандомизатор внутри неё закрыт условием `bcon_kind == 2`.

## Гейт открыт на время развёртки **[железо]**

Переписанный `bi::decide_beacon_kind` подменяет 0x801438 на 2 только на время вызова
развёртки (его также читают `bi_ap_mon_if::trig3_aw_or_dti` и `bi_cfg::tick_counters`).
Сборка `make fw DEFS=-DFORCE_BEACON_SWEEP=1`. В логе ucode:

```
bti_worker::bti_transmitter_beacon_sweep_flow() randomize beacon
bti_worker::bti_transmitter_beacon_sweep_flow() current_tsf: 0x0101cfff,
        Next tsf: 0x01056800, uniform_random: 128
set_tsf_event()
```

`uniform_random` 128, 102, 93, 137, 90; компаратор взводится каждый BI. Фазы начала BTI
по `MAC_MON`: `ff9, ff9, ffb, ff9, ff9`, интервалы 102 400 мкс, линк 987 Мбит/с.

## Ожидание TSF опросом **[железо]**

Вместо примитива ожидания — опрос `uc_read_tsf_lo` (0x9315e8: команда MAC 0x4d000005, три
`nop`, ответ в r55) со сторожем после `set_tsf_event`. Ucode работает, маяк ходит, линк
цел, но сторож исчерпывается всегда: длительность BTI растёт с ~1375 до ~11 550 мкс.
Причина — цель в прошлом/далёком будущем:

```
current=0x00513ffd  next=0x006d9800  rand=20
current=0x0052cffd  next=0x006f7400  rand=119
current=0x00545ffd  next=0x00700400  rand=36
```

`Next tsf` отстоит от текущего на ≈ 1,9 млн мкс (≈ 18 BI).

## Привязка накопителя к TSF **[железо]**

* Внутри `set_tsf_event` (`текущий TSF + (цель − TSF) mod BI + 200 мкс` через
  `uc_read_tsf64` 0x930cb0) — счётчик BTI замирает: `uc_read_tsf64` переключает общее
  теневое слово команды MAC (`[gp,-0x44]` = 0x8004e4) и шлёт команды в кольцо r25
  посреди такой же последовательности развёртки.
* В начале обработки BTI в переписанном `bi::decide_beacon_kind`, до собственных команд
  MAC (`make fw DEFS="-DFORCE_BEACON_SWEEP=1 -DRESYNC_TSF_BASE=1"`): работает.

```
current=0x01a8fffe  next=0x01abeffb  rand=188   +192 509 мкс
current=0x01ac2000  next=0x01ad2ffc  rand=68    +69 628 мкс
current=0x01af3ffe  next=0x01b05ffb  rand=72    +73 725 мкс
```

`next − current = rand · 1024`; накопитель идёт вровень с TSF (за 4 с +3 990 016 мкс),
счётчик BTI растёт. Интервалы маяка остаются 102 400 мкс, фаза `ff9`/`ffa`.

## Ограничения переписанных блоков ucode

* Статических переменных в C++-блоках ucode быть не может: переменная легла по 0x939270 —
  память кода, куда ucode не пишет, а в памяти данных ucode свободного места нет
  (в снимке `blob_uc_data` нет ни одного нулевого участка длиной 64 Б). Только стек.
* Образец-уловитель `*(.rodata .rodata.*)` в пуле скрипта размещения утаскивает сегмент
  данных, и запись `data` образа собирается пустой (драйвер: `__fw_handle_data: ERR data
  record too short: 4`, прошивка не грузится — внешне похоже на зависание ucode). Лечится
  `EXCLUDE_FILE`, который в этой версии ld относится только к ближайшему следующему
  образцу и пишется перед каждым.

## Не установлено

* Смысл аппаратного события компаратора TSF и его обработчик.
* Положение переданного кадра маяка внутри BTI при работающем рандомизаторе (метрика
  `MAC_MON [BTI] start time` его не показывает —
  [BEACONING.md §8](https://github.com/klukonin/SparRAW-firmware/blob/main/docs/BEACONING.md#8-отчёт-mac_mon-обе)).
* Способ сдвигать передачу маяка внутри BTI без перепрограммирования конфигурации BI.

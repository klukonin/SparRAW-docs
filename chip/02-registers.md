# 02. Периферийные регистры

Регистры Sparrow (wil6210), которыми пользуются прошивка (fw) и микрокод (ucode).
Два разных класса:

* **периферия RGF** — память-отображённые регистры 0x880000..0x88c200, видимые
  и процессорам, и хосту (debugfs);
* **регистровый файл MAC локального чтения** (`MSXD_LR_RGF`) — ARC-регистры
  `r36..r56` микрокода; с хоста не видны.

Версия: **[4.1]**, **[6.2]**, **[обе]**. Полный справочник по адресам — генерируемая
карта [REGS-62](https://github.com/klukonin/SparRAW-firmware/blob/main/6.2/ref/REGS-62.md)
(6.2); по блокам —
[HARDWARE-BLOCKS](https://github.com/klukonin/SparRAW-firmware/blob/main/docs/HARDWARE-BLOCKS.md)
и [MAC-REGISTERS](https://github.com/klukonin/SparRAW-firmware/blob/main/docs/MAC-REGISTERS.md).
Архитектура и карта памяти — [01-architecture.md](01-architecture.md); команды MAC —
[03-mac.md](03-mac.md).

---

## 1. Источник карты и доступ

* **Карта 6.2** строится автоматически: `tools/regmap.py` собирает обращения из
  дерева исходников, смысл берётся из `ref/REGS-KNOWN.txt` (ведётся руками, с
  источником в скобках), `make regs` выдаёт REGS-62. 870 регистров с обращениями,
  все описаны; **37 без доказанного смысла** (§9); у 47 записаны значения со
  стенда. Около четверти строк имеют вендорское имя (драйвер `RGF_*`,
  globals/wmiUT/debug-tools пака 11ad), около трети — смысл по лог-строкам,
  остальные — «по использованию».
* regmap не видит адресов, собранных сдвигом (`0x11<<19`), вычисляемых индексов
  и базы после ветвлений — такие регистры вносятся в REGS-KNOWN вручную.
* **Адресация ucode от базы 0x887000 отрицательными смещениями**
  (`mov r2,0x887000; st.as rX,[r2,-0x2d0]`) — поиск по абсолютному адресу такие
  обращения не находит [обе]. В 6.2 через эту базу `bi_mode_init_sequence` пишет
  0x886dc0, 0x886dac, 0x886d50, 0x886f78, 0x886f7c.
* **Чтение с хоста** — debugfs `mem_addr` + `mem_val`, ~5 мс на регистр, регион
  `rgf` без смещения. Снимок всего окна занимает ~17 мс, поэтому быстрые биты по
  периодичности не ищутся [4.1].
* **Запись с хоста**: штатно только ICR/DMA/ITR (`doff_io32`), остальное —
  `mem_write`. Запись в регистры, которыми управляет ucode, — борьба с прошивкой
  за ресурс ([HARDWARE-BLOCKS, «Доступ с хоста»](https://github.com/klukonin/SparRAW-firmware/blob/main/docs/HARDWARE-BLOCKS.md#доступ-с-хоста)).

---

## 2. Общая карта областей

Разбивка по страницам 0x100 из REGS-62 (подсистемы — по именам блоков, которые к
ним обращаются) [6.2]; для 4.1 карта по областям та же, если не сказано иное.

| диапазон | подсистема | что там |
|---|---|---|
| 0x880000–0x8800ff | USER_RGF: загрузка, GPIO | `chip_stepping_id`, `RGF_USER_USAGE_1..8` (адреса логов, OOB-биты), выбор функций GPIO |
| 0x880100–0x8801ff | SPI-флеш, состояние CPU | окно данных SPI 0x8800a8.., `spi_flash_ctrl/cmd/addr`, `RGF_USER_HW_MACHINE_STATE` 0x8801dc, `RGF_USER_USER_CPU_0` 0x8801e0, `RGF_USER_MAC_CPU_0` 0x8801fc |
| 0x880200–0x8802ff | таймеры, halt, почтовые ящики | таймеры планировщика A/B 0x880208..0x880258, счётчик времени 0x880254, halt 0x880270..0x880280, RF kill 0x880288, `RGF_USER_USER_SCRATCH_PAD` 0x8802bc (mbox_ctl WMI) |
| 0x880300–0x8804ff | почтовые ящики | кольца дескрипторов рабочего (WMI) и отладочного ящиков |
| 0x880a00–0x880aff | версии, платформа, ревизии | версия загрузчика, 0x880a3c платформа ASIC/FPGA, 0x880a8c `rev_id`, 0x880a90 `RGF_USER_FW_CALIB_RESULT`, 0x880abc `RGF_USER_CLKS_CTL_0` |
| 0x880b00–0x880cff | тактирование, сброс, user-ICR | `SW_RST_VEC_0..3` 0x880b04.., JTAG ID 0x880b34, **user-ICR** с базой 0x880b4c, карты режимов питания, OTP/XPM 0x880ce0.. |
| 0x881000–0x881dff | DMA | дескрипторные очереди `DMA_RGF.DESCQ<n>`, L2-offload, ICR DMA (EP TX/RX/MISC, fw-блоки 0x881c08/24/40), модерация ITR |
| 0x882000–0x882fff | PCIe | SerDes, L1SS, DBI, PERST, `RGF_PAL_UNIT_ICR` 0x88266c, отладочный ICR 0x882688, событие хосту 0x8826b0, LTSSM-ICR 0x882fcc, карты режимов питания |
| 0x883000–0x8851ff | PHY | RX/AGC, статистика приёма, BRP-измерения, запись/воспроизведение, TX (предыскажения, аттенюатор), SAR |
| 0x884800 | отладка ucode | `uc_assert_code` |
| 0x886000–0x8860ff | парсер MAC | свой/групповой адреса фильтра, `mac_irq_ctl_886028` |
| 0x886100–0x8862ff | режимы, фильтр | карты режимов 0x886100.., `MAC_PRS_CTRL_0`, `PRS_ICR` 0x88620c |
| 0x886400–0x886cff | очереди MAC | RX-очереди, TX, `mac_mgmt_seq_num` 0x886544, окно дескрипторов очередей, таблицы CID/MCS |
| **0x886d00–0x886dff** | **MAC_SXD** | IFS, таймеры BI, компаратор TSF, звонок fw→ucode, QSET, **порт core write 0x886dc8**, свой адрес |
| **0x886e00–0x886fff** | MAC: время, ICR, TXOP | тайминги LMAC 0x00, `MAC_HW_ICR`, `MAC_SXD_ERRORS`, отчёты PPDU, NAV, **TSF**, **мкс до TBTT**, пределы TXOP, слоты расписания, **TSF начала BI** |
| 0x887000–0x8878ff | счётчики MAC | командный регистр 0x887008, два блока ICR (векторы fw 10/11), настройки/селекторы счётчиков |
| 0x889000–0x8896ff | РЧ-контроллер, ABIF, ToF | `rfc_core_*`, датчики, синтезатор, `RGF_CAF_ICR` 0x88946c, ToF |
| 0x88a000–0x88afff | ABIF (`AGC_tbl`) | TX-таблица 0x88a004, RX-таблица (AGC) 0x88a208, режимы CAF, АЦП |
| 0x88b000–0x88bfff | `rgf_ext` | PCIe-расширение |
| 0x88c000–0x88c1ff | `mac_rgf_ext` | регистры расписания (`mac_ext_sched_*`), остаток окна 0x88c120, измерение 33 кГц |

---

## 3. Модель прерываний RGF_ICR

Каждый источник прерываний — блок `struct RGF_ICR` драйвера из 7 слов [обе]:
ICC +0x00, **ICR** +0x04 (причина, W1C), **ICM** +0x08 (причина с учётом маски —
её читает обработчик), **ICS** +0x0c (программно поднять причину), **IMV** +0x10
(маска, 1 = замаскировано), **IMS** +0x14 (замаскировать бит), **IMC** +0x18 (снять
маску). Модель подтверждена во всех блоках 6.2. Типовой обработчик читает ICM,
пишет то же значение в ICR; источник, который надо придержать, маскируется IMS на
время обработки и открывается IMC
([HARDWARE-BLOCKS, «Прерывания»](https://github.com/klukonin/SparRAW-firmware/blob/main/docs/HARDWARE-BLOCKS.md#прерывания--блоки-struct-rgf_icr-драйвер),
векторы fw — [HW-DRIVERS §2](https://github.com/klukonin/SparRAW-firmware/blob/main/docs/HW-DRIVERS.md#2-прерывания-fw-вектора-arc600-и-user-icr-обе)).

| база (ICC) | блок | кто обслуживает | биты / примечание |
|---|---|---|---|
| 0x880b4c | `RGF_USER_USER_ICR` | fw, вектор 4 | 2 таймер `u_schd`, 4 Awake TSF, **9 событие ucode** (ucode ставит через ICS 0x880b58), **18 WMI-ящик** (SW_INT_2), 19 отладочный ящик, 27 RF kill |
| 0x881bb4 / 0x881bd0 | `RGF_DMA_EP_TX_ICR` / `_RX_ICR` | драйвер | DMA хоста |
| 0x881bec | `RGF_DMA_EP_MISC_ICR` | fw поднимает прерывание хосту | ICS 0x881bf8: бит 28 FW_READY, 29 MBOX_EVT, 31 FW_ERROR; ICR бит 27 HALP — запрос хоста на пробуждение |
| 0x881c08 / 0x881c24 / 0x881c40 | fw-ICR DMA | fw | завершения TX-колец / PRING / планировщик vring |
| 0x881c68 | `RGF_DMA_PSEUDO_CAUSE` | драйвер | `MASK_SW` 0x881c6c, `MASK_FW` 0x881c70 |
| 0x88266c | `RGF_PAL_UNIT_ICR` | fw, вектор 16 | 4/5 PERST assert/deassert, 24 D3→D0, 29 вход в D3, 18 событие канала |
| 0x882688 | отладочный ICR PCIe | fw, вектор 15 | 3 дедлок PXE, 12 таймаут CPL |
| 0x8826a4 | событие хосту PCIe | fw | 0x8826b0 (как ICS): 0 FW ready, 2 «DRIVER is UP», 0x19 sysassert |
| 0x882fcc | `UNIT_ICR_PORT0_LTSSM_DBG_1` | fw, вектор 18 | 14 вход в L1, 19 выход из L1 |
| 0x88620c | `PRS` (парсер) | fw при подъёме MAC | сброс/открытие 0xa3ff; ucode печатает `PRS_ICR` при крахе |
| 0x886434, 0x886504, 0x88683c | блоки MAC | fw при подъёме MAC | 0x1f / 3 / 7 — сброс и открытие |
| **0x886dcc** | **fw→ucode** | ucode, L1-задача | **0 новая команда в кольце LMAC, 1 останов/смена режима MAC, 2 выход из сна** |
| 0x886e20 | `MAC_SXD.MAC_HW_ICR` | ucode (INT6) | сторож ucode квитирует бит 7; fw печатает его при сисассерте 0x26 |
| 0x886e3c | `MAC_SXD_ERRORS` | fw, вектор 6 | 15 «Awake TSF interrupt expired», 6 «LMAC CRASH» (ucode сам ставит через ICS 0x886e48) |
| 0x88700c / 0x887028 | счётчики MAC | fw, векторы 10 / 11 | только квитирование |
| 0x88946c | `RGF_CAF_ICR` | fw, вектор 13 | 2 PLL5 UNLOCK, 7 FS8 UNLOCK |

Имена `RGF_*` — из драйвера, безымянные базы установлены по раскладке и
использованию.

**Звонок fw→ucode (блок 0x886dcc) [6.2]:** ICR 0x886dd0 — ucode квитирует, fw ждёт
сброса бита; ICM 0x886dd4 — читается в начале L1-задачи ucode (биты 0..2 → разные
L1-события); ICS 0x886dd8 — fw пишет 1 после постановки команды в кольцо 0x804280,
2 при `POWER_MNGR__halt`, 4 при выходе из глубокого сна; IMV 0x886ddc — 0xf
маскирует всё (окно AW), 8 открывает биты 0..2; IMS 0x886de0 — ucode маскирует
источник на время обработки (биты 1, 2 — с откладыванием в фоновую задачу); IMC
0x886de4 — снятие маски после опустошения кольца и смены режима MAC. В 4.1 та же
точка: `basic_if__mailbox_send` звонит записью `[0x886d80+0x58] = 1`, фоновая
задача ucode проверяет биты 0/1 регистра 0x886dd0 перед сменой режима MAC.
События времени BI приходят к ucode не через этот блок, а через `r42`/`r54` (§5).

**Не ложатся в модель:** 0x886028 `mac_irq_ctl_886028` (вектор 12) — при подъёме
MAC пишется 0xffffff88, в обработчике −1; маска это или W1C — не установлено
(0x886018..24 заняты адресами фильтра). 0x886fbc/0x886fc0 по форме похожи на
ICC/ICR, но 0x886fc4 занят TSF начала BI.

---

## 4. Регистры USER_RGF

| адрес | имя | смысл | версия |
|---|---|---|---|
| 0x880000 | `chip_stepping_id` | сравнивается с JTAG ID Sparrow A0 0x0632072f / A1 0x1632072f → обход «PCIe serdes shlicht» | обе |
| 0x880004 | `RGF_USER_USAGE_1` | адрес кольца лога **fw** (0x843900 в обеих версиях); драйвер обнуляет до старта | обе |
| 0x880008 | `RGF_USER_USAGE_2` | адрес кольца лога **ucode** (4.1 0x80209c, 6.2 0x803234 — адреса ucode) | обе |
| 0x88000c / 0x880010 | `RGF_USER_USAGE_3/4` | ucode при старте пишет 0x800000 и gp+0x30 | 6.2 |
| 0x880018 | `RGF_USER_USAGE_6` | от хоста: бит 31 `BIT_USER_OOB_MODE`, бит 30 `BIT_USER_OOB_R2_MODE`, бит 0 «FW загружена хостом»; fw читает биты 31..29 целиком: 4 → R1 OOB, 2 → R2 OOB | обе |
| 0x88001c | `RGF_USER_USAGE_7` | переживает перезапуск FW: бит 0 «повторный старт», биты 1..8 — сохранённые настройки PM PCIe | 6.2 |
| 0x880020 | `RGF_USER_USAGE_8` | от хоста: PREVENT_DEEP_SLEEP, SUPPORT_T_POWER_ON_0, EXT_CLK; 6.2 не читает | 6.2 |
| 0x8801d8 | `hw_sm_wa_ctl` | «MAIN() HW statemachine WA»: 0x80000000, ожидание кода 0x15, 0x40000000 | 6.2 |
| 0x8801e0 | `RGF_USER_USER_CPU_0` | бит 1 сброс user-CPU; биты 6..8 — банки памяти под запись PHY | обе |
| 0x8801e8 | `RGF_USER_CPU_PC` | PC процессора (для крэшей) | обе |
| 0x8801fc | `RGF_USER_MAC_CPU_0` | управление MAC-CPU; 0x31 выпускает ucode из сброса | обе |
| 0x880208..0x880258 | `sys_timer_*` | таймеры планировщика; 0x880254 — свободно бегущий счётчик времени FW (не TSF) | 6.2 |
| 0x8802bc | `RGF_USER_USER_SCRATCH_PAD` | `wil6210_mbox_ctl` рабочего ящика WMI: tx-кольцо 0x8802e8 (5 записей), rx-кольцо 0x880318 (26 записей); за ним отладочный ящик 0x8803e8 | 6.2 |
| 0x8802e0 | scratch +0x24 | адрес дескриптора кольца команд ucode 0x840158 | 6.2 |
| 0x880a3c | `RGF_USER_BL` / платформа | 1 ASIC, 2 FPGA; начало `g_fw_dedicated_registers` | обе |
| 0x880a8c | `RGF_USER_FW_REV_ID` | baseband_type (3 A0 … 7 D0) и rf_type с 0x880a8e (1 Marlon, 2 Sparrow-R) | обе |
| 0x880a90 | `RGF_USER_FW_CALIB_RESULT` | от драйвера: RDAC и сигнатура 0x11 | обе |
| 0x880abc | `RGF_USER_CLKS_CTL_0` | бит 1: такт AHB от PLL 165 МГц | обе |
| 0x880b04..0x880b14, 0x880c18/0x880c2c | `SW_RST_VEC_*`, `EXT_SW_RST_VEC_*` | программный сброс блоков | обе |
| 0x880b34 | `RGF_USER_JTAG_DEV_ID` | JTAG ID (A0/A1/далее по маске) | обе |
| 0x880ce0, 0x880cec..0x880d64 | OTP/XPM | чтение OTP; драйвер OTP никогда не пишет | обе |

Заголовок кольца лога fw (адрес из 0x880004): слово write_ptr, затем
`u8 module_level_enable[16]`; запись в эти байты глушит шумные модули MAC_MON и
PHY_MON (host 0x90b904/0x90b908 при кольце 0x843900) [4.1]
([LMAC-PROTOCOL §12](https://github.com/klukonin/SparRAW-firmware/blob/main/docs/LMAC-PROTOCOL.md#12-глушение-шума-в-логе-прошивки-железо)).
Кольцо лога ucode [4.1]: 0x8020b0 (256 × 4 Б), индекс 0x80209c, маска разрешения
0x8020a0.

---

## 5. Регистровый файл MAC локального чтения (r36..r56)

ucode читает регистры MAC **командой MAC 0x1d**: она правит один полубайт теневой
копии селекторов (кэш в `[gp,-0x44]` = uc 0x8004e4), и через несколько тактов
результат появляется в ARC-регистре `r36..r56`. **Группа** в `MSXD_LR_RGF` =
регистр результата, **индекс** = значение селектора [обе]. Позиция полубайта:
биты 8..11 → `r41`, 12..15 → `r40`, 16..19 → `r47`, 20..23 → `r48`. Группу R55
читает отдельная команда 0x4d (0 = `SIFS_CNT`, 5 = `G2_TSF_LOW`, 0xa =
`GP_CLK2_TIMER`); в простое кольцо забито `0x4d000005` — непрерывное чтение TSF.
В ucode 4.1 — 65 запросов чтения из 825 записей в кольцо; из 69 разобранных чтений
67 названы вендорским файлом. «Local read» регистры в хостовом окне, по-видимому,
не видны. Идиома и число сайтов —
[MAC-REGISTERS](https://github.com/klukonin/SparRAW-firmware/blob/main/docs/MAC-REGISTERS.md#идиома-чтения-код).

Имена — из символов пака 11ad соседнего чипа wil6436 (1441 строка, выгрузка
[`6.2/ref/MSXD-LR-RGF.txt`](https://github.com/klukonin/SparRAW-firmware/blob/main/6.2/ref/MSXD-LR-RGF.txt)).
Совпадение проверено: группа R40 сошлась с индексами Sparrow 1:1 и по использованию
(TSF парой, 16-битный remaining time, 26 бит `bi_counter`); `rx_flow` ждёт ровно
`ppdu_report_event`, ответчик A-BFT — `rx_frame_event`. R42/R54/R47 подтверждены
использованием в коде и трассами; побитной проверки по всем группам нет.

| регистр | содержимое (индексы селектора) |
|---|---|
| r36..r39 | отчёты о последнем PPDU (`PPDU_REPORT_1..3`, `_MISC`); в R36 есть бит 7 `beacon_detected` (в ucode 4.1 не читается) |
| **r40** | 0 `BACKOFF_IFS_STATUS`, 3 `REMAINING_TIME_RD`, **4 `BI_COUNTER`**, 5/6 `TSF_LOW/HIGH`, 7 `REMAINING_BI`, 8/9 `GP_TIMERS_0_1/2_3`, a `PHY_SIGNALS` (самый читаемый, 15 мест в 4.1), b/c PLCP RX, d `BF_METRIC`, e `REMAINING_SLOT` |
| r41 | выбор очередей: `QUEUE_SEL_*`, `QSET_1..8_MASK_VECTOR`, b `MTP_Q_AVAIL`, c `BAP_Q_AVAIL` |
| **r42** | **`EVENT1_STATUS`** (события среды и времени), EVENT2/3, `EVENT_ENGINE_4_STATUS_7` |
| r43, r44 | backoff/IFS, TSF, SP; r44[1] `PHY_SIGNALS_AND_CCA_MONITOR` (cca, rx_on_air, tx_on_air) |
| r45 | 0 `MTP_TX_PERMISSION_RESP` (бит 31 valid), остаток времени, `BEACON_INTERVAL_CONTROL1` |
| r46 | конец передачи MTP, EVENT6, адрес источника |
| r47 | 0 `DIRECT_TX_CMD_STATUS`, **2 `LFSR_VAL`** (ГСЧ), статусы переключения RFC, `PMC_STATUS`, `SECTOR_CFG_STATUS` |
| r48 | поля принятого маяка: BI Control, TXSS, Capability, Timestamp, адреса |
| r49..r51, r56 | NAV: `NAV_MAX`, счётчики и адреса восьми записей NAV |
| r53 | BAP, `BCON_INTERVAL`, Duration |
| **r54** | **`EVENT4_STATUS`** (индикации GP-таймеров и служебных периодов), EVENT5/6 |
| r55 | `SIFS_CNT`, второй TSF/BI (`G2_*`), `GP_CLK0..2_TIMER` |

**BI_COUNTER** (r40, индекс 4): биты 0..25 — позиция внутри интервала маяка в мкс,
26..30 — резерв, **бит 31 — `bi_sync`** (счётчик BI синхронизирован), а не CCA/NAV
и не «принят чужой маяк». Ворота передачи маяка [4.1]: A — `bi_counter > BI−15 ||
bi_counter < 2500` (окно BTI), B — `bi_sync`; отсрочка по занятости среды идёт
программно через NAV внутри рандомизатора
([MAC-REGISTERS, «BI_COUNTER»](https://github.com/klukonin/SparRAW-firmware/blob/main/docs/MAC-REGISTERS.md#bi_counter-и-бит-31--bi_sync)).

**Регистры событий.** `r42` = `EVENT1_STATUS` — события среды и времени
(1 `slot_event`, 4 `bi1_event`, 6 `tsf_event`, 7 `txop_event`, 12 `rx_frame_event`,
14 `busy2_event`, 16 `ppdu_report_event`, 18/19 `backoff_event`/`busy_event`);
`r54` = `EVENT4_STATUS` — индикации окончания GP-таймеров и служебных периодов
(6/7/8 `gp0/1/2_end_ind`, 15 `ext_gp_0_1_2_end_ind`, 16/17 `ext_gp_3_4_5` /
`ext_gp_6_7_8_9_end_ind`). **Окна BI (BTI/A-BFT/AW/DTI) ведут GP-таймеры, а не
компаратор TSF**: AW→DTI — по биту 15 `r54`, DTI→BTI — по биту 4 `r42`; бит 6
`tsf_event` не ждёт ни одно из 168 мест ожидания. GP-таймеры программируются
командами MAC 0x4f..0x54 (длительность в тактах 165 МГц, управление), квитируются
0x49 (бит `r54` = idx+6). Полные таблицы битов, маска движка событий (команда MAC
0x27) и примитивы ожидания — [UCODE-EVENT-BITS](../research/UCODE-EVENT-BITS.md),
обзор — [03-mac.md §3.6](03-mac.md); команды GP-таймеров —
[MAC-COMMANDS §7.6](https://github.com/klukonin/SparRAW-firmware/blob/main/docs/MAC-COMMANDS.md#76-gp-таймеры-и-r54-0x49-0x4f0x54-0x670x68).

---

## 6. Блок MAC_SXD и время: 0x886d00..0x886fff

Полная таблица — REGS-62, сводка по TSF и таймерам —
[HARDWARE-BLOCKS, «Блок TSF и таймеров MAC»](https://github.com/klukonin/SparRAW-firmware/blob/main/docs/HARDWARE-BLOCKS.md#блок-tsf-и-таймеров-mac-0x886d180x886ec0).
Базовый период BI стенда — 100 TU = 102 400 мкс.

### 6.1 IFS и тайминги доступа к среде [обе]

Пары {такты 165 МГц, мкс}; задаются LMAC-командой 0x00 (тело +0x30,
`fui_cmds_hw_cfg_ifs_timing_cfg_s`,
[LMAC-PROTOCOL §3.1](https://github.com/klukonin/SparRAW-firmware/blob/main/docs/LMAC-PROTOCOL.md#31-команда-0x00-fui_cmds_hw_cfg_ifs_timing_cfg_s-по-body0x30-код-железо-пак)).
Проверено на железе до бита [4.1]:

| адрес | значение | смысл |
|---|---|---|
| 0x886d00 | 0x01730300 | SIFS 3 мкс (на время SLS ucode ставит 3·SIFS, затем возвращает) |
| 0x886d04 | 0x03400500 | slot 5 мкс |
| 0x886d08 | 0x0a000800 | SIFS+slot (PIFS) 8, 2·slot 10 |
| 0x886d0c | 0x01000100 | RIFS 1, SBIFS 1 |
| 0x886d10 | 0x100 при init | момент запуска ответа RX→TX, пересчитывается по каждому кадру из `SIFS_CNT` (R55) |
| 0x886e08..0x886e1c | 0x010400a1 … 0x0be41190 | поля тела LMAC 0x00 (задержки в тактах), константа 0x0be41190 одинакова в 4.1 и 6.2 |
| 0x886f70 / 0x886f74 | 0x07d007d0 | пределы TXOP 2000 мкс (в OOB 0x05000500) |

Такт 165 МГц: `us_from_ticks165` @0x931c98, делитель микросекундного тика MAC
0x886d58 = 0x5300a5 (165) штатно [обе]. 0x886d88 = 0x0c0a0f1e — операнд команды
MAC 0x1e, а не тайминги IFS; путь `short_txop_en` для деления предела TXOP мёртв.

### 6.2 Интервал маяка и таймеры BI

| адрес | имя | смысл | доказательство |
|---|---|---|---|
| **0x886d64** | `mac_beacon_interval` | [15:0] интервал в TU, init 100; **горячий**: одиночная запись с хоста останавливает маячный цикл; пишет сам ucode (`st_s r1,[r0,0x64]` @0x931798) | memscan 100→200 [4.1], опыт с записью [4.1] |
| **0x886d18** | `mac_bi_reload_us` | перезарядка таймера BI, мкс = **BI·1024 − 8** (0x18ff8 при BI 100) | серия BI 50/100/200: 0xc7f8/0x18ff8/0x31ff8 [6.2] |
| 0x886d1c | `mac_bi_reload_clk` | дробная часть той же перезарядки в тактах: 165 − 80 = 85 (0x55) | серия [6.2] |
| 0x886dc0 / 0x886dc4 | `mac_cmd0d_time_lo/hi` | 64-битный операнд команды MAC 0x0d: первый TBTT при запуске BI или текущий TSF при перезагрузке; **не регистр периода** | ревью кода [4.1], [6.2] |
| 0x886d50 | `mac_timed_wait_us` | 2000 при init BI, 3000 в фиксированном расписании | код [6.2] |

Помощник [4.1] `mac__program_deadline_slack` (0x930494) пишет 0x886d18/0x886d1c,
вызовы `(BI, 0x19, 0x50)` и `(BI, 7, 0x50)`. Горячая смена периода не удаётся ни
одиночной записью 0x886d64, ни согласованной записью пяти значений (0x886d64,
0x886dc0, 0x941afc/b00/b04) — маяк встаёт; перенос всех 11 ячеек периода маяк не
роняет, но период не меняет; штатный `wifi reload` пересчитывает всё [4.1]
([MAC-COMMANDS §9.1](https://github.com/klukonin/SparRAW-firmware/blob/main/docs/MAC-COMMANDS.md#91-период-маяка)).

### 6.3 Время: TSF, TBTT, начало BI

| адрес | имя | смысл |
|---|---|---|
| 0x886eb4 | `mac_timing_status` | бит 31 = значения времени MAC действительны; все читатели TSF крутятся на нём |
| 0x886eb8 / 0x886ebc | TSF low / high | текущий TSF, мкс (`TIMING_INDIRECT_REG_5`, `msrb_capture_ts_low`); старшее читается до и после младшего |
| **0x886ec0** | `mac_usec_to_tbtt` | **обратный отсчёт мкс до следующего TBTT**, 26 бит; фаза TSF в BI + значение ≈ 102 400; при невключённом MAC 0x3ffffff [6.2, железо] |
| **0x886fc4** / 0x886fc8 | `mac_bi_start_tsf_lo/hi` | **TSF начала текущего BI (TBTT)**; значения кратны BI (сетка от нуля TSF); переводится на следующий TBTT за ~7–8 мс до него [6.2, железо] |
| 0x886fcc / 0x886fd0 | `mac_awake_tsf_lo/hi` | момент пробуждения перед глубоким сном → бит 15 в 0x886e40 [6.2] |
| 0x88c120 | `mac_ext_window_remain` | [23:0] остаток текущего окна в тактах MAC; ucode не открывает окно приёма, если времени мало [обе] |

**Компаратор TSF** [обе]: 0x886d30/0x886d34 `mac_tsf_event_lo/hi` — значение,
0x886d38 `mac_tsf_event_ctrl` — 0x000c0050 взведён (бит 6), 0x000c0010 снят, 0 в
простое. В 4.1 его трогают только `set_tsf_event` (вызывается лишь рандомизатором
маяка) и `disable_tsf_event` (из обработчика конфига биконинга). Рандомизация маяка
через компаратор бесполезна: он взводится значением из прошлого (в ~34 раза меньше
текущего TSF), а бит 6 `tsf_event` не ждёт ни одно место ожидания; показание
`wil_tsf_probe.py` (TARGET − CUR) на передачу не влияет
([HARDWARE-BLOCKS, «Компаратор TSF»](https://github.com/klukonin/SparRAW-firmware/blob/main/docs/HARDWARE-BLOCKS.md#компаратор-tsf-код),
[BEACONING §9.2](https://github.com/klukonin/SparRAW-firmware/blob/main/docs/BEACONING.md#92-рандомизатор-маяк-не-сдвигает-41-железо--код)).

### 6.4 Константы старта MAC [6.2]

| адрес | запись при init BI | на стенде | вмешательство |
|---|---|---|---|
| **0x886d40** | −1 | 0x00ffffff (24 бита) | 0 или без любого байта — **линк рвётся** |
| **0x886d44** | 0x00028010 | 0x00028010 | 0 или без бита 15 — **AP перестаёт передавать**; без бита 4 — без эффекта; без бита 17 — iperf проседает |
| 0x886fc0 | −1 | 0x00ffffff | 0 — разрыв, ucode восстанавливает при переподключении |
| 0x886d14, d20, d24, d5c, d60, de8, dec, f60 | константы | не зависят от BI/канала/роли | запись эффекта не дала |

Смысл 0x886d40/0x886d44 не установлен, доказана только обязательность; гипотеза
«0x886d20 = предел/включение NAV» опровергнута опытом
([BENCH, «Регистры MAC»](https://github.com/klukonin/SparRAW-firmware/blob/main/6.2/docs/BENCH.md#регистры-mac)).

### 6.5 Прочее в блоке

| адрес | смысл | версия |
|---|---|---|
| 0x886d3c | маска событий, будящих ucode из `sleep` | 6.2 |
| 0x886d58 | делитель мкс-тика MAC (165; 40/38 в экономном режиме PLL) | 6.2 |
| 0x886d80 | лишь база: через неё пишется порт 0x886dc8 | 6.2 |
| 0x886d8c..0x886da8 | `QSET0..7_MASK_VECTOR` — маски очередей наборов 0..7 | обе |
| **0x886dc8** | `sxd_local_wr_data` — порт **uCode CORE WRITE** (§7) | обе |
| 0x886df0/f4, 0x886df8/fc | свой MAC-адрес, второй (групповой) адрес фильтра | 6.2 |
| 0x886e58..0x886e64 | отчёты о последнем PPDU, слова 1..4 | 6.2 |
| 0x886e84 | NAV по полю Duration: младшие 16 бит, под iperf до 1982 (≈TXOP − 18), бит 31 почти всегда [гипотеза: аппаратный NAV] | 6.2 |
| 0x886ecc | бит 3 — занятость MAC, fw ждёт снятия перед удалением очереди | 6.2 |
| 0x886f4c/50/54 | снимки sp/ilink1/ilink2 ucode для дампа при крахе | 6.2 |
| 0x886f5c | маска разрешённых очередей по qid; бит 31 — внутренняя очередь ucode | 6.2 |
| 0x886f84/88/8c, 0x88c050 | слоты расписания DTI (ESE): начало, конец, тип, состояние | 6.2 |
| 0x886f9c | `MTP_Q_AVAIL` (отображение R41[11]) | 6.2 |

---

## 7. Интерфейс core write (кратко)

Команда MAC — 32-битное слово `[31:24]` код, `[23:0]` параметр; порт
`MAC_RGF.MAC_SXD.LOCAL_REGISTER_IF.LOCAL_WR_REG.sxd_local_wr_data` = **0x886dc8**
(вендорское имя интерфейса — uCode CORE WRITE). ucode отправляет команду записью в
`r32` и кладёт копию в журнал `r25` (uc 0x800000, 1024 Б = 256 команд, host
0x940000, снимается без патчей через `blob_uc_data`). Бит 31 = «записать событием
PMC». С хоста команду отправить нельзя: порт хранит записанное, но триггер — запись
ядра ARC в `r32`. Набор кодов задан кремнием (77 кодов в 4.1 и 6.2, все общие).
fw тоже может писать порт по AHB: `hwm__probe_33khz_clk` шлёт через базу 0x886d80
команды 0x63001e04 и 0x63000108 (измерение такта 33 кГц, результат в 0x88c180)
[6.2]. Подробно — [03-mac.md](03-mac.md),
[MAC-COMMANDS](https://github.com/klukonin/SparRAW-firmware/blob/main/docs/MAC-COMMANDS.md).

---

## 8. Другие области (обзор)

| область | содержимое | подробно |
|---|---|---|
| DMA 0x881000..0x881dff | очереди `DMA_RGF.DESCQ<n>`, L2-offload, fw-ICR колец; ITR 0x881c5c.., TX-блок 0x881d34..0x881d40, RX-блок 0x881d44..0x881d50, `DMA_MISC_CTL` 0x881d6c (драйвер пишет через `doff_io32`) | [HARDWARE-BLOCKS, «DMA»](https://github.com/klukonin/SparRAW-firmware/blob/main/docs/HARDWARE-BLOCKS.md#dma-и-модерация-прерываний-itr-драйвер) |
| PCIe 0x882000..0x882fff, 0x88b000.. | SerDes, L1SS, DBI, PERST, PXE, карты режимов питания | [HW-DRIVERS §4](https://github.com/klukonin/SparRAW-firmware/blob/main/docs/HW-DRIVERS.md#4-pcie-и-питание-устройства) |
| PHY 0x883000..0x8851ff | AGC/RSSI, счётчики и статистика приёма (0x883900..), BRP-измерения, TX-предыскажения, SAR | [HARDWARE-BLOCKS, «Счётчики приёма PHY»](https://github.com/klukonin/SparRAW-firmware/blob/main/docs/HARDWARE-BLOCKS.md#счётчики-приёма-phy-код--железо), [HW-DRIVERS §6](https://github.com/klukonin/SparRAW-firmware/blob/main/docs/HW-DRIVERS.md#6-калибровки) |
| парсер и очереди MAC 0x886000..0x886cff | адреса фильтра, `PRS_ICR`, RX/TX-очереди, `mac_mgmt_seq_num` 0x886544 (SN кадров ucode: чтение [11:0], запись (SN+1)<<16 с битом 28) | [DATAPATH](https://github.com/klukonin/SparRAW-firmware/blob/main/docs/DATAPATH.md) |
| счётчики MAC 0x887000..0x8878ff | команды 0x887008 с ожиданием флагов готовности (иначе фатал 0x1187..0x1189), `mac_cnt_cfg_*`, `mac_cnt_sel_*` | REGS-62 |
| РЧ и ABIF 0x889000..0x88afff | `rfc_core_*`, бит 29 0x8890cc «RFC занят», маска RF-контроллеров 0x889488 биты 8..15, синтезатор, датчики, `RGF_CAF_ICR`, ToF 0x889500.., таблицы ABIF TX/RX (регион `AGC_tbl`) | [HW-DRIVERS §8–9](https://github.com/klukonin/SparRAW-firmware/blob/main/docs/HW-DRIVERS.md#8-abif-интерфейс-baseband--рч) |

---

## 9. Регистры без доказанного смысла [6.2]

37 регистров, у которых известно, кто и что пишет, но не известно, зачем (REGS-62,
[BENCH, «Регистры MAC»](https://github.com/klukonin/SparRAW-firmware/blob/main/6.2/docs/BENCH.md#регистры-mac)).

**MAC, пишутся при инициализации режима BI** (от BI, канала и роли не зависят;
при невключённом MAC нулевые):

| адрес | имя | значение | что известно |
|---|---|---|---|
| 0x886d14 | `mac_ifs_const_452` | 0x452 | пишется с IFS-регистрами LMAC 0x00; запись без эффекта |
| 0x886d20 | `mac_bi_init_arg0` | 5000 | гипотеза NAV опровергнута |
| 0x886d24 | `mac_bi_init_arg1` | 1 | парный к d20; без эффекта |
| 0x886d28 | `mac_bi_init_zero_28` | 0 | всегда 0 |
| 0x886d40 | `mac_bi_init_ones_40` | 0x00ffffff | **обязателен** (§6.4) |
| 0x886d44 | `mac_bi_init_const_44` | 0x00028010 | **бит 15 обязателен** (§6.4) |
| 0x886d5c | `mac_timing_const_5c` | 0x00040063 | гипотеза 4.1 «{clks, usec}» не доказана |
| 0x886d60 | `mac_timing_const_60` | 0x00010001 | без эффекта |
| 0x886dac | `mac_bi_init_zero_ac` | 0 | сразу за банком QSET, в него не входит |
| 0x886de8 | `mac_timing_const_e8` | 0x00060025 | без эффекта (повтор с 3×iperf) |
| 0x886dec | `mac_timing_const_ec` | 0x0002000b | без эффекта |
| 0x886f60 | `mac_slot_timing_886f60` | 0x03780378 | два поля по 888; без эффекта |
| 0x886f78, 0x886f7c | `mac_886f78/7c` | 0 | всегда 0 |
| 0x886fc0 | `mac_886fc0` | 0x00ffffff | 0 → разрыв; «сброс причин» или «никогда» — не установлено |

**MAC, частичный смысл:** 0x886028 — маска или W1C-квитирование (§3); 0x886e84 —
текущий NAV или предел (§6.5).

**USER_RGF:** 0x880074, 0x880078, 0x88007c (`gpio_cfg_3..5`: обнуляются при сбросе
GPIO, всегда 0); 0x880200 (fw_main ждёт бит 6 перед «HW statemachine WA»; на
стенде 0xc3 при работающем MAC, 0xc8 при выключенном радио); 0x880204
(`pcie__serdes_init` всегда пишет 1, так же в 4.1).

**PHY:** 0x8830ec, 0x8836c8, 0x8837b0, 0x8837d8, 0x8837dc, 0x8837f4, 0x883800,
0x883980, 0x8839bc, 0x883aa4, 0x883b78, 0x88405c, 0x884060, а также 0x88317c
(`phy_abif_override_flag`: ucode ставит 1 на время подмены регистров ABIF/TOF;
смысл бита не установлен). Зависят от роли: 0x8830ec (у станции 0x4c0), 0x8839bc
(растёт только у станции), 0x8836c8; 0x884060 всегда 0.

**РЧ:** 0x8890f8 `rfc_ctrl_f8`.

Итого 32 с пометкой «смысл не установлен» + 5 с частично неустановленным смыслом
(0x880200, 0x88317c, 0x886028, 0x886e84, 0x886fc0) = 37, что совпадает с итогом
REGS-62.

---

## 10. Расхождения источников

1. **Векторы 10/11 fw**: базы ICR 0x887000/0x88701c в разборе 4.1 против
   0x88700c/0x887028 в разборе 6.2 и REGS-62; регистры причин (0x887014/0x887030)
   совпадают, расходится только то, что считать базой блока
   ([HW-DRIVERS §2.1](https://github.com/klukonin/SparRAW-firmware/blob/main/docs/HW-DRIVERS.md#21-таблица-векторов-код)).
2. **`r25`**: в таблице
   [HARDWARE-BLOCKS, «Регистры ядра ARC»](https://github.com/klukonin/SparRAW-firmware/blob/main/docs/HARDWARE-BLOCKS.md#регистры-ядра-arc-используемые-ucode-код)
   он назван кольцом запросов к MAC; по
   [MAC-COMMANDS §2](https://github.com/klukonin/SparRAW-firmware/blob/main/docs/MAC-COMMANDS.md#2-журнал-r25)
   это журнал, а отправка — запись в `r32` (§7).

# 7. Беамформинг и установление линка

Глава — обзор того, как Sparrow находит соседа и наводит на него луч (SLS,
A-BFT, BRP), как из результата беамформинга получаются ассоциация и открытый
порт данных. Устройство в коде — документация прошивки:
[BF-ENGINE.md](https://github.com/klukonin/SparRAW-firmware/blob/main/docs/BF-ENGINE.md)
(потоки и состояния BF, BRP, SLS в DTI, rate search),
[HOST-INTERFACE.md](https://github.com/klukonin/SparRAW-firmware/blob/main/docs/HOST-INTERFACE.md)
(подключение станции, запуск PCP, P2P Find),
[6.2/docs/BENCH.md](https://github.com/klukonin/SparRAW-firmware/blob/main/6.2/docs/BENCH.md)
(поведение 6.2 на стенде).

Соседние главы: интервал маяка и окно A-BFT — [06-beaconing](06-beaconing.md);
регистры r37..r54 и маски событий — [02-registers](02-registers.md),
[03-mac](03-mac.md); команды LMAC (0x09, 0x0f, 0x14, 0x21, 0x23) —
[04-fw-ucode](04-fw-ucode.md); WMI, debugfs и драйверные патчи —
[05-host-interface](05-host-interface.md); различия версий —
[08-versions](08-versions.md).

**Обозначения.** Версия: **[4.1]**, **[6.2]**, **[обе]**. Доказанность:
**[железо]** — измерено на стенде (два узла wAP 60G 60°, узлы A и B),
**[код]** — по листингу, **[пак]** — по вендорским именам пака 11ad,
**[гипотеза]**. Host-адреса: ucode 0x940000 + (A − 0x800000), fw_data
0x900000 + (A − 0x800000), fw_peri 0x908000 + (A − 0x840000).

---

## 7.1 SLS/TXSS и ролевой гейт развёртки

SLS по 802.11ad — четыре фазы: **I-TXSS → R-TXSS → SSW-Feedback → SSW-ACK**.
Поле Sector Sweep принятого кадра лежит в буфере 0x804020 (бит 0 — Direction,
CDOWN, Sector ID, DMG Antenna ID) **[4.1, код]**
([STANDARD-MAPPING, «SLS»](https://github.com/klukonin/SparRAW-firmware/blob/main/docs/STANDARD-MAPPING.md#sls--sector-level-sweep)).

**Маска классов принятого кадра 0x801038** (24 байта, обнуляется на каждый
PPDU): биты SSW, SSW-Feedback, SSW-ACK, DMG RTS, BRP. Бит 1 — «развёртка
встречной стороны окончена» — ставится при `(ss_bitmap[0] ∨
my_bssid_beacon_detected) ∧ zero_cdown ∧ ppdu_report_event` и **подменяет
аппаратный `txss_end_ind`**: это единственный программный способ завершить
приёмную развёртку. С битом 1 ставится 0x8004b0 = 1 (host 0x9404b0) —
разрешение записать измеренный сектор в таблицу пира 0x8010d0; без него —
таймаут и откат РЧ **[4.1, код + железо]**. Путь через `ss_bitmap[0]`
(настоящий SSW) даёт бит 1 без совпадения BSSID; `other_beacon_detected` ucode
не проверяет нигде.

**Ролевой гейт** в `rx_funcs__handle_ppdu_report` **[4.1, код]**:

`принять ⟺ (Direction = 0 ∧ бит 6) ∨ (Direction = 1 ∧ бит 5)` слова роли
0x800604 (0x20 — TXSS-инициатор, пишет 0x93620c; 0x40 — TXSS-ответчик, пишет
0x936592). Это дословная ролевая семантика SLS, BSSID в гейте нет. Снимается
двумя однобайтовыми правками (`0x93620c: 20 d8 → 60 d8`, `0x936592: 40 d8 →
60 d8`) — не применялись. В 6.2 гейт тот же (бит «развёртка соседа окончена» в
0x8021e0, роль `[gp+0xcc]`, дополнительно `TA == 0x802188` и фаза `[gp+0xd0]`)
**[6.2, код + железо]**. Подробно —
[BF-ENGINE §8](https://github.com/klukonin/SparRAW-firmware/blob/main/docs/BF-ENGINE.md#8-sls-в-dti-62-код--железо).

Для широковещательного маяка «пировый» путь приёма (0x92e304) закрыт условием
`addr2_match_valid ∧ fcs_ok_partial ∧ ¬mcast ∧ ¬bcast ∧ ¬neighbour_indication`;
тем не менее маяки соседей принимаются и считаются (гл. 6).

**Число и порядок секторов [6.2].** Диапазон секторов для cid берётся из
таблицы, заполняемой из board-файла (не больше 64); порядок задаётся с хоста
`WMI_PRIO_TX_SECTORS_ORDER` 0x9A5 и держится, прямая запись в 0x802b68
затирается. Опыт «центральный сектор 35 последним» — без эффекта **[6.2, код +
железо]** ([BF-ENGINE §8.5](https://github.com/klukonin/SparRAW-firmware/blob/main/docs/BF-ENGINE.md#85-число-и-порядок-секторов-развёртки)).

## 7.2 Ответчик A-BFT

Ответчик A-BFT настраивается LMAC-командой 0x23 (`fui_abft_resp_ctrl_s`,
ucode 0x80208c..0x802098: `relaxation_period_cnt`, `mode`, `beacon_type`,
`relaxation_period`); WMI-команды для него в 4.1 нет **[4.1, код + железо]**.

| источник | mode | beacon_type | relaxation |
|---|---|---|---|
| загрузка (`lmac_if__lmac_ready_evt`) | 0 UNASSOC_ONLY | 2 ALL | 1 |
| PCP (`pcp_start`) | 2 ALWAYS | 1 DISCOVERY | 1 |
| запись с хоста `0x942090=2, 0x942094=2, 0x942098=0` | 2 ALWAYS | 2 ALL | 0 |

Загрузочная конфигурация означает, что **любой узел без линка уже отвечает в
A-BFT любого маяка** — буквальное прочтение стандарта (A-BFT служит тренировке
луча до ассоциации). Ассоциированная станция штатно отказывает каждый раз по
mode, поэтому в штатной паре ответчик мёртв. Запись с хоста даёт 22 ответа SSW
за 8 с против 1; прошивка перезаписывает её при рестарте ucode, `pcp_start`
или входе в find-search **[4.1, железо]**. Счётчики ответчика — 0x801e68
(host 0x941e68) +0x1c..+0x26.

## 7.3 BF-движок, BRP и quasi-omni

Потоки BF в ucode: SLS в BTI/A-BFT (по расписанию BI), I-TXSS и R-TXSS в DTI,
BRP-инициатор и BRP-ответчик, rate search. Имена состояний и подсостояний
(`ucode_bf_state_e`/`ucode_bf_substate_e`) взяты из пака 11ad, номера совпали
с 4.1 в коде и на стенде (например 13 DONE, 9 BRP_INIT, 10 BRP_RESP; подсостояния
48/49 BRP_INIT_SUCCESS/FAILURE, 51/52 BRP_RESP) **[обе, пак + код]** —
[BF-ENGINE §1–2](https://github.com/klukonin/SparRAW-firmware/blob/main/docs/BF-ENGINE.md#2-состояния-и-подсостояния-обе).

**BRP в 4.1 недостижим** **[4.1, код + железо]**: после успешного SLS бит CID в
0x801034 (host 0x941034, «сосед связан», ставит LMAC 0x0f из
`maintain_sm__on_start_maintain`) ведёт в BRP_INIT, а фоновый диспетчер делает
BRP-инициаторами **обе** стороны — взаимная блокировка, BRP-ответов 0. Без бита
узел в BRP не участвует. Пока BF висит, очередь соседа снята с передачи (u16
`0x801b2c[cid]`, регистр 0x886da4 → только broadcast). Политику повторов
выбирает `lm_main_sm__on_maintain_started` по `m_bss_mode` (`[mid+0x50]`,
0x805940: 1 — станция/PCP не запущен, 2 — PBSS-PCP, 3 — AP): 2/3 → ABORT, иначе
RETRY_FOREVER —
[BF-ENGINE §4–7](https://github.com/klukonin/SparRAW-firmware/blob/main/docs/BF-ENGINE.md#4-развилка-после-sls-бит-связан-0x801034-41-код).

**6.2**: BRP выключается штатно — `WMI_BF_CONTROL` 0x9aa, `brp_mode == 0` → SLS
→ DONE без BRP; раскладка команды в 6.2 не совпадает с `wmi.h` мейнлайна
**[6.2, код]**. Ответчик BRP в 6.2 работает на железе (BRP_RESP_SUCCESS,
BRP_INIT_SUCCESS на обоих узлах) **[6.2, железо]**.

**Quasi-omni: глухота станции 6.2 в DTI** **[6.2, код + железо]**. После
успешного BRP `brp_update_omni_sector_idx` 0x923970 ставит индекс RX-сектора
станции в «направленный omni» (96 + sta) вместо quasi-omni из board (0x7fff); в
4.1 механизма нет. На стенде «направленный omni» плохой: станция не слышит RSS,
данные и ACK от AP. Стандарт (§11.1.3.7) разрешает станции, обученной на приём, quasi-omni вместо направленного приёма в течение dot11MinBHIDuration после TBTT; про DTI он этого прямо не говорит. Исправление
в ucode — блок `src/c/uc/brp_update_omni_sector_idx.S` дерева 6.2 (всегда
0x7fff, заплата 5 байт): пинг 192/200 и 190/200 против 54 и 62 на стоке —
[6.2/docs/BENCH.md, «Направленный omni станции»](https://github.com/klukonin/SparRAW-firmware/blob/main/6.2/docs/BENCH.md#направленный-omni-станции),
[6.2/docs/RX-BRP-UC.md](https://github.com/klukonin/SparRAW-firmware/blob/main/6.2/docs/RX-BRP-UC.md).
Разрывы, остающиеся после quasi-omni, объясняются неверным периодом BI станции
(§7.6), а не публикатором MAC_MON.

## 7.4 Таблица РЧ-секторов

Единственный писатель таблицы секторов — `rf_sector_params_write` @0x8d1174,
зовётся из обработчика `WMI_SET_RF_SECTOR_PARAMS_CMDID` (**0x9A1**): банки RX
0x10000 и TX 0x8000, слот сектора 0xc0, шесть u32-полей (по числу и ширине
совпадают с `struct wmi_rf_sector_info` драйвера) **[4.1, код]** —
[research/RF-SECTOR-TABLE](../research/RF-SECTOR-TABLE.md).

## 7.5 PBSS штатно

Сеть без точки доступа в 802.11ad — PBSS. Пара собрана на 4.1 **[железо]**:
координатор — hostapd `pbss=1`, станция —
wpa_supplicant `pbss=1`; согласовано 2310 Мбит/с, измерено 593…988 Мбит/с,
задержка 0,7…1,6 мс. OpenWrt `pbss` не пробрасывает, демоны запускаются вручную
после `/etc/init.d/wpad stop`. Маячит только координатор; передачи роли PCP в
прошивке 4.1 нет (`psc_*` — энергосбережение) —
[research/PBSS-ON-HW](../research/PBSS-ON-HW.md),
[research/WMI-PCP-FACTOR](../research/WMI-PCP-FACTOR.md).

## 7.6 BI станции и MaxLostBeacons

Стандарт требует, чтобы станция приняла период маяка BSS при присоединении
(802.11-2020 §11.1.3.3.1) и ждала маяк не дольше `dot11BeaconPeriod ×
dot11MaxLostBeacons` (объявляется в элементе DMG Operation). Ни одна из
стоковых версий этого не делает **[обе]**:

| версия | откуда период станции |
|---|---|
| 4.1 | зашитые 100 TU (при BI точки 200 у станции 0x886d64 = 0x64) |
| 6.2 (сборка из RouterOS) | глобал fw 0x8002d4 из WMI **0x803** («задать BI»), берётся в `l2_mgr__connect` |

Мейнлайн-драйвер шлёт 0x803 как эхо 0x12345678 → станции 6.2 программируется BI
0x5678 TU (~22,7 с) → нет `bi1_event` → слот MAC_MON залипает (ложные «плохие
маяки» каждые 2 с), пропажа AP не обнаруживается **[6.2, железо]**. Исправление:
патч драйвера 914 (0x803 с `bss->beacon_interval`), правка
`src/c/fw/l2_mgr__pcp_start_flow.S` (AP копирует BI в BSS +0x4e, иначе Probe
Response сообщал 100) и патч 915 (MaxLostBeacons из DMG Operation в порог
`bad_beacons_num_threshold`). С ними станция берёт BI точки (100 и 200), пропажа
AP → разрыв за 20 BI, iperf3 981/1010 Мбит/с —
[6.2/docs/BENCH.md, «BI станции и детекторы потери связи»](https://github.com/klukonin/SparRAW-firmware/blob/main/6.2/docs/BENCH.md#bi-станции-и-детекторы-потери-связи),
[STANDARD-MAPPING, «Интервал маяка и потеря связи у станции»](https://github.com/klukonin/SparRAW-firmware/blob/main/docs/STANDARD-MAPPING.md#интервал-маяка-и-потеря-связи-у-станции).

**Скан после подключения [6.2, железо]**: `WMI_CONNECT` идёт через
`scan_mngr`, скан не останавливается, станция уходит с канала и теряет маяки →
цикл переподключений ~3 с; обход — скан одним каналом (`scan_freq`/`freq_list`
= 58320) ([05 §5.10](05-host-interface.md#510-особенности-эксплуатации)).

## 7.7 Board-файлы

**[обе, железо]**:

* драйвер всегда грузит `/lib/firmware/wil6210.brd` (`wil->board_file` нигде не
  присваивается), RF-данных в DTS нет: качество 60 ГГц под OpenWrt определяет
  brd, а не device tree;
* валидный формат — 3588 байт, заголовок `06000000 34000000 "0126"`; стоковый brd
  linux-firmware отличается от вендорского на 1574 байта (антенный кодбук AWV,
  2048–3584);
* соответствие модель → brd (из строк RouterOS): LHG → `lhg-span2.8`, wAP 60G →
  `wap60g-60deg`, wAP 60Gx3 → `wap60g-omni`, Cube → `cube`, nRAY → `nray`;
  строка модели «LHGG-60ad» в `/tmp/sysinfo/model` приходит из DTS сборки, по ней
  brd не выбирается;
* рантайм-патчер RouterOS — регулятор мощности вниз по tx-power, не калибровка;
  вендорские brd уже содержат tx = rx = 15 (максимум), сток linux-firmware
  слабее (tx = 9);
* ревизия Sparrow D0 («plus», HW version 0x2) просит `wil6210_sparrow_plus.fw`,
  которого нет в linux-firmware — патч 902.

В 6.2 из board берутся диапазоны секторов развёртки (§7.1) и quasi-omni
(0x7fff). Разбор секций —
[HW-DRIVERS §9](https://github.com/klukonin/SparRAW-firmware/blob/main/docs/HW-DRIVERS.md#9-board-файл-и-рч-интерфейс),
[research/BRD-RF-REGS](../research/BRD-RF-REGS.md); сборка OpenWrt (в том числе
патч backports `399-cfg80211-allow-60ghz-chantype`) —
[research/DRIVER-PATCHES-OPENWRT](../research/DRIVER-PATCHES-OPENWRT.md).

## 7.8 Пропускная способность и показания iw

**[4.1, железо]**: под нагрузкой линк даёт **~985 Мбит/с** (`iw` тогда
показывает 2310); **`iw` в простое врёт** — 27,5 Мбит/с и −72 дБм означают
состояние без трафика, а не деградацию физики; мерить трафиком (`iperf3`). При
A/B-замерах первый прогон после записи `bcon_kind` отбрасывается (проседание до
~735 Мбит/с) — [research/LINK-THROUGHPUT](../research/LINK-THROUGHPUT.md).

| конфигурация | результат |
|---|---|
| 4.1 AP↔STA, штатно | ~985 Мбит/с, RTT 0,7 мс |
| 4.1 PBSS | 593…988 Мбит/с |
| 6.2 сток, AP↔STA | нестабилен, пинг 54–62 из 200 |
| 6.2 quasi-omni + патчи 914/915 | 981 / 1010 Мбит/с |

`power_save` включён по умолчанию и даёт RTT 170–370 мс;
`iw dev <if> set power_save off` (или патч 904) → 0,77 мс.

## 7.9 Сводка рычагов линка с хоста

| рычаг | адрес / способ | эффект | версия, статус |
|---|---|---|---|
| ответчик A-BFT на любой маяк | запись 0x942090 = 2, 0x942094 = 2, 0x942098 = 0 | ответы SSW | 4.1, железо |
| без BRP | `WMI_BF_CONTROL` brp_mode = 0 | SLS → DONE | 6.2, код |
| порядок секторов | `WMI_PRIO_TX_SECTORS_ORDER` 0x9A5 | держится; эффекта на стенде нет | 6.2, железо |
| содержимое секторов | `WMI_SET_RF_SECTOR_PARAMS` 0x9A1 | запись таблицы секторов | 4.1, код |
| BI станции | патч 914 (0x803 = BI из BSS) | станция берёт BI точки | 6.2, железо |
| порог потери | патч 915 (MaxLostBeacons) | порог из DMG Operation | 6.2, железо |
| качество РЧ | верный brd (`wap60g-60deg` для wAP 60G) | основной рычаг качества | обе, железо |

## Не установлено

* Поле ↔ смещение шести полей слота RF-сектора.

# 5. Интерфейс с хостом

Глава — обзор того, что видно и доступно со стороны хоста (драйвер wil6210 на
OpenWrt): шина и окна памяти, протокол WMI обеих изученных версий, нештатные
смыслы отдельных команд и событий, рычаги без правки прошивки, лог прошивки и
трасса ucode, особенности эксплуатации стенда и патчи драйвера. Подробные
таблицы команд, событий и обработчиков — в документации прошивки
([оглавление](https://github.com/klukonin/SparRAW-firmware/blob/main/docs/README.md)):
[WMI.md](https://github.com/klukonin/SparRAW-firmware/blob/main/docs/WMI.md) (номера команд и
событий, диспетчеры),
[HOST-INTERFACE.md](https://github.com/klukonin/SparRAW-firmware/blob/main/docs/HOST-INTERFACE.md)
(что делают обработчики: подключение, PCP/AP, скан, P2P Find, отключение).

Пометки версии: **[4.1]** — FW 4.1.0.1000, **[6.2]** — FW 6.2.0.1000,
**[4.1+6.2]** — проверено в обеих, **[5.2]** — linux-firmware 5.2.0.18.
Доказанность: **[код]** — по листингу, **[железо]** — проверено на стенде,
**[гипотеза]** — не проверено.

Соседние главы: процессоры и память — [01-architecture](01-architecture.md);
регистры — [02-registers](02-registers.md); MAC и кольцо команд —
[03-mac](03-mac.md); протокол прошивка↔ucode — [04-fw-ucode](04-fw-ucode.md);
биконинг — [06-beaconing](06-beaconing.md); beamforming и линк —
[07-beamforming-link](07-beamforming-link.md); версии — [08-versions](08-versions.md).

---

## 5.1 Шина PCIe и загрузка

Sparrow подключён к хосту как конечная точка PCIe. Прошивка настраивает канал
под топологию («EP under Serdes»; «EP under switch» во многих местах «Not
supported yet»), выключает L0s, исправляет MaxReadReq больше 512 Б, поддерживает
L1SS [6.2]; устройство PCIe в 4.1 и 6.2 одно и то же — подробности в
[HW-DRIVERS §4](https://github.com/klukonin/SparRAW-firmware/blob/main/docs/HW-DRIVERS.md#4-pcie-и-питание-устройства).

Прошивка грузится из файловой системы хоста: драйвер берёт
`/lib/firmware/wil6210.fw` и `wil6210.brd` через `request_firmware`, во
флеш/OTP радио он не пишет — подменой прошивки радиомодуль не испортить; при
неудачной загрузке CPU радио остаётся остановленным, хост жив. Board-файл всегда
один — `wil6210.brd`, имя из DTS или параметра не берётся.

Порядок шагов `main()` прошивки (23 шага от `boot__init_subsystems`, читающего
бит OOB, до «SCHEDULER STARTS»; последним `l2mgr__init` шлёт хосту `WMI_READY`) —
[HW-DRIVERS §1](https://github.com/klukonin/SparRAW-firmware/blob/main/docs/HW-DRIVERS.md#1-загрузка-main-прошивки).

Слово `RGF_USER_USAGE_7` (0x88001c) переживает перезапуск прошивки: бит 0
отличает повторный старт (выполняется `BUG_5382_WA`), бит 1 — «настройки PM
PCIe хоста сохранены», биты 2..8 — ASPM L0s/L1, Clock PM и подвключения L1SS из
LnkCtl [6.2]
([6.2/ref/REGS-62.md](https://github.com/klukonin/SparRAW-firmware/blob/main/6.2/ref/REGS-62.md)).

## 5.2 Окна памяти и адреса хоста

### 5.2.1 Пересчёт адресов

Драйвер описывает память устройства таблицей `sparrow_fw_mapping`; «адрес
хоста» — то, что принимают debugfs-файлы `mem_addr`/`mem_write`. Полная карта
областей (`fw_code`, `fw_data`, `fw_peri`, `rgf`, `AGC_tbl`, `rgf_ext`,
`mac_rgf_ext`, `upper`, `uc_code`, `uc_data`) —
[HARDWARE-BLOCKS, «Карта памяти»](https://github.com/klukonin/SparRAW-firmware/blob/main/docs/HARDWARE-BLOCKS.md#карта-памяти-драйвер).

| пространство | адрес хоста |
|---|---|
| код прошивки (fw_code, 256 К) | 0x8c0000 + A |
| данные прошивки (fw_data, 32 К) | `0x900000 + (A − 0x800000)` |
| периферийная память прошивки (fw_peri, 128 К) | `0x908000 + (A − 0x840000)` |
| данные ucode (uc_data, 16 К) | `0x940000 + (A − 0x800000)` |
| регистры | тождественно (0x88xxxx) |

**У прошивки и ucode разные пространства 0x800000.** Совпадение адресов между
ними — не одна переменная [4.1+6.2]. Регистр `gp`: fw 0x800170 [4.1] /
0x800184 [6.2], ucode 0x800528 в обеих.

### 5.2.2 Ловушка `wmi_addr_remap`

`wmi_addr_remap` в драйвере перебирает только области с `fw == true`;
`uc_code`/`uc_data` помечены `fw = false`. Поэтому линкерный адрес ucode
0x801438, переданный в `mem_addr`/`mem_write`, попадает в `fw_data` и даёт
host 0x901438 — **данные прошивки**: запись молча портит чужую переменную.
Писать надо по `0x940000 + (A − 0x800000)`, то есть 0x941438 (область
`upper`) [4.1]. После записи адрес перечитывается через `mem_val` или
`blob_uc_data`. Та же ловушка в чтении кольца ucode закрыта флагом `fw` в
патче 909.

### 5.2.3 Файлы debugfs

Каталог `/sys/kernel/debug/ieee80211/phy0/wil6210/`.

| файл | что | откуда |
|---|---|---|
| `mem_addr` + `mem_val` | чтение 32-битного слова по адресу хоста; только адреса с `fw == true` | штатно |
| `mem_val` на запись | не работает: подключён к `memread_fops`, запись даёт EINVAL при правах 0644 | штатно |
| `mem_write` | запись слова: `echo "0xАДРЕС 0xЗНАЧЕНИЕ"` | патч 911 |
| `blob_fw_code`, `blob_fw_data`, `blob_fw_peri`, `blob_upper`, `blob_rgf`, `blob_uc_code`, `blob_uc_data` | снимки областей 256К/32К/128К/548К/40К/128К/16К; `ls` показывает размер 0, но читаются | штатно |
| `wmi_send` | отправка произвольной WMI-команды из файла (`dd bs=64`; busybox `printf \x` портит нулевые байты) | штатно |
| `discovery_mode` | записываемое поле (`WIL_FIELD(discovery_mode, 0644, doff_u8)`) | штатно |
| `fw_log_level`, `uc_trace`, `fw_watchdog_ms` | уровни лога, собранная трасса ucode, сторож | патчи 906, 910, 907 (§5.11) |

Подробности `mem_write` и трассировки —
[research/TRACING-TOOLKIT](../research/TRACING-TOOLKIT.md).

`CONFIG_DEV_COREDUMP` в ядре OpenWrt не включён, штатный дамп после ассерта
(823 296 Б) выбрасывается; снимок состояния снимается через `blob_*` при
`no_fw_recovery=1` [5.2].

Поиск параметров в памяти — серия дампов `blob_*` с фильтрами
increased/decreased/changed/unchanged и контрольным дампом
(`wil_memscan.py`); так найдены ячейки периода маяка и хранение канала
индексом с нуля [4.1]
([MAC-COMMANDS §9.1, §9.3](https://github.com/klukonin/SparRAW-firmware/blob/main/docs/MAC-COMMANDS.md#93-прочие-ячейки-найденные-методом-дампов)).

### 5.2.4 Регистры обмена `RGF_USER_USAGE_*`

| адрес | имя | смысл | версия |
|---|---|---|---|
| 0x880004 | `RGF_USER_USAGE_1` | прошивка публикует адрес кольца лога FW; драйвер обнуляет до старта (`wil_clear_fw_log_addr`) | [4.1+6.2] |
| 0x880008 | `RGF_USER_USAGE_2` | адрес кольца лога ucode | [4.1+6.2] |
| 0x88000c | `RGF_USER_USAGE_3` | ucode при старте пишет 0x800000 (начало данных ucode, кольцо MAC) | [6.2] |
| 0x880018 | `RGF_USER_USAGE_6` | от хоста: бит 31 `BIT_USER_OOB_MODE`, бит 30 `BIT_USER_OOB_R2_MODE`, бит 0 «прошивка загружена хостом» | [4.1+6.2] |
| 0x88001c | `RGF_USER_USAGE_7` | переживает перезапуск FW; PM PCIe (§5.1) | [6.2] |
| 0x880020 | `RGF_USER_USAGE_8` | хост: бит 0 PREVENT_DEEP_SLEEP, 1 SUPPORT_T_POWER_ON_0, 2 EXT_CLK; 6.2 к нему не обращается | [6.2] |

Живые значения на стенде: `0x880004 → 0x00843900` (кольцо FW в `fw_peri`),
`0x880008 → 0x0080209c` [4.1] и `0x803234` [6.2]. Источники —
[HARDWARE-BLOCKS, «Публикация адресов логов»](https://github.com/klukonin/SparRAW-firmware/blob/main/docs/HARDWARE-BLOCKS.md#публикация-адресов-логов-драйверкод)
и [6.2/ref/REGS-62.md](https://github.com/klukonin/SparRAW-firmware/blob/main/6.2/ref/REGS-62.md).

### 5.2.5 Прерывания хост↔прошивка

| регистр | направление | биты |
|---|---|---|
| 0x880b50 `RGF_USER_USER_ICR.ICR` | хост → прошивка | 2 — таймер планировщика, 4 — Awake TSF, 9 — событие ucode, **18 — рабочий почтовый ящик (WMI)**, 19 — отладочный ящик, 27 — RF kill |
| 0x881bf8 `RGF_DMA_EP_MISC_ICR.ICS` | прошивка → хост | 28 FW_READY, 29 MBOX_EVT (событие WMI), 31 FW_ERROR |
| 0x881bf0 `RGF_DMA_EP_MISC_ICR.ICR` | хост → прошивка | 27 HALP — запрос пробуждения |

Раскладка ICR/ICM/ICS/IMV/IMS/IMC совпадает со `struct RGF_ICR` драйвера.
Бит 18 обслуживает общий обработчик user-ICR (вектор 4 ARC), который ставит
задачу разбора почтового ящика — [HW-DRIVERS §2.2](https://github.com/klukonin/SparRAW-firmware/blob/main/docs/HW-DRIVERS.md#22-user-icr-хоста-0x880b50-код),
[HARDWARE-BLOCKS, «Прерывания»](https://github.com/klukonin/SparRAW-firmware/blob/main/docs/HARDWARE-BLOCKS.md#прерывания--блоки-struct-rgf_icr-драйвер).

Регистры ассерта (в окне `upper`): 0x91f020 — код ассерта, 0x91f024 — адрес
места ассерта как смещение в `fw_code` [5.2].

## 5.3 WMI: каркас

**Транспорт.** Команды хоста приходят в рабочий почтовый ящик (бит 18
user-ICR), события уходят по MBOX_EVT. Вход команд — большой switch по
идентификатору, каждая ветка печатает имя команды и зовёт обработчик
(`host_if__wmi_cmd_dispatch` [4.1], `wmi__host_cmd_dispatch` [6.2]); выход
событий — `operational_if__send_evt` [4.1] и три отправителя в 6.2
(`low_sme__send_evt2sw`, `wmi_evt__post`, `fw_sysapi_mgr__send_evt2host`).

| | 4.1 | 6.2 |
|---|---|---|
| команд в диспетчере | 68 | 102 |
| только в этой версии | 1 (0x859 `WMI_BRP_RF_CHAINS_LIMIT`) | 35 |

## 5.4 Карта команд

**Группы команд, общих для обеих версий:** соединение и ключи, скан, PCP/AP
(`BCON_CTRL`, `PCP_START/STOP`, `GET_PCP_FACTOR`, SSID и канал), P2P и
обнаружение (`P2P_CFG`, `START_LISTEN/SEARCH`, `DISCOVERY_START/STOP`), кольца
данных и BA, BF и RF-сектора, поддержание линка (`LINK_MAINTAIN_CFG_*`,
`RS_CFG`), питание, служебные (`ECHO`, `TEMP_SENSE`, `UNIT_TEST`, `FW_VER`,
XPM), ESE (0xa01). **Новое в 6.2:** управление BF с хоста (`WMI_BF_TRIG`,
`WMI_BF_CONTROL`, `WMI_PRIO_TX_SECTORS_*`), фиксированное расписание (TDMA),
станции с хоста (`NEW_STA`/`DEL_STA`), ToF/AoA, РЧ-калибровки, CCA, rate search.
Полная таблица «id → ветка → обработчик» и перечни команд, которых нет ни в
одной версии либо нет в `wmi.h`, —
[WMI.md](https://github.com/klukonin/SparRAW-firmware/blob/main/docs/WMI.md#полная-таблица-команд).

Для безролевой связи главное в 6.2 — `WMI_BF_CONTROL`/`WMI_BF_TRIG`: штатное
управление политикой и запуском BF (`brp_mode=0` проводит SLS→DONE без BRP);
в 4.1 политику повторов BF выбирает прошивка по `m_bss_mode`
([07-beamforming-link](07-beamforming-link.md)). `WMI_PRIO_TX_SECTORS_ORDER/NUMBER`
задают число и порядок секторов маяка [6.2].

**Совместимость ABI с мейнлайн-драйвером.** Числовые идентификаторы событий
4.1 сверены с перечислением драйвера по 43 местам отправки: 28/28 совпали;
0x1865 в прошивке называется `VRING_EN`, в драйвере `RING_EN` — тот же номер
[4.1] ([4.1/docs/UCODE-BEACON.md §7](https://github.com/klukonin/SparRAW-firmware/blob/main/4.1/docs/UCODE-BEACON.md#7-совместимость-wmi-4101000-с-mainline-драйвером)).
Размер `WMI_READY` — 0x10 в 4.1 против 0x14 в 6.2; жёсткого рукопожатия версий
нет. Поверхность WMI двух вендорских сборок на базе 6.2 почти одна (31 общая
команда) — [WMI.md, «Сборки 6.2 разных производителей»](https://github.com/klukonin/SparRAW-firmware/blob/main/docs/WMI.md#сборки-62-разных-производителей).

**Подкоманды UT (0x900).** `WMI_UNIT_TEST` ведёт в диспетчер
`UT_HW_DRIVERS_cmd_handler` с девятью таблицами переходов (ABIF, CAR, MAC-SXD,
PHY, RFC, PCIe) [6.2]; имена перечисления из пака 11ad совпадают с нумерацией
Sparrow лишь частично —
[6.2/docs/UT-DRIVERS.md](https://github.com/klukonin/SparRAW-firmware/blob/main/6.2/docs/UT-DRIVERS.md);
соответствие «код UT → функция» 4.1 — [research/UT-COMMANDS](../research/UT-COMMANDS.md).

## 5.5 События

| id | событие | замечание |
|---|---|---|
| 0x1001 / 0x1801 | `WMI_READY` / `WMI_FW_READY` | |
| 0x1002 / 0x1003 | `WMI_CONNECT` / `WMI_DISCONNECT` | |
| 0x1840 | `WMI_RX_MGMT_PACKET` | кадры управления, которые прошивка отдаёт хосту |
| 0x1860 | `WMI_DATA_PORT_OPEN` | |
| **0x15 / 0x16** | `WMI_PBSS_JOINED` / `WMI_PBSS_LEAVE` | нет в `wmi.h` мейнлайна, §5.6.2 |
| 0x1918 / 0x1919 / 0x191a | `PCP_STARTED/STOPPED`, `WMI_PCP_FACTOR` | 0x191a драйвер не разбирает |
| 0x1803 | `WMI_ECHO_RSP` | в 6.2 шлёт обработчик «задать BI», §5.6.1 |
| 0x1a01 | `WMI_SCHEDULING_SCHEME` | ответ на ESE CFG |
| 0xffff | `WMI_COMMAND_NOT_SUPPORTED` | |

Полная выгрузка событий 6.2 по трём отправителям —
[WMI.md, «События прошивка → хост»](https://github.com/klukonin/SparRAW-firmware/blob/main/docs/WMI.md#события-прошивка--хост);
для 4.1 полной выгрузки нет.

## 5.6 Нештатные смыслы команд и событий

### 5.6.1 `WMI_ECHO` (0x803) = «задать BI станции» [6.2]

В 6.2 из пакета RouterOS ветка 0x803 ведёт в обработчик `beacon_interval`
(0x8de7b0), который задаёт beacon_interval станции с выравниванием и шлёт
`WMI_ECHO_RSP`. Мейнлайн-драйвер при старте шлёт эхо 0x12345678 → глобал
прошивки 0x8002d4 = 0x5678 → станции программируется BI 0x5678 TU (~22,7 с) →
нет `bi1_event` → пропажа AP не обнаруживается; RouterOS шлёт 0x803 со
значением 500. В 4.1 обработчик — `wmi_handler_echo`, интервал зашит (100)
[железо]. Исправление по 802.11-2020 11.1.3.3.1 — патч драйвера 914 (0x803 с
`bss->beacon_interval` перед `WMI_CONNECT`) и правка
`l2_mgr__pcp_start_flow` (AP копирует BI в BSS +0x4e) — см.
[07-beamforming-link §7.8](07-beamforming-link.md#78-bi-станции-и-maxlostbeacons) и
[6.2/docs/BENCH.md](https://github.com/klukonin/SparRAW-firmware/blob/main/6.2/docs/BENCH.md#интервал-маяка-станции).

### 5.6.2 `WMI_PBSS_JOINED_EVENTID` (0x15) и `LEAVE` (0x16) [4.1+6.2]

Вендорский режим «Direct Connection»: член PBSS ассоциируется с соседом без
эфирных Assoc-кадров и сообщает об этом событием 0x15 вместо
`WMI_CONNECT_EVENTID`. Тело 12 байт: [0] — байт mid+148, [2..7] — MAC соседа,
[8] — nettype mid, [9] — cid. Отправитель — `l2mgr__send_pbss_joined_evt`
@0x8eea68 [4.1] («fake key installation» → KEY_ASSOC → DATA_PORT_OPEN).
Писателя `[conn+0x1c]=1`, включающего этот путь, в прошивке нет — функция
вендора недоведена, но путь цел [4.1, железо]. Номера 0x15/0x16 — наследие
Wilocity, в перечислении драйвера свободны. Мейнлайн-драйвер пишет
`Unhandled event 0x0015`; патч 913 превращает 0x15 в синтетический
`wmi_connect_event`. Рецепт подъёма прямого линка —
[07-beamforming-link §7.6.2](07-beamforming-link.md#762-прямой-линк-до-associated-4-записи).

### 5.6.3 ESE CFG, идентификатор 0xa01 [4.1+6.2]

Штатный 802.11ad TDMA (ESE): ветка 0xa01 диспетчера ведёт в
`wmi_handler_ese_cfg` @0x8f2c6c → `ese_cfg__apply` @0x8e7418
(`fw_scheduled_dti.cpp`) [4.1] и в `wmi_ese_cfg` с ответом 0x1a01 [6.2]. В
`wmi.h` номер называется `WMI_SCHEDULING_SCHEME_CMDID`, **ни один .c драйвера
его не шлёт**. Тело: до 3 аллокаций {source_aid, dest_aid, allocation_type,
block_duration 16 бит}; результат — элемент Extended Schedule (id 0x90) в
маяке. Отправка через `wmi_send` на стоковой 4.1 проходит, сосед разбирает
элемент («Known ESE») [железо]; влияние на доступ к среде не проверено.
Подробно — [research/ESE-TDMA-4100](../research/ESE-TDMA-4100.md),
[ESE.md](https://github.com/klukonin/SparRAW-firmware/blob/main/docs/ESE.md).

### 5.6.4 `WMI_PCP_START` и `abft_len` [4.1]

Обработчик 0x918 (`wmi_handler_pcp_start` @0x8e79dc) перекладывает поля в
структуру `wmi_bcon_ctrl_cmd` и зовёт обработчик 0xf (`wmi_handler_bcon_ctrl`
@0x8e7924); главные поля (`bcon_interval`, `network_type`, `max_assoc_sta`)
доходят корректно, а `abft_len` не копируется ни по одному пути — отсюда
отрицательный результат опыта «abft_len 0 → 6 не меняет A-BFT». `bi == 0`
останавливает PCP ([research/WMI-PCP-START-LAYOUT](../research/WMI-PCP-START-LAYOUT.md)).

### 5.6.5 `WMI_GET_PCP_FACTOR` (0x91b) [4.1]

Прошивка отдаёт событием 0x191a одно 32-битное слово, упакованное из
сохранённых возможностей DMG (`mid+0xa6`), — метрику пригодности на роль PCP.
Автомата Handover в прошивке нет, но метрику можно опрашивать и сравнивать на
хосте ([research/WMI-PCP-FACTOR](../research/WMI-PCP-FACTOR.md)).

### 5.6.6 Прочее

* `WMI_DISCOVERY_START` (0x916): discovery-маяки + ответчик A-BFT
  ALWAYS/DISCOVERY + немедленный `mid__acquire_link` в SEARCHING;
  мейнлайн-драйвер его не посылает [4.1].
* `WMI_LINK_MAINTAIN_CFG_WRITE` (0x842): порог `bad_beacons_num_threshold`
  задаётся на соединение и после переподключения шлётся снова; порог 200 вместо
  20 снимает разрывы 6.2↔6.2 [6.2, железо]. Патч 915 делает то же по стандарту —
  `dot11MaxLostBeacons` из элемента DMG Operation (EID 151, байт 9 тела)
  read-modify-write после подключения.
* `WMI_START_SCAN` с `discovery_mode=1` — §5.7.
* Ad-hoc: диспетчер типа сети в connect-пути принимает ADHOC(2) и
  ADHOC_CREATOR(4); роль в слое маяка определяется `bss_mode ∈ {2,3}`, а не
  типом сети: `WMI_PCP_START(ADHOC_CREATOR)` даёт `bss_mode=2` и маяк.

## 5.7 Рычаги со стороны хоста без патча прошивки

Все значения — host-адреса (§5.2). Записи в ОЗУ требуют `mem_write` (патч 911)
и не переживают сброс прошивки; `wifi reload` устройство **не** сбрасывает —
записи в ОЗУ ucode его переживают, чистое состояние даёт только питание.

| рычаг | как включить | эффект | версия, статус |
|---|---|---|---|
| `discovery_mode` | `echo 1 > …/discovery_mode` + активный скан | по коду: `WMI_START_SCAN` с `discovery_mode=1` → `add_discovery_bcon` @0x8c22e4 → ucode-команда 0x0b с `bi_mode=2` → AW пропускается. На железе: период A-BFT = 1 (против 4), AW снят; передача discovery-маяков так не запускается | [4.1]; AW — [железо] |
| `oob_mode=1` | параметр модуля (только `insmod … oob_mode=1`, sysfs 0444) | бит 31 `RGF_USER_USAGE_6` → `pf_mode_en` [0x803468]=1 → ответчик A-BFT (ALWAYS, ALL, relaxation 1); путь «Discovery A-BFT link up» есть **только** в OOB; OOB выключает детекторы разрыва и PS, MSDU 7976, dwell скана 1,5 с | [4.1, железо]; `oob_mode=2` на Sparrow не действует |
| OOB в 6.2 | тот же регистр | прошивка читает биты 31..29 целиком: 4 → R1 OOB (выкл. PM), 2 → R2 | [6.2, код] |
| ответчик A-BFT вручную | `0x942090=2`, `0x942094=2`, `0x942098=0` | ALWAYS, ALL, relaxation 0; счётчики в 0x941e68+0x1c..+0x26 | [4.1, железо]; перезаписывается при рестарте ucode, `pcp_start`, входе в find-search |
| `special_flags` бит 9 | `0x903474 = 0x200` после загрузки прошивки | маяк соседа без PCP Association Ready не отбрасывается. Работают биты 0,1,2,3,6,7,9,10; **бит 10 рвёт все MID по Probe Request**; кандидат `0x28a` | [4.1, железо] |
| рандомизатор маяка | `0x941438 = 2` (`bcon_kind`) | уходит команда MAC 0x30 с задержками 58..182; AW снят, `MAC_MON [DTI]` не пишется; маяк не сдвигается | [4.1, железо]; [06](06-beaconing.md) |
| Clustering Control | `0x905a3c = 0x81000000` | B0 CC Present: маяк длиннее на 8 октетов, партнёр разбирает; бит держится | [4.1, железо] |
| период длинного BTI | байт `+0x04` блока конфигурации BI (host 0x94143c) | 2 → чередование длинного и короткого BTI | [4.1, железо] |
| AW без DTI | ucode `0x800458 = 0` или `0x80049c != 0` (host 0x940458 / 0x94049c) | снимают окно AW, не трогая DTI | [4.1, железо] |
| прямой линк без ролей | узел с conn в READY_FOR_ASSOC, `oob_mode=1`: `0x905940=1` (`m_bss_mode`), `conn+0x18=1`, `conn+0x1c=1`, затем `conn+0xfc=0x00090b02` (последним) | следующий A-BFT link-up доводит до ASSOCIATED + `WMI_PBSS_JOINED` + `DATA_PORT_OPEN`; для юникаста `0x941034=0` на обоих | [4.1, железо]; [07 §7.6](07-beamforming-link.md#76-безролевой-линк) |
| MCS вручную | `0x940658 = 0x80\|MCS` (на cid), затем `0x940520 = 1<<cid`, `0x940630 = 2` | фиксирует MCS; авто-MCS в прямом линке не работает | [4.1, железо] |
| ручной BF к соседу | байт причин `0x91f848 = 0x00001000` (cid0) + трафик | SLS-перенастройка при `0x801034=0` проходит и закрывается сама | [4.1, железо] |
| порог плохих маяков | `WMI_LINK_MAINTAIN_CFG_WRITE` через `wmi_send` | 200 вместо 20 | [6.2, железо] |
| BF-политика | `WMI_BF_CONTROL brp_mode=0`, `WMI_BF_TRIG` | SLS без BRP; запуск BF с хоста | [6.2, код] |
| сектора маяка | `WMI_PRIO_TX_SECTORS_ORDER/NUMBER` | число и порядок секторов | [6.2, код] |
| выбор PCP | `WMI_GET_PCP_FACTOR` | метрика пригодности на роль PCP | [4.1, код] |

Механизм OOB и ответчика A-BFT —
[ROLELESS-LINK.md](https://github.com/klukonin/SparRAW-firmware/blob/main/docs/ROLELESS-LINK.md);
`special_flags` — там же, §1.5.

Замечания:

* Безролевой A-BFT-линк (два PCP с одинаковым или разным SSID, оба
  `oob_mode=1`) доходит до READY_FOR_ASSOC на стоковой прошивке; хост о нём не
  узнаёт. Смешанная пара 6.2 + 4.1 в OOB тоже проходит Discovery A-BFT.
* В рецепте прямого линка важен порядок: гейт `conn_main_sm__linkup_ntf`
  проверяется только на переходе WAIT_FOR_LINKUP→READY.
* `m_bss_mode=1` включает политику BF RETRY_FOREVER (выбор в
  `lm_main_sm__on_maintain_started` @0x8d9c6e по `m_bss_mode`) — поэтому нужна
  запись `0x941034=0`.
* Сброс прошивки с хоста: запись нулей в байты уровней лога (`0x90b904 = 0`) —
  патч 907 считает это рестартом и запускает `wil_fw_error_recovery`;
  `recovery` в debugfs только продолжает уже начатое.

## 5.8 Лог прошивки и трасса ucode

У устройства два независимых кольца: прошивки (адрес в `RGF_USER_USAGE_1`,
на стенде 0x843900 в `fw_peri`, host 0x90b900) и ucode (`RGF_USER_USAGE_2`).
Раскладка, включение и опыты —
[research/TRACING-TOOLKIT](../research/TRACING-TOOLKIT.md); адреса колец —
[HARDWARE-BLOCKS, «Публикация адресов логов»](https://github.com/klukonin/SparRAW-firmware/blob/main/docs/HARDWARE-BLOCKS.md#публикация-адресов-логов-драйверкод).

**Кольцо прошивки.** Заголовок `{u32 write_ptr; u8 module_level_enable[16];}`
(b0 err, b1 warn, b2 info, b3 verbose), затем записи. Прошивка пишет только при
ненулевых байтах уровней; вендорские образы отдают их нулевыми. Кольцо держит
~300–320 записей (~1,5 с на маячащем AP) [4.1]. Прошивка 5.2 отдаёт уровни
0x07070707 уже включёнными, но без таблиц строк её лог не расшифровать [5.2].
Включение — патчи 905 (параметр `fw_log_level`, нужен сброс) и 906 (debugfs
`fw_log_level` без сброса); 909 взводит тем же уровнем кольцо ucode.

**Модули** (индекс = байт уровня) [4.1]: 0 SYSTEM, 1 DRIVERS, 2 MAC_MON,
3 HOST_CMD, 4 PHY_MON, 5 INFRA, 11 CONN_MGR, 14 POWER_MNGR. MAC_MON и PHY_MON
вытесняют всё остальное меньше чем за секунду, их глушат записью байтов уровней
(`0x90b904`, `0x90b908`, little-endian); любой сброс прошивки перевзводит уровни
драйвером (`wil_log_ring_arm`) и затирает ручную маску —
[LMAC-PROTOCOL §12](https://github.com/klukonin/SparRAW-firmware/blob/main/docs/LMAC-PROTOCOL.md#12-глушение-шума-в-логе-прошивки-железо).
Модули ucode 6.2: SYSTEM, TX, RX, ISR, BCON, BEAMFORM, SXD_UTILS, RSSI, CALIBS,
DRIVERS.

**Формат записи.** [4.1]: 32-битный заголовок — биты 0..19 смещение строки,
20..23 модуль, 24..25 уровень, 26..27 число аргументов, 28 `is_string`;
аргумент-строка кодируется `0x01000000 | offset`. [6.2]: за каждым заголовком
(fw и ucode) идёт **слово времени**; заголовок `0xa4/0xa8/0xac000000 |
байт_времени << 18 | смещение`, модуль и уровень не кладутся (проверяются по
таблице разрешений 0x803238). Старшая тетрада 0xa/0xb одинакова в обеих
версиях — **по биту 31 формат не различать**; декодеры (`wil_fw_log.pick_ts`,
`wil_uc_collect.py`) выбирают формат по монотонности слова за заголовком.

**Таблицы строк** — записи контейнера **101** (fw) и **100** (ucode). Они есть
только в полных образах из пакета RouterOS; лог стриппованного образа читается
по таблице из полного образа той же версии ([08-versions §8.1](08-versions.md#81-контейнер-образа)).

**Кольцо ucode.** [4.1]: индекс 0x80209c (по модулю 256, его публикует
`RGF_USER_USAGE_2`), байт разрешения 0x8020a0 (в вендорском 4.1 уже 0x07),
записи 0x8020b0 (256 слов); заголовка с 16 байтами уровней нет, аргументы лежат
перед словом-заголовком; читается из `blob_uc_data`. [6.2]: индекс 0x803234,
кольцо 0x803248, заголовок `0xac000000 | …`. Кольцо держит ~1,5 с; вычерпывание
из userspace теряет записи и даёт ложные «пропуски маяков», поэтому патч 910
собирает поток в драйвере (буфер 1 МБ пачками {magic, число слов, ktime},
debugfs `uc_trace`; за 67,9 с потеряно одно событие из 668). Чтение `uc_trace`
буфер не освобождает — окно замера задаётся по времени пачек
(`wil_uc_collect.py --since/--until`).

**Что видно.** MAC_MON [4.1] — поинтервальный отчёт BTI/AW/DTI, `tx bcon`,
`rx bcon`, `detected`, `bcon bitmap`, ATIM (в 6.2 эта телеметрия вырезана);
ucode — цикл интервала `L1_TASK PRE TBTT` → `sending event=0x17` →
`BI_AP_MONITOR_SM: AW EVENT` → `DTI EVENT` и строки рандомизатора. Кольцо команд
MAC — первый килобайт `blob_uc_data`, снимается без патчей циклом `dd`
([03-mac](03-mac.md)).

## 5.9 Трассировщик WMI-команд

**[4.1]** Логгер команд хоста (функция @0x8da54e,
1444 Б, 32 идентификатора WMI) печатает на каждую команду пару строк
`HOST CMD 0x… for MIDID n` / `---->> [HOST CMD] WMI_…_CMDID`: обмен драйвера с
прошивкой виден без патча драйвера. Сброс прошивки (в том числе
`/etc/init.d/wpad restart`) затирает маску уровней, поэтому последовательность
команд при старте AP так не снимается —
[research/TRACING-TOOLKIT](../research/TRACING-TOOLKIT.md#трассировщик-команд-wmi-в-логе-fw-код--железо).

| инструмент (SparRAW-tools) | назначение |
|---|---|
| `host/wil_fw_log.py` | разбор кольца fw или ucode из снимка (`-a АДРЕС --region fw_peri\|uc_data`), `--extract-strings IMAGE.fw` |
| `host/wil_uc_stream.py` | склейка потока из снимков по `write_ptr` |
| `host/wil_uc_collect.py` | разбор пачек сборщика `uc_trace` |
| `host/wil_mac_ring.py` | снятие кольца команд MAC |

## 5.10 Особенности эксплуатации

| особенность | суть | обход |
|---|---|---|
| ложные сбросы сторожа 907 | 907 считает FW зависшей, если `write_ptr` не движется 5 с; верно только для AP 4.1 (MAC_MON пишет каждый BI). На 6.2 и на STA 4.1 до ассоциации лог молчит → «FW log stuck … recovering» каждые 6–18 с | `echo 0 > …/fw_watchdog_ms`; контроль — `dmesg \| grep -c recovering` |
| скан 6.2 после подключения | `WMI_CONNECT` идёт через `scan_mngr`, скан не останавливается: STA уходит на другой канал, теряет маяки → LOST_LINK, цикл ~3 с | `scan_freq 58320`, `freq_list 58320` в wpa_supplicant; `uci wireless.radio0.scan_list=58320` |
| зависание в `wil_reset` | узел умирает внутри `wil_reset` на втором сбросе, после `wil_get_bl_info` и до `wil_set_oob_mode`; ни ping, ни HW-watchdog; похоже на залипание PCIe/AXI; встречается и на 6.2 при загрузке модулей | цикл питания, повторный — с паузой 60 с |
| тихий рестарт прошивки | под нагрузкой на передачу прошивка перезапускается без ассерта (0x91f020/24 = 0) и без события; признак — нулевые байты уровней и сброшенный `write_ptr`; TX-кольцо остаётся DRV_XOFF | патчи 907 (сторож) и 908 (снос TX-колец при сбросе) |
| `rmmod wil6210` | самодедлок на мьютексе, задача в D, `reboot` не завершается | прошивку менять подменой файла + цикл питания; параметры модуля — файлом для `insmod` в `rc.local` |
| мягкий reboot на 6.2 | зависает | цикл питания |
| ассерт 0x12aa [5.2] | linux-firmware 5.2.0.18 под iperf3 падает каждые ~9 с, 274–367 Мбит/с; на 4.1 ассертов нет, 971 Мбит/с | 5.2 не использовать |
| `wifi reload` | устройство не сбрасывает, записи в ОЗУ ucode живут | питание перед опытом |
| следы прошлых опытов | записи в память, таблицы ucode, зависшие станции делают замеры несравнимыми | перед каждым опытом — питание обоих узлов |
| `oob_mode` через modprobe | `modprobe` OpenWrt (kmodloader) молча теряет параметры | только `insmod` под `setsid` |
| формат образа для OpenWrt | мейнлайн `fw_handle_record` даёт -EINVAL на записях 100/101/102 | `fw_strip_for_openwrt.py` ([08-versions §8.1](08-versions.md#81-контейнер-образа)) |
| имя образа для D0 | Sparrow D0 просит `wil6210_sparrow_plus.fw`, которого нет → `-ENOENT` | патч 902 |
| board-файл | качество 60 ГГц определяет `.brd`, не DTS | board под модель ([07 §7.9](07-beamforming-link.md#79-board-файлы)) |
| netconsole в rc.local | цикл `rmmod/insmod netconsole` рвёт список планировщика ядра, узлы падают | модуль грузить штатно; сигналы о прошивке — строка 907 и побайтовая сверка `blob_*_code` |
| PS по умолчанию | `NL80211_CMD_SET_POWER_SAVE` уходит до подъёма FW и игнорируется; RTT 170–370 мс вместо 0,8 | [патч 904](../../SparRAW-driver/openwrt/patches/ath/904-wil6210-no-default-ps.patch) |

## 5.11 Патчи драйвера SparRAW

Серия quilt-патчей для `package/kernel/mac80211`; правка только через quilt
([SparRAW-driver](../../SparRAW-driver/README.md)). Базовое дерево — backports
7.2 OpenWrt master, таргет ipq40xx
([research/DRIVER-PATCHES-OPENWRT](../research/DRIVER-PATCHES-OPENWRT.md)).

| № | что добавляет | где |
|---|---|---|
| 900 | `NL80211_IFTYPE_ADHOC` в `interface_modes` и `wil_mgmt_stypes` | [06](06-beaconing.md) |
| 901 | `join_ibss`/`leave_ibss` поверх PBSS: creator → `wmi_pcp_start(ADHOC_CREATOR)` | [06](06-beaconing.md) |
| 902 | запасное имя образа: `wil6210_sparrow_plus.fw` → `wil6210.fw` | §5.10 |
| 904 | PS выключен по умолчанию | §5.10 |
| 905 | параметр `fw_log_level`: байты уровней кольца лога FW после FW ready | §5.8 |
| 906 | debugfs `fw_log_level` — включение лога без сброса | §5.8 |
| 907 | сторож `fw_watchdog_ms`: нулевые байты уровней или стоящий `write_ptr` → `wil_fw_error_recovery()` | §5.10 |
| 908 | снос TX-колец в `wil_reset()` (иначе очередь навсегда DRV_XOFF) | §5.10 |
| 909 | взвод кольца ucode тем же уровнем | §5.8 |
| 910 | сборщик трассы ucode `uc_trace_ms` + debugfs `uc_trace` | §5.8 |
| 911 | debugfs `mem_write` | §5.2 |
| 912 | параметр `ibss_creator`: создать ячейку или присоединиться (`ADHOC`) | [06](06-beaconing.md) |
| 913 | событие 0x15 `WMI_PBSS_JOINED` → connect | §5.6.2 |
| 914 | станция берёт BI из BSS (`wmi_set_sta_bcon_int`, 802.11-2020 11.1.3.3.1) | §5.6.1 |
| 915 | `dot11MaxLostBeacons` из DMG Operation (11.1.3.1) | §5.6.6 |

Описание каждого — в шапке файла патча
([openwrt/patches/ath](../../SparRAW-driver/openwrt/patches/ath/)).

## Противоречия

* Описание патча 909 говорит, что кольцо ucode имеет «тот же заголовок», что
  кольцо fw; разбор логгера ucode 4.1 (0x925548) показывает раскладку без
  16-байтового заголовка уровней (§5.8). Действует раскладка по коду.

## Не установлено

* Полная выгрузка событий прошивка → хост для 4.1.
* Назначение ветки 0xfff диспетчера 6.2.
* Влияет ли аллокация ESE (0xa01) на реальный доступ к среде.

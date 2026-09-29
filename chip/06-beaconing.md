# 6. Интервал маяка и биконинг

Глава — обзор того, как Sparrow строит интервал маяка (BI) 802.11ad, кто
переключает его окна, где лежат период и расписание, как устроена передача
маяка и что осталось от распределённого биконинга по патенту US8520648B2.
Механика в коде (адреса обеих версий, листинги, разбор рандомизатора и
NAV-отсрочки, опыты на железе) —
[BEACONING.md](https://github.com/klukonin/SparRAW-firmware/blob/main/docs/BEACONING.md);
раскладка блока конфигурации 4.1 —
[4.1/docs/BEACON-CONFIG.md](https://github.com/klukonin/SparRAW-firmware/blob/main/4.1/docs/BEACON-CONFIG.md);
соответствие стандарту (ATI/ATIM, A-BFT, кластеризация) —
[STANDARD-MAPPING.md](https://github.com/klukonin/SparRAW-firmware/blob/main/docs/STANDARD-MAPPING.md).

Соседние главы: регистры MAC — [02-registers](02-registers.md); кольцо команд
MAC и события r42/r54 — [03-mac](03-mac.md); протокол прошивка↔ucode —
[04-fw-ucode](04-fw-ucode.md); WMI и debugfs — [05-host-interface](05-host-interface.md);
беамформинг и линк — [07-beamforming-link](07-beamforming-link.md); версии —
[08-versions](08-versions.md).

**Обозначения.** Версия: **[4.1]**, **[6.2]**, **[обе]**. Доказанность:
**[железо]** — измерено на стенде (два узла wAP 60G, узлы A и B), **[код]** —
по листингу, **[гипотеза]** — не доказано. Адреса ucode — в линкерном
пространстве; host-адрес для `mem_write` = 0x940000 + (A − 0x800000).

---

## 6.1 Структура BI по 802.11ad и её отражение в чипе

| окно стандарта | что в нём | как видно в чипе |
|---|---|---|
| BTI | PCP/AP передаёт DMG Beacon по секторам (I-TXSS) | `bti_duration`, `tx bcon` = 63 (развёртка) |
| A-BFT | состязательный период, ответчики делают R-TXSS в случайном слоте | ответчик A-BFT ([гл. 7, §7.2](07-beamforming-link.md#72-ответчик-a-bft-и-режим-oob)) |
| ATI | Announcement Transmission Interval | «окно AW», `aw_duration` |
| DTI | данные (CBAP и SP) | `dti_duration` |

Ключевые величины **[4.1, железо]**:

| величина | значение |
|---|---|
| сумма окон при BI = 100 TU | ровно 102 400 мкс (BTI 1364…1954, AW 1003…1006, DTI 99 442…100 032) |
| BTI | двумодален: 1366 либо 1951 мкс; +585 мкс — окно A-BFT, выделяемое не в каждом BI (§6.5) |
| окно AW | 1000 мкс по константе `power_mngr__recalc_aw` (165 000 тактов в MAC-командах 0x68/0x67); снятое окно отдаёт ~1005 мкс в DTI |
| такт MAC | 165 МГц (165 тактов/мкс, `us_from_ticks165`), проверено по IFS-регистрам 0x886d00..0x886d0c |

**ATI в эфир не объявляется** **[4.1, код]**: бит `AT_preset` поля Beacon
Interval Control гасится при сборке маяка; окно AW открывается только локально,
поэтому unicast-ATIM уходят без ACK
([STANDARD-MAPPING, «ATI и механизм ATIM»](https://github.com/klukonin/SparRAW-firmware/blob/main/docs/STANDARD-MAPPING.md#ati-и-механизм-atim)).
11 подряд неудачных ATIM ведут к разрыву (`tx_ageing_detector` →
`conn__disconnect`); детектор вооружён только вне режима OOB.

## 6.2 Автомат BI

Автомат интервала маяка — экземпляр общего движка `basic_sm` (формат таблиц —
[04-fw-ucode](04-fw-ucode.md)). Состояния: 0 BTI, 1 ABFT, 2 DTI (начальное),
3 AW; события: DTI_TO_BTI, BTI_TO_ABFT, BHI_TO_DTI, BHI_TO_AW, AW_TO_DTI
(BHI = BTI + A-BFT); имена — из строк 6.2, нумерация совпадает с 4.1.

| | 4.1 | 6.2 |
|---|---|---|
| имя автомата | `BI_AP_MONITOR_SM` | `bi_sm` |
| объект / экземпляр | 0x801de8 | 0x800600 (gp+0xd8) |
| таблица переходов / описатель | 0x800adc, запись 5 Б | описатель 0x801adc (20 Б) |
| блок счётчиков окон | счётчики 0x801e1c (DTI) / 0x801e1e (AW) | 0x802ec0 (host 0x942ec0): +0x04 номер BI, +0x12 текущее окно |

**События подают аппаратные таймеры, а не компаратор TSF** **[4.1, код]**:
AW→DTI ведут GP-таймеры (r54 бит 15 `ext_gp_0_1_2_end_ind`), DTI→BTI — r42
бит 4 (`bi1_event`, путь TBTT). GP-таймеры программируются командами кольца
MAC 0x4f..0x54 и квитируются командой 0x49; по квитированиям и маркерам
`0xcafNNN` в кольце смена окон видна без патчей **[4.1, железо]**. Ни одно из
168 мест ожидания ucode не ждёт бит 6 r42 `tsf_event`.

Подробно (обработчики переходов, входы диспетчера, блок счётчиков 6.2, блоки
BI/BTI/AW микрокода) —
[BEACONING §6](https://github.com/klukonin/SparRAW-firmware/blob/main/docs/BEACONING.md#6-автомат-bi-и-окна),
[MAC-COMMANDS, приложение В.1](https://github.com/klukonin/SparRAW-firmware/blob/main/docs/MAC-COMMANDS.md#в1-автомат-bi-микрокод-41),
[MAC-COMMANDS §7.6](https://github.com/klukonin/SparRAW-firmware/blob/main/docs/MAC-COMMANDS.md#76-gp-таймеры-и-r54-0x49-0x4f0x54-0x670x68).

## 6.3 Окно AW: три гейта

Вход в AW решает `bi_ap_mon_if__trigger_bhi` @0x930564 **[4.1, код + железо]**:

| гейт | условие «AW нет» | host-адрес | кто пишет |
|---|---|---|---|
| A | uc 0x800458 == 0 | 0x940458 (в образе 1) | только хост |
| B | uc 0x80049c != 0 | 0x94049c (в образе 0) | только хост |
| C | `bcon_kind` (0x801438) == 2 | 0x941438 | команда 0x0b |

Гейт C делает **рандомизатор маяка и окно AW взаимоисключающими**; роли PCP/STA
в гейте нет. Цена окна — 1009,6 мкс из 102 400 (0,98 %), со снятым окном —
5,0 мкс; выигрыш ниже разброса замера трафика. Для обучения секторов окно не
нужно, оно несёт учёт сна пиров
([research/AW-WINDOW](../research/AW-WINDOW.md),
[STANDARD-MAPPING, гейт окна AW](https://github.com/klukonin/SparRAW-firmware/blob/main/docs/STANDARD-MAPPING.md#гейт-окна-aw--bi_ap_mon_if__trigger_bhi-0x930564)).

## 6.4 Период маяка и конфигурация BI

**Период.** Командой MAC период не задаётся. Горячий регистр — **0x886d64**
(период в TU, пишет ucode): запись на лету останавливает маячный цикл. Длительность
BI для ucode — 0x801434 (мкс); ещё около десятка ячеек пропорциональны периоду,
но ни одна из них поодиночке период не меняет; штатно его меняет только
`wifi reload` **[4.1, железо]**. Таблица ячеек и опыты —
[MAC-COMMANDS §9.1](https://github.com/klukonin/SparRAW-firmware/blob/main/docs/MAC-COMMANDS.md#91-период-маяка).
Период станции из маяка точки не берётся ни в 4.1, ни в 6.2 —
[гл. 7, §7.8](07-beamforming-link.md#78-bi-станции-и-maxlostbeacons).

**Расписание окон от периода не зависит.** `bi_build_schedule_tables` @0x927f00
при инициализации строит таблицы длительностей (шаг 2624), из которых
`ucode_cmd__bi_config` по индексу FSS выбирает значения 0x801afc/0x801b00 и
длину A-BFT 0x801b04 = (A-BFT Length + 1)·B[FSS] **[4.1, код + железо]** —
[research/ABFT-WINDOW §2](../research/ABFT-WINDOW.md#2-таблицы-расписания-код--железо).

**Блок конфигурации BI** `fui_bi_cfg_params_s` — 32 байта, приходит из тела
LMAC-команды 0x0b (`CMD_BCON_MGT`); 4.1: 0x801438 (host 0x941438), 6.2: 0x8022a8.

| смещ. | поле |
|---|---|
| +0x00 | **`bi_mode`** («bcon_kind»): 0 только приём, 1 обычный маяк, 2 discovery при активном скане, 3 discovery при direct scan |
| +0x04 | `n_bis_abft` — период окна A-BFT в интервалах (живьём 4) |
| +0x07 | FSS (живьём 0x0f) — индекс в таблицы расписания |
| +0x0a | A-BFT Length |
| +0x14 | флаги: CC Present / Discovery Mode / FragmentedTXSS / PCP Assoc Ready |
| +0x18, +0x1c | 64-битный накопитель TSF рандомизатора |

Полная раскладка и источники полей —
[4.1/docs/BEACON-CONFIG.md](https://github.com/klukonin/SparRAW-firmware/blob/main/4.1/docs/BEACON-CONFIG.md),
[LMAC-PROTOCOL §3.2](https://github.com/klukonin/SparRAW-firmware/blob/main/docs/LMAC-PROTOCOL.md#32-команда-0x0b-fui_bi_cfg_params_s-и-дескрипторы-развёртки-код-пак).

## 6.5 Окно A-BFT и его рычаги

Длинный BTI в 4.1 — окно A-BFT: в конце развёртки маяка счётчик периода
0x80142c доходит до нуля, взводится байт 0x801e69, в кольцо MAC уходит
последовательность окна и включается `abft_responder_worker` **[4.1, код +
железо]**. Таймеры GP0/GP1/GP2, программируемые после развёртки, задают
геометрию A-BFT **[6.2, код]**.

| рычаг | адрес (host) | эффект | статус |
|---|---|---|---|
| период окна `n_bis_abft` | 0x94143c, байт +0x04 | 1 — длинные все BI, 2 — чередование, 4 — сток; при 1 темп ответов SSW 2,4 → 9,8 в с | **[4.1, железо]** |
| длина окна `0x801b04` | 0x941b04 | 95 600 → 585 мкс, 191 200 → 1135, 286 800 → 1691, линейно | **[4.1, железо]**, см. «Противоречия» |
| A[idx] `0x801afc` | 0x941afc | не влияет | **[4.1, железо]** |
| `abft_length` (+0x0a) | 0x941440 | не влияет ни на длину окна, ни на темп SSW | **[4.1, железо]** |
| `abft_len` драйвера (`WMI_PCP_START`) | — | в 4.1 не доходит до прошивки ([05 §5.6.4](05-host-interface.md#564-wmi_pcp_start-и-abft_len-41)) | **[4.1, железо + код]** |

Опыты — [research/ABFT-WINDOW](../research/ABFT-WINDOW.md);
`abft_length` и асимметрия узлов —
[STANDARD-MAPPING, «A-BFT»](https://github.com/klukonin/SparRAW-firmware/blob/main/docs/STANDARD-MAPPING.md#a-bft).

**Асимметрия длинного интервала** **[4.1, железо]**: при побайтово одинаковой
конфигурации BI у двух PCP длинный интервал устойчиво различается в пять раз
(≈ 89 мс против ≈ 17 мс); `abft_length`, CC Present, `special_flags`, 0x941b04
её не объясняют. Базу и результат рычага снимают на одном узле и серией;
`bti_duration` берут из лога прошивки, а не выборкой из `blob_fw_peri`
(структура двухбуферная).

**6.2** **[код]**: обработчик `wmi_pcp_start_cmd_handler` 0x8ec840 читает
`abft_len` по +0x12 (1…8, хранится как len−1) и `max_assoc_sta` по +0x0d; строка
лога «wmi_pcp_start_cmd_handler: set abft_len = %d».

## 6.6 Путь маяка: от WMI до эфира

Конфигурация задаётся однократно: `WMI_PCP_START` / `WMI_BCON_CTRL` →
`l2_mgr__pcp_start_flow` (`m_bss_mode` = 3 для AP, иначе 2) → LMAC-команда
0x1b (в 6.2 выбор автомата в `l1_task__main_step`, 0x92cdc8, `ld [gp,-0x34]`)
и LMAC-команда 0x0b (содержимое маяка и блок конфигурации BI). Дальше ucode и MAC передают маяк сами: на интервал 1,00
события PRE TBTT и 1,00 события awake peers, команд прошивки, связанных с
маяком, — 0,00 **[4.1, железо]**.

Передача: событие TBTT (`kind = 0`) → автомат BI → развилка «что передавать»
(6.2: `bti__prepare_sweep_set` 0x935b9c, прежнее имя `mac_cmd_0x0f__935b9c`)
→ развёртка `bti_worker__bti_transmitter_beacon_sweep_flow` (4.1 0x923de0,
6.2 0x923cb4) → шаг по секторам. Момент выдачи задают ожидания внутри
развёртки: `slot_event | busy2` (последний барьер — 0x924058 [4.1]), затем
`txop_event`. Путь `kind = 1 → bti_transmitter_bi2_flow` маяк не передаёт —
он обслуживает окна GP для BF-метрики. Запретительной проверки «STA не
маячит» в 6.2 нет: три гейта (`m_bss_mode`, 0x8014c0, 0x8004f4) питаются
командами хоста.

Подробно —
[BEACONING §2.2, §3](https://github.com/klukonin/SparRAW-firmware/blob/main/docs/BEACONING.md#3-путь-передачи-маяка).

## 6.7 Распределённый биконинг: патент и рандомизатор

**Патент US8520648B2** («Beacon transmission techniques in directional
wireless networks», Intel, Carlos Cordeiro; подача 2010-06-14, выдан
2013-08-27, статус «Expired — Fee Related» по данным Google Patents): в каждом
BI станция выбирает `delay ~ U(0, RangeMax)`, ждёт TBTT, отсчитывает delay;
услышав чужой маяк — откладывает свой до следующего BI, иначе передаёт серию
направленных маяков. Целевая среда — mmWave IBSS; в каждом BI маячит
фактически одна станция. Слабость схемы: «не услышал» ≠ «никто не передавал»
— в направленных 60 ГГц соседа слышно только при наведённом RX-луче. Текст и
сопоставление с кодом — [research/PATENT-US8520648-SPEC](../research/PATENT-US8520648-SPEC.md),
[BEACONING §4.3](https://github.com/klukonin/SparRAW-firmware/blob/main/docs/BEACONING.md#43-сверка-с-патентом-us8520648b2).
Стандартный флаг *Decentralized PCP/AP Clustering* прошивка 4.1 безусловно
гасит, в эфир он не объявляется **[4.1, код + железо]**.

**Рандомизатор** стоит в начале развёртки маяка и включается только при
`bi_mode == 2` (discovery-маяк при активном скане) и `kind == 0`:

```
uniform = ((r47 × 0xbd) >> 32) + 10      ; r47 — аппаратный LFSR, 10…198
накопитель TSF += uniform << 10          ; задержка uniform × 1024 мкс
set_tsf_event(накопитель)                ; компаратор TSF; в 6.2 — MAC-команда 0x30
```

Маяк PCP идёт с `bi_mode = 1` (`pcp_start` гасит discovery_mode) —
детерминированно по TBTT **[обе, код]**. Однобайтовый патч гейта (для девяти
образов Sparrow с рандомизатором) —
[4.1/docs/DISTBCN-PATCH.md](https://github.com/klukonin/SparRAW-firmware/blob/main/4.1/docs/DISTBCN-PATCH.md).

**Состояние на железе** **[4.1, железо]**:

| опыт | результат |
|---|---|
| запись `0x941438 = 2` с хоста (без патча) | гейт открыт, команда 0x30 идёт (задержки 58…182), связь цела; побочно снят AW и перестаёт писаться `MAC_MON [DTI]` |
| однобайтовый патч гейта | `randomize beacon` в 197 из 197 снимков против 0 из 900 на стоке, `uniform_random` 10…198, среднее 104,2 |
| сдвиг маяка | нет: интервалы ровно 102 400 мкс, корреляция `next_tsf − current_tsf` с `uniform_random` r = 0,012, фаза начала BTI дрожит ≤ 2 мкс |

**Почему рандомизация «немая»**: путь передачи не ждёт события компаратора TSF
(развёртка ждёт только `slot_event|busy2` и `txop_event`); база накопителя не
привязана к TSF и цель лежит в прошлом; починка базы (C++-переписанная развилка
`bi::decide_beacon_kind`) не помогает; автомат на TBTT может сидеть в AW и идти
мимо развилки. Расписание маяка задаёт конфигурация BI в MAC (GP-таймеры,
TBTT), а не компаратор TSF — менять надо её. Метрика MAC_MON `[BTI] start
time` для оценки сдвига кадра внутри BTI непригодна. Разбор —
[BEACONING §9.2](https://github.com/klukonin/SparRAW-firmware/blob/main/docs/BEACONING.md#92-рандомизатор-маяк-не-сдвигает-41-железо--код),
журнал опытов — [research/RANDOMIZER-ROOT-CAUSE](../research/RANDOMIZER-ROOT-CAUSE.md).

Точка вставки задержки маяка — последнее ожидание перед выдачей кадра
(0x924058); варианты — пауза на GP0 перед 0x92400c либо сдвиг TBTT через слова
BI-конфига 0x8005a8/0x8005ac **[гипотеза]**, не применялись
([BEACONING §3.3](https://github.com/klukonin/SparRAW-firmware/blob/main/docs/BEACONING.md#33-ожидания-внутри-развёртки-код)).

Рандомизатор есть во всех версиях 4.1…wil6436 7.5 и отсутствует в 1.4, 2.2 и
Terragraph 10.x; в 6.2 добавлено фиксированное расписание — [08-versions §8.5](08-versions.md#85-биконинг-через-версии).

## 6.8 NAV-отсрочка маяка

Второй, штатный механизм расхождения маяков лежит в той же развёртке сразу за
гейтом рандомизатора: пока программный NAV не истёк, проход развёртки
пропускается (потолок 30 мс, затем маяк уходит принудительно). NAV взводится
только от кадров чужого BSS (r38 бит 28 `neighbour_indication`) **[обе код;
4.1 железо]**.

| узел (оба маячат, разные BSS) | чужих маяков | NAV-задержек | размах интервалов DTI |
|---|---|---|---|
| A | 2965 | 0 | 1169 мкс |
| B | 3236 | **14 850** | 5914 мкс |

Маяк реально сдвигается штатно, без патча. В конфигурации «AP + станция»
маячит один узел, и NAV не взводится. Код, состояние `g_nav_db` 0x800994 и
возможные правки —
[BEACONING §5](https://github.com/klukonin/SparRAW-firmware/blob/main/docs/BEACONING.md#5-nav-отсрочка-маяка-обе-код-41-железо).

## 6.9 MAC_MON и маяки соседей

Модуль `MAC_MON` лога прошивки печатает раз в BI отчёт ucode: начало и
длительность BTI/AW/DTI, `tx bcon`, `rx bcon`, `detected`, `bcon bitmap`
**[4.1, железо]**. Приёмная половина распределённого биконинга работает
штатно: станция, не маяча, принимает 64 маяка за развёртку и детектирует
соседа (AP 63/0/0, станция 0/64/1). `bcon bitmap` мертва по коду: писателя у
неё нет; единственный потребитель отчёта — `bad_beacons_detector`. В 6.2
отчёт публикуется по 0x853940. Раскладка и двойной буфер —
[BEACONING §8](https://github.com/klukonin/SparRAW-firmware/blob/main/docs/BEACONING.md#8-отчёт-mac_mon-обе),
[4.1/docs/UCODE-BEACON.md §5](https://github.com/klukonin/SparRAW-firmware/blob/main/4.1/docs/UCODE-BEACON.md#5-телеметрия-mac_mon-41),
[research/MACMON-SCHEDULE](../research/MACMON-SCHEDULE.md).

Маяк соседа, не объявляющего PCP Association Ready, молча отбрасывается в
`discovery__handle_rx_frame` и до хоста не доходит — в безролевой сети это
теряет все маяки. Бит 9 `special_flags` (host **0x903474** = 0x200) передаёт
их в `scan_mngr__dband_beacon_ind` **[4.1, железо]**
([ROLELESS-LINK §1.5](https://github.com/klukonin/SparRAW-firmware/blob/main/docs/ROLELESS-LINK.md#15-special_flags--0x803474-special_mode_flags_s-пак-код-железо)).

## 6.10 IBSS на железе

**[4.1, железо + код]**:

* оба узла в IBSS (`iw ... set type ibss`, патчи драйвера 900/901) маячат сами:
  блок конфигурации ucode как у AP (`bcon_kind = 1`), `tx bcon 63`;
* узлы друг друга слышат (`beacon_neighbor_cnt` 2965 и 3236), NAV-отсрочка
  работает (§6.8);
* **прошивка IBSS не реализует**: ADHOC и ADHOC_CREATOR сливаются в один класс
  с P2P, старт ветвится только на AP (bss_mode 3) и PCP (bss_mode 2), ответы на
  Probe Request отданы хосту; IBSS-биконинга и слияния TSF нет;
* присоединяющийся узел (`ibss_creator=N`, патч 912) всё равно маячит сам;
  путь через `connect` закрыт — в DMG-маяке нет SSID.

Реалистичный путь к одноранговой сети — PBSS и безролевой линк
([гл. 7](07-beamforming-link.md)). Опыты —
[research/IBSS-ON-HW](../research/IBSS-ON-HW.md); BSSID и ad-hoc в коде —
[ROLELESS-LINK §6](https://github.com/klukonin/SparRAW-firmware/blob/main/docs/ROLELESS-LINK.md#6-ibss-и-bssid-в-41-код).

## 6.11 Clustering Control

**[4.1, код + железо]**: поле Clustering Control передаётся и разбирается
корректно (8 октетов при бите CC Present), но его единственное содержимое —
A-BFT Responder Address; логики кластера (Cluster ID, роли члена, Beacon SP
Index, выбора свободного слота) нет. Штатно CC Present гасится; с хоста поле
включается записью `0x905a3c = 0x81000000`, партнёр удлинённый маяк переваривает
([STANDARD-MAPPING, «Clustering Control в 4.1»](https://github.com/klukonin/SparRAW-firmware/blob/main/docs/STANDARD-MAPPING.md#clustering-control-в-41)).

## 6.12 P2P Find и discovery-маяки

В прошивке 4.1 есть подсистема однорангового обнаружения (`find_mngr.cpp`,
`find_main_sm.cpp`): LISTEN/SEARCH по каналам, discovery-маяки с окном A-BFT,
приём чужих discovery-маяков и `mid__acquire_link` к найденному узлу; запуск
штатно — только через P2P-device интерфейс
([HOST-INTERFACE, «Обнаружение: P2P Find»](https://github.com/klukonin/SparRAW-firmware/blob/main/docs/HOST-INTERFACE.md#обнаружение-p2p-find-find_main_sm--штатный-безролевой-путь)).
`psc_*` — согласование энергосбережения, не передача роли PCP.

| рычаг | эффект | статус |
|---|---|---|
| debugfs `discovery_mode = 1` + активный скан | `add_discovery_bcon` → uc 0x0b; discovery-маяк 16 секторов; период A-BFT = 1; окно AW снято | [железо]; цепочка — по коду |
| передача discovery-маяков | через `wil_p2p_listen()` → `wmi_p2p_cfg` → `wmi_start_listen()`, нужен запущенный P2P-device; без wpa_supplicant не запущен | не достигнуто |
| WMI 0x916 `DISCOVERY_START` | discovery-маяки + ответчик ALWAYS/DISCOVERY + `mid__acquire_link` в SEARCHING | [код]; мейнлайн не шлёт |

Опыт с `discovery_mode` — [research/IBSS-ON-HW §5](../research/IBSS-ON-HW.md#5-режим-обнаружения-как-побочный-путь-железо).

## 6.13 Детерминированная карта маяков **[гипотеза]**

Альтернатива случайной задержке — детерминированная позиция узла в расписании
маяков, вычисляемая консистентным хэшированием с ключом от GTK соты: члены,
знающие GTK, считают одну карту без переговоров, при входе/выходе узла
переезжает малая доля позиций. Опоры в чипе:

* номер интервала и текущее окно читаются с хоста (6.2: 0x942ec0 +0x04 и
  +0x12, §6.2) — для привязки задержки к номеру BI;
* носитель сдвига — TBTT (0x8005a8/0x8005ac) либо GP0-пауза (§6.7);
* носитель механизма — расписание: ESE 4.1 (`WMI_ESE_CFG_CMDID`, драйвер его не
  шлёт) или фиксированное расписание 6.2
  ([6.2/docs/FIXED-SCHED.md](https://github.com/klukonin/SparRAW-firmware/blob/main/6.2/docs/FIXED-SCHED.md));
  отказ Terragraph от рандомизатора в пользу детерминированного расписания
  согласуется с этим путём. В патенте — случайный backoff, не карта.

Узел без GTK (стадия обнаружения) карту не посчитает — нужен режим входа;
второй вход хэша (MAC/AID) не выбран.

## 6.14 Сводка рычагов биконинга с хоста

| рычаг | адрес / способ | эффект | версия, статус |
|---|---|---|---|
| снять AW | 0x940458 = 0 или 0x94049c ≠ 0 | AW 1009 → 5 мкс, DTI +1005 | 4.1, железо |
| рандомизатор | 0x941438 = 2 | команда 0x30 идёт; AW снят; `MAC_MON [DTI]` глохнет; маяк не сдвигается | 4.1, железо |
| период A-BFT | 0x94143c байт +0x04 | длинный BTI в каждом N-м BI | 4.1, железо |
| длина A-BFT | 0x941b04 | линейно (см. «Противоречия») | 4.1, железо |
| период BI | 0x886d64 | запись на лету останавливает маяк | 4.1, железо |
| CC Present | 0x905a3c = 0x81000000 | поле Clustering Control в эфире | 4.1, железо |
| маяки соседей к хосту | 0x903474 = 0x200 | снимает гейт PCP Assoc Ready | 4.1, железо |
| NAV-отсрочка | узлы в разных BSS, оба маячат | маяк пропускается до 30 мс | 4.1, железо (штатно) |

## Противоречия

* **Длина окна A-BFT через 0x801b04.** Серия записей 95 600 / 191 200 / 286 800
  дала линейный рост прибавки к BTI (585 → 1135 → 1691 мкс); запись 20 000 в
  0x941b04 на узле A «длинный» интервал ≈ 87 мс не изменила. Замеры относятся к
  разным величинам (прибавка +585 мкс к BTI против аномально длинного BHI), обе
  могут быть верны.
* **Единица длины A-BFT.** Линейная аппроксимация опыта даёт ≈ 172,8 единиц на
  мкс ([research/ABFT-WINDOW](../research/ABFT-WINDOW.md)); по коду GP0 =
  `[0x801b04]` + 495 тактов, то есть единица — такт 165 МГц. Расхождение ~5 %
  не объяснено.
* **`bi_mode` при `discovery_mode` + активном скане.** По коду
  `add_discovery_bcon` ставит 2, и во время активного скана
  `0x801438[0] = 2` наблюдается
  ([BEACONING, «Метод»](https://github.com/klukonin/SparRAW-firmware/blob/main/docs/BEACONING.md#метод));
  отдельный замер при discovery_mode + скане показал 0 при снятом AW —
  вероятно, значение держится только на время dwell.

## Не установлено

* Держится ли `bi_mode = 2` после окончания скана и можно ли штатно удерживать
  узел в discovery-биконинге.
* Чем ограничен размах NAV-отсрочки (5914 мкс против `nav_max_us` 5000) и
  отчего асимметричны узлы A/B — в NAV и в длине длинного интервала.
* Доходит ли `abft_len` драйвера до разбора по +0x12 в 6.2.

# Регистры Talyn 7.5 → Sparrow: что применимо

Вход: карта обращений Talyn 7.5.3.115 fw+uc (`tl75_{fw,uc}.tsv`, 994 адреса); снимки Sparrow
(`../sparrow_rgf/`: RGF 0x880000…0x889fff и AGC_tbl в 4 состояниях — down, vendor 6.2 AP, наш AP, наш AP
на канале 3; rgf_ext 0x88b000…0x88bfff и mac_rgf_ext 0x88c000…0x88c1ff — vendor 6.2 AP, 2026-10-23; наш образ на .12:
STA без трафика ×2 через 1 с, STA под iperf, down — `our_*.bin`); чтения ядром fw за пределами карты
драйвера (`../sparrow_rgf/fw_probe.tsv`, отладочная WMI 0xfe0 поверх open);
регистры нашей прошивки Sparrow (`inventory2.tsv`); каталог имён Talyn
(`talyn-regs.tsv`). Скрипт `sparrow_applic.py`, результат `sparrow_applicability.tsv` (строка на адрес:
класс, имена Talyn/Sparrow, значения Sparrow, константы Talyn, совпадение, функции Talyn), сводка по блокам
0x100 — `sparrow_applicability_blocks.txt`.

## Итог

| класс | адресов | смысл |
|---|---|---|
| A | 451 | прошивка Sparrow тоже обращается к этому адресу |
| B | 307 | в железе Sparrow есть, прошивка Sparrow не трогает (305 по снимкам, 2 — чтением fw: 0x88c204, 0x88c364) |
| C | 236 | у Sparrow нет: 151 — 0xdeadbeaf в снимках, 79 — чтение ядром не завершается, 6 — чтение вешает чип |

Поведение по снимкам (A+B, колонка 13): const 538, live 37 (меняется за 1 с в простое — счётчики, таймеры),
state 179 (зависит от down/AP/STA/iperf/канала).

**Класс — только про адрес, не про смысл.** Пример: фильтр 13 отводов у Talyn на 0x883d74…80, у Sparrow
на 0x883d64…70 (сдвиг −0x10). Поэтому 0x883d70 попал в A (у Sparrow — хвост фильтра 0x1fc, у Talyn —
`phy_dbg_clk`), а 0x883d80 — в B (у Sparrow там другое слово, 0xe4e4e4e4). Автоматический поиск сдвигов
(та же константа в ±0x100) других случаев не нашёл. Смысл подтверждают имена с обеих сторон (колонки 6–7),
совпадение констант (колонка 11) или опыт на стенде.

## A — общее (451)
USER/CAR, PCIe, DMA, MAC, MAC-SXD, PHY-RX, ABIF. В 45 случаях Talyn пишет другие константы, почти все —
маски прерываний ICR/IMS/IMC (другие биты), указатели в память fw, `MAC_BI_INIT_*`,
`MAC_RX2TX_RESP_START`, `DPAL_CPL_TO_CTL`. Это отличия настройки, а не железа.

## B — есть у Sparrow, не используется (305), где интересно
- **0x887100…0x8878ff (57 B + 7 A):** таблица, которую Talyn заполняет после перехода XTAL→PLL
  (`FUN_ram_0006483c` из `Switch_from_XTAL_to…`). У Sparrow в 49 из 57 адресов уже лежат те же значения,
  что пишет Talyn (у остальных 8 константа Talyn статически не извлечена). Блок общий, у Sparrow он в сбросе
  настроен так же.
- **ABIF/RF-CAF 0x889200…0x8896ff, 0x88ac00…0x88afff (~110):** `abif_*`, `rf_caf_*`, каналы и
  ATB/RXVGA. Константы Talyn здесь с Sparrow не совпадают ни разу, где сравнимы: поканальные слова
  0x88ae10…18, 0x88ae5c…68 (0x1f80 против 0x1f87 и т. п.), петля RFC (`rf_caf_rfc_loopback_test`
  0x889300/04, 0x88911c). Блок есть, но настройка своя.
- **PHY 0x883000…0x8841ff (~60):** в том числе `phy_tof_config` (0x883174, 0x883ea0) — регистры
  FTM/ToF Talyn у Sparrow есть (для pro FTM); `phy_init_load_table` 0x883580.
- **MAC:** `mac_enable_multi_network` 0x88613c (у Sparrow 0) — кандидат «несколько сетей» в pro;
  `hwm_init` 0x886c58 (у Sparrow 0xa028, у Talyn 0x14050).

## C — нет у Sparrow, по снимкам (151)
- **PHY 0x8843xx…0x8858xx (77):** CB2/канальная настройка (`phy_set_channel`, 0x8851c0), упаковка IQ
  (`phy_pack_iq_config_bytes`), разбор памяти BRP (`phy_brp_mem_parse_config`, `phy_rx_brp_mem_parse`),
  EVM-метр (`phy_rx_evm_config`), DC VGA (`phy_vga_dc_measure`), таблицы `phy_init_load_table`, радар
  (`phy_radar_cc_delay_config_apply`). Это и есть EDMG/CB2-тракт, TRN/BRP Talyn и sensing.
- **USER 0x880000…0x880eff (29):** SPI-хост sensing, OTP-автомат, AHB-логгер, таймер TSF
  (`tsf_timer_*`, 0x880cxx).
- **MAC 0x8864xx/0x8869xx/0x886axx (11):** FLM (`mac_configure_flm`), флаги MAC.
- **ABIF 0x8896xx/0x8897xx/0x88afxx (15):** `abif_init`, поканальная настройка, 0x88aff4.
- **DMA 0x881dxx (5), PCIe-ext 0x88b1xx (11, `pcie_ext_err_chk_isr`), mac_rgf_ext 0x88c1xx (3).**

## Вне карты драйвера Sparrow — проверено на стенде (2026-10-23)
Драйвер Sparrow отображает rgf, AGC_tbl, rgf_ext и mac_rgf_ext до 0x88c200, остальное отвечает INVALID.
Чтение ядром fw (по одному адресу, после каждой пробы — чистая загрузка):
- **mac_rgf_ext у Sparrow длиннее карты драйвера: 0x88c000…0x88c438.** Ненулевые 0x88c368 = 0x10001100,
  0x88c36c = 0x5, 0x88c434 = 0x1a0, 0x88c438 = 0x82000022; остальное 0. 0x88c43c…0x88c4ff — 0xdeadbeaf.
  Talyn: 0x88c204 (`vrvr_bi2_wait_terminated`, uc) и 0x88c364 (`mac_parser_4`, пишет 0) — у Sparrow есть.
- **0x88c500 и выше (0x88c500/600/700/800 проверены): чтение вешает весь чип** — хост читает 0xffffffff,
  нужен power-cycle. Разборщик MAC Talyn 0x88c800…0x88c814 (`mac_parser_9`) — нет.
- **ext_user_rgf 0x88d000+, CPR 0x88e000+, 0x89a000+, dma_ext_rgf 0x8b0000+:** чтение не завершается
  (ядро висит, RGF с хоста читается). AHB-дескрипторы `user_ahb_*`, CPR, `main_switch_xtal_to_sys_clk`, eDMA/SCM — нет.
- Пачка из 16 таких чтений с выключенными прерываниями повесила узел целиком (хост), отсюда правило: вне
  известных областей читать только по одному адресу и только на стенде с power-cycle.

## Запись в неиспользуемые регистры Sparrow (P3, стенд 2026-10-23)
Регистры класса B, в которые Talyn пишет известную константу (без ABIF/RF, тактов/PLL, PCIe, DMA, ICR и
функций start/trigger/reset/tx). Одна команда poke с выключенными прерываниями: исход → константа Talyn →
0xffffffff → 0 → исход, после каждой записи чтение (`../sparrow_rgf/p3_write.tsv`). Все восстановлены,
связь после каждого цела.

| адрес | функция Talyn | Sparrow: исход | записываемые биты | константа Talyn |
|---|---|---|---|---|
| 0x883174 | `phy_tof_config` (читает в снимок ToF, пишет 1) | 0x30 | [7:0] | 0x1 |
| 0x883580 | `phy_init_load_table` | 0 | [7:0] | 0xf6 |
| 0x8835b8 | `phy_config_set` | 0x4014 | [18:0] | 0x0 |
| 0x883ea0 | `phy_tof_config` | 0 | [31:0] | 0x20000/0x20001 |
| 0x883f30 | 3 функции PHY | 0 | [31:0] | 0x1/0x5 |
| 0x8840f4 | тесты ut_hwd_phy | 0 | [0] | 0x0 |
| 0x886c58 | `hwm_init` | 0xa028 (41000) | [19:0] | 0x14050 (82000) |
| 0x88613c | `mac_enable_multi_network` | 0 | [31:0] | 0x1f |
| 0x88c364 | `mac_parser_4` | 0 | [31:30], [23:0] | 0x0 |

Регистры с записываемыми 32 битами (0x883ea0, 0x883f30, 0x88613c) могут оказаться простыми ячейками
без действия — запись этого не различает. Действие констант Talyn (multi-network, ToF, 0x886c58 = 82000)
не проверялось: нужен опыт с наблюдаемым эффектом.

## Выводы для Sparrow
1. EDMG/CB2, разбор BRP Talyn, EVM-метр, радар, CPR, eDMA, AHB-дескрипторы, разборщик MAC 0x88c800 —
   регистров у Sparrow нет (C, проверено на стенде).
2. Переносить можно логику, которая работает на A/B: ABIF/RF-CAF, таблицу 0x8873xx…0x8878xx, ToF PHY,
   multi-network MAC, расписание DTI (0x886f8c). Сначала проверять смысл (имена с обеих сторон, константы,
   стенд), а не только адрес.
3. Числа Talyn для общих регистров (константы в колонке 10) — кандидаты, не эталон: у Sparrow свой
   тракт и сброс.

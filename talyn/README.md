# Talyn (QCA6436, 802.11ay)

Talyn по начинке почти Sparrow — fw и uc на том же ARC600, общий драйвер wil6210, — поэтому его поддержка
ведётся в рамках SparRAW. Здесь — результаты разбора прошивки Talyn.
Основная версия разбора: **Talyn 7.5.3.115** (MikroTik wireless 6.48.6 … 7.24.4).

## Источники
- Образы из пакетов MikroTik. Talyn: 7.1.3.980, 7.1.3.1158, 7.5.3.115; Talyn TG (Terragraph): 10.1.0.692, 10.7.0.9, 10.11.0.92.
- Ядро fw и uc — ARC600 (ARCompact, `mul64`), как у Sparrow.

## Конвейер
1. `seg/<образ>/` — сегменты по AHB-адресу, `map.txt` (AHB → линковка): fw_code 0x900000→0x0, fw_data 0xa00000→0x800000, uc_code 0xa38000→0x0, uc_data 0xa78000→0x800000; у 7.5 доп. fw_code 0x9a0000→0xa0000.
2. `run_all.sh` — Ghidra 12.1.3 headless (ARCompact:LE:32:default), проекты `proj/<образ>_{fw,uc}`; окно регистров 0x880000…0x8bffff — блок RGF.
3. `run_pass2.sh` — дозапуск дизассемблера с инструкций, ссылающихся на строки журнала (`seed/`), имена по строкам журнала (`names/fwlog_names.py`, ссылки limm 0x01000000|offset в записи 0x65 fw / 0x64 uc), выгрузка `out3/`.
4. `regmap/regmap.py` — карта обращений к регистрам (база mov/add + смещение ld/st).

## Talyn 7.5.3.115 — итоги
| | функций | инструкций | с именами из журнала | регистров (обращений) |
|---|---|---|---|---|
| fw | 3879 | 142968 | 1598 | 898 (2922) |
| uc | 1078 | 65146 | 299 | 275 (936) |

Модули (из таблиц строк): fw — 166 исходных файлов, uc — 86. Наиболее «тяжёлые» по числу именованных функций:
fw — hw_drivers_abif (144), hw_drivers_phy (49), calib_tx_power (27), marlon_r_if (21), hw_drivers_rf_caf (21), mid, layer2_mgr, lmac_if, edma_mgr;
uc — hw_drivers_abif (30), wa_rx_unsupported (15), hw_drivers_phy (15), sensing_mgr (9), beamforming_flows_brp (7).
Новое против Sparrow: edmg_mgr (uc), channels_switch_sm / rm_ch_switch_sm, sensing_manager / sensing_mgr (радар), tof_mgr / hw_flows_phy_tof (FTM), rate_search, linear_power, thermal_throttling, multi_network_mgr, wbe_driver, hw_drivers_tlna2_rfc.

Выгрузки: `out3/tl_7.5.3.115_{fw,uc}/` (funcs.txt, listing.dis, decomp.c), `names/tl_7.5.3.115_*.txt`, `regmap/tl75_{fw,uc}.tsv`.

## Смена канала и CB1/CB2 (Talyn 7.5, fw)
- `hwm_ANALOG_CHANNEL_SWITCH` @0x31e10 (hw_modes_analog_manager.c): стоп тракта → синтезатор → `file_hw_drivers_phy_11(ch)` → `file_hw_drivers_phy_20(1, cb2)` (cb2 = 5 < ch_enum < 10) → ожидание 0x339 итераций → пуск. Такты (PLL/CAR) не трогаются.
- `file_hw_drivers_phy_11` @0x24f88: 0x8851c0 = индекс канала; 0x883b3c = {0x6b,0x68,0x65,0x62,0x62,0x62}000101 для CB1 (enum 0..5), {0x69,0x66,0x63,0x60}000101 для CB2 (enum 6..9 = каналы 9–12).
- `file_hw_drivers_phy_20` @0x2599c: 13-отводный фильтр 0x883d74/78/7c — CB1 `d8d8ecfc 3b573b00 ecd8d800` (+ 0x883d80[7:0]=0xfc), CB2 `11fdf9fd 4c574c30 f9fd1130` (0xfd); 0x883d80 бит 8 = cb2.
- Сопоставление со Sparrow (снимок nc0): блок 13-отводного фильтра у Sparrow на 0x883d64…70 (сдвиг −0x10), значения CB1 те же; у Sparrow 0x883d70 = 0x1fc (бит 8 = 1). 0x883b3c у Sparrow — те же значения для каналов 1 (0x6b) и 3 (0x65). 0x8851c0 у Sparrow отсутствует (0xdeadbeaf).

## Talyn 7.5 vs Sparrow: какие регистры есть в железе Sparrow
Снимок RGF Sparrow (nc0, 0x880000…0x889fff, 6139 «живых» слов из 10240, остальное 0xdeadbeaf) против карты обращений Talyn 7.5:
- fw: из 719 регистров Talyn в этом окне у Sparrow есть 592 (82 %), нет 127;
- uc: из 233 есть 208, нет 25.
Отсутствуют у Sparrow блоками: 0x8844xx–0x8858xx (цифровой TX-тракт/DPD, EVM-метр, CB2: 0x8844c4…fc, 0x8851c0, 0x8854xx–0x8858xx), 0x880c00 (13), 0x880000/0x880e00, 0x881d00, 0x8864xx/0x886axx (MAC), 0x8896xx/0x8897xx (ABIF).

## Инструменты и данные
Конвейер разбора (Ghidra headless, имена по журналу, карты регистров, диспетчер WMI) — SparRAW-tools `talyn/`.
Образы, сегменты, выгрузки, карты обращений и снимки стенда в открытые репозитории не входят.
Модуль ARCompact для Ghidra — SparRAW-tools `re/ghidra` (install_arc.sh).

## Команды WMI (Talyn 7.5 fw)
Диспетчер — switch по `hdr.id` (r2) в коде 0x3ce9e…0x3dd1e (Ghidra не выделила его в функцию): двоичный поиск cmp/beq + таблицы переходов `ldw.x.as` (16-битные смещения в fw_data 0x805dfc…). Восстановление — `wmi/wmi_switch.py`, результат `wmi/tl_7.5.3.115_wmi.tsv` (id, адрес case, строка журнала «[HOST CMD] …», первый вызов не из журнала — кандидат в обработчик; у части case это вспомогательная функция).
- 116 команд с отдельной веткой (+ ветка по умолчанию).
- Сравнение с Sparrow 6.2 (наш rewrite): общих 72, только у Talyn 44 (радар/sensing 0x100/0x101, EDMA-кольца 0x9c0…0x9c5, TOF 0x991…0x999, RS_CFG_EX/RS_ENABLE, thermal throttling 0x940/0x941, link stats 0xa0c, AOA 0x923, FORCE_DISABLE_EDMG 0x859, TPC_SET_MAX_AVG_TX_POWER 0xa20 и др.), только у Sparrow 18 — `wmi/compare_sparrow62.txt`.
- Ограничение: несколько id сравниваются в коде (0xf003, 0xf007, 0xf049, 0xf04d), но их ветки разбором не атрибутированы — проверить вручную.

## Счётчики MAC: остаток TXOP и расписание DTI (Talyn 7.5 uc, для сверки со Sparrow)
- Чтение остатка: команда MAC `0x1d` с селектором 3 в [15:12] (`0x1d003000`), результат — r40 (16 бит, мкс). Места: `txop_initiator_open_txop_flow` @0x35668 (если r40 < 0x31 — TXOP не открывать, как у Sparrow «REMAINING_TIME_RD ≤ 0x30»), `cf_end_flow` (CF-End шлётся, только если r40 > TXTIME(CF-End)+9), ответ на направленный кадр, edmg_mgr, rx_flow. Вывод: селектор 3 = остаток текущего TXOP/NAV-периода в мкс (по смыслу использования; на стенде не проверено).
- Другие селекторы `0x1d`: 0 (часто), 4 (TSF/время; internal_tx, sensing, multi_network), 7 (multi_network), 0xb/0xc/0xc000 (edmg_mgr, r40 & 0x3ffff).
- Расписание DTI (SP из ESE) — аппаратный планировщик MAC: команда `0x5a000000`/`0x5a001000` (выкл/вкл), статус слота `0x886f8c` ([3:0] id слота, бит 7 и бит 6 — признаки), `dti_worker::scheduled_dti_allocation_event` @0x2c120 (лог «starting slot id=%d at tsf_low»), TSF — `0x886eb8`. Тот же регистр у Sparrow: `HW_MAC_SCHED_SLOT_STATUS` 0x886f8c, команда 0x5a — в rewrite (uc/bi.c, bti.c, tail.c событие 0x29). Следствие: конец SP знает планировщик (слот), а не селектор 3.
- Регистры на Sparrow (снимок nc0, AP): 0x886eb8 (TSF lo) живой, 0x886ec0 (до TBTT, мкс) = 0x1aca, 0x886f8c = 0 (расписание не включено).

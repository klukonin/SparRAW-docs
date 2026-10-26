# Мощность TX и TPC у Talyn; что делают драйвер MikroTik и RouterOS

Разбор образа Talyn 7.5.3.115 (fw, Ghidra, имена по лог-строкам; метка **[код]**) и пакета RouterOS `wireless`
6.48.6 arm64 (модуль `lib/modules/3.3.5/drivers/net/wil6210.ko` — 32-битный ARM, `nova/bin/wireless`). Мощность у
Sparrow — [../research/TX-POWER.md](../research/TX-POWER.md).

## Talyn: замкнутая TPC

Модули fw: `calib_tx_power` (27 лог-строк), `hw_flows_tx_power_sens`, `hw_drivers_tlna2_rfc`, `linear_power`,
`time_averaging`. Функции: `calib_tx_power_*` (калибровка мощности, выбор «lineup»), `tpc_psk_*` (база lineup по
температуре и TSSI), `time_averaging_update_pout_post_tpc`, `tx_power_vout_to_pout` **[код]**.

**Применение — индекс строки по MCS.** `calib_tx_power_set_gain_idx` запоминает новый индекс усиления и сразу зовёт
`abif_set_mcs_range_all` → `abif_set_mcs_range` → `abif_set_mcs(mcs, индекс, таблица)`: 5-битное поле [4:0] слова
по MCS — таблица 0: `0x8891dc + 4·mcs`, таблица 1: `0x88919c + 4·mcs` (MCS 0…15). Диапазоны MCS (`abif_set_mcs_range`,
режим 0/1/2): 0…5 (или 1…5), 6…9 (7…11), 10…12 (12…15) — в зависимости от SC/EDMG; `calib_tx_power_set_gain_idx_sel`
задаёт индекс для третьего диапазона отдельно. Кадры берут строку усиления по своему MCS — мощность меняется на ходу,
без перенастройки РЧ **[код]**. У Sparrow то же место — `RFC_TX_POWER_TBL` 0x8891b4…0x8891e4 (13 слов, MCS 0…12).

**Датчик мощности (TSSI).** РЧ-блок TLNA2 (вероятно, «Talyn LNA 2») имеет датчик TX (`tlna2_tx_sens_*`):
запуск — РЧ-регистры 0xb4, 0x1d8, 0x1a0, 0x1a4, 0x1bc, 0x1b4, 0x1ac; диапазоны по антеннам — 0x3a4 + 4·(ант/2);
триггер — 0x214 = 1; готовность — 0x210 ≠ 0 и 0x20c бит 0; результат — 0x218 + 4·антенна + 0x80·k; настройка — 0x398;
сброс — 0x1dc, 0x52c. Напряжение `vout` по антеннам (LUT 0x857710 / 0x857790 общей памяти) переводится в мощность:
`pout = (vout·10 >> n) − 0x280` (десятые доли) и `tx_power_get_dc_lut_val`. Замеры ведёт ucode по командам LMAC
0x46, 0x47, 0x60 (`tx_power_sens_send_msg_*`, конфигурация 0x6c Б, шаблоны 0x808338 / 0x8082f8 / 0x808420); результаты
выборок — 0x8578dc (записи по 0x18 Б), лучшая — `tx_power_gain_sample_result` → `tx_power_fill_lineup_entry` **[код]**.
У РЧ Sparrow-R таких регистров нет: в 0x224…0x2a0 у него вторая таблица усилений.

**Предел средней мощности с хоста.** `WMI_TPC_SET_MAX_AVG_TX_POWER` (0xa20) → `wmi_tpc_set_max_avg_tx_power`: значение
s8 в (−101, 101) → `time_averaging_set_delta_limit`, ответ 0x1a20 со статусом (1 — вне диапазона) **[код]**. В Sparrow
6.2 этой команды нет.

## Драйвер wil6210 RouterOS

TPC нет. Что относится к мощности **[код]**:

* **Talyn: выключение калибровки мощности fw.** В обработчике поднятия линка (рядом с `netif_carrier_on`) для
  устройства Talyn (тип 2), если в объекте драйвера по +0x26020 есть адрес, печатается «disable TX_POWER calibration»
  и в память устройства по этому адресу + 0x14 пишется 1 (`writel`). Откуда берётся адрес, не установлено.
* «rf-power» — имя GPIO 36 (питание РЧ-модуля), не мощность TX.
* Board-файл: поиск секций по ключу (`0xa000900d`, `0xc00c900e`, `0xc00c900d`); запись с адресом 0x8ffff2 в секции
  `0xa000900d` драйвер переключает 0 ↔ 255 («change slow control»).

`nova/bin/wireless`: `tx-power` — только `patchBoardFileRFRegs` (строка 7 board до загрузки прошивки, см.
[../research/BRD-RF-REGS.md](../research/BRD-RF-REGS.md)), `manual-tx-power` — проверки пределов карты. Смена мощности
в RouterOS — перезапуск интерфейса.

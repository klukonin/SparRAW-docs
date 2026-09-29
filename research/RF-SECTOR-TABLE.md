# Таблица РЧ-секторов в прошивке 4.1.0.1000

Запись таблицы секторов РЧ с хоста (статический разбор, структура сверена с заголовком
драйвера). Секции секторов board-файла (`0xC00C900E` RX, `0xC00C900D` TX), omni-запись и
переписывание секторов драйвером RouterOS —
[HW-DRIVERS.md](https://github.com/klukonin/SparRAW-firmware/blob/main/docs/HW-DRIVERS.md)
(раздел о board-файле); команды WMI RF-секторов обеих версий —
[WMI.md](https://github.com/klukonin/SparRAW-firmware/blob/main/docs/WMI.md).

## Писатель таблицы **[код]**

`rf_sector_params_write` @0x8d1174 (330 Б) — единственный писатель; зовётся из
`wmi_handler_set_rf_sector_params::set_rf_sector_params` (`wmi_handlers.cpp`), то есть
достижим командой `WMI_SET_RF_SECTOR_PARAMS_CMDID` (0x9A1).

```c
slot = sector_idx * 0xc0;
if (sector_type == 0) {                    /* WMI_RF_SECTOR_TYPE_RX */
    rfc_field_write(m, (0x10000 + slot + 0x00) >> 3, f0);
    rfc_field_write(m, (0x10000 + slot + 0x20) >> 3, f1);
    rfc_field_write(m, (0x10000 + slot + 0x60) >> 3, f2);
    rfc_field_write(m, (0x10000 + slot + 0x40) >> 3, f3);
    rfc_field_write(m, (0x10000 + slot + 0xa0) >> 3, f4);
    rfc_field_write(m, (0x10000 + slot + 0x80) >> 3, f5);
    if (commit) rf_sector_commit_rx(m, sector_idx & 0xff);
} else {                                   /* WMI_RF_SECTOR_TYPE_TX */
    /* то же с банком 0x8000 и rf_sector_commit_tx */
}
```

Вспомогательные (все `hw_drivers_rfc.c`): `rfc_field_write` @0x8d113c,
`rf_sector_commit_rx` @0x8d073c, `rf_sector_commit_tx` @0x8d0ef4.

## Раскладка

| что | значение |
|---|---|
| банк приёмных секторов | 0x10000 |
| банк передающих секторов | 0x8000 |
| слот сектора | 0xc0 (192) Б = 24 регистра |
| полей в секторе | 6 по 32 бита, шаг 0x20 |
| адресация | в 8-байтных единицах (`>> 3`) |

Шесть полей по числу и ширине совпадают со `struct wmi_rf_sector_info` драйвера:
`psh_hi`, `psh_lo` (фазы по 2 бита на РЧ-цепь), `etype0..2` (биты индекса усиления
краевого усилителя), `dtype_swch_off` (усилители распределения + биты переключателя X16).
Порядок записи — 0x00, 0x20, 0x60, 0x40, 0xa0, 0x80.

## Применение

Содержимое каждого сектора задаётся с хоста без патча прошивки. В WMI рядом есть
`WMI_SECTOR_SWEEP_TYPE_BCON` (отдельный тип развёртки для маяков) и
`WMI_SET_SELECTED_RF_SECTOR_INDEX_CMDID` (фиксация сектора при отключённых TXSS/BRP) —
путь к управляемому направленному маяку.

## Не установлено

* Соответствие «поле структуры ↔ смещение в слоте» (доказаны только число и ширина полей).

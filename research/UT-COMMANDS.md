# Диагностические команды UT прошивки 4.1: код → функция

Соответствие «код подкоманды `WMI_UNIT_TEST` → целевая функция» для прошивки 4.1.0.1000
по таблицам диспетчеров UT (`UT_HW_DRIVERS_cmd_handler`: 0x1xx ABIF, 0x2xx CAR, 0x4xx PHY,
0x5xx RFC; `ut_hw_flows`: 0x3xx; ответ — `ut_hw__post_response` 0x8e5b94). Блоки UT 4.1 —
[4.1/docs/MISC.md](https://github.com/klukonin/SparRAW-firmware/blob/main/4.1/docs/MISC.md);
подкоманды модуля драйверов 6.2 —
[6.2/docs/UT-DRIVERS.md](https://github.com/klukonin/SparRAW-firmware/blob/main/6.2/docs/UT-DRIVERS.md).

Ниже — цели с тремя и более вызывающими: это обычные функции драйверов, которые вызывают
и калибровки, поэтому имя по коду команды (`ut_hw_drivers_cmd_0x410`) для них не
присваивается, а соответствие хранится здесь. Цели с одним-двумя вызывающими, для которых
команда — единственная точка входа, названы по команде в дереве
(`ut_hw_drivers_cmd_0x131` и т. п.). Имена в таблице — текущие имена блоков дерева 4.1
(`src/asm/fw/blocks.json`), в скобках — каталог исходного файла.

| функция | имя блока | код команды | вызывающих всего |
|---|---|---|---|
| 0x8c7794 | `u_schd__cancel` (u_schd) | 0x022 | 52 |
| 0x8e4694 | `fw_read_timer` (_unknown) | 0x011 | 48 |
| 0x8cc674 | `hwd_abif__rgf_88a200_8cc674` (hw_drivers_abif) | 0x10f | 19 |
| 0x8cf4c8 | `hwd_phy__rgf_883000_8cf4c8` (hw_drivers_phy) | 0x410 | 16 |
| 0x8cb8d8 | `hwd_abif__rgf_88af80_8cb8d8` (hw_drivers_abif) | 0x10a | 14 |
| 0x8cf4fc | `hwd_phy__rgf_884000` (hw_drivers_phy) | 0x406 | 14 |
| 0x8cfcdc | `hwd_phy__rgf_883000_8cfcdc` (hw_drivers_phy) | 0x403 | 10 |
| 0x8cfd48 | `hwd_phy__rgf_883000_8cfd48` (hw_drivers_phy) | 0x402 | 10 |
| 0x8cf588 | `hwd_phy__set_mode` (hw_drivers_phy) | 0x412 | 9 |
| 0x8cc3c8 | `hwd_abif__set_agc_field` (hw_drivers_abif) | 0x119 | 9 |
| 0x8cf958 | `set_reg_883150` (hw_drivers_phy) | 0x411 | 9 |
| 0x8cf74c | `hwd_phy__rgf_883000_8cf74c` (hw_drivers_phy) | 0x423 | 8 |
| 0x8cfc80 | `hwd_phy__set_883900` (hw_drivers_phy) | 0x421 | 6 |
| 0x8cfc9c | `set_reg_88395c` (hw_drivers_phy) | 0x420 | 6 |
| 0x8cfd88 | `hwd_phy__rgf_883000_8cfd88` (hw_drivers_phy) | 0x401 | 6 |
| 0x8cc834 | `hwd_abif__get_agc_field` (hw_drivers_abif) | 0x132 | 6 |
| 0x8cf8e8 | `hwd_phy__snapshot_status` (hw_drivers_phy) | 0x422 | 6 |
| 0x8d04b0 | `tail_hwd_rfc_write_core` (hw_drivers_rfc) | 0x509 | 6 |
| 0x8cd6ac | `vring__reg_write_locked_28` (_unknown) | 0x032 | 6 |
| 0x8cbd8c | `hwd_abif__set_88ae60_flags` (hw_drivers_abif) | 0x12b | 5 |
| 0x8cc6cc | `get_reg88a204_s1_w32` (hw_drivers_abif) | 0x12a | 5 |
| 0x8cbb20 | `hwd_abif__set_sel_nibbles` (hw_drivers_abif) | 0x10d | 5 |
| 0x8d006c | `hwd_phy__rgf_884100` (hw_drivers_phy) | 0x40e | 5 |
| 0x8cfdc0 | `hwd_phy__mode_flags_by_index` (hw_drivers_phy) | 0x432 | 5 |
| 0x8cd6dc | `vring__reg_write_locked_3c` (_unknown) | 0x019 | 5 |
| 0x8cb9dc | `hwd_abif__program_rf_clk` (hw_drivers_abif) | 0x106 | 4 |
| 0x8cca94 | `hwd_abif__rgf_88a200_8cca94` (hw_drivers_abif) | 0x110 | 4 |
| 0x8cf184 | `hwd_phy__rgf_885000_8cf184` (hw_drivers_phy) | 0x41c | 4 |
| 0x8d02ac | `hwd_phy__wait_ready_884054` (hw_drivers_phy) | 0x408 | 4 |
| 0x8d0388 | `set_reg_884064` (_unknown) | 0x40a | 4 |
| 0x8ccfb0 | `hwd_abif__rgf_88a000_8ccfb0` (hw_drivers_abif) | 0x10e | 4 |
| 0x8cbf98 | `hwd_abif__adjust_5bit_pair` (hw_drivers_abif) | 0x112 | 4 |
| 0x8d01e8 | `hwd_phy__write_884000` (hw_drivers_phy) | 0x407 | 4 |
| 0x8cf52c | `hwd_phy__rgf_883000_8cf52c` (hw_drivers_phy) | 0x413 | 4 |
| 0x8cb7c0 | `set_reg_88af00` (_unknown) | 0x102 | 3 |
| 0x8cb9a8 | `hwd_abif__rgf_88ae00` (hw_drivers_abif) | 0x109 | 3 |
| 0x8cbcb8 | `hwd_abif__rgf_889400` (hw_drivers_abif) | 0x107 | 3 |
| 0x8cbf84 | `hwd_abif__rgf_88a200_8cbf84` (hw_drivers_abif) | 0x113 | 3 |
| 0x8cca78 | `hwd_abif__set_88af10` (hw_drivers_abif) | 0x115 | 3 |
| 0x8ccb7c | `hwd_abif__rgf_889300_8ccb7c` (hw_drivers_abif) | 0x125 | 3 |
| 0x8ccc78 | `hwd_abif__rgf_889300_8ccc78` (hw_drivers_abif) | 0x124 | 3 |
| 0x8cd590 | `hwd__wait_pll_lock` (_unknown) | 0x202 | 3 |
| 0x8cd018 | `hwd_abif__rgf_88a000_8cd018` (hw_drivers_abif) | 0x10c | 3 |
| 0x8cbf5c | `hwd_abif__get_88af10` (hw_drivers_abif) | 0x114 | 3 |
| 0x8cc0c8 | `hwd_abif__set_agc208_w4` (hw_drivers_abif) | 0x11b | 3 |
| 0x8cce38 | `hwd_abif__set_agc004_w2` (hw_drivers_abif) | 0x11e | 3 |
| 0x8ccdac | `hwd_abif__set_agc004_w7` (hw_drivers_abif) | 0x11f | 3 |
| 0x8cccf4 | `hwd_abif__set_agc004_w4` (hw_drivers_abif) | 0x120 | 3 |
| 0x8d0318 | `rgf_reg_884000` (_unknown) | 0x409 | 3 |
| 0x8cf83c | `phy__program_channel_regs` (hw_drivers_phy) | 0x414 | 3 |
| 0x8cf0e4 | `hwd_phy__get_885110` (hw_drivers_phy) | 0x41e | 3 |
| 0x8cfc0c | `hwd_phy__rgf_885000_8cfc0c` (hw_drivers_phy) | 0x426 | 3 |
| 0x8cf7b4 | `hwd_phy__read_sar_sample` (hw_drivers_phy) | 0x428 | 3 |
| 0x8cf198 | `hwd_phy__rgf_885000_8cf198` (hw_drivers_phy) | 0x435 | 3 |
| 0x8d04bc | `hwd_rfc__rgf_889100` (hw_drivers_rfc) | 0x50a | 3 |
| 0x8d08bc | `hwd_rfc__read_sector_chain` (hw_drivers_rfc) | 0x50d | 3 |
| 0x8d07c8 | `hwd_rfc__read_sector_reg` (hw_drivers_rfc) | 0x511 | 3 |
| 0x8d36b4 | `calib_sar__enable_path` (_unknown) | 0x406 | 3 |
| 0x8d4d5c | `hwm__apply_mode_flags` (_unknown) | 0x306 | 3 |
| 0x8d50e4 | `hwd_phy__power_up_seq` (_unknown) | 0x307 | 3 |
| 0x8d508c | `hwm__apply_channel_config` (_unknown) | 0x30a | 3 |

Коды 0x011, 0x019, 0x022, 0x032 ведут на функции общего назначения (`fw_read_timer`,
`vring__reg_write_locked_*`, `u_schd__cancel`) с десятками вызывающих; отнесение этих
кодов к диспетчеру UT **[гипотеза]**.

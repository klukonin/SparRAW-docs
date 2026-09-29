# Что Terragraph добавил в прошивку радио: диф 7.5 → 10.11

Сравниваются `wil6436.fw` 7.5.0.77 (дженерик WiGig) и `wil6210-IF2IF-10.11.4.87.fw`
(cnWave/Terragraph). Метаданные у образов разные: у 7.5 — символьные карты
(globals XML), у 10.11 — таблицы лог-строк. Поэтому диф строится по инвентарю
подсистем: имена функций из лог-строк 10.11 против имён из символов 7.5.
Тракт биконинга по версиям — [CROSS-VERSION-BEACON](CROSS-VERSION-BEACON.md).

## IF2IF — флейвор, а не отдельный стек

IF2IF против стандартной 10.11: общих шаблонов 3884, собственных у IF2IF 2,
отсутствующих 39; все различия — РЧ-калибровка (`IF2IF override
hwm_analog_channel_switch`, `calib_silent_rssi`). Меш-механика есть во всей
линейке 10.11; IF2IF, WMI_ONLY, DEV — варианты board и сборки.

## Дельта 7.5 → 10.11: TDMA-MAC

Из 1684 подсистем 10.11 461 (27 %) специфичны для Terragraph и отсутствуют в
7.5. Токены `tgf`, `lsm_`, `tdm_`, `mtpo`, `bwgd`, `superframe`, `gps_`, `1pps`,
`dn_pcie`/`cn_pcie`, `link_impair`, `ignit` дают 0 совпадений в символах и
`wmi.xml` 7.5, тогда как база DMG (beamforming, sector_sweep, bcon, scheduler)
в 7.5 есть: Terragraph — надстройка над штатным DMG.

| группа | функций | назначение |
|---|---|---|
| `tgf*` | 307 | Terragraph Framework: планировщик кадров, распределитель слотов, планирование BF-сканов, интерфейс драйвера (`tgfaddlinks`, `tgfsendslotallocstomicrocode`, `tgfbeginframescheduling`) |
| `lsm*` | 57 | автомат состояния линка: ignition и teardown линка DN↔DN (`lsm_w4_send_assoc_req`, `lsm_link_up`, `lsm_eve_link_failed`) |
| `fb_*` | 29 | API управления и расписания (`fb_mgmt_pkt_manager`, `fb_scheduling_api`, `fb_attach`) |
| `tdm*` | 25 | TDMA-тракт данных: слоты ucode, `tdm_3p_adapter`, `handle_tx/rx_slot_start`, `ucode_slot_control` |
| `mtpo*` | 23 | калибровка фазового сдвига между модулями антенн (multi-tile phase offset) |
| `gps/pps` | 23 | синхронизация времени сети: GPS 1PPS → TSF (`tgfgpsprocessgpstime`, `tgftsfcorrectgpsdrift`) |
| `efw_param_*` | 22 | рантайм-конфигурация узла и линка с хоста (`efw_param_mtpo_enabled`, `efw_param_link_impair_config`) |

Новые команды WMI под роли Terragraph: `wmi_tdm_set_dn_pcie_params_cmd`,
`wmi_tdm_set_cn_pcie_params_cmd` (DN — distribution node, CN — client node).

## Вывод

«Меш на уровне прошивки» для этого чипа — GPS/PPS-синхронный TDMA-MAC:
суперкадровый планировщик (`tgf`), автомат ignition на линк (`lsm`), раскладка
слотов в ucode (`tdm`) и южный API к хосту (`tgf`/`TG_SB_*`). Пересылки пакетов в
прошивке нет: `forward`/`disable_forward` относятся к управлению потоком,
L3-пересылка остаётся на хосте (Open/R). Прошивка превращает радио из одиночного
DMG-линка в набор синхронных линков с TDMA-планированием; топологию и маршрутизацию
ведёт хост.

Для дженерик-wil6436 меш уровня Terragraph требует TDMA-слоя прошивки
(≈ 460 функций). Меш без TDMA — асинхронные линки PBSS/STA (штатные в 7.5) и
хостовая маршрутизация — обходится без правки прошивки, но линки не
координируются во времени (интерференция, меньшая плотность), тогда как
выигрыш Terragraph именно в GPS-TDMA.

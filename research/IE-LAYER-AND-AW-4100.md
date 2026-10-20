# Слой IE, события состояния системы и приём окна AW в fw 4.1.0.1000

Дополнение к документации прошивки по 4.1: сборщики информационных элементов, перечень
системных событий L2MGR, место приёма окна AW и формы адресов кода в данных.

Механизм окна AW (длительность, `ps_assoc_mgr::aw_ie_reception` и правило «сужать сразу,
расширять через 4 BI», LMAC 0x25, `aw__set_new_tbtt`) —
[docs/BEACONING.md §6.5](https://github.com/klukonin/SparRAW-firmware/blob/main/docs/BEACONING.md);
путь длительности AW, гейт окна AW и `ie__push_awake_window` —
[docs/STANDARD-MAPPING.md](https://github.com/klukonin/SparRAW-firmware/blob/main/docs/STANDARD-MAPPING.md);
сборка DMG-маяка `tx_mgmt_build_step` и Clustering Control — там же;
рассылка системных событий в 6.2 (`sys_state__broadcast`) —
[docs/L2-MANAGER.md](https://github.com/klukonin/SparRAW-firmware/blob/main/docs/L2-MANAGER.md).

Метки: **[код]** — по листингу 4.1, **[гипотеза]**.

## 1. Сборщики IE **[код]**

Кластер 0x8eca28…0x8ed28e (между `frame_builder.cpp` и `connection.cpp`/`layer2_mgr.cpp`) и
связанные функции:

| адрес | имя | что делает |
|---|---|---|
| 0x8da098 | `ies_block__merge` | слияние блоков IE, «Exclude the P2P IE from frame type…» |
| 0x8c30fc | `mid__update_aw_ie` | обновление элемента AW, гейт по константе 0x9d, иначе ассерт `mid.cpp:898`; перевыпуск маяка через `tx_bcon` |
| 0x8dce44 | `app_ie__print_ies` | перебор IE: «IE index / IE Type / IE Length» |
| 0x8de250 | `remove_app_ie` | удаление IE приложения |
| 0x8e1fe8 | `mid__replace_rsn_ie` | замена RSN IE |
| 0x8e4c34 | `tx_mgmt_srvs__build_dmg_beacon` | «TX MGMT SRVS: Build DMG Beacon len %d», «Before IE push»; общий путь обычного (`tx_bcon`) и discovery-маяка (`add_discovery_bcon`) |
| 0x8ecaec, 0x8ecd10 | `mgmt_tx__append_probe_resp_ies`, `mgmt_tx__probe_resp_ies` | SSID IE в Probe Response, в т. ч. «use host SSID» |
| 0x8ef7b8 | `ie__push_edca` | EDCA IE, 0x14 Б |
| 0x8ef804 | `ie__push_qos_capability` | QoS Capability IE, 3 Б |
| 0x8ef82c | `ie__push_awake_window` | `{id 0x9d, len 2, get_aw()}` — 4 байта |
| 0x8f1de4 | `ie__classify_vendor_oui` | разбор вендорских IE (WSC) по OUI |

Соседние строители того же семейства (ID / длина): 0xbe / 4, 0x97 / 0xa, 0x94 / 0x11 (с MAC в
теле — элемент, который `tx_bcon` кладёт по `bi_mode`).

Порядок сборки тела DMG-маяка в `tx_mgmt_build_step` 0x8ec89c: `mgmt_tx__set_frame_control`
0x8e8974 → 8 байт нулей (`memset0_words` 0x8c1a78) → u16 `[gp+0x54]` по +0x15 →
`mgmt_tx__push_bi_control` 0x8e8960 (+0x17) → `BUILDER_push_dmg_parameters` 0x8e89b8 (+0x1d) →
при `[+0x17] & 1` — `mgmt_tx__push_clustering_control` 0x8e894c → элемент AW. Элемент AW
добавляется, если `r16 ≠ 0` (результат заголовка) или слово `0x80345c + 0xc` ненулевое —
практически всегда; discovery-маяк собирается тем же путём и тоже несёт AW.

Соседний кластер 0x8d1ebc…0x8d2826 (`hwf_calib__prepare_hw` … `if_gain__apply_delta`, между
`hw_drivers_sdp.c` и `hw_flows_if_gain_calibration.c`) — калибровка RF/baseband
([docs/HW-DRIVERS.md](https://github.com/klukonin/SparRAW-firmware/blob/main/docs/HW-DRIVERS.md)).

## 2. Системные события L2MGR **[код]**

`L2MGR__system_state_evt_handler` 0x8e3a08 — широковещатель события (зовётся с константой
события; печатает `%s, is idle = %s, is AP = %s`). Имена берутся из массива указателей
**0x800fbc** (host 0x900fbc), 25 записей:

| № | событие | № | событие |
|---|---|---|---|
| 0 | BOOT_DONE | 13 | LISTEN_START |
| 1 | CHANNEL_SWITCH | 14 | SEARCH_START |
| 2 | SCAN_START | 15 | DISCOVERY_START |
| 3 | SCAN_DONE | 16 | DISCOVERY_DONE |
| 4 | ASSOC_START | 17 | PAUSE |
| 5 | ASSOC_DONE | 18 | RESUME |
| 6 | DATA_PORT_OPEN | 19 | HIGH_FALSE_ALARM_RATE |
| 7 | UNASSOC_START | 20 | SHALLOW_SLEEP_ENTER |
| 8 | UNASSOC_DONE | 21 | SHALLOW_SLEEP_EXIT |
| 9 | PCP_START | 22 | MAC_MONITOR |
| 10 | PCP_START_DONE | 23 | DISABLE_PS_MODE |
| 11 | PCP_STOP | 24 | ENABLE_PS_MODE |
| 12 | PCP_STOP_DONE | | |

Источники (места вызова с константой события):

| событие | источник |
|---|---|
| BOOT_DONE | `l2mgr__init` 0x8ed398 |
| SCAN_START / SCAN_DONE | `l2_mgr::wmi_scan_cmd` / `l2_mgr::scan_complete_handle` |
| ASSOC_START | `l2_mgr::connect` |
| ASSOC_DONE | `l2mgr__publish_assoc_done` (вызов @0x8e9c36) |
| DATA_PORT_OPEN | `l2mgr__send_data_port_open_evt` 0x8f04b0 |
| UNASSOC_DONE | `l2mgr__disconnect_done` 0x8eb270 |
| PCP_START(_DONE), PCP_STOP(_DONE) | `l2_mgr::pcp_start`, `l2_mgr::pcp_stop` |
| LISTEN_START / SEARCH_START | `l2_mgr::wmi_cmd_handler_start_listen` / `_start_search` |
| DISCOVERY_START / DISCOVERY_DONE | `l2mgr__start_discovery` 0x8f3354 / `l2mgr__send_discovery_stopped` 0x8f05d0 |
| MAC_MONITOR | хвост широковещателя 0x8ee760 |
| SHALLOW_SLEEP_EXIT | `ps__post_shallow_sleep_state` 0x8f0d08 |

Подписчик `phy_monitor__on_system_state` 0x8e3880 (`phy_monitor.cpp`) ветвится по событию
(< 0x17, иначе ассерт) через байтовую таблицу 0x802cf8: 7 тел на 23 события — 0x8e389a
{2, 13, 14, 15}, 0x8e38b2 {3, 5, 6, 8, 10, 12, 16}, 0x8e38ba {4, 7, 9, 11}, 0x8e38ce {18, 21},
0x8e38dc {17, 20}, 0x8e38e4 {19}, 0x8e38f8 {0, 1, 22}.

LISTEN/SEARCH/DISCOVERY поднимаются обработчиками WMI 0x914 (`WMI_START_LISTEN_CMDID`),
0x915 (`WMI_START_SEARCH_CMDID`), 0x916 (`WMI_DISCOVERY_START_CMDID`), 0x917
(`WMI_DISCOVERY_STOP_CMDID`); mainline-драйвер шлёт 0x914, 0x915 и 0x917 из `p2p.c`
(remain-on-channel и поиск пиров P2P). Ограничения из строк прошивки: «If connected or in GO
mode do nothing» (listen), «If connected or in GO mode it's prohibit to run the search flow»
(search) — обнаружение только у несвязанного узла не в роли GO.

## 3. Где вызывается приём AW **[код]**

`ps_assoc_mgr::aw_ie_reception` 0x8c34f4 прямых вызывающих не имеет; единственный переход —
`bleq.d 0x8c34f4` @0x8dd290 внутри `ps_assoc_mgr__set_awake_tsf` 0x8dd268 («ps_assoc_mgr:: set
m_awake_tsf_64»). Эта функция — метод из таблицы методов `ps_assoc_mgr` в `fw_data`
(host 0x900740…0x900774: 0x900758 = 0, 0x90075c, 0x900760, 0x900764, 0x900768, …; слова в форме процессора, §4: 0x80075c = 0x15b68 → `ps_assoc_mgr__vt04`
0x8d5b68, 0x800760 = 0x1d268 → 0x8dd268, 0x800764 = 0x1d210 → 0x8dd210, 0x800768 = 0x1d1d8 →
0x8dd1d8, 0x800774 = 0x1d4e4 → 0x8dd4e4; 0x800758 = 0). Она ветвится по второму аргументу:

| значение | ветка |
|---|---|
| 2 | `vring_schd__prohibit_cids(~awake)` → `vring_schd__find_runnable` 0x8c2fe4 → `TX_API__awake_peer_ntf` 0x8c3658 (путь PS_AWAKE_PEER_EVT, [docs/DATAPATH.md](https://github.com/klukonin/SparRAW-firmware/blob/main/docs/DATAPATH.md)) |
| 3, 4 | ветки 0x8dd2b4, 0x8dd2d8 |
| 5 | `ps_assoc_mgr::aw_ie_reception` |

Функция лежит между `ps_nonassoc_sm__cancel_timer` 0x8dd1a0 и `PS_CONNECTION__psc_complete`
0x8dd554. Адрес 0x800740 кладётся в поле +0x2c объекта 0x84edf4 (host 0x916df4, существует
только во время работы) кодом @0x8c185c (`mov r1,0x84edf4;
mov r0,0x800740; st r0,[r1,0x2c]`).

**[гипотеза]** Значения аргумента — номера событий энергосбережения; при чтении их как номеров
системных событий (§2) ветка 5 соответствовала бы ASSOC_DONE, но ветка 2 совпадает с путём
PS_AWAKE_PEER_EVT, что говорит против этого прочтения. Какое событие поднимает ветку 5, не
установлено.

Приём маяка соседа без ассоциации: `find_main_sm__bcon` 0x8e9d48 кладёт MAC и IE соседа в пул
0x843134; элемент AW соседа в нём присутствует, но `set_aw` оттуда не вызывается.

## 4. Адреса кода в данных: две формы **[код]**

`fw_code` отображён для процессора с нуля (линкерные 0x000000…0x040000), для хоста — с 0x8c0000
([docs/HARDWARE-BLOCKS.md](https://github.com/klukonin/SparRAW-firmware/blob/main/docs/HARDWARE-BLOCKS.md)).
Указатели на код в `fw_data` и в таблице векторов записаны в форме процессора: `j 0x774` в
векторе = 0x8c0774, запись 0x1d268 = 0x8dd268. Поиск указателей в хостовой форме в 4.1 находит
1 указатель на код в `fw_data` и 13 адресов кода в данных; в форме процессора — 686 и 1117;
на 173 из 1069 участков вне функций указывает запись таблицы. Код вне функций дизассемблера
(46 232 Б, 22 % `fw_code`) имеет входы из таблицы векторов, из таблиц обработчиков в `fw_data`
или является продолжением больших функций (переходы извне внутрь участка, например
`brhs r15,0x5` из 0x8da4c8).

## Не установлено

* Номер события, поднимающего ветку 5 `ps_assoc_mgr__set_awake_tsf` (приём AW).

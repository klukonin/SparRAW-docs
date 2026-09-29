# Одноранговый режим Wilocity: ABI и логика есть, включение отсутствует

Тезис: семейство Wilocity/Sparrow несёт одноранговый режим (ADHOC/PBSS с распределённым
биконингом), у которого в драйвере и в образах отсутствует только включение, а
определения ABI и логика прошивки сохранены. Линия Terragraph пошла другим путём —
TDMA-планировщик. Механика биконинга в прошивке —
[BEACONING.md](https://github.com/klukonin/SparRAW-firmware/blob/main/docs/BEACONING.md);
межверсионное сравнение — [CROSS-VERSION.md](https://github.com/klukonin/SparRAW-firmware/blob/main/docs/CROSS-VERSION.md);
причины, по которым распределённый режим не стал продуктом, —
[WHY-MESH-ABANDONED.md](WHY-MESH-ABANDONED.md).

## ABI сохранён **[драйвер]**

Мейнлайн-драйвер (OpenWrt 25.12.5 / backports-6.18.26, `wmi.h`):
* `enum wmi_network_type`: INFRA = 1, **ADHOC = 2, ADHOC_CREATOR = 4**, AP = 0x10,
  P2P = 0x20, WBE = 0x40 (`wmi.h:311`);
* режимы трансляции PBSS (мостовой слой VRING AP↔PBSS↔STA): `WMI_VRING_DS_PBSS`,
  `WMI_NWIFI_TX_TRANS_MODE_AP2PBSS/STA2PBSS`, `WMI_NWIFI_RX_TRANS_MODE_PBSS2AP/PBSS2STA`
  (`wmi.h:897/905/1176`);
* `WMI_DIS_REASON_IBSS_MERGE = 0x0E` (`wmi.h:2364`).

Те же определения — в ABI Wilocity 2017 (`wigig_public_2017/.../wmi.h`): полный
`wmi_network_type_e`, `DS_PBSS_MODE`, `NWIFI_*_PBSS_TRANS_MODE`, `IBSS_MERGE = 0xe`. С
эпохи Wilocity ABI не менялся.

## Логика есть в прошивке **[код]**

* Сетевые типы ADHOC/ADHOC_CREATOR распознаются на MID (`mid__log_network_type`:
  4.1 0x8e1b34, 6.2 0x8e696c; в 6.2 одинаково у UBNT и RouterOS); networkType ADHOC даёт
  `m_bss_mode = 2`
  ([BEACONING.md §2.2](https://github.com/klukonin/SparRAW-firmware/blob/main/docs/BEACONING.md#22-кто-ставит-bi_mode-обе-код)).
* PBSS Direct Connection (`WMI_PBSS_JOINED/LEAVE` 0x15/0x16), список станций по AID,
  режим обнаружения (LMAC 0x24).
* Слой соседей в ucode: ATIM, doze, допуск к передаче пиру (`get_tx_eligibility`),
  окно AW ([4.1/docs/UCODE-BEACON.md](https://github.com/klukonin/SparRAW-firmware/blob/main/4.1/docs/UCODE-BEACON.md)).
* Рандомизатор отсрочки маяка присутствует в 4.1, 5.2, 6.2 и wil6436 7.5, отсутствует в
  1.4.1.7698, 2.2 и Terragraph 10.x
  ([BEACONING.md §4.2](https://github.com/klukonin/SparRAW-firmware/blob/main/docs/BEACONING.md#42-гейт-и-адреса-по-версиям-код));
  в ucode 6.2 из кода адресуются и `bti_worker`, и `dti_worker`, и `fixed_scheduling`
  ([6.2/docs/FIXED-SCHED.md](https://github.com/klukonin/SparRAW-firmware/blob/main/6.2/docs/FIXED-SCHED.md)).
* Роли назначаются на MID; пул MID есть.

В 4.1.0.1000 телеметрия распределённого режима адресуется из кода fw:
`[NNL] tx bcon|rx bcon|detected`, `bcon bitmap`, `tx atim pass/fail/counter` —
`lmac_if__mac_monitor_report` 0x8d865c (блок счётчиков 0x854de0…0x854e4c: tx/rx bcon,
bitmap, atim, rts/cts/dts, nav, availability); `AWAKE PEERS EVENT: awake peers vector` —
`lmac_if__awake_peers_evt` 0x8d8078; `ies_block::merge()` — `ies_block__merge` 0x8da098;
`availability_vec` — `FWQ_TX_AVAILABILITY__add_packet` 0x8c23b8 /
`FWQ_TX_AVAILABILITY__remove_packet` 0x8de304; `discovery.cpp` — четыре функции
([WHY-MESH-ABANDONED.md](WHY-MESH-ABANDONED.md)).

## Включения нет **[драйвер + образы]**

* `wiphy->interface_modes` (`cfg80211.c:2684`) без ADHOC; в `wil_cfg80211_ops` нет
  `.join_ibss/.leave_ibss`.
* Соответствие `wil_iftype_nl2wmi{ADHOC → WMI_NETTYPE_ADHOC}` (`cfg80211.c:333`) оставлено;
  рядом `MONITOR → ADHOC /* FIXME */`.
* Ни в одном образе корпуса нет записи concurrency (`wil_fw_record_concurrency`, magic
  `0xfedccdef`, comment type = 1): RouterOS и mainline Sparrow, wil6436, Terragraph TG,
  cnWave IF2IF. Без неё драйвер запрещает виртуальные интерфейсы
  (`n_iface_combinations == 0` → «virtual interfaces not supported», `cfg80211.c:708`).
* Образы Sparrow из RouterOS не несут comment-записей brd/RF, только capabilities
  (`0xabcddcba`).

Возврат включения со стороны драйвера —
[DRIVER-IBSS-PATCH.md](DRIVER-IBSS-PATCH.md).

## Тракт данных PBSS — прямой **[драйвер]**

`txrx.c`: в PBSS юникаст идёт через `wil_find_tx_ucast` → VRING по адресу пира; на каждый
CID своё TX-кольцо (`wil_ring_init_tx(vif, evt->cid)` на событие соединения, `wmi.c`).
Широковещательный кадр в PBSS дублируется по кольцам всех станций (отдельного
bcast-VRING нет, `txrx.c:2342`). Данные между членами PBSS идут peer-to-peer, PCP их не
ретранслирует: один PBSS — «звезда» по маяку и таймингу и меш по данным.

## Имена в паке 11ad (wil6436 7.5) **[пак]**

`ucode_image_globals.xml` и `wmi/wmi.xml` пака 11ad (соседний чип wil6436, общая
родословная):
* `BSS_IBSS_MODE = 4` — отдельный `bss_mode` для IBSS (в Sparrow IBSS сведён к PBSS с
  `bss_mode = 2`);
* синхронизация TSF по пирам: `peer_tsf_reseted`, `m_upm_unsynced_peer`;
* слой пиров ucode: `m_peers_pm_eligibility_vec`, `peer_cid_vec`, `peer_enable_mask`,
  `psc_awake_peer_vector`, `m_awake_peer`, `m_peers_curr_psc`;
* обнаружение: `g_discovery_mode_on`, `g_discovery_type`, `BI_TX_DISCOVERY_BCON`,
  `ABFT_RESPONDER_VALID_BEACON_TYPE_DISCOVERY`, `PEER_STA_TYPE_DMG/EDMG`;
* WMI: `WMI_DIRECT_SCAN` (`direct_scan_mac_addr`, `direct_snr_thr` — поиск пира по MAC),
  `WMI_SET_MULTI_DIRECTED_OMNIS_CONFIG`, `WMI_NETTYPE_ADHOC/ADHOC_CREATOR`, режимы
  трансляции PBSS.

## Версии корпуса

| источник | FW | таблицы строк |
|---|---|---|
| TP-Link Talon AD7200 (2017) | 1.4.1.7698 | нет (записи 6/2/7) |
| RouterOS 6.41.2 (wil6210-d0) | 6.1.0.40 | нет |
| RouterOS 6.42.1 | 4.1.0.1000 | есть (100/101) |
| RouterOS 6.46.x | 6.2.0.1000 | есть (100/101/102) |

Таблицы строк fw 4.1 и 6.2 разные (2179 против 2474 строк). В 4.1 есть 35 строк
слоя соседей, отсутствующих в таблице fw 6.2: `[NNL] | tx bcon | rx bcon | detected`,
`bcon bitmap`, `tx atim pass/fail/counter`, `rx atim`, `bcons_atim_fail_vec`,
`bad_beacons_num_threshold`, `availability_vec`, NAV/backoff, RTS/CTS/DTS,
`SYS_STATE_EVT_DISCOVERY_START/DONE`, `discovery.cpp`, `AWAKE PEERS EVENT`,
`Ucode->PS_AWAKE_PEER_EVT`, `conn_mgr::awake_peer_ntf`, `ies_block::merge()`. Отчёт
MAC_MON и сам механизм при этом есть в обеих версиях
([BEACONING.md §8](https://github.com/klukonin/SparRAW-firmware/blob/main/docs/BEACONING.md#8-отчёт-mac_mon-обе)):
различие — в диагностике fw, а не в механизме ucode.

Язык fw 4.1 — C++ с низкоуровневыми частями на C: в таблице строк 421 `class::method`,
81 файл `.cpp` против 34 `.c` (`SDP_GPIO_srvs.c`, `arc_action_point.c`,
`calib_cfg_schemes.c`, `cnt_handler.c` и др.). Ассерты 4.1 несут `file.cpp:line`; карта
«функция → файл» — `4.1/ref/FN-FILEMAP.txt` в SparRAW-firmware.

## Линия Terragraph

`wil6210_tg.fw` (10.11) и cnWave IF2IF — другая линия прошивок с TDMA
(`fixed_scheduling`, PTP, PRING) без рандомизатора
([UNPACK-1.8-and-ubnt.md](UNPACK-1.8-and-ubnt.md)). Многоузловой 60 ГГц там достигнут
TDMA-планировщиком, а не одноранговым режимом Wilocity.

## Не установлено

* Работал ли одноранговый режим Wilocity как симметричный многоузловой или как PBSS с
  одним PCP; доходит ли ADHOC_CREATOR до всех нужных состояний автоматов.
* Есть ли в ucode Sparrow аналог синхронизации TSF по пирам (поименован только в wil6436).
* Эталон с включённым режимом (история mainline wil6210 до удаления ADHOC, драйверы и
  прошивки Talon AD7200 эпохи 1.4) не найден.

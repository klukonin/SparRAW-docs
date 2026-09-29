# IBSS/adhoc на wil6210: устройство и опыт на стенде

Как прошивка трактует типы сети ADHOC/ADHOC_CREATOR и что получается при переводе двух узлов в
IBSS. Путь `networkType → bss_mode → bi_mode` и роль режима обнаружения —
[docs/BEACONING.md §2.2, §2.4](https://github.com/klukonin/SparRAW-firmware/blob/main/docs/BEACONING.md);
обработка ADHOC в 4.1 (делегирование Probe Req хосту, отсутствие IBSS-биконинга и слияния) —
[docs/ROLELESS-LINK.md §6](https://github.com/klukonin/SparRAW-firmware/blob/main/docs/ROLELESS-LINK.md);
события `WMI_PBSS_JOINED` / `WMI_PBSS_LEAVE` —
[4.1/docs/UCODE-BEACON.md](https://github.com/klukonin/SparRAW-firmware/blob/main/4.1/docs/UCODE-BEACON.md).

Метки: **[код]**, **[железо]**, **[гипотеза]**. Стенд: два узла 4.1.0.1000, узел A и узел B.

## 1. ADHOC — метка поверх PBSS **[код]**

* Единственная развилка по типу сети при старте PCP — `l2_mgr__pcp_start_flow` (4.1 0x8dc544,
  6.2 0x8e0778): `bss_mode = (network_type == 0x10 /* AP */) ? 3 : 2`. ADHOC (0x02),
  ADHOC_CREATOR (0x04) и P2P (0x20) дают `bss_mode = 2` и одинаково запускают PCP.
* Обработчик типа сети (`mid__log_network_type`, 4.1 0x8e1b34, 6.2 0x8e696c) только печатает
  тип и сохраняет его в структуре MID по +0x14; тип уходит в вендорский IE Wilocity
  (`vendor_ie__build_wilocity`, 6.2 0x8e73a0) и в события
  `WMI_PBSS_JOINED`/`LEAVE`, поведение не меняет.
* Отдельного пути «найти чужую ячейку и присоединиться» в прошивке нет.
* Примитивы PBSS для равноправных членов (6.2): список членов по AID (`pbss__add_sta` 0x8e01a4 →
  `pbss__alloc_sta_entry` 0x8e0148), обновление списка с посекторной настройкой BF для пира
  (`pbss_sta_list_update` 0x8d88c4, 572 Б), событие прямого соединения
  (`l2mgr__send_pbss_joined_evt` 0x8f6dac, `l2mgr__send_pbss_leave_evt` 0x8f89ec).

## 2. Роли по MID **[код]**

* MID выдаются из пула 0x804e7c (`mid__open`, 6.2 0x8c34d0): строки «Reject MID opening - already exist»,
  «Cannot open additional MID - no resources» — число одновременных MID ограничено; поиск
  контекста по номеру MID — `mid_list__by_mid_fw` (6.2 0x8cc3dc).
* Запреты ролей проверяются для текущего MID: `validate_connect` (6.2 0x8fb348) — «WMI Connect is
  not allowed in PCP mode» при `bss_mode == 2` этого MID; `l2_mgr__validate_scan`
  (6.2 0x8ebbc8) — «blocked, in PCP/AP mode».
* Следствие **[гипотеза]**: на одном узле допустимы MID0 = PCP своего PBSS и MID1 = STA чужого
  PBSS («двойной PCP»). Не проверено: работа двух независимых BI на одном радио, прямое
  соединение через границу PBSS, разрешение такой комбинации интерфейсов в cfg80211
  (`wil_cfg80211_iface_combinations`).

## 3. Оба узла в IBSS: маячат, но друг друга не слышат **[железо]**

Узлы переведены в IBSS (`iw dev … set type ibss`, `ibss join MESH60 58320`); тип интерфейса
разрешён драйверным патчем `900-adhoc-iftype`.

* Каждый узел начинает маячить сам. Блок конфигурации BI в ucode (0x801438) заполняется так же,
  как у точки доступа:
  ```
  до IBSS:  00 00 00 00  00 00 00 00  00 00 00 00
  в IBSS:   01 00 00 00  04 00 00 0f  00 01 01 00
  ```
  `bi_mode = 1`, период A-BFT 4, индекс расписания 15, один слот; `MAC_MON` — `tx bcon 63` на обоих.
  Раскладка блока — [4.1/docs/BEACON-CONFIG.md](https://github.com/klukonin/SparRAW-firmware/blob/main/4.1/docs/BEACON-CONFIG.md).
* Маяки в эфире есть: узел, возвращённый в приёмный режим, находит маяк соседа сканированием
  (`capability: IBSS (0x0006)`, −49 дБм против −72 дБм в линке AP–STA).
* При обоих узлах в IBSS на одном канале: `rx bcon 0`, `detected 0`, `bcon bitmap` нулевая на
  обоих — окна BTI перекрываются, каждый передаёт в своём BTI и не слушает в это время.
* С включённым гейтом рандомизатора (`bi_mode = 2`) слышимость не появляется: рандомизатор
  маяк не сдвигает ([docs/BEACONING.md §9.2](https://github.com/klukonin/SparRAW-firmware/blob/main/docs/BEACONING.md)).
  Сдвиг маяков двух маячащих узлов в разных BSS даёт NAV-отсрочка (§5, §9.3 там же).

## 4. Присоединение к ячейке **[железо + код]**

* Драйвер (`wil_cfg80211_join_ibss`) безусловно звал
  `wmi_pcp_start(..., WMI_NETTYPE_ADHOC_CREATOR, ...)` — оба узла всегда создавали ячейку.
  Драйверный патч `912-wil6210-ibss-join-mode.patch` добавляет параметр модуля `ibss_creator`:
  при `N` отправляется `WMI_NETTYPE_ADHOC` («IBSS joined» вместо «IBSS created» в логе ядра).
* На прошивку это не влияет (§1): присоединяющийся узел 30 с подряд передаёт свои 63 маяка,
  `rx bcon 0`, `detected 0`, станций 0 — ведёт себя как создатель.
* Обход через `connect`: маяк создателя объявляет тип DMG = AP (capability 0x0006, по маске 0x3
  — 2; `iw` показывает «IBSS», разбирая бит по не-DMG правилам). `connect` завершается
  `-ENOENT` («Unable to find BSS»): в DMG-маяке нет SSID — он передаётся только в Probe Response,
  а `cfg80211_get_bss` ищет по SSID.

В 802.11ad классического IBSS нет: его место занимает PBSS с координатором (PCP) и передачей
роли (PCP handover, `psc_if.cpp`, `pcp_psc_sm.cpp`, `sta_psc_sm.cpp`). Рабочая пара PBSS через
hostapd/wpa_supplicant с `pbss=1` — [PBSS-ON-HW.md](PBSS-ON-HW.md).

## 5. Режим обнаружения как побочный путь **[железо]**

`echo 1 > discovery_mode` плюс активное сканирование заставляют прошивку отправить в ucode
настройку интервала:
```
bcn_tx_init_bi_cfg(): discovery_mode = 1
Send CMD_BCON_MGT: flags.pcp_assoc_ready = 0, discovery_mode = 1
lmac_if_discovery_mode_cfg_handler() send command to uCode discovery_mode = 1
bcn_tx_init_ss_params: discovery beacon total sectors 16
```
Блок конфигурации заполняется иначе, чем в IBSS: период A-BFT 1 (против 4), счётчик окна AW
стоит на нуле при идущих BTI и DTI (режим обнаружения сам снимает AW), discovery-маяк —
16 секторов вместо 63. Сканирование — приёмная операция; передача discovery-маяков идёт через
`wil_p2p_listen()` → `wmi_p2p_cfg(PEER2PEER)` → `wmi_start_listen()` и требует запущенного
интерфейса P2P-device (создаётся, `wdev 0x2`, но без wpa_supplicant не запускается).

Подсистема P2P Find (`find_mngr.cpp`, `find_main_sm.cpp`: чередование LISTEN/SEARCH по каналам,
приём discovery-маяков, список увиденных устройств, разбор Probe Req с WFA/P2P IE и атрибутом
DEVICE ID) на стенде в этом опыте не исполнялась:
```
FIND MNGR: Detect discovery beacon BSSID: %08x%04x discovery_mode=%d
FIND:: Start LISTENING on channel %d for %d msec
FIND:: Start SEARCHING on channel %d for %d msec
FIND_SM:: DMG Beacon info added: ADDR-%04X%08X
```

## 6. Запуск hostapd с `pbss=1` на стандартной конфигурации OpenWrt **[железо]**

OpenWrt не пробрасывает `pbss` в `/etc/config/wireless`; демон запускается вручную. За основу
взят конфиг рабочей точки доступа (`/var/run/hostapd-phy0.conf`: `hw_mode=ad`, `channel=1`,
`stationary_ap=1`, `beacon_int=100`, `bssid=`, `ctrl_interface=`) плюс `pbss=1`.

| попытка | конфиг | запуск | исход |
|---|---|---|---|
| 1 | урезанный, без `ctrl_interface`/`bssid` | `-B` | узел перестал отвечать, нужен перезапуск питанием |
| 2 | полный | передний план, таймаут 12 с | `interface state UNINITIALIZED->ENABLED`, `AP-ENABLED`, узел жив |
| 3 | полный | `-B` | узел перестал отвечать через несколько секунд |

Падение происходит не при инициализации, а через несколько секунд работы. Рабочая процедура
(с обязательной остановкой `wpad`) — [PBSS-ON-HW.md](PBSS-ON-HW.md).

## Не установлено

* Причина падения узла в попытках 1 и 3 (§6).
* Есть ли в ucode логика слияния ячеек (`IBSS_MERGE` = 0xe в перечислении причин разрыва WMI).
* Работоспособность схемы «двойной PCP» (§2).

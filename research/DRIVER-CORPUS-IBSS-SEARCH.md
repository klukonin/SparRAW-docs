# Корпус драйверов wil6210: IBSS не реализован ни в одном

Проверка по всему локальному корпусу бинарных драйверов, не сохранилась ли
реализация `join_ibss`/`leave_ibss` в каком-либо вендорском драйвере. Результат
отрицательный.

## Корпус

| источник | ядро | размер `.ko` | символов |
|---|---|---|---|
| TP-Link Talon AD7200 (2017) | 3.4.103 | — | 436 |
| Netgear R8900 | 3.10.20 | — | 1127 |
| Terragraph cnWave | 4.14.140 | — | — |
| RouterOS, 12 версий (6.41.2 → 6.49) | 3.3.5 | 66–88 КБ | 187 |
| RouterOS 7.2 | 5.6.3 | — | — |
| эталон: OpenWrt backports 6.18.26 | — | — | исходник |

## Результат

| драйвер | операций `wil_cfg80211_*` | символов с `ibss` |
|---|---|---|
| Talon AD7200 (2017) | 18 | 0 |
| Netgear R8900 | 17 | 0 |
| Terragraph cnWave | 45 | 0 |
| RouterOS 6.42.1 / 7.2 | слоя cfg80211 нет | 0 |

`join_ibss`/`leave_ibss` не реализованы ни в одном драйвере корпуса.
Соответствие `{NL80211_IFTYPE_ADHOC, WMI_NETTYPE_ADHOC}` в `wil_iftype_nl2wmi` —
наследие WMI-ABI, к пути cfg80211 не подключённое. Число операций cfg80211 со
временем только росло (18 → 45), ничего не удалялось. Драйверная сторона IBSS —
новый код (патчи 900, 901, 912 — [DRIVER-PATCHES-OPENWRT](DRIVER-PATCHES-OPENWRT.md)).

## Драйвер RouterOS без cfg80211

`wil6210.ko` RouterOS не содержит ни одной `wil_cfg80211_*`; вместо них —
`wil_sta_open`, `wil_sta_xmit_commit`, `wil_fast_path_xmit`, `wil_do_ioctl`,
`wil_change_l2mtu`, `cfg_keepalive_worker`. RouterOS не использует
nl80211/cfg80211: управление идёт из userspace-бинаря `nova/bin/wireless` через
ioctl, модуль транслирует его в WMI (`wmi_buffer`, `wmi_event_worker`).

«MESH» в RouterOS — не DMG-меш: строки `MESH: addNetDev/authorized/tx/rx/delete
mesh`, `ApPeer::gotBss: non MT in mesh mode`, `wants MESH support` в
`nova/bin/wireless` относятся к собственной L2-меш-функции поверх протокола
AP/STA вендора (на любом интерфейсе), а не к распределённому биконингу DMG. Там
же — управление 60 ГГц: `wil6210.fw`, `wap60g-omni`, `wap60g-sa-omni`,
`wap60g-sa-dir`, `drivers/net/wil6210.ko`.

## Применимость корпуса

* 12 версий `wil6210.ko` RouterOS — эволюция стека управления прошивкой без
  cfg80211 (альтернативный путь управления).
* `.ko` Talon AD7200 (2017) — ближайший к эпохе Wilocity драйвер с cfg80211
  (18 операций).
* Ghidra-проект `nova/bin/wireless` (3696 функций) — логика выбора board-файла
  для wAP 60G ([BRD-RF-REGS](BRD-RF-REGS.md)).

## Не установлено

* Использовал ли вендор RouterOS распределённый режим через собственный путь
  ioctl → WMI. Косвенно: потолок 8 пиров в прошивке совпадает с практическим
  пределом клиентов wAP 60G. Проверка — построение команд WMI в `wil6210.ko` и
  путь ioctl в `nova/bin/wireless`.

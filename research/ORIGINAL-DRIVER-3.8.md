# Первый драйвер wil6210 (Linux 3.8, 2012) и тип сети ADHOC

Исходный драйвер Wilocity — 14 файлов из `linux-3.8.tar.xz` (cdn.kernel.org), первая версия
драйвера в ядре, copyright «Qualcomm Atheros, Inc. 2012». Метки: **[код]** — исходники драйвера,
**[железо]**.

## 1. Что в драйвере есть и чего нет **[код]**

* `wiphy->interface_modes = STATION | AP | MONITOR` — ADHOC не объявлен.
* `wil_cfg80211_ops`: scan, connect, disconnect, change_virtual_intf, get_station,
  set_monitor_channel, add/del/set_default_key, start_ap, stop_ap; `join_ibss`/`leave_ibss` нет.
* Есть отображения `{NL80211_IFTYPE_ADHOC, WMI_NETTYPE_ADHOC}` и
  `{NL80211_IFTYPE_MONITOR, WMI_NETTYPE_ADHOC} /* FIXME */` — те же рудименты присутствуют и в
  backports-6.18.26.

## 2. Тип сети всегда ADHOC **[код]**

`main.c`, `__wil_up()`:

```c
u16 wmi_nettype = wil_iftype_nl2wmi(wdev->iftype);   /* обычное отображение */
rc = wil_reset(wil);
...
/* FIXME Firmware works now in PBSS mode(ToDS=0, FromDS=0) */
wmi_nettype = wil_iftype_nl2wmi(NL80211_IFTYPE_ADHOC);   /* безусловная подмена */
switch (wdev->iftype) { STATION / AP / P2P_CLIENT / P2P_GO / MONITOR ... }
```

Первый драйвер для любого типа интерфейса сообщал прошивке `networkType = WMI_NETTYPE_ADHOC`
(0x02). Причина — в комментарии: прошивка той эпохи работала в режиме PBSS (ToDS = 0,
FromDS = 0). ADHOC/PBSS был штатным режимом ранней связки драйвер + прошивка.

## 3. Современный драйвер: PBSS даёт `bss_mode = 2` **[код]**

backports-6.18.26, `_wil_cfg80211_start_ap()`:

```c
u8 wmi_nettype = wil_iftype_nl2wmi(wdev->iftype);   /* AP → WMI_NETTYPE_AP 0x10 */
...
if (pbss)
        wmi_nettype = WMI_NETTYPE_P2P;               /* флаг PBSS → 0x20 */
```

В прошивке `bss_mode = (network_type == 0x10) ? 3 : 2`, так что P2P (0x20) даёт тот же
`bss_mode = 2`, что ADHOC (0x02) и ADHOC_CREATOR (0x04). Флаг PBSS доступен без патча драйвера:
`NL80211_ATTR_PBSS` в cfg80211, `pbss=1` в hostapd и wpa_supplicant. Связь `bss_mode` с
`bi_mode` маяка и гейтом рандомизатора —
[docs/BEACONING.md §2.2](https://github.com/klukonin/SparRAW-firmware/blob/main/docs/BEACONING.md):
у обычного маяка PCP `bi_mode` всегда 1 независимо от `bss_mode`, так что сам по себе PBSS
рандомизатор не включает.

**[железо]** PBSS через `wpa_supplicant` на linux-firmware 5.2.0.18 и ванильном драйвере
поднимается (PCP ↔ STA): путь `pbss → WMI_NETTYPE_P2P → bss_mode = 2` работает без патча
драйвера. Рабочая пара на 4.1.0.1000 — [PBSS-ON-HW.md](PBSS-ON-HW.md).

## 4. Замечания по совместимости 4.1.0.1000 с современным драйвером

* События WMI сверены (28/28 ID совпадают), но структуры могли меняться: например, `WMI_READY` —
  0x10 байт в 4.1 против 0x14 в 6.2.
* Драйвер грузит `wil6210.brd`; формат board-файлов эпохи 4.1 может отличаться — см.
  [docs/HW-DRIVERS.md](https://github.com/klukonin/SparRAW-firmware/blob/main/docs/HW-DRIVERS.md).
* В linux-firmware 5.2.0.18 нет таблиц строк (записи 100/101), поэтому наличие подсистем в 5.2
  проверяется только по коду; рандомизатор в 5.2 присутствует
  ([4.1/docs/DISTBCN-PATCH.md](https://github.com/klukonin/SparRAW-firmware/blob/main/4.1/docs/DISTBCN-PATCH.md)).

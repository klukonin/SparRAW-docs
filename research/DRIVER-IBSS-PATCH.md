# Драйверная часть IBSS поверх PBSS: спецификация правок wil6210

Спецификация правок драйвера wil6210 (backports-6.18.26 / OpenWrt 25.12.5, номера строк —
по этой версии), которые открывают ячейку IBSS на прошивке 4.1.0.1000. Правки реализованы
серией патчей репозитория SparRAW-driver (`openwrt/patches/ath/`):

| правка | патч |
|---|---|
| 1, 2 — iftype ADHOC и его подтипы mgmt | `900-wil6210-adhoc-iftype.patch` |
| 3 — `join_ibss` / `leave_ibss` | `901-wil6210-join-ibss.patch`, выбор роли — `912-wil6210-ibss-join-mode.patch` (параметр `ibss_creator`) |
| 4 — событие PBSS Direct Connection | `913-wil6210-pbss-joined-event.patch` (только 0x15) |

Смежное в документации прошивки: networkType → `m_bss_mode` и цепочка до маяка —
[BEACONING.md §2](https://github.com/klukonin/SparRAW-firmware/blob/main/docs/BEACONING.md#2-конфигурация-биконинга);
прямой линк без ролей и `WMI_PBSS_JOINED` —
[ROLELESS-LINK.md](https://github.com/klukonin/SparRAW-firmware/blob/main/docs/ROLELESS-LINK.md);
совместимость WMI 4.1 с mainline-драйвером (28/28 номеров событий, 12 имён без пары в драйвере) —
[4.1/docs/UCODE-BEACON.md §7](https://github.com/klukonin/SparRAW-firmware/blob/main/4.1/docs/UCODE-BEACON.md#7-совместимость-wmi-4101000-с-mainline-драйвером).

## Исходное состояние драйвера

* В драйвере уже есть соответствие `{NL80211_IFTYPE_ADHOC, WMI_NETTYPE_ADHOC}`
  (`cfg80211.c:333`, `wil_iftype_nl2wmi`); networkType доходит до прошивки через него.
* `wiphy->interface_modes` (`cfg80211.c:2684`) ADHOC не содержит, `join_ibss`/`leave_ibss`
  в `wil_cfg80211_ops` нет.
* Виртуальные интерфейсы требуют записи concurrency в образе (`cfg80211.c:708` → -EINVAL
  «virtual interfaces not supported»); один adhoc-интерфейс на основном vif её не требует.

## Правка 1 — iftype ADHOC

`cfg80211.c:2684`, `wil_cfg80211_init()`:
```c
 wiphy->interface_modes = BIT(NL80211_IFTYPE_STATION) |
                          BIT(NL80211_IFTYPE_AP) |
+                         BIT(NL80211_IFTYPE_ADHOC) |
                          BIT(NL80211_IFTYPE_P2P_CLIENT) | ...
```

## Правка 2 — подтипы mgmt для ADHOC

`wil_mgmt_stypes[]` (`cfg80211.c:~274`): запись `[NL80211_IFTYPE_ADHOC]` по образцу AP —
PROBE_REQ/RESP, ACTION, AUTH, ASSOC_REQ/RESP, DISASSOC, REASSOC (узел ячейки и отвечает,
и запрашивает).

## Правка 3 — `.join_ibss` / `.leave_ibss`

```c
static int wil_cfg80211_join_ibss(struct wiphy *wiphy, struct net_device *ndev,
                                  struct cfg80211_ibss_params *params)
{
    /* создатель ячейки:
     *     wmi_pcp_start(vif, bi, WMI_NETTYPE_ADHOC_CREATOR, chan, ...)
     *     (тело wil_cfg80211_start_ap)
     * присоединяющийся:
     *     WMI_NETTYPE_ADHOC (тело wil_cfg80211_connect либо wmi_pcp_start)
     * В обоих случаях m_bss_mode в прошивке = 2. */
}
static int wil_cfg80211_leave_ibss(...)   /* wmi_pcp_stop / wmi_disconnect */
```

Роль в реализации выбирается параметром модуля `ibss_creator` (патч 912): без него оба
узла создают каждый свою ячейку и друг друга не видят.

## Правка 4 — события PBSS Direct Connection

`wmi.c`, таблица `wmi_evt_handlers[]`:
```c
+ {WMI_PBSS_JOINED_EVENTID /* 0x15 */, wmi_evt_pbss_joined},
+ {WMI_PBSS_LEAVE_EVENTID  /* 0x16 */, wmi_evt_pbss_leave},
```

Номера 0x15/0x16 лежат вне обычного диапазона событий драйвера (минимальные 0x200 и
0x1001) и в `wmi.h` свободны. Тела событий 4.1 (`l2mgr__send_pbss_joined_evt` @0x8eea68,
12 Б; `l2mgr__send_pbss_leave_evt` @0x8f0720, 10 Б) — в
[4.1/docs/UCODE-BEACON.md](https://github.com/klukonin/SparRAW-firmware/blob/main/4.1/docs/UCODE-BEACON.md#события-pbss-direct-connection-id-0x15--0x16).
Байт [0] тела 0x15 трактуется в документации прошивки как AID, в патче 913 — как индекс
канала (по образцу `wmi_connect_event`).

`wmi_evt_pbss_joined` повторяет действия `wmi_evt_connect` для нового пира: запись в
`wil->sta[cid]` и `wil_ring_init_tx(vif, cid)`. Без этого TX-кольцо пира не создаётся, а
юникаст в PBSS идёт через `wil_find_tx_ucast` по адресу пира. `wmi_evt_pbss_leave` —
обратное действие.

## Правка 5 (не требуется для одного интерфейса) — iface combinations

Несколько vif одновременно требуют в образе записи `wil_fw_record_concurrency`
(comment-запись, magic `0xfedccdef`). Её нет ни в одном образе корпуса
([WILOCITY-DORMANT-MESH.md](WILOCITY-DORMANT-MESH.md)).

## Порядок проверки на железе

1. Образ `wil6210_4.1.0.1000_openwrt.fw` (без записей 100/101/102, CRC вендорский —
   [4.1/docs/DISTBCN-PATCH.md](https://github.com/klukonin/SparRAW-firmware/blob/main/4.1/docs/DISTBCN-PATCH.md))
   как `/lib/firmware/wil6210.fw` + board `wil6210.brd`; загрузка до WMI_READY.
2. Модуль с патчами; `iw dev <if> set type ibss`, `iw dev <if> ibss join …`.
3. Телеметрия `MAC_MON`: `tx bcon / rx bcon / detected` на обоих узлах.

## Ограничения

* Ячейка ADHOC сама по себе не включает рандомизированную отсрочку маяка: гейт
  рандомизатора — `bi_mode == 2` (discovery-маяк при активном скане), а не `m_bss_mode`;
  у обычного маяка `bi_mode` = 1
  ([BEACONING.md §4.2](https://github.com/klukonin/SparRAW-firmware/blob/main/docs/BEACONING.md#42-гейт-и-адреса-по-версиям-код)).
  Результат на железе — [IBSS-ON-HW.md](IBSS-ON-HW.md).
* Диапазон отсрочки рандомизатора зашит (189 + 10) и не масштабируется от числа секторов.
* Структуры ucode для соседей (AW, ATIM) рассчитаны не более чем на 8 пиров.

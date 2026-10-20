# Таблица соседей при обнаружении (4.1.0.1000)

Что прошивка сохраняет о соседе на этапе P2P Find, до какой-либо ассоциации.
Автомат FIND_MAIN_SM, режимы LISTEN/SEARCH, discovery-маяки с A-BFT и
Probe Request после BF —
[docs/HOST-INTERFACE.md, «Обнаружение: P2P Find»](https://github.com/klukonin/SparRAW-firmware/blob/main/docs/HOST-INTERFACE.md);
номера команд и событий WMI —
[docs/WMI.md](https://github.com/klukonin/SparRAW-firmware/blob/main/docs/WMI.md);
останов сессии — [docs/MLME.md](https://github.com/klukonin/SparRAW-firmware/blob/main/docs/MLME.md).
Все выводы — **[код]**.

## Поток

```
WMI_START_LISTEN 0x914 / WMI_START_SEARCH 0x915
   -> l2_mgr__wmi_cmd_handler_start_listen / _start_search
        (запрещено, если узел CONNECTED или в роли GO)
   -> l2mgr__start_discovery 0x8f3354: SYS_STATE_EVT_DISCOVERY_START (0xf)
        + start_find_session(mid + 0x234)
   -> приём DMG-маяка соседа: find_main_sm__bcon 0x8e9d48
   -> BF удался:              find_main_sm__bf_ntf 0x8e9e60
   -> l2mgr__send_discovery_stopped 0x8f05d0: SYS_STATE_EVT_DISCOVERY_DONE (0x10)
        + WMI_DISCOVERY_STOPPED_EVENTID 0x1917
```

Драйвер шлёт эти команды из `p2p.c` (`wmi_start_listen`, `wmi_start_search`,
`wmi_stop_discovery`) — обнаружение запускается штатными операциями Wi-Fi Direct.

## `find_main_sm__bcon`: запись о соседе

1. Поиск отправителя в списке сессии (`find_session + 0x3c`,
   `search_scan_elem_in_list` 0x8f020c); если есть — выход.
2. Запись из пула **0x843134** (поля пула +0x0c/+0x10 — занято/предел); при
   исчерпании — `FIND_SM:: dband info pool is empty !!!! %x`.
3. Лог `FIND_SM:: DMG Beacon info added: ADDR-%04X%08X`.
4. `memcpy(entry + 0x08, mac, 6)`.
5. Если разобранный элемент `[ctx+0x0c]` не пуст и его длина (байт +9) ≥ 1 —
   тело копируется в `entry + 0x10`, длина — в `entry + 0x30` (SSID).
6. Вставка в список (`llist__insert` 0x8c1fc4).

Раскладка записи: `+0x08` MAC[6], `+0x10` SSID, `+0x30` длина SSID. На этом шаге
у прошивки есть MAC соседа и разобранные IE его маяка — без ассоциации.

## `find_main_sm__bf_ntf`

* Индекс пира `[ctx+8]`, проверка `< 8` (иначе ассерт `find_main_sm.cpp:462`) —
  потолок 8 соседей.
* Два байтовых массива по 8 записей: `find_session+0x20` (индекс) и `+0x2c`
  (состояние; 8 — ожидание Probe Response).
* Probe Request с широковещательным BSSID (байты 0xff), адресованный MAC соседа
  из `[ctx+0x10]`; лог `bf_ntf:: send probe_req with bcast BSSID...ssid_len= %d`.

## Следствие

Приём окна AW соседа (`ps_assoc_mgr::aw_ie_reception`) подписан на
`SYS_STATE_EVT_ASSOC_DONE`, то есть требует ассоциации. В `find_main_sm__bcon`
сосед уже опознан и IE его маяка разобраны — это точка, где параметры соседа
(например, AW) доступны на каждом принятом маяке без ассоциации **[гипотеза:
точка врезки]**.

## Не установлено

* Какой именно элемент лежит в `[ctx+0x0c]` (по форме — разобранный IE: байт +9
  длина, тело с +0x0c).
* Содержит ли маяк соседа на этой стадии элемент AW.

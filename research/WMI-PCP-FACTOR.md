# Событие WMI_PCP_FACTOR_EVENTID (0x191A) в прошивке 4.1

Метрика пригодности на роль PCP, которую прошивка 4.1.0.1000 отдаёт хосту по запросу.
Драйвер событие не разбирает. Номера команды и события в таблице диспетчера —
[WMI.md](https://github.com/klukonin/SparRAW-firmware/blob/main/docs/WMI.md) (0x91b
`WMI_GET_PCP_FACTOR_CMDID`).

## Путь **[код]**

`WMI_GET_PCP_FACTOR_CMDID` (0x91B) → `wmi_handler_get_pcp_factor` @0x8f2cdc (80 Б,
`layer2_mgr`):

```
evt = operational_if__alloc_evt(mid)              ; 0x8c2b68
m   = mid_list__by_mid(mid)                       ; 0x8ca1b0
pcp_factor_evt_step(m+0x48, m+0xa6, g_8004xx, evt)       ; 0x8c4a40
fwlog(11, 2, "<<---- [HOST EVENT] WMI_PCP_FACTOR_EVENTID for devid %d.")
operational_if__send_evt(evt, 0x191a, 4, mid, 0, 0)   ; 0x8e0874
```

## Содержимое **[код]**

`pcp_factor_evt_step` @0x8c4a40 собирает на стеке одно 32-битное слово и кладёт его в
буфер события; событие — 4 байта, это слово. Слово упаковывается из битовых полей структуры
по `m+0xa6` (сохранённые возможности DMG) аксессорами битовых полей 0x8e81e0, 0x8e8258,
0x8e836c, 0x8e8534 ([FW-UTILITIES.md](https://github.com/klukonin/SparRAW-firmware/blob/main/docs/FW-UTILITIES.md)):

| источник | преобразование |
|---|---|
| `[+0x07]`, `[+0x08]` | 16 бит, сдвиг вправо на 7, маска 7 бит |
| `[+0x0f]`, `[+0x10]` | 16 бит, сдвиг вправо на 3, байт |
| `[+0x0f]` | один бит (сдвиг переменный) |
| `[+0x10]` | сдвиг вправо на 4, один бит |
| `gp[-0x17]` | ещё один бит, аргумент |

По смыслу — фактор выбора PCP 802.11ad (число ассоциированных станций, признак питания,
возможности PHY), упакованный в одно слово для сравнения кандидатов **[гипотеза]**.

## На железе **[железо]**

Команда без полезной нагрузки (8 байт заголовка `struct wmi_cmd_hdr`: `mid`, `reserved`,
`command_id`, `fw_timestamp`) через debugfs `wmi_send`:

```
драйвер:  wil_write_file_wmi: 0x091b[0] -> 0
прошивка: HOST CMD 0x0000091B / ---->> [HOST CMD] WMI_GET_PCP_FACTOR_CMDID
прошивка: <<---- [HOST EVENT] WMI_PCP_FACTOR_EVENTID for devid 0
драйвер:  wmi_event_handle: Unhandled event 0x191a
```

## Передачи роли PCP в прошивке нет **[код]**

Поиск по таблице строк fw 4.1 (2292 строки) по словам handover / takeover / successor /
cluster даёт три совпадения, все про эту метрику:

```
[host_if.cpp   ] ---->> [HOST CMD] WMI_GET_PCP_FACTOR_CMDID
[layer2_mgr.cpp] <<---- [HOST EVENT] WMI_PCP_FACTOR_EVENTID for devid %d.
[mid.cpp       ] MORDI. TAKE m_dmg_bcon_additional_ies
```

Автомата передачи роли и обмена кадрами Handover Request/Response нет. Подсистема
`psc_if.cpp` / `pcp_psc_sm.cpp` / `sta_psc_sm.cpp` — согласование энергосбережения (PSC,
строки про `bi_start_time`, `number_of_awake_bi`, `sleep_cycle`), а не передача роли
([STATE-MACHINES.md §3](https://github.com/klukonin/SparRAW-firmware/blob/main/docs/STATE-MACHINES.md#3-подсистема-энергосбережения)).
Метрика доступна с хоста одной командой, поэтому выбор координатора можно строить на
хосте: опросить фактор у себя и у соседа и сравнить.

## Не установлено

* Точный смысл каждого поля слова.
* Чтение значения на хосте: нужен разбор события 0x191a в драйвере или чтение почтового
  ящика.

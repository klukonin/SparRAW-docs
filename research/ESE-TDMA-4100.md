# ESE-TDMA в прошивке 4.1.0.1000

Штатное расписание 802.11ad (Extended Schedule Element, ESE): поле Allocation в маяке PCP
задаёт слоты DTI по AID источника и назначения. Общее устройство подсистемы, команда хоста
0xa01 и разбор 6.2 — [docs/ESE.md](https://github.com/klukonin/SparRAW-firmware/blob/main/docs/ESE.md);
пара «команда — обработчик» — [docs/WMI.md](https://github.com/klukonin/SparRAW-firmware/blob/main/docs/WMI.md).
Здесь — подробности 4.1, которых в документации прошивки нет (там тела 4.1 не читались), и опыт
на стенде.

Метки: **[код]** — по листингу 4.1, **[железо]** — проверено на стенде, **[гипотеза]**.

## 1. Модуль `fw_scheduled_dti.cpp` **[код]**

| функция 4.1 | адрес | роль |
|---|---|---|
| `wmi_handler_ese_cfg` | 0x8f2c6c | ветка диспетчера WMI для 0xa01 (`cmp_s r0,0xa01` @0x8da824 → 0x8daa22, строка имени команды `---->> [HOST CMD] WMI_ESE_CFG_CMDID` по 0x1006828, смещение 0x06828 в таблице строк fw, — `mov_s r2,0x1006828` @0x8daa22); обёртка: находит MID и зовёт `ese_cfg__apply` |
| `ese_cfg__apply` | 0x8e7418 (528 Б) | разбор тела команды, таблица аллокаций `mid+0x2b4`, элемент ESE для маяка |
| `fw_scheduled_dti__update_ie` | 0x8e5708 | переиздание элемента в маяке: база времени `mac__mul_886fc4_by_886d64` (TBTT), `ies_block__find_ie`, затем `tx_bcon__update_bcon` |
| `bcon__refresh_dti_ie` | 0x8e5784 | вызывающий `fw_scheduled_dti__update_ie`; зовётся из `ps__on_awake_peers` |
| `parse_ese` | 0x8db3cc | разбор ESE из принятого кадра; вызывается через `dti__parse_ese_at692` 0x8db5b4 из `discovery__handle_rx_frame` |
| `allocation_type` | 0x8dcd68 | лог-дамп полей одного поля Allocation (зовёт `parse_ese`) |

Имя команды в строках прошивки — `WMI_ESE_CFG_CMDID`; в перечислении mainline-драйвера тот же
номер 0xa01 называется `WMI_SCHEDULING_SCHEME_CMDID`, и ни один отправитель драйвера его не шлёт.

## 2. Тело команды 0xa01 в 4.1 **[код]**

```
+0x00  u8   число аллокаций (проверка на 3)
далее по 6 байт на аллокацию:
  +0  u8   source_aid
  +1  u8   destination_aid
  +2  u8   allocation_type (младшие биты, bmsk 0x2 → биты 0..2)
  +3  u8   назначение не установлено
  +4  u16  allocation_block_duration (LE)
```

Лог-строки обработчика: `ESE CFG CMD. ESE: %d`, `ESE CFG CMD. Source AID: %d, Destination AID: %d`,
`ESE CFG CMD. allocation_type: %d, allocation_block_duration: %d, allocation_id: %d`. Переполнение
таблицы (`brlt r0,0x3`) даёт строку `Number of DTI allocations exceeded: %d. Ignoring allocation` и
`fw_sysassert_fatal` с кодом 0x11bb.

Выход: элемент **ID 0x90** (144, DMG Extended Schedule element), длина `n × 15` — по 15 байт на
поле Allocation; раскладка поля (из `allocation_type`) совпадает с 6.2: биты [6:4] байта +0 —
allocation_type, бит 7 — pseudo_static, +4 source_aid, +5 destination_aid, +6 u32 начало (TSF),
+0xa u16 allocation_block_duration, далее number_of_blocks.

Типы слотов в перечислении wil6436 (`wmi.xml`): `WMI_SCHED_SLOT_SP` = 0, `CBAP` = 1, `IDLE` = 2,
`ANNOUNCE_NO_ACK` = 3, `DISCOVERY` = 4; режимы объявления ESE: `WMI_ADVERTISE_ESE_IN_BEACON` = 1,
`IN_ANNOUNCE_FRAME` = 2. Соответствие этих номеров полю allocation_type 4.1 не проверено.

## 3. Отправка с хоста и приём у соседа **[железо]**

Стенд: два узла на **стоковой** 4.1.0.1000 (узел AP и узел STA), прошивка не модифицировалась.
Команда отправляется через debugfs `wmi_send`: заголовок `struct wmi_cmd_hdr` (mid, reserved,
command_id LE16, fw_timestamp LE32) плюс тело.

```python
hdr   = struct.pack('<BBHI', 0, 0, 0x0a01, 0)
alloc = struct.pack('<BBBBH', 1, 2, 0, 0, 1000)   # src=1, dst=2, type=0, dur=1000
open('cmd.bin', 'wb').write(hdr + bytes([1]) + alloc)
```
```
cat cmd.bin > /sys/kernel/debug/ieee80211/phy0/wil6210/wmi_send
```

`printf` из busybox на устройстве портит нулевые байты: тело готовится файлом на хосте и
копируется на узел (`scp -O`), иначе драйвер получает мусор (`0x0000[3]` вместо `0x0a01[7]`).

Результат: драйвер — `wil_write_file_wmi: 0x0a01[7] -> 0`; приёмник на стоковой прошивке
разбирает элемент из маяка:

```
MOD13    INFO [PRS] ESE EI ID=144, EI_LEN=15 (pos_to_end 15)
CONN_MGR INFO Known ESE
```

Путь «хост → маяк → разбор у соседа» работает на штатной прошивке без патчей. Изменение
расписания передач при этом не проверялось — доказан только разбор элемента.

## 4. Связь с планировщиком колец **[гипотеза]**

Автомат `sm_pring_connectivity` (привязка колец к PRING, `sm_pring__bind_vring` 0x8c3fdc,
`sm_pring__slot_optimized` 0x8e4ef4 — «PRING #%d slot optimized recognized at %d», закрытие кольца
при простое PRING, лог «PRING #%d inactivity recognized»; вызовы привязки из 0x8c6300 и 0x8e4df4) описан в
[docs/DATAPATH.md §4.1](https://github.com/klukonin/SparRAW-firmware/blob/main/docs/DATAPATH.md).
Предположение: аллокация ESE задаёт, какой пир активен в слоте, а автомат PRING открывает и
закрывает кольцо к нему. Прямой связи в коде (вызова из `ese_cfg__apply`/`parse_ese` в автомат
PRING) не найдено.

## 5. Происхождение

* В 4.1 одновременно присутствуют распределённый биконинг (развёртка с рандомизатором,
  `bad_beacons_detector`) и штатный ESE-TDMA; фиксированного расписания (`fixed_scheduling`,
  гранты) в 4.1 нет — строк нет вовсе, оно появляется в 6.2
  ([6.2/docs/FIXED-SCHED.md](https://github.com/klukonin/SparRAW-firmware/blob/main/6.2/docs/FIXED-SCHED.md))
  и в линии wil6436/Terragraph.
* Mainline-драйвер знает `WMI_PORT_ALLOCATE` (0x911) и `fixed_scheduling`, а ESE-команду — только
  как неиспользуемый идентификатор 0xa01. Для задания расписания штатными средствами хоста
  отправитель надо добавить в драйвер.

## Не установлено

* Смысл байта +3 аллокации в теле команды.
* Влияет ли аллокация на реальный доступ к среде (расписание передач), а не только на
  содержимое маяка.
* Можно ли накапливать таблицу `mid+0x2b4` повторными командами; поведение при
  `source_aid = 0xff` (в коде есть ветка «Received Broadcast AID»).
* Полная раскладка таблицы `mid+0x2b4` в 4.1.
* Какой программой хоста команда посылалась в эпоху 4.1.

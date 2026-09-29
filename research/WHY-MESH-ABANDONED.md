# Почему распределённый режим Wilocity не стал продуктом

Разбор причин, по которым распределённый (состязательный, с ATIM и рандомизированной
отсрочкой маяка) одноранговый режим семейства Wilocity/Sparrow не попал в продукты, по
коду 4.1.0.1000 и документальному следу патента. Механика режима —
[BEACONING.md](https://github.com/klukonin/SparRAW-firmware/blob/main/docs/BEACONING.md),
сохранность ABI и логики — [WILOCITY-DORMANT-MESH.md](WILOCITY-DORMANT-MESH.md), патент —
[PATENT-US8520648-SPEC.md](PATENT-US8520648-SPEC.md) и
[BEACONING.md §4.3](https://github.com/klukonin/SparRAW-firmware/blob/main/docs/BEACONING.md#43-сверка-с-патентом-us8520648b2).

## Подсистемы 4.1 по исходным файлам **[код]**

Карта «функция → файл» — `4.1/ref/FN-FILEMAP.txt` в SparRAW-firmware.

| файл | функции |
|---|---|
| `discovery.cpp` | `discovery__handle_rx_frame` 0x8c7c80, `find_mngr__enter_listen` 0x8edbbc, `find_mngr__enter_search` 0x8f01ac, `start_find_session` 0x8f12f8 |
| `bad_beacons_detector.cpp` | `bad_beacons_detector__configure` 0x8c5a34, `__init` 0x8d5f8c, `__is_bad` 0x8d70fc |
| `tx_bcon.cpp` | `tx_bcon__update_bcon` 0x8c2190, `add_discovery_bcon` 0x8c22e4 |
| `fw_scheduled_dti.cpp` | `dti__parse_ese_at692` 0x8db5b4, `fw_scheduled_dti__update_ie` 0x8e5708, `ese_cfg__apply` 0x8e7418 |
| `sm_pring_connectivity.cpp` | `sm_pring__bind_vring` 0x8c3fdc, `sm_pring__timeout` 0x8e43c8, `sm_pring__connectivity_update` 0x8e4ea8, `sm_pring__slot_optimized` 0x8e4ef4, `sm_pring__state_step` 0x8e5358 |
| `find_main_sm` / `find_mngr` / `mlme_sm` | `find_main_sm__bcon` 0x8e9d48 / `find_mngr__cfg_offload` 0x8c5df4 / `mlme_notify` 0x8da490, `rx_pkt_handler` 0x8df208, `mlme_sm__send_disassoc` 0x8ee7fc |

* Распределённый MAC исполняется в ucode: блок счётчиков соседей публикует ucode
  (`bi_window_transition_prep` 0x9370b8), fw только печатает его
  (`lmac_if__mac_monitor_report` 0x8d865c). Приём и передача маяков, окна ATIM, backoff/NAV
  — реальное время ucode; fw — конфигурация и диагностика.
* `discovery__handle_rx_frame` работает при `bss_mode ∉ {2, 3}` (узел не PCP/AP) — путь
  члена сети, ищущего соседей; `discovery_mode` — двухбитное поле (`cfg >> 4 & 3`) плюс
  глобальные переопределения (0x803474 & 0x200, 0x803468).
* `tx_bcon.cpp` — старт и останов маяка, параметризованы режимом (2/3) и `discovery_mode`.

## Признаки в коде

1. **Ненадёжность маяков.** Отдельный модуль `bad_beacons_detector.cpp` и счётчики
   `bad_bcons_cnt`, `bad_beacons_num_threshold`, `bcons_atim_fail_vec`: детектор плохих
   маяков с порогами нужен, когда маяки регулярно теряются. В распределённом режиме каждый
   узел и шлёт, и слушает маяки; при узких лучах 60 ГГц это частые промахи.
2. **Состязательный доступ поверх направленного mmWave.** Счётчики MAC_MON — портрет
   CSMA-подобного доступа: `tx atim pass/fail/counter`, `rx atim`, `rx rts | tx cts | tx dts`,
   `nav accumulator`, `backoff`, `cf end`, `false_alarms.out_txop_ad / total_fa`,
   `rx off duration`, `ka pass/fail`. Несущая вне луча не слышна, скрытый узел при узких лучах
   неустраним.
3. **Предел масштаба.** `bad_beacons_detector` ведёт массив не более чем на 8 соседей
   (assert `7 < idx`, `bad_beacons_detector.cpp:5243`, запись 0x24 Б); структуры ucode (AW,
   ATIM) — тоже 8 пиров
   ([BEACONING.md §6.5](https://github.com/klukonin/SparRAW-firmware/blob/main/docs/BEACONING.md#65-окно-aw-ati)).
4. **Логика в микрокоде.** Доводка тайминга и доступа требует правки ucode, непереносимой
   между ревизиями кремния; диагностика вынесена в fw.
5. **Планируемый доступ в том же образе.** В 4.1 рядом присутствуют `fw_scheduled_dti.cpp`
   (разбор ESE — стандартные аллокации DTI 802.11ad,
   [ESE.md](https://github.com/klukonin/SparRAW-firmware/blob/main/docs/ESE.md)) и
   `sm_pring_connectivity.cpp`. В 6.2 добавлено фиксированное расписание
   ([6.2/docs/FIXED-SCHED.md](https://github.com/klukonin/SparRAW-firmware/blob/main/6.2/docs/FIXED-SCHED.md)),
   линия Terragraph (10.x) ушла в полный TDMA без рандомизатора. Механизм распределённого
   биконинга при этом сохранён в 4.1, 5.2, 6.2 и wil6436 7.5; в таблице строк fw 6.2 нет
   части диагностики (см. [WILOCITY-DORMANT-MESH.md](WILOCITY-DORMANT-MESH.md#версии-корпуса)).

## Патент и противоречие с физикой

Схема патента US8520648B2 — CSMA для маяков: выждать случайную задержку, проверить, не
принят ли чужой маяк, и передавать только при его отсутствии. Соответствие коду —
[BEACONING.md §4.3](https://github.com/klukonin/SparRAW-firmware/blob/main/docs/BEACONING.md#43-сверка-с-патентом-us8520648b2).
В направленном 60 ГГц условие «принят ли чужой маяк» зависит от того, наведён ли приёмный
луч на соседа: вне луча маяк не детектируется, избежание столкновений теряет основание.
Отсюда патологии, видимые в счётчиках: `bad_beacons_detector` с порогами,
`bcons_atim_fail_vec`, `false_alarms.total_fa`, NAV/backoff.

Родственная заявка US20140169288A1 (Cordeiro, Intel) предполагала три архитектуры DMG:
Infrastructure BSS, IBSS и PBSS. Экосистема 802.11ad сошлась на PBSS как одноранговом
эквиваленте, затем многоузловые сети ушли в планируемый TDMA (Terragraph); ветка IBSS для
DMG осталась рудиментом — ABI и логика на месте, включения нет.

## Неинженерные факторы

Код распределённого режима в 4.1 доведён до полной полевой телеметрии
(`bad_beacons_detector` с порогами, `bcons_atim_fail_vec`, `availability_vec`, счётчики
NAV/backoff/FA) — такую диагностику строят при доводке продукта, а не на макете.

* Патент принадлежит Intel (подача 2010-06-14, переуступка от Cordeiro 2010-06-22, выдан
  2013-08-27, расчётный срок до 2031); статус по Google Patents — «Expired — Fee Related»
  (пошлины не уплачены, прекращён досрочно). Intel свернул 60-ГГц направление. Статус по
  Google Patents — индикатор, не юридическое заключение; у семейства возможны живые
  продолжения и зарубежные аналоги.
* Wilocity после покупки Qualcomm вела свою линию; Terragraph выбрал TDMA.

Причины композитные:
* технические — проверка «услышал чужой маяк» ломается направленностью, потолок 8 соседей,
  логика в ucode;
* правовые и деловые — чужой патент, уход держателя с рынка, отсутствие продукта-носителя.

Планируемый доступ выиграл и по существу (детерминизм на mmWave), но распределённая схема
не опровергнута — она не доведена. С тех пор изменились управление лучом, 802.11ay и
доказанный многоузловой 60 ГГц.

## Следствия для Sparrow

* Реалистичная топология на Sparrow без правки ucode — один PBSS: PCP-якорь и прямые
  линки данных между членами ([WILOCITY-DORMANT-MESH.md](WILOCITY-DORMANT-MESH.md#тракт-данных-pbss--прямой-драйвер),
  [PBSS-ON-HW.md](PBSS-ON-HW.md)).
* Распределённый режим требует принять его родовые ограничения (8 соседей, потери маяков).

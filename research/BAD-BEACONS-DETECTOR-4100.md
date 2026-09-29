# Детектор плохих маяков: решающее правило (4.1.0.1000)

Детектор `bad_beacons_detector` — часть сопровождения связи `conn_mgr`: на каждый
CID считает подряд идущие «плохие» интервалы маяка и при достижении порога
инициирует разрыв. Место детектора в link maintain, его включение, пороги из
WMI, связь с отчётом MAC_MON —
[docs/MLME.md](https://github.com/klukonin/SparRAW-firmware/blob/main/docs/MLME.md),
[docs/L2-MANAGER.md](https://github.com/klukonin/SparRAW-firmware/blob/main/docs/L2-MANAGER.md),
[docs/BEACONING.md §8](https://github.com/klukonin/SparRAW-firmware/blob/main/docs/BEACONING.md#8-отчёт-mac_mon-обе);
раскладка отчёта MAC_MON 4.1 (0x854de0…) —
[4.1/docs/UCODE-BEACON.md §5.2](https://github.com/klukonin/SparRAW-firmware/blob/main/4.1/docs/UCODE-BEACON.md);
порог 20 и сравнение с dot11MaxLostBeacons —
[docs/STANDARD-MAPPING.md](https://github.com/klukonin/SparRAW-firmware/blob/main/docs/STANDARD-MAPPING.md).

## `bad_beacons_detector__handle_bcons_info` fw 0x8caeb4 **[код]**

Вызывается из `conn__feed_bcons_info`; `lm` — запись детектора CID (из
`bad_beacons_detector__is_bad`, при нуле или `lm+0x08 == 0` выход).

```c
cid  = *(u8*)lm;
info = *(u8**)(lm + 4);
valid = *(u8*)0x854e0a;                 /* MAC_MON: маяк принят за BI            */
if (valid == 0)                     bad = 1;
else if (*(u8*)0x854e00 == 0)       bad = 0;      /* rx snr valid == 0: SNR не проверяется */
else bad = (*(s32*)(info + 0x20 + cid*0x24) >= *(s32*)0x854e04);
                                                  /* порог SNR CID >= rx snr BI            */
cnt = (u8*)(info + 0x1e4 + cid);
*cnt = bad ? *cnt + 1 : 0;
if (*cnt > *(u32*)(info + 0x1c + cid*0x24))       /* bad_beacons_num_threshold CID */
    LOG("Link-Loss detected for cid=%d. Initating disconnect flow."), conn__disconnect(...);
```

Печатает `cid | valid_bcon_received | bti_max_bcon_rx_snr` и
`snr_valid | bad_bcons_cnt | bad_beacons_num_threshold`.

* Интервал «плохой», если маяк за BI не принят, либо принят, SNR валиден и не
  выше порога CID.
* Пороги — в записи на CID с шагом 0x24 (`+0x1c` число интервалов, `+0x20`
  порог SNR); счётчики — байтовый массив `+0x1e4 + cid`.
* Массивы рассчитаны на 8 CID (ассерт `cid > 7` в `bad_beacons_detector.cpp`,
  строки 0x31/0x3c) — тот же потолок 8 пиров, что у AW, ATIM и таблицы
  обнаружения ([DISCOVERY-PEER-TABLE-4100](DISCOVERY-PEER-TABLE-4100.md)).
* 0x854e00, 0x854e04, 0x854e0a — поля отчёта MAC_MON (`rx snr valid`,
  `rx snr`, `detected`), а не конфигурация детектора.

## Следствие для распределённого режима

В направленном 60 ГГц маяки соседа без согласованного луча теряются; детектор,
охраняющий линк STA↔PCP, в распределённой схеме считает такие интервалы
плохими и рвёт связь по порогу — конкретный механизм «хронической потери
маяков». Приёмная сторона MAC_MON (`detected`) при этом работает штатно
([docs/BEACONING.md §8](https://github.com/klukonin/SparRAW-firmware/blob/main/docs/BEACONING.md#8-отчёт-mac_mon-обе)).

## Не установлено

* Полная раскладка записи CID (`+0x1e0` и прочие поля).

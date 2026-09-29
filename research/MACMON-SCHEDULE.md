# Расписание интервала маяка по отчёту MAC_MON: измерения на стенде

Измерения окон BTI/AW/DTI по штатному отчёту `MAC_MON` лога прошивки 4.1.0.1000, без патчей.
Устройство отчёта (публикация ucode, событие 0x17, обработчик fw) —
[docs/BEACONING.md §8](https://github.com/klukonin/SparRAW-firmware/blob/main/docs/BEACONING.md);
раскладка блока счётчиков 0x854de0 с писателями и читателями полей —
[4.1/docs/DASHBOARD.md §7](https://github.com/klukonin/SparRAW-firmware/blob/main/4.1/docs/DASHBOARD.md);
чтение лога с хоста — [OBSERVABILITY.md](OBSERVABILITY.md).

Метки: **[железо]**, **[код]**. Узел AP, один подключённый клиент.

## 1. Вид отчёта

```
MAC_MON [BTI] | start time 0x000000004ebc3ff9 | duration 1364
MAC_MON [NNL] | tx bcon 63 | rx bcon 0 | detected 0
MAC_MON [NNL] | rx snr valid 0 | rx snr -256
MAC_MON [NNL] | bcon bitmap 0x00000000 00000000
MAC_MON [AW ] | start time 0x000000004ebc454d | duration 1004
MAC_MON [NNL] | backoff 1 | rx atim 0 | bcons_atim_fail_vec 0x0
MAC_MON [DTI] | start time 0x000000004ebc4939 | duration 100032
```

## 2. Длительности окон **[железо]**

12 подряд идущих интервалов (мкс):

| окно | мин | макс | среднее |
|---|---|---|---|
| BTI | 1364 | 1954 | 1561 |
| AW | 1003 | 1006 | 1004 |
| DTI | 99 442 | 100 032 | 99 835 |

Сумма трёх окон в каждом интервале — ровно 102 400 мкс (BI = 100 TU): длительности в
микросекундах, окна замощают интервал без зазора. AW постоянно (~1 мс, разброс 3 мкс). BTI
двумодален: 1365 либо 1951 мкс (+585 мкс) при неизменном `tx bcon = 63`.

Станция отчитывается о той же структуре интервала (включая двумодальность BTI 1363/1951 мкс):
`tx bcon 0 | rx bcon 64 | detected 1` — она синхронизирована с чужим интервалом.

## 3. Период длинного BTI — байт +0x04 блока конфигурации BI **[железо]**

Длинный BTI приходит строго периодически: на стоке из 83 интервалов разрывы между длинными —
ровно 4. Период задаёт байт **+0x04** блока конфигурации BI (0x80143c, host 0x94143c; его читает
`bi_cfg__tick_counters` как перезарядку счётчика). Запись `mem_write` на живом узле:

| значение | интервалов | длинных | один длинный на | разрывы |
|---|---|---|---|---|
| 4 (сток) | 83 | 21 | 4,0 | все ровно 4 (7 наблюдений) |
| 8 | 82 | 12 | 6,8 | отрезки коротки, разрыв 8 не наблюдается |
| 2 | 82 | 41 | 2,0 | 25 разрывов ровно по 2 (`L s L s …`) |

Прибавка длинного BTI — окно A-BFT; его длина задаётся не `abft_len` драйвера (подъём до 6 с
перезапуском hostapd длину не меняет), а числом слотов `cfg[+0x0a]` и длиной слота из блока
конфигурации ucode (запись 0x801b04 меняет длину линейно). В логе прошивки при длинных BTI строк
про A-BFT, discovery или sweep нет. Раскладка блока —
[4.1/docs/BEACON-CONFIG.md](https://github.com/klukonin/SparRAW-firmware/blob/main/4.1/docs/BEACON-CONFIG.md);
сводка рычагов — [docs/BEACONING.md §9.4](https://github.com/klukonin/SparRAW-firmware/blob/main/docs/BEACONING.md);
отдельно по окну A-BFT — [ABFT-WINDOW.md](ABFT-WINDOW.md).

## 4. Учёт маяков соседей

* Прямое чтение `blob_fw_peri` совпадает с печатью лога: AP — байты 0x854e08…0x854e0a = 63, 0, 0
  (`tx bcon 63 | rx bcon 0 | detected 0`); станция — 0, 64, 1 **[железо]**.
* `bcon bitmap` (+0x04/+0x08, 64 бита) на обоих узлах всегда нулевая; писателя у поля нет
  ни в fw, ни в ucode **[код]**.
* Детектор потери связи `bad_beacons_detector__handle_bcons_info` 0x8caeb4 карту не читает
  **[код]**:
  ```c
  good = detected && snr_valid && (snr > threshold[cid]);   /* snr = 0x854de0+0x24 */
  if (good) bad_bcons_cnt[cid] = 0; else bad_bcons_cnt[cid]++;   /* ctx + 0x1e4 + cid */
  if (bad_bcons_cnt[cid] > limit[cid])                      /* ctx + 0x1c + cid*0x24 */
      conn__disconnect(cid, 1, 1);   /* «Link-Loss detected for cid=%d» */
  ```
  (порог и лимит — слова с шагом 0x24 на cid от `ctx+0x20` и `ctx+0x1c`; по листингу 0x8caeb4).

## 5. Блок конфигурации BI у станции **[железо]**

У станции все 32 байта по 0x801438 и производные 0x801afc/0x801b00/0x801b04 нулевые — команду
настройки интервала (LMAC 0x0b) она не получала. Чтобы станция маячила, недостаточно выставить
`bi_mode`: блок надо заполнить целиком.

## Не установлено

* Кто и при каких условиях заполнял бы `bcon bitmap`.

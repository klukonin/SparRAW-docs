# Биты событий ucode 4.1 (r42, r54), команда MAC 0x27 и примитивы ожидания

Дополнение к документации прошивки: полная таблица битов EVENT1 (r42) и EVENT4 (r54),
пять примитивов ожидания ucode 4.1, раскладка маски команды MAC 0x27 и опкоды GP-таймеров
с обоснованием. Имена битов — из регистрового файла `MSXD_LR_RGF` пака 11ad
(`ucode_image_globals.xml`, выгрузка — `4.1/ref/MSXD-LR-RGF.txt` в SparRAW-firmware).
Адреса — ucode 4.1.0.1000.

Уже описано в документации прошивки:
* ожидания внутри развёртки маяка, итог «168 мест ожидания, ни одно не ждёт `tsf_event`» —
  [BEACONING.md §3.3](https://github.com/klukonin/SparRAW-firmware/blob/main/docs/BEACONING.md#33-ожидания-внутри-развёртки-код),
  разбор причин немой рандомизации — [§9.2](https://github.com/klukonin/SparRAW-firmware/blob/main/docs/BEACONING.md#92-рандомизатор-маяк-не-сдвигает-41-железо--код);
* квитирование r54 командой 0x49, коды GP-таймеров 0x4f..0x54, группы 0x67/0x68 —
  [MAC-COMMANDS.md §7.6](https://github.com/klukonin/SparRAW-firmware/blob/main/docs/MAC-COMMANDS.md#76-gp-таймеры-и-r54-0x49-0x4f0x54-0x670x68);
* бит 30 r41 (`QSET_1_MASK_VECTOR`) = «маяк в очереди 30» —
  [BEACONING.md §3.2](https://github.com/klukonin/SparRAW-firmware/blob/main/docs/BEACONING.md#32-ворота-передачи-маяка-обе-код)
  (развёртка кладёт маяк в очередь 30: `mov r0,0x1e` @0x92403e; «есть трафик, кроме маяка» —
  `bmsk.f 0,r41,0x1d` в `qset1_mask__any_pending` и `bclr.f 0,r41,0x1e` в `background_bi_step`);
* полные переписи мест ожидания — `4.1/ref/WAIT-SITES-UC.txt` (через примитивы) и
  `4.1/ref/WAIT-SITES-INLINE.txt` (встроенные самоциклы), утилита `tools/wait_sites.py`.

## r42 — EVENT1_STATUS **[пак + код]**

| бит | имя | где проверяется в ucode 4.1 |
|---|---|---|
| 0 | `sifs_event` | `rx_flow__run` |
| 1 | `slot_event` | развёртка маяка (маска 0x4002), `txss__tx_sector_frame`, `txrx_common_helper` |
| 2 | `start_sifs_rifs_bifs_event` | — |
| 3 | `cfg_event` | — |
| 4 | `bi1_event` | `l1_task` (TBTT) |
| 5 | `bi2_event` | `bti__bi2_event_step` (маска 0x1020) |
| 6 | `tsf_event` | нигде |
| 7 | `txop_event` | развёртка маяка (0x924168, 0x9241b8), `rx_flow_step`, `tx_mtp_queue_gate`, `cf_end_flow` |
| 8 | `slot2_event` | `sxd_hal__wait_slot2_busy2` (0x4100) |
| 9 | `gp_timers_event` | `rx_flow`; добавляется примитивами при aux ≠ 0 (ниже) |
| 10 | `plcp_valid_event` | — |
| 11 | `sfd_sync_event` | `rx_flow__run` (0x6800) |
| 12 | `rx_frame_event` | `abft_responder_worker`, `brp_flow_step` |
| 13 | `cca_event` | `rx_flow__run` (0x6800) |
| 14 | `busy2_event` | развёртка маяка, `bcon_txss_sweep_step`, `internal_tx_step`, `bi_manager_rx_helper`, `cf_end__tx_flow` |
| 15 | `tx_end_event` | 17 голых спинов |
| 16 | `ppdu_report_event` | `rx_flow`, `rx_funcs__handle_ppdu_report`, `abft_responder_substep`, `bti_worker_bi2_step` |
| 17 | `queue_set_event` | — |
| 18 | `backoff_event` | `l1_task`, `txss`, `background_bi_step` |
| 19 | `busy_event` | там же; `bi_manager__run_rx_flow_by_kind` (путь kind = 1) |
| 20..23 | `comb_event1..4` | — |

Смысл имён сходится с местами проверки независимо в шести случаях (TBTT в `l1_task`,
приём отчёта PPDU в `rx_flow`, кадр SSW у ответчика A-BFT и др.). Живое значение r42 на
AP: `0x00f2c3bf` / `0x00f2c1bf` (различие в бите 9).

## r54 — EVENT4_STATUS **[пак + код]**

| бит | имя | бит | имя |
|---|---|---|---|
| 0 | `txss_end_ind` | 10 | `backdoor_toggle_ind` |
| 1 | `rxss_bf_metric_ind` | 11 | `backdoor_lock_ind` |
| 2 | `rxss_measurement_ind` | 12 | `service_period_1_ind` |
| 3 | `rxss_timeout_ind` | 13 | `service_period_2_ind` |
| 4 | `service_period_0_ind` | 14 | `service_period_3_ind` |
| 5 | `rxss_last_cycle_ind` | 15 | `ext_gp_0_1_2_end_ind` |
| 6 | `gp0_end_ind` | 16 | `ext_gp_3_4_5_end_ind` |
| 7 | `gp1_end_ind` | 17 | `ext_gp_6_7_8_9_end_ind` |
| 8 | `gp2_end_ind` | 18 | `brp_tx_ind` |
| 9 | `nav_ind` | | |

Опросы без ожидания: `bti_bi2__poll_gp0_end` @0x929020 (`r54 & 0x40`) и
`bti_bi2__poll_gp2_end` @0x923dd4 (`r54 & 0x100`), оба из `bti_transmitter_bi2_flow`.
Развёртка секторов (`txss__*`) ждёт пары «`busy` в r42 + `txss_end`/`gp*_end` в r54».

## Примитивы ожидания **[код]**

| примитив | адрес | регистр | действие |
|---|---|---|---|
| `uc_sleep_until_event(mask, aux)` | 0x93187c | r42 | взводит движок событий, пишет маску в 0x886d3c, `sleep` |
| `uc_wait_event_busy(mask, aux)` | 0x931808 | r42 | то же, вместо `sleep` — цикл `tst r42,mask; beq` |
| `uc_spin_until_event` | 0x9317fc | r42 | голый спин `tst r42,mask; jne [blink]; b .`, ничего не программирует |
| `uc_spin_until_event4` | 0x9317f0 | r54 | голый спин по r54 (зовут `abft_responder_worker`, `bi_dti__sleep_and_resume`, `cf_end__pulse_883150`, `mac_mode_kick`) |
| `uc_wait_event1_or_event4(m42, m54, cmd)` | 0x9318f4 | r42 и r54 | ждёт любой из двух масок; при cmd ≠ 0 предварительно шлёт команду MAC `0x23010000 \| cmd` (7 мест, в том числе `internal_tx__await_rx_frame_*`, `mac_mode_brp_step`) |

`uc_sleep_until_event`:

```
tst   r42,r0                ; событие уже есть — выход
bne   эпилог
breq  r1,0,0x9318b8         ; aux == 0: обход программирования движка и бита 9
  ... st.ab (r1 | 0x23010000),[r25]
  ... st.ab 0x02000200,[r25]
  bset r13,r13,0x9          ; 0x9318b4: бит 9 gp_timers_event
0x9318b8: st r13,[0x886d3c] ; регистр разрешения пробуждения
  ... st.ab 0x21000002,[r25]
st.as r13,[gp,-0x17]        ; 0x8004c4: ожидаемая маска
st.as ilink2,[gp,-0x27]     ; 0x800494: адрес возврата из прерывания
sleep 0
st    0,[0x886d3c]
```

**Бит 9 добавляется к маске только при aux ≠ 0.** При aux = 0 движок событий не
программируется и страховочного таймаута нет; так же устроен `uc_wait_event_busy`
(`beq.d` на 0x931824 обходит `bset` на 0x931850). Штатные вызывающие используют aux = 0xa
(`cf_end__tx_flow`, бит 14) и 0 (`rx_flow__run`, бит 16). `uc_sleep_until_event(1<<9, 0)`
усыпляет ucode без возврата **[железо]**: при aux = 0 движок событий не взведён. Вызов с
aux = 0 и маской бита, который может не прийти (например, `1<<6`), оставляет ucode без
страховки. Формулировка «бит 9 добавляется всегда» в BEACONING.md §9.2 этим уточняется.

Сводка масок у вызовов (по `WAIT-SITES-UC.txt`):

| примитив | вызывающий | маска | биты |
|---|---|---|---|
| busy | `txrx_common_helper`, развёртка маяка, `txss__tx_sector_frame` | 0x4002 | slot, busy2 |
| busy | `bti_worker_bi2_step`, `abft_responder_substep`, `rx_flow_step`, `rx_flow__handle_frame`, `rx_funcs__handle_ppdu_report` и др. | 0x10000 | ppdu_report |
| busy | `rx_flow__run` | 0x6800 | sfd_sync, cca, busy2 |
| busy | `sxd_hal__wait_slot2_busy2` | 0x4100 | slot2, busy2 |
| busy | `sxd_hal__wait_slot2_busy` 0x931e74 | 0x80100 | slot2, busy |
| busy | `bti__bi2_event_step` | 0x1020 | bi2, rx_frame |
| busy | `tx_flows__program_mac_tx`, `uc_wait_two_events` | 0x4000 | busy2 |
| sleep | `cf_end__tx_flow` (дважды) | 0x4000 | busy2 |
| sleep | `rx_flow__run` | 0x10000 | ppdu_report |

Голых спинов по r42 — 26: 17 ждут `tx_end`, 5 — `busy2`, 2 — `txop`, по одному `slot` и
`ppdu_report`. Встроенные самоциклы (102): r45 `MTP_TX_PERMISSION_RESP` бит 31 — 40,
r43 `MTP_QUERY_RESPONSE_1` бит 31 — 28, r47 `DIRECT_TX_CMD_STATUS` (бит 0 `cmd_ready`,
бит 17) — 16, r53 `BAP_IF_0` — 10, r40 `BACKOFF_IFS_STATUS` — 4, r42 бит 7 `txop` — 2,
r54 бит 6 `gp0_end` — 1, r49 `EVENT_ENGINE2_STATUS` — 1.

Два дефекта разбора масок, закрытые в `wait_sites.py`: маску часто дописывают в слоте
задержки вызова (`bl.d wait; bset_s r0,r0,0xe`) — слот исполняется после перехода, и
разбор назад от вызова его не видит; сдвиг после загрузки (`mov_s r0,0x41; asl r0,r0,0x8`
= 0x4100) при разборе назад встречается раньше значения и должен накапливаться.

## Команда MAC 0x27 — маска движка событий **[пак + код]**

Раскладка слова (`dump_only__EVENT_ENGINE_4_STATUS_7_REG_st` пака): `[31:24]` — код,
`[23:0]` — маска:

```
b0..b9  gp0..gp9_expired_event
b10     rx_ppdu_timeout        b14  rx_frame
b11     early_ppdu_report      b15  tsf_nid_event
b12     early_mpdu_rep         b16  bi1_nid_event
b13     abort_flush            b17  bi2_nid_event
```

`uc_wait_two_events` @0x9316b4 строит маску из двух номеров событий:

```
9316ba  asl  r2,0x1,r1        ; 1 << номер1
9316c2  bset r2,r2,r0         ; | 1 << номер0
9316c6  xor  r0,r2,0x7f       ; дополнение по 7 битам
9316ce  asl_s r0,r0,0x8       ; сдвиг на 8
9316d0  or   r1,r0,0x27000000
```

Номера 0..6 ложатся в разряды 8..14 (`gp8`, `gp9`, `rx_ppdu_timeout`,
`early_ppdu_report`, `early_mpdu_rep`, `abort_flush`, `rx_frame`); номер 6 — `rx_frame`.
`tsf_nid_event` (разряд 15) этим примитивом недостижим. Образец 0x9245b2: `0x27004000`
(разряд 14, `rx_frame`), затем `0x02004000` той же маской (0x02 маршрутизирует событие),
затем ожидание r42 бит 14. Все 18 констант `0x27xxxxxx` ucode 4.1 декодируются
осмысленно. С учётом 15 мест выбора события через 0x27 всего 183 места ожидания
(81 через примитивы + 102 встроенных самоцикла), ни одно не ждёт события TSF.

## Опкоды GP-таймеров: обоснование **[код]**

По `bti_transmitter_bi2_flow` @0x923c74 и соседям:

| опкод | смысл | подтверждение |
|---|---|---|
| `0x49 <маска>` | квитирование r54: 0x40 `gp0_end`, 0x80 `gp1_end`, 0x100 `gp2_end` | четыре независимые тройки |
| `0x4f <D>` / `0x52 <ctl>` | длительность / управление GP0 | 0x923ce0 + 0x923cf4 + 0x923d08, опрос r54 бит 6 |
| `0x50 <D>` / `0x53 <ctl>` | GP1 | `bti_tx__program_bi2_regs`: `0x50 \| (0x3de − a)`, `0x5300070e`, `0x49000080` |
| `0x51 <D>` / `0x54 <ctl>` | GP2 | 0x92b58c: `0x51 \| …`, `0x5400070e`, `0x49000100`, опрос r54 бит 8 |

Хвост развёртки 0x9241d8…0x924300 (окно A-BFT): GP0 = `[0x801b04] + 495`,
GP1 = `[0x801afc] + 165`, GP2 = 0x339 = 825 — кратные 165 (3×, 1×, 5×): единица таймера —
**165 тактов на микросекунду** (такт MAC 165 МГц).

## Пересчёт адресов пака 11ad

Глобалы ucode в XML пака даны в системном пространстве `0xa78000+`, ucode адресует их как
`0x800000+`: **вычитать 0x278000**. Проверено на образе 7.5.0.77 из пака:
`g_bi_cfg_params` XML 0xa7c1d8 → `ld r8,[0x8041d8]`; `g_respect_neighbor_bcon`
XML 0xa7858c → `ldb r3,[0x80058c]`.

# Типы сети ADHOC в пути connect и старта PCP (6.2.0.1000)

Как прошивка 6.2.0.1000 обрабатывает `WMI_CONNECT_CMD.networkType = ADHOC (2)` и
`ADHOC_CREATOR (4)`, и сверка с другой сборкой 6.2 (UBNT). Все выводы — **[код]**
по двум образам 6.2: из пакета RouterOS (адреса ниже) и UBNT.

Цепочка `networkType → bss_mode → bi_mode` в слое маяка и таблица типов
`mid__type_of` 4.1 —
[docs/BEACONING.md §2.2](https://github.com/klukonin/SparRAW-firmware/blob/main/docs/BEACONING.md#22-кто-ставит-bi_mode-обе-код),
[docs/ROLELESS-LINK.md §6](https://github.com/klukonin/SparRAW-firmware/blob/main/docs/ROLELESS-LINK.md);
команда LMAC `trigger_bf` —
[docs/LMAC-PROTOCOL.md](https://github.com/klukonin/SparRAW-firmware/blob/main/docs/LMAC-PROTOCOL.md);
номера WMI — [docs/WMI.md](https://github.com/klukonin/SparRAW-firmware/blob/main/docs/WMI.md).

## 1. Путь connect и диспетчер типа сети

```
WMI_CONNECT
  → 0x8ec434                 обработчик команды connect
      → validate_connect 0x8fb348   проверяет состояние линка (+0x50, +0x2ac, флаг +10);
                                    networkType не фильтрует
      → l2_mgr__connect 0x8f1e18
          networkType = *(u8*)cmd   (первый байт WMI_CONNECT_CMD)
          → 0x8e696c(link_ctx, networkType)
```

Второй вход: `connect_multi_omni_cb` 0x8c7300 → `l2_mgr__connect`.

Диспетчер 0x8e696c (в сборке UBNT — 0x8e60e4):

```c
void nettype_dispatch(link_ctx, networkType) {
  *(u32*)(link_ctx + 0x14) = networkType;
  bss_cfg(link_ctx + 0x48, networkType, 0);          /* безусловно */
  switch (*(link_ctx + 0x14)) {
    case 1:    log("INFRA_STA");                 break;
    case 2:    log("WMI_NETTYPE_ADHOC");         break;
    case 4:    log("WMI_NETTYPE_ADHOC_CREATOR"); break;
    case 0x10: log("INFRA_AP");                  break;
    case 0x20: log("P2P");                       break;
    default:   log("Error Illegal Network Type"); assert(0x11ec); return;
  }
}
```

* ADHOC (2) и ADHOC_CREATOR (4) — принятые ветки; прочие значения, кроме 1, 0x10,
  0x20, ведут в ассерт. Раннего фильтра по типу сети в пути нет.
* Тип сохраняется в `link_ctx+0x14` и попадает в кадр: построитель 0x8cccbc
  копирует `*(link+0x14)` в дескриптор (лог `<network_type:%d>`).
* `bss_cfg` 0x8e73a0 строит фиксированный набор DMG-IE; по типу сети в видимом
  теле не ветвится. Сам `switch` выбирает только лог-строку.
* Сборка UBNT: двойник 0x8e60e4 структурно совпадает — тот же `st +0x14`, вызов
  `bss_cfg(+0x48)`, та же лесенка 4/1/2/0x10/0x20 и тот же номер ассерта 0x11ec;
  различаются только адреса лог-строк.

## 2. Роль в слое маяка определяет `bss_mode`

Предикат роли 0x8da488 (`mid__state_is_2_or_3`): `vif+0x50 ∈ {2, 3}`. Тот же
признак проверяют `assoc_ready_check_trig` 0x8c3db0 и DMG Information flow
0x8d86f0 («Not PCP AP» при ложном предикате).

`l2_mgr__pcp_start_flow` 0x8e0778: `bss_mode = (networkType == 0x10) ? 3 : 2`
(запись `m_bss_mode` — `bss_set_mode` 0x8c4eec);
только P2P (0x20) дополнительно вызывает 0x8d8750. ADHOC_CREATOR (4) не
отвергается и получает `bss_mode = 2`, предикат роли истинен, выполняется весь
путь PCP:

```
WMI_PCP_START 0x8ec840 → l2_mgr::pcp_start 0x8f706c → 0x8e0778
  → pcp_start 0x8f7110 → 0x8c2778 → Build DMG Beacon 0x8e9ccc
```

`bcn_tx_init_ss_params` 0x8c4664 кладёт в маяк набор TX-секторов. Построение и
передача маяка по типу сети и по `m_is_pcp_ap` не гейтятся.

`m_is_pcp_ap` (`gp+0xce`) ставит `update_pcp_ap` 0x8eac58 по состоянию автомата
(10 → TRUE, 12 → FALSE) — флаг калибровки, а не гейт маяка.

Следствие для драйвера: создатель ячейки шлёт `WMI_PCP_START(ADHOC_CREATOR)`,
присоединяющийся — `WMI_CONNECT(ADHOC)`; разделение ролей то же, что AP/STA.

## 3. Инициирование BF с хоста без роли

`WMI_SLS` 0x8fc11c → `lmac_if_trigger_bf` 0x8dc228: BF к произвольному пиру —
`bf_cmd_type = 0` по MAC (поиск соединения 0x8cc024), `1` по CID; в ucode уходит
команда LMAC 0x09 `{1, bf_cmd_type, cid}`. Привязки к роли PCP/AP нет. Результат
BF доставляется и в `find_main_sm::bf_ntf`, и в `conn_main_sm::bf_ntf`; строки
`BF_SUBFLOWS_DONE_DTI_TXSS` и `BF_SUBFLOWS_DONE_ABFT_TXSS` подтверждают обе
стороны развёртки.

## 4. Возможности и WMI_READY

* `WMI_FW_VER` 0x9004 (0x8dd130): база 0x3c + массив битовых масок возможностей
  (число масок — 0x8f3560; лог «Number of FW capability bit masks»,
  «bitmask %d = 0x%08X»); формат совпадает с `struct wmi_fw_ver_event` мейнлайна.
* `WMI_READY` 0x1001 (0x8f5d60), размер 0x14: `sw_version = 1000`, MAC,
  `numof_additional_mids` (0x8f41ac).
* Бита возможности «adhoc» нет: тип сети выбирает драйвер, прошивка его
  принимает. Нумерация команд и событий мейнлайновая; проверки версии
  интерфейса, отвергающей драйвер, нет.

## Не установлено

* Где тип сети ADHOC даёт отличное от PBSS поведение ниже диспетчера.
* Доходит ли создатель ячейки ADHOC_CREATOR до состояния 10 автомата
  (флаг калибровки `m_is_pcp_ap`).

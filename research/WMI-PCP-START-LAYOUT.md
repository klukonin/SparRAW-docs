# WMI_PCP_START в прошивке 4.1: раскладка и поле abft_len

Как прошивка 4.1.0.1000 разбирает `WMI_PCP_START_CMDID` (0x918) драйвера и почему поле
`abft_len` не влияет на окно A-BFT. Ветки диспетчера —
[WMI.md](https://github.com/klukonin/SparRAW-firmware/blob/main/docs/WMI.md)
(0x918 → `wmi_handler_pcp_start`, 0xf `WMI_BCON_CTRL_CMDID` → `wmi_handler_bcon_ctrl`);
измерение `abft_length` на железе —
[STANDARD-MAPPING.md](https://github.com/klukonin/SparRAW-firmware/blob/main/docs/STANDARD-MAPPING.md#abft_length-на-длину-окна-не-влияет-железо).

## Две структуры драйвера

```c
struct wmi_pcp_start_cmd {        /* 0x918, 16 Б */
    __le16 bcon_interval;         /* +0x00 */
    u8 pcp_max_assoc_sta;         /* +0x02 */
    u8 hidden_ssid;               /* +0x03 */
    u8 is_go;                     /* +0x04 */
    u8 edmg_channel;              /* +0x05 */
    u8 raw_mode;                  /* +0x06 */
    u8 reserved[3];               /* +0x07 */
    u8 abft_len;                  /* +0x0a */
    u8 ap_sme_offload_mode;       /* +0x0b */
    u8 network_type;              /* +0x0c */
    u8 channel;                   /* +0x0d */
    u8 disable_sec_offload;       /* +0x0e */
    u8 disable_sec;               /* +0x0f */
};
struct wmi_bcon_ctrl_cmd {        /* 0xf, 20 Б */
    __le16 bcon_interval;         /* +0x00 */
    __le16 frag_num;              /* +0x02 */
    __le64 ss_mask;               /* +0x04 */
    u8 network_type;              /* +0x0c */
    u8 pcp_max_assoc_sta;         /* +0x0d */
    u8 disable_sec_offload;       /* +0x0e */
    u8 disable_sec;               /* +0x0f */
    u8 hidden_ssid;               /* +0x10 */
    u8 is_go;                     /* +0x11 */
    u8 abft_len;                  /* +0x12 */
    u8 reserved;
};
```

## `wmi_handler_pcp_start` @0x8e79dc — перекладка в BCON_CTRL **[код]**

Обработчик 0x918 собирает на стеке `wmi_bcon_ctrl_cmd` и передаёт её обработчику 0xf:

| поле BCON_CTRL (стек) | источник в PCP_START |
|---|---|
| +0x00 `bcon_interval` | +0x00 |
| +0x02 `frag_num`, +0x04 `ss_mask` | 0 |
| +0x0c `network_type` | +0x0c |
| +0x0d `pcp_max_assoc_sta` | +0x02 |
| +0x0e `disable_sec_offload` | +0x0e |
| +0x0f `disable_sec` | +0x0f |
| +0x10 `hidden_ssid` | +0x03 |
| +0x11 `is_go` | +0x04 |

`channel` (+0x0d) пишется в глобал 0x8033dc; затем `bl wmi_handler_bcon_ctrl`.
`abft_len` (+0x0a), `edmg_channel`, `raw_mode`, `ap_sme_offload_mode` не копируются.

## `wmi_handler_bcon_ctrl` @0x8e7924 **[код]**

```c
u16 bi = *(u16*)(cmd + 0x00);
if (bi == 0) { log("BCON CMD: PCP Stop mid=%d"); l2_mgr__pcp_stop(mid); }
else {
    v = cmd[0x0d];
    if (1 < v && v < 9) gp[0x268] = v;          /* max assoc sta, диапазон 2..8 */
    sec = (cmd[0x0f] == 0);
    log("bi %d, max assoc sta %d, sec_en %d", bi, gp[0x268], sec);
    log(..., cmd[0x11]);
    gp[0x150] = bi;
    *(u32*)0x80344c = (cmd[0x0e] == 0);          /* sec offload */
    *(u32*)0x803448 = sec;
    l2_mgr__pcp_start(mid, cmd[0x0c], cmd[0x10], cmd[0x11]);  /* network_type, hidden, is_go */
}
```

Читаются +0x00, +0x0c…+0x11 — ровно поля `wmi_bcon_ctrl_cmd`; `abft_len` (+0x12) не
читается.

## Следствие

Ни через `WMI_PCP_START`, ни через `WMI_BCON_CTRL` поле `abft_len` в прошивку 4.1 не
попадает: подъём `abft_len` с 0 до 6 с перезапуском hostapd длину окна A-BFT не меняет.
Длину окна задают поля конфигурации BI ucode
([4.1/docs/BEACON-CONFIG.md](https://github.com/klukonin/SparRAW-firmware/blob/main/4.1/docs/BEACON-CONFIG.md)).
Остальные поля `wmi_pcp_start_cmd` драйвера mainline прошивка 4.1 принимает корректно.

## Не установлено

* Строки обработчика в живом логе: он исполняется сразу после сброса прошивки, когда
  байты уровней лога ещё нулевые (за 45 снимков подряд, 5559 строк, поймана только
  остановка PCP). Проверка — `WMI_PCP_START` через debugfs `wmi_send` без сброса
  (с ненулевым `bcon_interval`: нулевой останавливает PCP).

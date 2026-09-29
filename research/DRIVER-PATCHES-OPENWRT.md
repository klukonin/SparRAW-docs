# Драйверные патчи wil6210 для OpenWrt: база, метод, первые патчи

Патчи драйвера накладываются штатным quilt поверх мейнлайновой серии пакета
`package/kernel/mac80211` OpenWrt. Полный перечень серии (900…915) —
[../chip/05-host-interface.md §5.11](../chip/05-host-interface.md); здесь — база
сборки, метод и подробности первых патчей.

## База

* OpenWrt master (2026-09, коммит 820fbf4), `package/kernel/mac80211`,
  **backports 7.2**; места правок wil6210 структурно совпадают с backports 6.18.26.
* Таргет `ipq40xx` — апстрим-поддержка устройств RouterBOARD на IPQ4019 с 60 ГГц
  (файл образов субтаргета в `target/linux/ipq40xx/image/`, общий DTS
  `qcom-ipq4019-lhgg-60ad.dts`, `DEVICE_PACKAGES += kmod-wil6210`).
* Вся серия (152 мейнлайновых патча + свои) применяется `quilt push -a` без
  отказов.

Устройства, различающиеся только board-данными:

| модель | суффикс профиля OpenWrt | board-пакет | brd |
|---|---|---|---|
| LHGG-60ad | `_lhgg-60ad` | `…-lhg` | lhg-span2.8 |
| wAP 60G | `_wap-60g` | `…-wap60g-60deg` | wap60g-60deg |
| wAP 60Gx3 | `_wap-60gx3` | `…-wap60gx3` | wap60g-omni |

Имена профилей — по соглашению OpenWrt `<вендор>_<модель>` (DEVICE_MODEL в нижнем
регистре, пробелы → дефисы).
В образах раздельные `hostapd-openssl` и `wpa-supplicant-openssl`, `wpad` нет.

## Метод

```sh
make defconfig
make -j4 package/kernel/mac80211/prepare QUILT=1 V=s   # распаковка backports
BD=build_dir/target-*/linux-*/mac80211-regular/backports-7.2
cd $BD
export QUILT_PATCHES=patches
quilt push -a                       # все мейнлайновые патчи
quilt new ath/900-<имя>.patch       # свой патч поверх серии
quilt add drivers/net/wireless/ath/wil6210/<file>
#   ...правка...
quilt refresh
cp patches/ath/900-*.patch <openwrt>/package/kernel/mac80211/patches/ath/
```

Патчи — в семействе `ath/` (`drivers/net/wireless/ath/wil6210`), нумерация с 900
(после мейнлайновых 0xx/4xx). `.quiltrc` в стиле OpenWrt:
`-p ab --no-timestamps --no-index`.

## Патчи 900–902

**900-wil6210-adhoc-iftype.** `BIT(NL80211_IFTYPE_ADHOC)` в
`wiphy->interface_modes` и запись `wil_mgmt_stypes[NL80211_IFTYPE_ADHOC]`.
Соответствие `wil_iftype_nl2wmi{ADHOC → WMI_NETTYPE_ADHOC}` в драйвере уже
есть, поэтому `iw dev X set type ibss` принимается и `__wil_up` передаёт
прошивке `networkType = ADHOC`.

**901-wil6210-join-ibss.** `.join_ibss`/`.leave_ibss` в `wil_cfg80211_ops`.
`join_ibss` поднимает сеть как создатель ячейки через
`wmi_pcp_start(vif, bi, WMI_NETTYPE_ADHOC_CREATOR, chan, …)` (ad-hoc на этом
железе — надстройка над PBSS, `vif->pbss = true`), ставит SSID (`wmi_set_ssid`),
сообщает `cfg80211_ibss_joined`; `leave_ibss` → `wmi_pcp_stop`. Гейт
`vif->mid == 0`. Поля `cfg80211_ibss_params` (ssid, bssid, chandef, ssid_len,
beacon_interval, privacy), сигнатуры операций, `cfg80211_ibss_joined` и
`wmi_pcp_start` (7 аргументов) сверены с `backports-7.2/include/net/cfg80211.h`.
Путь в прошивке — [ADHOC-CONNECT-PATH](ADHOC-CONNECT-PATH.md).

**902-wil6210-fw-name-fallback.** Драйвер выбирает имя образа по ревизии
кремния: Sparrow D0 («plus», HW version 0x2, wAP 60Gx3) просит
`wil6210_sparrow_plus.fw`, которого в linux-firmware нет → `error -2`, образ не
грузится, дальше падают WMI и hostapd; ревизия 0x1 (wAP 60G) работает. Драйвер
вендора грузит один образ на все ревизии Sparrow. Патч при `-ENOENT` повторяет
запрос с дженерик-именем:

```
wil6210_sparrow_plus.fw     -> wil6210.fw
wil6210_sparrow_plus_ftm.fw -> wil6210_ftm.fw
прочее                      -> NULL (без подмены)
```

`wil6436.fw` не подменяется (другой чип). FTM здесь — Factory Test Mode
(`MODULE_PARM_DESC(ftm_mode, " Set factory test mode")`), поэтому заводской образ
боевым не подменяется; ToF-ranging — другое (`WMI_FW_CAPABILITY_FTM`,
`WMI_TOF_FTM_PER_DEST_RES_EVENTID` 0x1995). В собранном модуле присутствует
строка `<%s> is missing, using <%s>`.

Сборка `wil6210.ko` под ipq40xx/arm_cortex-a7 (ядро 6.18.52, backports 7.2):
`nm` показывает `wil_cfg80211_join_ibss`/`_leave_ibss`.

## Порядок сборки

`tools/install` + `toolchain/install` → `target/linux/compile` (без собранного
ядра любой kmod падает сразу) → `package/…/compile` → `make`. Toolchain собирать
до компиляции пакетов, иначе сборка падает на отсутствии
`staging_dir/host/bin/openssl`. Сборка `openssl` при `-j6` иногда падает на гонке
(`.d.tmp: No such file`) — лечится повтором.

## `compat_version` при sysupgrade

Образы ipq40xx объявляют `DEVICE_COMPAT_VERSION := 1.1`
(`target/linux/ipq40xx/image/Makefile`, миграция swconfig → DSA). Устройство
берёт свою версию из `uci get system.@system[0].compat_version`, при отсутствии
опции — `1.0` (`base-files/lib/upgrade/fwtool.sh`). Опцию никто не записывает,
поэтому `sysupgrade` всегда требует `-n`. Решение — `files/etc/uci-defaults/99-compat-version`,
проставляющий 1.1 при первой загрузке (свежая установка уже на DSA).

Сбои драйвера и прошивки на стенде (зависание в `wil_reset`, тихий рестарт
прошивки, `rmmod`) и патчи против них (904, 907, 908) —
[../chip/05-host-interface.md §5.10](../chip/05-host-interface.md).

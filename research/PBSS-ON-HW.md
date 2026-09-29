# Одноранговая сеть PBSS на стенде (4.1.0.1000)

802.11ad не содержит классического IBSS; одноранговая форма DMG — **PBSS** (Personal BSS)
с координатором PCP. Пара PCP + станция на прошивке 4.1.0.1000 и штатном драйвере
поднимается демонами hostapd/wpa_supplicant с `pbss=1`. Безролевой линк без PCP записями с
хоста — [ROLELESS-LINK.md](https://github.com/klukonin/SparRAW-firmware/blob/main/docs/ROLELESS-LINK.md);
PBSS на 6.2 (`ASSOC-REJECT status_code=1`) —
[6.2/docs/BENCH.md](https://github.com/klukonin/SparRAW-firmware/blob/main/6.2/docs/BENCH.md).

## Результат **[железо]**

| | |
|---|---|
| координатор (PCP) | узел B, `hostapd` с `pbss=1` |
| станция | узел A, `wpa_supplicant` с `pbss=1` |
| SSID / канал | PBSS60 / 1 (58320 МГц) |
| согласованная скорость | 2310 Мбит/с |
| пропускная способность | 988, 662, 725, 615, 593 Мбит/с (5 прогонов) |
| задержка | 0,7…1,6 мс |

Лог ядра станции:
```
wil_print_connect_params:   BSSID: <MAC PCP>
  SSID: PBSS60
  Auth Type: OPEN_SYSTEM
  PBSS: 1
wmi_evt_connect: successful connection to CID 0
```

`MAC_MON`: на координаторе `tx bcon 63`, на станции `tx bcon 0 | rx bcon 63 | detected 1`.
Маячит только координатор.

## Подъём

OpenWrt не пробрасывает `pbss` из `/etc/config/wireless`, демоны запускаются вручную.
**`wpad` останавливается заранее** — иначе netifd отбирает интерфейс и hostapd сразу
получает «Remove interface».

Координатор — конфиг, который OpenWrt генерирует для рабочей точки доступа
(`/var/run/hostapd-phy0.conf`), плюс `pbss=1`:
```
driver=nl80211
beacon_int=100
hw_mode=ad
channel=1
stationary_ap=1
interface=phy0-sta0
bssid=<MAC интерфейса>
ssid2="PBSS60"
ctrl_interface=/var/run/hostapd-pbss
wpa=0
pbss=1
```
```sh
/etc/init.d/wpad stop
ifconfig phy0-sta0 down; iw dev phy0-sta0 set type ap; ifconfig phy0-sta0 up
hostapd -B /tmp/pbss.conf
ip addr add <адрес B>/24 dev phy0-sta0
```

Станция:
```
ctrl_interface=/var/run/wpa_supplicant-pbss
network={
	ssid="PBSS60"
	key_mgmt=NONE
	pbss=1
}
```
```sh
/etc/init.d/wpad stop
ifconfig phy0-ap0 down; iw dev phy0-ap0 set type managed; ifconfig phy0-ap0 up
wpa_supplicant -B -i phy0-ap0 -c /tmp/wpa-pbss.conf
ip addr add <адрес A>/24 dev phy0-ap0
```

## Особенности

* Без остановки `wpad` hostapd получает «Remove interface 'phy0'» — внешне похоже на сбой
  PBSS.
* `AP-DISABLED` и `CTRL-EVENT-TERMINATING` в конце вывода — реакция на завершение демона
  по `timeout`, а не сбой.
* Урезанный конфиг координатора (без `ctrl_interface` и `bssid`) приводил к зависанию
  станции; с полным конфигом не повторялось.
* Единичное падение станции на первом прогоне iperf3 не воспроизвелось (четыре прогона
  подряд чистые). При подъёме PBSS драйвер трижды подряд сбрасывает прошивку (видно в
  netconsole) — вероятное окно неустойчивости.

## Чего PBSS не даёт

Распределённого биконинга нет: маячит только координатор. Механизм смены координатора
802.11ad (PCP handover) в прошивке 4.1 отсутствует; есть только метрика пригодности на
роль PCP — [WMI-PCP-FACTOR.md](WMI-PCP-FACTOR.md).

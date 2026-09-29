# Návrh monitoringu malej siete s Loki, syslog-ng a Grafana

Prototyp centralizovaného logovania a monitoringu malej SMB siete
(Fortinet, MikroTik, Cisco) na Raspberry Pi 4 / SFF PC.

## 1. Architektúra

```
┌────────────┐  syslog UDP/514   ┌────────────┐   tail    ┌──────────┐  push   ┌──────┐   query  ┌─────────┐
│ MikroTik   │ ─────────────────▶│            │──────────▶│          │────────▶│      │◀────────▶│         │
│ RouterOS   │                   │ syslog-ng  │  /var/log │ promtail │  HTTP   │ Loki │          │ Grafana │
├────────────┤  syslog TCP/601   │ (splitter) │           │          │         │      │          │ (dash-  │
│ Fortinet   │ ─────────────────▶│            │           │          │         │      │          │ boards, │
│ FortiGate  │                   └────────────┘           └──────────┘         └──────┘          │ alerts) │
├────────────┤  syslog UDP/514                                                                     └─────────┘
│ Cisco      │ ─────────────────▶ (same pipeline)
│ IOS/IOS-XE │
└────────────┘
```

Všetky komponenty bežia ako kontajnery cez `docker compose`. syslog-ng
prijíma logy, rozdeľuje ich podľa zdrojovej podsiete/hostname do
samostatných súborov per vendor → promtail ich tailuje a posiela do
Loki s labelmi (`vendor`, `host`, `job`) → Grafana slúži na
vyhľadávanie (LogQL) a alerting.

## 2. Rýchly štart

```bash
cd network-monitoring
docker compose up -d
```

- Grafana: http://<host>:3000 (admin / change_me — **zmeňte pred nasadením**)
- Loki API: http://<host>:3100
- syslog-ng: UDP/514 (MikroTik/Cisco), TCP/601 (Fortinet, spoľahlivé doručenie)

Upravte `syslog-ng/syslog-ng.conf` — podsiete vo `filter f_mikrotik`,
`f_fortinet`, `f_cisco` musia zodpovedať skutočným manažment IP vašich
zariadení (alebo použite `host()` filter podľa hostname, ak zariadenia
posielajú svoje meno).

## 3. Konfigurácia zariadení (odosielanie syslog)

### MikroTik RouterOS
```
/system logging action add name=remote-loki target=remote remote=<syslog-ng-IP> remote-port=514
/system logging add topics=info,error,warning,critical action=remote-loki
/system logging add topics=firewall action=remote-loki
/system logging add topics=vpn action=remote-loki
```

### Fortinet FortiGate (CLI)
```
config log syslogd setting
    set status enable
    set server "<syslog-ng-IP>"
    set port 601
    set mode reliable
    set format default
end
config log syslogd filter
    set severity information
end
```
(Reliable/RFC5424 cez TCP odporúčané pre FortiGate kvôli integrite
logov o VPN/policy udalostiach.)

### Cisco IOS / IOS-XE
```
logging host <syslog-ng-IP> transport udp port 514
logging trap informational
logging source-interface <mgmt-interface>
logging on
```

> Ak nemáte fyzický prístup k reálnym zariadeniam, použite virtuálne
> obrazy: MikroTik CHR (Cloud Hosted Router), Fortinet FortiGate VM
> (evaluation license), Cisco CSR1000v/IOSv v GNS3 alebo EVE-NG.

## 4. Testovacie scenáre (pre kapitolu "Testovanie a merania")

Cieľom je vygenerovať kontrolovanú sadu udalostí, overiť že sa objavia
v Loki/Grafana so správnymi labelmi, a zmerať end-to-end latenciu.

| # | Scenár | Ako vygenerovať | Čo overiť v Grafana |
|---|--------|------------------|----------------------|
| 1 | VPN nadviazanie/zrušenie (IPsec/SSL) | Pripojiť/odpojiť VPN klienta na FortiGate/MikroTik | Log s `vendor=fortinet` obsahujúci VPN event, čas medzi eventom a jeho zobrazením v Grafana |
| 2 | ACL/firewall deny | Pokus o zakázaný prístup cez firewall pravidlo | Log s `vendor=mikrotik|fortinet|cisco`, filter podľa `deny`/`drop` |
| 3 | Interface up/down | Fyzicky/administratívne shutdown rozhrania | `%LINK-3-UPDOWN` (Cisco) alebo ekvivalent u MikroTik/Fortinet |
| 4 | Autentifikačné zlyhanie | Nesprávne prihlásenie na management rozhranie | Log obsahujúci "authentication failure"/"login failed" |
| 5 | Záťažový test | `logger`/skript generujúci N správ/s | Meranie CPU/RAM syslog-ng + promtail + Loki počas záťaže |

### Meranie výkonu (RPi4 vs SFF)
- **Latencia**: časová značka udalosti na zariadení vs. čas dostupnosti v Loki (query `{vendor="..."} |= "marker"` ihneď po vygenerovaní).
- **Priepustnosť**: počet spracovaných správ/s pred stratou/zahodením (sledovať `syslog-ng-ctl stats` a Promtail metriky na `:9080/metrics`).
- **Zdrojová náročnosť**: `docker stats` počas idle a počas záťažového testu (scenár 5).
- **Storage**: veľkosť `/loki` po X dňoch prevádzky pri danom objeme logov → extrapolácia na 30/90 dní retention.

## 5. Loki vs. Elasticsearch — body na porovnanie (pre analytickú kapitolu)

| Kritérium | Loki | Elasticsearch |
|---|---|---|
| Indexovanie | Iba metadata/labels, nie full-text | Full-text index každého poľa |
| Nároky na zdroje (RAM/CPU) | Nízke — vhodné pre RPi4/SFF | Výrazne vyššie, needoporúča sa na RPi |
| Rýchlosť vyhľadávania (grep-like dotazy) | Pomalšie pri veľkých objemoch bez labelov | Rýchlejšie vďaka plnému indexu |
| Zložitosť nasadenia/údržby | Nízka (single binary/Docker) | Vyššia (cluster, shards, mapping) |
| Vhodnosť pre malé SMB nasadenie | Áno — nízke náklady, "log aggregation ako Prometheus pre logy" | Skôr pre stredné/veľké nasadenia s vyššími nárokmi na full-text search |
| Ekosystém alertingu | Grafana Alerting priamo nad LogQL | Kibana/Watcher, komplexnejšie |

Odporúčanie pre report: Loki je vhodnejší pre PoC/malé nasadenie
kvôli nízkym HW nárokom; Elasticsearch sa oplatí až pri potrebe
komplexného full-text vyhľadávania alebo pri raste objemu logov nad
kapacitu jedného malého zariadenia.

## 6. Bezpečnostné poznámky
- Zmeniť predvolené Grafana heslo pred akýmkoľvek nasadením mimo lab.
- Zvážiť TLS syslog (port 6514) namiesto plaintext UDP/TCP pre produkčné nasadenie.
- Obmedziť prístup k portom 514/601/3100/3000 iba z manažment VLAN (firewall pravidlá na hoste alebo Docker network policies).
- Retencia logov (720h v `loki-config.yml`) by mala zohľadňovať lokálnu legislatívu/interné politiky pre uchovávanie prevádzkových logov.

## 7. Štruktúra repozitára
```
network-monitoring/
├── docker-compose.yml
├── syslog-ng/
│   └── syslog-ng.conf
├── promtail/
│   └── promtail-config.yml
├── loki/
│   └── loki-config.yml
├── grafana/
│   ├── provisioning/
│   │   ├── datasources/loki.yml
│   │   └── dashboards/dashboards.yml
│   └── dashboards/         # sem uložiť exportované JSON dashboardy
└── README.md
```

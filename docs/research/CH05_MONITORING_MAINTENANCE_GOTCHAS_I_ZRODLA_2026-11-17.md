# CH05 — monitoring, maintenance i recovery: źródła, gotchas oraz walidacja

**Data badań:** 2026-11-17. **Blueprint:** rozdział 5.1–5.9. **Lekcja:** [Operational Monitoring and Maintenance](../chapters/JUNOS_Rozdzial_05_Operational_Monitoring_and_Maintenance.md).

To notatka redakcyjna dla autora/Codexa. **Gotchas zostały już wplecione w odpowiednie punkty wykładu**, nie stanowią osobnego banku pytań egzaminacyjnych. Własne przypadki szkoleniowe są nazwane jako własne i nie są exam dumps. Nie przeprowadzono runtime testów EVE-NG.

## 18 kluczowych pułapek z udokumentowanymi podstawami

| Nr | Sekcja | Gotcha i znaczenie | Oficjalne źródło |
|---:|---|---|---|
| 1 | 5.1 | `show configuration` ≠ rzeczywisty operacyjny stan protokołu/interfejsu. Sam `up/up` nie oznacza zdrowego dataplane. | [Operational Monitoring](https://www.juniper.net/documentation/us/en/software/junos/cli/topics/topic-map/junos-operational-mode-commands-monitoring.html) |
| 2 | 5.1 | `show route`: `destinations` ≠ `routes`; aktywność/hidden i wybrana tablica mają znaczenie. `show route | count` liczy linie outputu, nie prefiksy. | [show route](https://www.juniper.net/documentation/us/en/software/junos/cli-reference/topics/ref/command/show-route.html), [show route hidden](https://www.juniper.net/documentation/us/en/software/junos/cli-reference/topics/ref/command/show-route-hidden.html) |
| 3 | 5.2 | `monitor traffic` jest obserwacją ruchu dostępnego w danym punkcie (często RE), nie automatycznie kopią wszystkich pakietów transit przechodzących przez PFE. | [monitor traffic](https://www.juniper.net/documentation/us/en/software/junos/cli-reference/topics/ref/command/monitor-traffic.html) |
| 4 | 5.2 | `monitor start messages` śledzi dopisania do logu, nie uruchamia logowania protokołu; `monitor stop` kończy podgląd, nie usuwa historycznych logów. | [monitor start](https://www.juniper.net/documentation/us/en/software/junos/cli-reference/topics/ref/command/monitor-start.html), [monitor stop](https://www.juniper.net/documentation/us/en/software/junos/cli-reference/topics/ref/command/monitor-stop.html) |
| 5 | 5.2 | `monitor interface` ma ograniczenia obsługi terminali (na Junos Evolved SSH może być wspierane, a console/Telnet nie). | [monitor interface](https://www.juniper.net/documentation/us/en/software/junos/cli-reference/topics/ref/command/monitor-interface.html) |
| 6 | 5.3 | Błędy CRC/input i drops/output nie są synonimami; liczniki zwykle są skumulowane i bez dwóch pomiarów nie określisz tempa awarii. | [Understanding Packet Drops](https://www.juniper.net/documentation/us/en/software/junos/junos-overview/topics/concept/using-show-commands-for-packet-drops.html) |
| 7 | 5.3 | DOM: Rx local odpowiada torowi Tx peer → lokalny odbiornik. Progi zależą od typu SFP i sprzętu, a nie jednego uniwersalnego dBm. | [Digital Optical Monitoring](https://www.juniper.net/documentation/us/en/software/junos/chassis/topics/concept/digital-optical-monitoring.html) |
| 8 | 5.3 | `clear interfaces statistics` zmienia punkt odniesienia, może utrudnić analizę NMS i zachowuje się inaczej przy failover RE. | [clear interfaces statistics](https://www.juniper.net/documentation/us/en/software/junos/cli-reference/topics/ref/command/clear-interfaces-statistics.html) |
| 9 | 5.4 | Ping z RE do gateway **nie potwierdza** end-to-end ruchu klienta z innym source IP/VRF/security policy. | [ping CLI](https://www.juniper.net/documentation/us/en/software/junos/cli-reference/topics/ref/command/ping.html) |
| 10 | 5.4 | Gwiazdka pośredniego hopa w traceroute nie jest dowodem utraty wszystkich pakietów transit: ICMP TTL exceeded może być filtrowany/rate-limited. | [traceroute CLI](https://www.juniper.net/documentation/us/en/software/junos/cli-reference/topics/ref/command/traceroute.html) |
| 11 | 5.5 | Jeden package Junos nie pasuje do EX/QFX/MX/SRX. Wybór release wg PID, release notes, hardware/feature support i JTAC KB21476. | [Software install overview](https://www.juniper.net/documentation/us/en/software/junos/junos-install-upgrade/topics/topic-map/software-install-and-upgrade-overview.html) |
| 12 | 5.5 | `no-validate` pomija sprawdzenie zgodności config; nie jest dobrym defaultem, ale część przejść (SRX3xx z 23.4R2-S3 lub 24.2R2 do 24.4R1) wymaga konkretnych wyjątków, w tym `no-copy no-validate reboot`. | [software add](https://www.juniper.net/documentation/us/en/software/junos/cli-reference/topics/ref/command/request-system-software-add.html) |
| 13 | 5.5 | GRES/redundant RE/ISSU/VMHost/SRX cluster/Virtual Chassis mają różne sposoby upgrade; HA nie oznacza zero-impact i nie zawsze działa zwykłe `software add`. | [software add](https://www.juniper.net/documentation/us/en/software/junos/cli-reference/topics/ref/command/request-system-software-add.html) |
| 14 | 5.5 | Snapshot wykonany po aktualizacji może nadpisać obraz zapasowy na alternate root, usuwając możliwość prostego powrotu do starego release. | [Back Up Installation Using Snapshots](https://www.juniper.net/documentation/us/en/software/junos/junos-install-upgrade/topics/topic-map/backing-up-install-using-snapshot.html) |
| 15 | 5.5 | `rollback 1` dotyczy **konfiguracji**, `request system software rollback` — software; nie są zamienne i nie wszystkie downgrade są wspierane. | [rollback config](https://www.juniper.net/documentation/us/en/software/junos/cli-reference/topics/ref/command/rollback.html), [software rollback](https://www.juniper.net/documentation/us/en/software/junos/cli-reference/topics/ref/command/request-system-software-rollback.html) |
| 16 | 5.6 | `request system reboot` vs `halt` vs fizyczne wyłączenie; RE/VC/cluster zmieniają impact. `show system reboot` umożliwia odczyt zaplanowanego restartu. | [reboot](https://www.juniper.net/documentation/us/en/software/junos/cli-reference/topics/ref/command/request-system-reboot.html), [halt](https://www.juniper.net/documentation/us/en/software/junos/cli-reference/topics/ref/command/request-system-halt.html) |
| 17 | 5.7 | Klasyczne recovery może używać `boot -s` i `recovery`, ale zabezpieczenie console jako insecure, FIPS i Junos OS Evolved mogą zmienić/zablokować proces. | [Recover Root Junos](https://www.juniper.net/documentation/us/en/software/junos/user-access/topics/topic-map/recovering-root-password.html), [Junos Evolved](https://www.juniper.net/documentation/us/en/software/junos/user-access-evo/user-access/topics/topic-map/recovering-root-password.html) |
| 18 | 5.7 | `plain-text-password` uruchamia prompt i późniejszy hash; `encrypted-password` wymaga poprawnego już zaszyfrowanego inputu. Błąd może spowodować lockout. | [Recover Root Junos](https://www.juniper.net/documentation/us/en/software/junos/user-access/topics/topic-map/recovering-root-password.html) |

## Inne kwestie zweryfikowane przed publikacją

- `show system memory` istnieje w CLI, ale jest opisane jako narzędzie zaawansowane/Customer Support: [CLI Reference](https://www.juniper.net/documentation/us/en/software/junos/cli-reference/topics/ref/command/show-system-memory.html). Nie wymagać, by każda platforma pokazywała identyczne pola.
- `junos-ping` występuje jako predefiniowana aplikacja SRX: [Security Policy documentation](https://www.juniper.net/documentation/us/en/software/junos/security-policies/topics/topic-map/configuring-unified-policies.html). W labie 5.9 wciąż weryfikuj dokładny release i istniejące policies/zone.
- `show security flow session summary` jest udokumentowane dla SRX: [Monitoring Sessions](https://www.juniper.net/documentation/us/en/software/junos/flow-packet-processing/topics/topic-map/security-flow-session-and-error-handling.html).
- `request system storage cleanup dry-run` jest oddzielną operacją od `request system storage cleanup`: [Junos CLI Reference](https://www.juniper.net/documentation/us/en/software/junos/cli-reference/topics/ref/command/request-system-storage-cleanup.html).

## Warunki przyszłego runtime validation

1. **Przed uruchomieniem** zweryfikuj prawa do obrazów Junos/vSRX i rzeczywisty vNIC mapping, ilość RAM i dostępne interfejsy EVE.
2. Skonfiguruj **nowy** lab CH05 bez naruszania CH02/CH03/CH04 i Cisco.
3. Zapisz `show version`, precheck oraz faktyczne ICMP PC-A → PC-B przy poprawnej security policy; nie zakładaj PASS.
4. Oddzielnie testuj port `disable`, missing static route, policy deny i parser reject. Używaj niezależnej konsoli.
5. Przy `commit confirmed` **nie uruchamiaj ponownie `commit check`**, bo może nieświadomie potwierdzić pending commit (zob. CH03).
6. Zbierz outputy i czasy przed, w trakcie i po fault. Nie wymyślaj logów.
7. Dla DOM i hardware counters wymagany jest rzeczywisty EX/QFX z optyką; vSRX nie ma fotodetektora.
8. Do upgrade potrzebny jest **legalny obraz dla dokładnego modelu**, release path i procedura; nie uruchamiaj samego `software add` podczas ćwiczenia routing fault.
9. Root password recovery wykonuj tylko na własnej odseparowanej instancji z faktyczną konsolą i autoryzacją.
10. Repozytorium publiczne nie może zawierać credentiali, `.qcow2`, private keys ani danych produkcyjnych.

## Ilustracja

[Wikimedia: SFP-front-RX-TX.jpg](https://commons.wikimedia.org/wiki/File:SFP-front-RX-TX.jpg), oryginalna fotografia Adamantios, oznaczenia Tx/Rx MichaelFrey, CC BY-SA 3.0, hotlink w sekcji 5.3.5a. Pozostałe obrazy to autorskie schematy Mermaid zawarte w Markdown.

**Stan źródeł:** publiczna dokumentacja Juniper/HPE dostępna 2026-11-17; aktualizuj informacje o upgrade i CVE przed każdą realną zmianą.

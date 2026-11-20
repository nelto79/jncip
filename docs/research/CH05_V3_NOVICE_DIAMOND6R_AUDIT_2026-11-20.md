# CH05 v3 — audyt blueprintu, dydaktyki i ryzyka wykonywania laboratoriów

**Data redakcji:** 2026-11-20  
**Kanon:** `docs/chapters/JUNOS_Rozdzial_05_Operational_Monitoring_and_Maintenance.md`  
**Archiwum:** `docs/archive/JUNOS_Rozdzial_05_pre_beginner_rewrite_2026-11-20.md`  
**Profil czytelnika:** inżynier sieci z ~1 rokiem doświadczenia, bez wymagania Cisco/CCNA/JNCIA/JNCIS.  
**Status:** SOURCE REVIEWED / STATIC EDITORIAL REVIEW / **NOT RUNTIME VALIDATED**.

## Pokrycie blueprintu

| Cel | Rozdział | Sprawdzana kompetencja |
|---|---|---|
| 5.1 `show` | §5.1 | Konfiguracja ≠ operational ≠ service, RIB/FIB, alarmy i procesy |
| 5.2 `monitor` | §5.2 | Obserwacja zmian, monitor start/stop, limited RE capture |
| 5.3 Liczniki i błędy | §5.3 | Dwa pomiary, CRC/input/errors/drops, physical DOM vs vSRX |
| 5.4 Ping/traceroute/telnet/SSH | §5.4 | Source, routing-instance, return path, granice testu ICMP |
| 5.5 Junos installation/upgrade | §5.5 | Model/release/image/validation/backup/snapshot/HA/MOP |
| 5.6 Boot/reboot/halt | §5.6 | Impact, rozpoznanie historii, brak reboot jako pierwszego fix |
| 5.7 Root recovery | §5.7 | Wymóg autoryzowanej konsoli, różnice klasyczny/Evolved |
| 5.8 Diagnostyka | §5.8 | Objaw → hipoteza → test → przyczyna → fix → test |
| 5.9 Lab | §5.9 | R1 → R5/R1/R2 → pełny DIAMOND-6R |
| Rozszerzenia | §5.TS/5.TEST/5.REF | 6 case studies, 24 pytania z odpowiedziami, field guide |

## Audyt statyczny

- Około **15 418 słów**, około **116 979 znaków** po korekcie poleceń commit, **94 bloki CLI**, **3 diagramy Mermaid**.
- Wszystkie bloki `~~~junos` mają bezpośrednio nad nimi nazwę urządzenia (R1–R6 albo odrębny fizyczny EX-ACCESS01) oraz tryb pracy.
- Obecne wszystkie wymagane nagłówki 5.1–5.9, zamknięte bloki kodu, brak błędnych backslash-backticks.
- Referencyjna adresacja wszystkich 6 segmentów DIAMOND-6R i interfejsów R1–R6 zachowana. Nie ma historycznych hostów PC-A/PC-B; jeśli pojawia się VPCS, wyłącznie dla wyjaśnienia różnicy.
- R5/R6 są routerami: lokalne pingi nie dowodzą tranzytu od realnych komputerów za nimi.
- W konfigurowanych przykładach `commit confirmed N comment "CH05 controlled lab change"` i `commit comment "CH05 post-check verified"` zastosowano jawne opisy na wypadek release wymagającego commit comments.
- Podczas oczekującego `commit confirmed` nie wykonujemy kolejnego `commit check` — w Junos może on potwierdzić zmianę.
- Nie wykonano rzeczywistych testów vSRX ani wymiany obrazów systemowych.

## Krytyczne ograniczenia laboratoryjne

1. Potwierdzić Junos release i rzeczywiste mapowanie ge-0/0/x na sześciu vSRX przed wprowadzaniem konfiguracji.
2. Każda instancja musi mieć bazowy login i poprawne root-authentication oraz niezależną konsolę; nie próbować globalnego factory reset ani nadpisywania istniejących konfiguracji.
3. Baseline z poziomu 2 wymaga trasy R5→10.12.0.0/30 i drogi powrotnej R2→192.0.2.0/24, host-inbound ping na R2 oraz jawnej polityki R1 CE→CORE.
4. DIAMOND-6R używa tras statycznych i `qualified-next-hop`; ruch R5→R6 przez R1/R2/R4 może mieć inną drogę powrotną po awarii górnego łącza. **Nie gwarantować end-to-end failover tylko na podstawie zapasowej trasy R1**.
5. W SRX nowa sesja zależy od policy, a stara może przetrwać jej zmianę; fault injection wymaga sprawdzania nowego flow.
6. `show interfaces diagnostics optics` wymaga faktycznej wspieranej wkładki fizycznej. vSRX nie emuluje wiarygodnego DOM ani realnych laserów.
7. `monitor traffic` zwykle nie jest pełnym capture pakietów PFE transit; brak widoczności nie znaczy brak forwardingu.
8. Nie uruchamiać w tym labie faktycznego software add, request reboot/halt, globalnego clear counters, proces restart lub root recovery. To osobne, autoryzowane procedury na klonach/sprzęcie.
9. RUNTIME VALIDATED wymaga rzeczywistych outputów i czasu pre/post dla każdego fault na konkretnym modelu/release.

## Źródła szczególnie istotne dla gotchas

- [Junos show interfaces extensive](https://www.juniper.net/documentation/us/en/software/junos/cli-reference/topics/ref/command/show-interfaces-extensive.html)
- [monitor traffic](https://www.juniper.net/documentation/us/en/software/junos/cli-reference/topics/ref/command/monitor-traffic.html)
- [Digital Optical Monitoring](https://www.juniper.net/documentation/us/en/software/junos/chassis/topics/concept/digital-optical-monitoring.html)
- [Static Route Preferences and Qualified Next Hops](https://www.juniper.net/documentation/us/en/software/junos/static-routing/topics/topic-map/static-route-prefer-qualified-next-hop.html)
- [SRX Security Policies](https://www.juniper.net/documentation/us/en/software/junos/security-policies/topics/concept/policy-overview.html)
- [request system software add](https://www.juniper.net/documentation/us/en/software/junos/cli-reference/topics/ref/command/request-system-software-add.html)
- [commit and confirmed commit comments](https://www.juniper.net/documentation/us/en/software/junos/cli/topics/topic-map/junos-configuration-commit.html)
- [Recover Root Password](https://www.juniper.net/documentation/us/en/software/junos/user-access/topics/topic-map/recovering-root-password.html)

**Wynik:** rozdział gotowy do czytania i oceny dydaktycznej. Wszystkie działania laboratoryjne pozostają **PLANNED**, bez przypisywania fikcyjnego PASS.

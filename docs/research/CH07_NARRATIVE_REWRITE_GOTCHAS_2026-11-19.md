# CH07 — rejestr wyjątków, źródeł i walidacji przebudowanej lekcji

**Data:** 19 listopada 2026  
**Kanoniczny rozdział:** \`docs/chapters/JUNOS_Rozdzial_07_Protocol_Independent_Routing.md\` (v2.0)  
**Archiwum:** \`docs/chapters/JUNOS_Rozdzial_07_Protocol_Independent_Routing_v1_ARCHIVE.md\`  
**Status:** redakcja zakończona, pokrycie 7.1–7.8 sprawdzone; **NOT RUNTIME VALIDATED**. Nie uruchomiono EVE-NG ani testów na konkretnym wydaniu Junos.

Celem rejestru jest udokumentowanie, dlaczego w tekście ucznia pojawiają się nieintuicyjne zachowania i jakie dowody pozwolą je zweryfikować. Nie zastępuje narracji rozdziału i nie jest bankiem pytań egzaminacyjnych.

| Temat | Zjawisko, które może zmylić inżyniera Cisco | Materiał producenta | Gdzie wpleciono w lekcję | Test runtime |
|---|---|---|---|---|
| Aggregate | Jeden active contributor może podtrzymywać /16, chociaż część usług jest niedostępna; domyślnie reject next-hop | [Configuring Route Aggregation](https://www.juniper.net/documentation/us/en/software/junos/static-routing/topics/topic-map/config-route-aggregation.html) | 7.1 i LAB 7.8 fazy 5, 8 | Odczyt RIB/FIB, usunięcie contributorów, przywrócenie |
| Generated | Generated może odziedziczyć next-hop primary contributor, więc pakiety do nieistniejącego fragmentu summary mogą być wysyłane do upstream | [Conditionally Generated Routes](https://www.juniper.net/documentation/us/en/software/junos/static-routing/topics/concept/policy-generated-routes.html) | 7.1 i LAB 7.8 fazy 5, 11 | Capture WAN-A dla 10.70.99.25 |
| Aggregate/Generated route tag | Obie rodziny mogą być pokazane jako \`[Aggregate/130]\`, mimo że ich semantyka różni się | [Route Preferences](https://www.juniper.net/documentation/us/en/software/junos/routing-overview/topics/concept/routing-protocols-route-preferences-overview.html), [generate](https://www.juniper.net/documentation/us/en/software/junos/cli-reference/topics/ref/statement/generate-edit-routing-options.html) | 7.1 | Compare configuration, extensive route |
| Martians | \`0.0.0.0/0 exact allowed\` nie zakazuje default; prywatne RFC1918 nie są automatycznie martians | [Recognize Martian Addresses](https://www.juniper.net/documentation/us/en/software/junos/static-routing/topics/topic-map/recognize-martian-addr-routing.html), [show route martians](https://www.juniper.net/documentation/us/en/software/junos/cli-reference/topics/ref/command/show-route-martians.html) | 7.2 i LAB faza 13 | Odczyt outputu, bez zmian w głównym R1 |
| RIB group | Samo zdefiniowanie \`rib-groups\` nie powoduje importu; trzeba przypiąć do route source | [interface-routes](https://www.juniper.net/documentation/us/en/software/junos/cli-reference/topics/ref/statement/interface-routes-edit-routing-options.html), [rib-groups](https://www.juniper.net/documentation/us/en/software/junos/cli-reference/topics/ref/statement/rib-groups-edit-routing-options.html) | 7.3/7.4 i LAB fazy 6, 9 | Porównanie primary/secondary RIB i connected |
| Secondary route lifecycle | Secondary route importowana przez RIB group nie jest niezależna, gdy primary znika | [import-rib CLI](https://www.juniper.net/documentation/us/en/software/junos/cli-reference/topics/ref/statement/import-rib-edit-routing-options.html) | 7.4 i LAB 9 | Zniknięcie connected i rollback |
| Import-policy RIB group | Kontekst polityki import RIB group ≠ BGP import; filtry nie powinny naruszać primary | [RIB Groups](https://www.juniper.net/documentation/us/en/software/junos/cli-reference/topics/ref/statement/rib-groups-edit-routing-options.html) | 7.4 opcjonalny wariant | Test per-model; verify global inet.0 unmodified |
| Load balance | \`load-balance per-packet\` w wielu Junos to per-flow hashing, nie literalne naprzemienne wysyłanie każdego pakietu | [Configuring Per-Flow Load Balancing](https://www.juniper.net/documentation/us/en/software/junos/sampling-forwarding-monitoring/topics/concept/policy-configuring-per-flow-load-balancing.html), [load-balance CLI](https://www.juniper.net/documentation/us/en/software/junos/cli-reference/topics/ref/statement/load-balance-edit-policy-options-policy-statement-then.html) | 7.5 i LAB 12 | Dwie equal-cost paths; wiele flows; ingress/egress capture |
| Forwarding-table export | To policy instalacji FIB, nie reklamowanie do BGP; musi istnieć prawdziwa multipath eligibility | [Routing Policies](https://www.juniper.net/documentation/us/en/software/junos/routing-policy/topics/concept/policy-routing-policies-overview.html) | 7.5 i LAB 12 | RIB vs FIB i działania linków |
| FBF | \`then routing-instance\` klasyfikuje ingress pakiet, nie dodaje nowej zwykłej trasy ani security permit | [FBF Overview](https://www.juniper.net/documentation/us/en/software/junos/routing-policy/topics/concept/firewall-filter-option-filter-based-forwarding-overview.html), [FBF Example](https://www.juniper.net/documentation/us/en/software/junos/routing-policy/topics/example/filter-based-forwarding-example.html) | 7.6 i LAB 7 | Rzeczywisty capture PC-A/PC-B na UP-A/UP-B |
| FBF default filter | Brak termu NORMAL może dropować niedopasowany ruch przez implicit discard | [Firewall Filter Components](https://www.juniper.net/documentation/us/en/software/junos/routing-policy/topics/concept/firewall-filter-components.html) | 7.6, 7.7, LAB faza 10 | Zamierzone usunięcie termu, \`commit confirmed\`, rollback |
| FBF platform differences | EX/QFX/MX/vSRX mają odmienne możliwości, kierunki, działania i punkty processingu | [FBF for EX](https://www.juniper.net/documentation/us/en/software/junos/routing-policy/topics/concept/filter-based-forwarding-ex-series.html), [APBR for SRX](https://www.juniper.net/documentation/us/en/software/junos/application-identification/topics/topic-map/security-application-advanced-policy-based-routing.html), [Feature Explorer](https://apps.juniper.net/feature-explorer/) | 7.3, 7.6, 7.8 | Zweryfikować CLI i rzeczywisty flow/NAT na obrazie |
| SRX stateful security | Routing instance i stateless filter nie zastępują security zones/policies, sesji, return path | [Security Policies](https://www.juniper.net/documentation/us/en/software/junos/security-policies/topics/concept/policy-overview.html) | 7.3, 7.6–7.8 | \`show security flow session\`, host-to-host |
| Bezpieczny rollback | \`commit check\` w oczekującym \`commit confirmed\` może potwierdzić zmianę | [Junos CLI Configuration](https://www.juniper.net/documentation/us/en/software/junos/cli/topics/topic-map/junos-cli-configuration.html) | 7.7, 7.8 | Odrębna konsola, brak dodatkowego check po confirmed |

## Zakres i styl

- Sekcje **7.1–7.8** zachowane. W 7.8 pełny scenariusz: preflight, adresacja, Linux test nodes, SRX security, baseline, aggregate/generated, RIB group, FBF, pięć rodzajów testów i cleanup.
- Narracja oparta na jednej topologii greenfield OFFICE z dwoma WAN. Opcjonalne przykłady martian i ECMP mają jawnie wydzielone fazy.
- Dodatkowe pytania integrujące wiedzę znajdują się po objaśnieniach, nie jako słownik i lista nakazów „na pamięć”.
- Przy każdym bloku Junos i Linux określono nazwę urządzenia i tryb. Wartości do testów są dokumentacyjne/izolowane.
- Diagramy Mermaid powstały od zera na potrzeby lekcji i są uzupełnione tekstem alternatywnym.
- Porównania do Cisco służą wykorzystaniu wiedzy inżyniera, nie są tylko translacją CLI.
- **Nie wykonano testów runtime.** Nie ma dowodów licznika FBF, egress tcpdump, statusu SRX flow ani konkretnego \`show version\`. Laboratorium pozostaje PLANNED.

## Lista kontrolna do testów wykonywanych później

Po uruchomieniu w EVE-NG zbierz rzeczywiste: wersję Junos i model vSRX, mapowanie portów QEMU→Junos, \`show route\` dla summary, connected w primary i secondary, resolved/unresolved FBF default, output flow sessions, tcpdump oddzielnie WAN-A i WAN-B, stan system commit, wyniki commit confirmed rollback. Dla ECMP pokaż liczbę kwalifikowanych next-hopów w FIB oraz kilka niezależnych przepływów, nie jedną serię ping.

Nie traktować zmian w obrazach EVE-NG, poprawnej składni Markdown lub \`commit check\` jako dowodu runtime. Zasoby sprzętowe i licencjonowanie vSRX trzeba sprawdzić przed faktyczną aktywacją labu.

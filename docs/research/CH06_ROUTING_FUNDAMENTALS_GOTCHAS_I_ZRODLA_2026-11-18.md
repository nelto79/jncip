# CH06 — Routing Fundamentals: rejestr badań, gotchas i pochodzenie źródeł

**Data:** 2026-11-18. **Blueprint:** [6.1–6.11](../JUNOS_JNCIA_JNCIS_JNCIP_unified_blueprint.md). **Podręcznik ucznia:** [Rozdział 6](../chapters/JUNOS_Rozdzial_06_Routing_Fundamentals.md).

To jest **rejestr redakcyjny dla autora i Codexa**, a nie dodatkowy bank pytań. Rzeczywiste haczyki i wyjaśnienia zostały wplecione w odpowiednie sekcje wykładu jako sytuacje Cisco → Junos, z testami CLI. Pytania problemowe w rozdziale 6 są **autorskie**, chyba że wyraźnie wskazano inaczej; nie są kopią egzaminu.

## 18 ważnych gotchas / sytuacji z oficjalnych źródeł

| # | Sekcja | Obserwacja i praktyczne znaczenie | Źródło producenta |
|---:|---|---|---|
| 1 | 6.1 | Forwarding podejmuje niezależną decyzję hop-by-hop; ping bramy SRX (host-bound) ≠ transit ping przez SRX i security policies | [SRX Flow Processing](https://www.juniper.net/documentation/us/en/software/junos/flow-packet-processing/topics/topic-map/security-srx-devices-processing-overview.html) |
| 2 | 6.2 | `destinations` ≠ `routes`; Active/Inactive/Hidden mają inną semantykę; `show route | count` liczy linie outputu | [show route](https://www.juniper.net/documentation/us/en/software/junos/cli-reference/topics/ref/command/show-route.html), [Junos Route Tables](https://www.juniper.net/documentation/us/en/software/junos/static-routing/topics/topic-map/config_junos_routing_table.html) |
| 3 | 6.2 | `inet.0` ≠ `inet6.0` ≠ `NAME.inet.0`; izolowane routing instances nie dziedziczą global routes automatycznie | [Routing Instances](https://www.juniper.net/documentation/us/en/software/junos/routing-overview/topics/concept/routing-instances-overview.html) |
| 4 | 6.3 | RIB active i route w RE forwarding table ≠ gwarancja poprawnej adjacency ARP/ND, SRX policy i hardware ASIC programming | [RE/PFE tables](https://www.juniper.net/documentation/us/en/software/junos/junos-overview/topics/concept/junos-software-routing-forwarding-table-overview.html), [Day One Troubleshooting](https://www.juniper.net/documentation/en_US/day-one-books/topics/concept/troubleshooting.html) |
| 5 | 6.4 | W Junos static=5, OSPF internal=10, external=150, BGP=170; Cisco IOS XE ma inne typical AD, eBGP/iBGP nie przenoszą tego podziału 1:1 | [Default Route Preference](https://www.juniper.net/documentation/us/en/software/junos/routing-overview/bgp/topics/concept/routing-protocols-default-route-preference-values.html) |
| 6 | 6.4 | `preference` nie jest OSPF cost ani BGP Local Preference, a BGP `Preference2` jest tiebreaker/inna reprezentacja | [Route Preferences](https://www.juniper.net/documentation/us/en/software/junos/routing-overview/topics/concept/routing-protocols-route-preferences-overview.html) |
| 7 | 6.5 | **LPM wybiera /26 nad /24 nad /0** niezależnie od preference między różnymi prefiksami. Preference wybiera aktywną trasę **dla tego samego prefixu** | [Routing Table Config](https://www.juniper.net/documentation/us/en/software/junos/static-routing/topics/topic-map/config_junos_routing_table.html) |
| 8 | 6.5 | Bardziej szczegółowy `discard` może blackholować ruch pomimo prawidłowego default route; `discard` zwykle nie wysyła ICMP, `reject` tak | [discard](https://www.juniper.net/documentation/us/en/software/junos/cli-reference/topics/ref/statement/discard-edit-routing-options.html), [static](https://www.juniper.net/documentation/us/en/software/junos/cli-reference/topics/ref/statement/static-edit-routing-options.html) |
| 9 | 6.6 | `virtual-router` to nie to samo co instance-type `vrf` dla usług VPN; routing instance wymaga przypisania interfejsów | [Virtual Routing Instances](https://www.juniper.net/documentation/us/en/software/junos/multicast-l2/topics/topic-map/virtual-routing-instances.html) |
| 10 | 6.7 | `Direct/0` route do sieci i `Local/0` route do adresu własnego interfejsu to odrębne wpisy; static IPv6 należy do `routing-options rib inet6.0 static` | [IPv4/IPv6 static routes](https://www.juniper.net/documentation/us/en/software/junos/static-routing/topics/topic-map/config_static-routes.html) |
| 11 | 6.8 | Dynamic routing umożliwia automatyczną wymianę wiedzy, ale zależy od konwergencji, policy i stanu protokołów; nie jest magicznym sposobem naprawy złego next-hop | [Route Preferences Overview](https://www.juniper.net/documentation/us/en/software/junos/routing-overview/topics/concept/routing-protocols-route-preferences-overview.html) |
| 12 | 6.10 | Wiele next-hopów pod jednym static prefix bez określenia kwalifikowanych parametrów nie gwarantuje primary/standby ani ECMP | [Qualified Next Hops](https://www.juniper.net/documentation/us/en/software/junos/static-routing/topics/topic-map/static-route-prefer-qualified-next-hop.html) |
| 13 | 6.10 | Floating static z higher preference nie wykryje blackhole za dostępnym bezpośrednim next-hop; trzeba dopasować BFD/RPM/SLA lub dynamic routing | [BFD Static Routes](https://www.juniper.net/documentation/us/en/software/junos/static-routing/topics/topic-map/bfd-static-routes.html) |
| 14 | 6.10 | `resolve` umożliwia recursive/indirect next-hop; normalny static wymaga bezpośredniej kwalifikacji next-hop; recursive resolution może wybrać niespodziewaną drogę | [resolve](https://www.juniper.net/documentation/us/en/software/junos/cli-reference/topics/ref/statement/resolve-edit-routing-options.html) |
| 15 | 6.10 | `resolve` i `retain` są niekompatybilne dla jednej static route | [retain](https://www.juniper.net/documentation/us/en/software/junos/cli-reference/topics/ref/statement/retain-edit-routing-options.html) |
| 16 | 6.10 | `no-readvertise` nie uruchamia/wyłącza BGP; static nie rozgłasza się bez właściwej polityki eksportu, `no-readvertise` ogranicza eligibility | [Readvertise](https://www.juniper.net/documentation/us/en/software/junos/cli-reference/topics/ref/statement/readvertise-edit-routing-options.html) |
| 17 | 6.10 | `passive` może zachować route pomimo nieosiągalnego next-hop i powodować odrzucenie ruchu; `retain` dotyczy specjalnej retencji w FIB, nie zdrowia fizycznego linku | [Static Route Control](https://www.juniper.net/documentation/us/en/software/junos/static-routing/topics/topic-map/static-route-prefer-qualified-next-hop.html) |
| 18 | 6.11 | Na SRX routing i stateful security policy są oddzielne; dobry RIB/FIB nie dowodzi permit session. W dual stack IPv4 ARP ≠ IPv6 ND | [SRX Security Policies](https://www.juniper.net/documentation/us/en/software/junos/security-policies/topics/topic-map/security-policy-configuration.html), [IPv6 ND](https://www.juniper.net/documentation/us/en/software/junos/neighbor-discovery/topics/topic-map/ipv6-neighbor-discovery.html) |

## Dodatkowe źródła edukacyjne

- [Day One: Beginner's Guide to Learning Junos](https://www.juniper.net/documentation/en_US/day-one-books/junos-beginners-guide.pdf) — publiczna książka producenta; zagadnienia route preference, ECMP i static.
- [show route forwarding-table](https://www.juniper.net/documentation/us/en/software/junos/cli-reference/topics/ref/command/show-route-forwarding-table.html) — opcje `family inet6`, `destination`.
- [show route](https://www.juniper.net/documentation/us/en/software/junos/cli-reference/topics/ref/command/show-route.html) — wariant `exact`, kontekst table i route states.
- [VPCS manual](https://github.com/GNS3/vpcs/blob/master/man/vpcs.1) — ograniczenia związane z konfiguracją IPv6 w niektórych wariantach VPCS; stąd hosty Linux w CH06.

## Prawdziwość, licencje i testy

- Oficjalny JNCIA Practice Exam nie został w tym zadaniu uruchomiony; nie oznaczamy autorskich pytań jako kopii tego egzaminu.
- Case studies wynikają z połączenia dokumentacji i doświadczenia operatorskiego Cisco/Juniper, są **samodzielnymi konstrukcjami dydaktycznymi**.
- Wszystkie diagramy Mermaid są oryginalne. Nie osadzano przypadkowych zdjęć routerów — nie wspierają rozumienia RIB/FIB/LPM.
- Konfiguracja LAB CH06 i pokazane wyniki to **plan**, nie faktyczny EVE run. Nie oznaczaj ich jako PASS bez wykonanych testów.
- W produkcji dobór platformy/softu/opcji (szczególnie `resolve`, `retain`, `passive`, `next-table`, `qualified-next-hop`) trzeba potwierdzić w Feature Explorer i Release Notes dla konkretnego modelu.
- Nie uruchamiaj `discard`, `reject`, `delete static`, `commit confirmed` na sieciach klientów bez change approval i recovery.
- W repozytorium nie zapisujemy prywatnych adresów management klientów, haseł, tokenów, obrazu QCOW2, niepublikowanych konfiguracji czy danych SLA.

**Wzorzec na następne rozdziały:** najpierw aktywna kwerenda gotchas z dokumentacji producenta, potem narracja na konkretnym objawie, analiza Cisco ↔ Junos, weryfikacja CLI i izolowany lab. Pytania oraz odpowiedzi w tekście wykładu, nie na końcu jako sztuczny bank zadań.

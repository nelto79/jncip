# CH08 v3 — audyt redakcyjny, techniczny i laboratoryjny DIAMOND-6R

**Data:** 20 listopada 2026.  
**Dokument:** \`docs/chapters/JUNOS_Rozdzial_08_Routing_Policy_and_Firewall_Filters.md\`  
**Status:** EDITORIAL REVIEW COMPLETE / SOURCE CHECKED / **NOT RUNTIME VALIDATED**.  
**Zasada:** Weryfikacja merytoryczna na oficjalnej dokumentacji nie jest tym samym, co uruchomienie poleceń na konkretnym obrazie vSRX.

## 1. Dlaczego wykonano kolejną przebudowę

Czytelnik porównał wcześniejszą wersję rozdziału 8 z udostępnioną wersją przygotowaną przez innego agenta. Główna różnica dydaktyczna: wzorcowa wersja prowadzi od **jasnej reguły Junos → przyczyny → konsekwencji → komend → dowodu diagnostycznego**. Poprzedni tekst wielokrotnie zaczynał sztuczną historię, przez co rozpraszał tok wykładu. Wersja v3 usuwa wymuszone opowieści i utrzymuje jeden przykład techniczny przez cały rozdział.

Wzorcowy rozdział dostarczony przez użytkownika służył jako przykład układu, nie jako źródło niezweryfikowanych faktów i wymyślonych outputów.

## 2. Kluczowe korekty faktograficzne

| Temat | Korekta w rozdziale v3 | Źródło producenta |
|---|---|---|
| Default BGP export | Typowy klasyczny Junos eksportuje aktywne trasy BGP, ale Direct/Static nie zaczynają automatycznie być reklamowane bez odpowiedniej policy | [Default Routing Policies](https://www.juniper.net/documentation/us/en/software/junos/routing-policy/topics/concept/policy-routing-policies-actions-defaults.html) |
| BGP rejected routes / \`keep all\` | Domyślnie Junos zwykle zachowuje odrzucone otrzymane BGP routes jako hidden; \`keep all\` nie jest warunkiem każdego podglądu hidden. \`keep none\` ma odrębne znaczenie | [keep (BGP)](https://www.juniper.net/documentation/us/en/software/junos/cli-reference/topics/ref/statement/keep-edit-protocols-bgp.html) |
| \`then local-preference\` | Akcja modyfikuje atrybut, ale sama nie gwarantuje terminalnej akceptacji; dalej może działać reject albo protocol default | [Actions in Routing Policy Terms](https://www.juniper.net/documentation/us/en/software/junos/routing-policy/topics/concept/policy-configuring-actions-in-routing-policy-terms.html) |
| \`route-filter\` | Najbardziej szczegółowy wzorzec w jednym termie, lokalne actions i opcjonalny \`walkup\` nie są prostą pierwszą linią ACL | [Route Filter Walkup](https://www.juniper.net/documentation/us/en/software/junos/routing-policy/topics/concept/walkup-route-filter-policy-option-overview.html) |
| Stateless filter count | W zwykłym stateless filter pasujący term z samym \`count\` może implicit accept; pakiety bez dopasowania na końcu filter trafiają do implicit discard | [Firewall Filter Components](https://www.juniper.net/documentation/us/en/software/junos/routing-policy/topics/concept/firewall-filter-components.html) |
| SRX \`lo0\` | Nie traktujemy przykładowego filtra \`PROTECT-RE\` jako bezpiecznej kompletnej konfiguracji bez sprawdzenia OOB, BGP, SSH, NTP i pozostałych host-bound flows | [Guidelines for Firewall Filters](https://www.juniper.net/documentation/us/en/software/junos/routing-policy/topics/concept/firewall-filter-stateless-guidelines-for-applying.html) |
| uRPF | Strict/loose i zachowanie default route/feasible paths zależą od platformy i topologii asymetrycznej; nie obiecujemy identycznego wyniku na vSRX/MX/EX | [Unicast RPF](https://www.juniper.net/documentation/us/en/software/junos/security-services/topics/topic-map/interfaces-configuring-unicast-rpf.html) |
| OSPF/IS-IS import | Nonexternal LSDB/routing nie da się dowolnie odrzucić przez \`import reject\`; wsparcie IS-IS import jest wersyjne | [OSPF policy](https://www.juniper.net/documentation/us/en/software/junos/ospf/topics/topic-map/configuring-ospf-routing-policy.html), [IS-IS import](https://www.juniper.net/documentation/us/en/software/junos/cli-reference/topics/ref/statement/import-edit-protocols-isis.html) |
| \`test policy\` | Nie jest pełnym symulatorem BGP peering/advertisement, operuje na właściwych istniejących trasach i ma własny default accept | [Routing Policy Tests](https://www.juniper.net/documentation/us/en/software/junos/routing-policy/topics/concept/policy-testing-routing-policies.html) |

## 3. Nowe laboratorium — DIAMOND-6R

Źródło prawdy: [README](../../labs/reference-diamond-6vsrx/README.md), [Mermaid](../../labs/reference-diamond-6vsrx/DIAMOND_TOPOLOGY.md).

- Sześć vSRX, bez VPCS w tym rozdziale.
- R5 CE-A zastępuje PC-A na \`192.0.2.10/24\`; R6 CE-B zastępuje PC-B na \`198.51.100.10/24\`.
- Wszystkie cztery łącza R1–R4 pozostają bez zmian: 10.12/30, 10.13/30, 10.24/30, 10.34/30.
- Dodatkowe źródła prefiksów \`172.16.50/51\` i \`172.16.60/61/62\` pozwalają pokazać rzeczywiste dwie drogi BGP, dwa różne LOCAL_PREF i odrzucone NLRI.
- Sześć peerings eBGP: R5–R1, R1–R2, R1–R3, R2–R4, R3–R4, R4–R6.
- R1 preferuje /60 górą R2 i /61 dołem R3; R4 preferuje odpowiadający ruch powrotny do /50 górą i /51 dołem.
- Statefulness SRX wymaga testowania określonych par source/destination; routing bez reverse-path nie gwarantuje odpowiedzi.
- Pięć kontrolowanych awarii: brak końcowego reject, neighbor overrides group, count→implicit accept, implicit discard na końcu filtra, security-policy deny. Dodatkowe osobne rozważania uRPF/lo0.

## 4. Statyczny audyt Markdown

- Rozdział v3 ma 14/14 sekcji blueprintu 8.1–8.14.
- Ok. 19 tys. słów, 8 diagramów Mermaid, 120 bloków CLI. Każdy blok Junos ma bezpośrednio wskazany router i tryb, lub jednoznaczną listę węzłów.
- Balans znaczników kodu sprawdzony; formatowanie inline backticks poprawione przed publikacją.
- Wszystkie diagramy są autorskie; nie kopiowano grafiki zewnętrznej. Nie przedstawiono żadnego fikcyjnego outputu CLI jako wyniku runtime.
- Referencyjna topologia 4×vSRX + 2×VPCS pozostaje dla starszych rozdziałów; bieżący 6R nie zmienia po cichu ich adresacji.

## 5. Co nadal trzeba zweryfikować w EVE-NG

Przed oznaczeniem LAB jako RUNTIME VALIDATED zebrać: wersję Junos każdego vSRX, port mapping, BGP sessions na wszystkich sześciu węzłach, route advertisements R5/R6, import/hidden na R1, dwa LOCAL_PREF i dwie symetryczne ścieżki, security flow sessions, działanie static firewall filters/count, zachowanie po \`commit confirmed\` i rzeczywisty rollback. Osobno potwierdzić dostępność uRPF, fail-filter i ochrony \`lo0\` na danym obrazie.

**Nie używać informacji o „runtime PASS” bez widocznych outputów tych testów.**

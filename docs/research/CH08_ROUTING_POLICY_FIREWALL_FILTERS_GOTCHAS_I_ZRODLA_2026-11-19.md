# CH08 — Routing Policy & Firewall Filters: źródła, gotchas i zakres walidacji

**Data kwerendy:** 19 listopada 2026  
**Podręcznik:** [JUNOS_Rozdzial_08_Routing_Policy_and_Firewall_Filters_Agent.md](../chapters/JUNOS_Rozdzial_08_Routing_Policy_and_Firewall_Filters_Agent.md)  
**Blueprint:** 8.1–8.14 [A → S → P]  
**Status:** SOURCE-REVIEWED / GENERATED; **NOT RUNTIME VALIDATED**. Nie uruchamiano Junos/vSRX ani EVE-NG.  
**Zasada oceny:** materiały publiczne Juniper/HPE mają pierwszeństwo przed community; nie używano exam dumps.

## Co zostało potwierdzone w źródłach

| Mechanizm | Pułapka dla inżyniera Cisco | Potwierdzenie | Gdzie omówiony |
|---|---|---|---|
| Domyślne BGP export | BGP bez export policy nie redystrybuuje dowolnie static/direct; default BGP redistributes active BGP routes według reguł protokołu | [Default Routing Policies](https://www.juniper.net/documentation/us/en/software/junos/routing-policy/topics/concept/policy-routing-policies-actions-defaults.html) | 8.1, 8.12 |
| OSPF/IS-IS default export | „Reject everything” dotyczy redistribution; nie tłumi automatycznie własnego LSA/LSP flooding | [Default Routing Policies](https://www.juniper.net/documentation/us/en/software/junos/routing-policy/topics/concept/policy-routing-policies-actions-defaults.html) | 8.1, 8.4, 8.12 |
| OSPF import | OSPF import policy jest przeznaczona dla tras external, nie zwykłego intra-area topology | [Protocol Support](https://www.juniper.net/documentation/us/en/software/junos/routing-policy/topics/concept/policy-protocol-support-import-export-policies.html) | 8.4, 8.12 |
| Terminating actions | `accept` i `reject` kończą przetwarzanie polityk, `next term` oraz `next policy` zmieniają flow | [Policy Actions](https://www.juniper.net/documentation/us/en/software/junos/routing-policy/topics/concept/policy-default-actions-default-policies.html), [Evaluation](https://www.juniper.net/documentation/us/en/software/junos/routing-policy/topics/concept/policy-routing-policies-evaluation-method.html) | 8.3 |
| Term bez conditions | Dopasowuje wszystkie rozpatrywane trasy; wcześniejszy match-all accept powoduje shadowing | [Policy Actions](https://www.juniper.net/documentation/us/en/software/junos/routing-policy/topics/concept/policy-default-actions-default-policies.html) | 8.5 |
| BGP inbound reject | Rejected NLRI może pozostać jako hidden; `keep all/none` wpływa na retencję i zmianę import | [import (BGP)](https://www.juniper.net/documentation/us/en/software/junos/cli-reference/topics/ref/statement/import-edit-protocols-bgp.html) | 8.2, 8.14 |
| BGP advertised view | `show route advertising-protocol bgp ...` przedstawia export protocol-side; nie jest packet capture na wire | [CLI Reference](https://www.juniper.net/documentation/us/en/software/junos/cli-reference/topics/ref/command/show-route-advertising-protocol.html) | 8.2, 8.11, 8.14 |
| Route-filter LPM | W obrębie tego samego termu longest matching prefix może „przysłonić” szerszą regułę `orlonger`, gdy najdłuższy qualifier zawiedzie | [Route Filters Evaluation](https://www.juniper.net/documentation/us/en/software/junos/routing-policy/topics/concept/policy-configuring-route-lists-for-use-in-routing-policy-match-conditions.html) | 8.6 |
| `prefix-list` vs `prefix-list-filter` | Pierwszy użyty jako routing policy match nie działa automatycznie `orlonger` | [Prefix Lists](https://www.juniper.net/documentation/us/en/software/junos/routing-policy/topics/concept/policy-configuring-prefix-lists-for-use-in-routing-policy-match-conditions.html) | 8.6 |
| Stateless filter implicit discard | Brak ostatniego pasującego termu kończy się packet drop, inaczej niż routing policy defaults | [Evaluation](https://www.juniper.net/documentation/us/en/software/junos/routing-policy/topics/concept/firewall-filter-stateless-evaluate-packets.html) | 8.8, 8.9 |
| Nonterminating count/log | Term z samym `then count`/`log` może zakończyć się niejawnym accept; konieczny explicit `next term`, jeśli kolejny term ma widzieć pakiet | [Firewall Filter Components](https://www.juniper.net/documentation/us/en/software/junos/routing-policy/topics/concept/firewall-filter-components.html) | 8.8, 8.9 |
| Firewall filter family | `inet` i `inet6` są oddzielne; nie wolno uogólniać obsługi ACL IPv4 na IPv6 | [Application Points](https://www.juniper.net/documentation/us/en/software/junos/routing-policy/topics/concept/firewall-filter-application-points.html) | 8.9 |
| uRPF strict/loose | RPF wykonuje reverse lookup **source IP** i może legalnie odrzucić ruch asymetryczny | [Understanding uRPF](https://www.juniper.net/documentation/us/en/software/junos/security-services/topics/topic-map/interfaces-configuring-unicast-rpf.html) | 8.10 |
| uRPF default | uRPF jest domyślnie disabled; obsługa ECMP i trybu różni się według platformy | [rpf-check CLI](https://www.juniper.net/documentation/us/en/software/junos/cli-reference/topics/ref/statement/rpf-check-edit-interfaces.html) | 8.10 |
| SRX stateful policy | Stateless accept nie bypassuje SRX security policies/flow/zone | [SRX Traffic Processing](https://www.juniper.net/documentation/us/en/software/junos/flow-packet-processing/topics/topic-map/security-srx-devices-processing-overview.html) | 8.7, 8.9, 8.14 |
| SRX BGP host-inbound | Established wymaga host-inbound protocol dla peeringu na strefowym interfejsie | [BGP Peering Sessions](https://www.juniper.net/documentation/us/en/software/junos/bgp/topics/topic-map/bgp-peering-sessions.html) | 8.14 |
| `test policy` | Domyślne accept dla narzędzia nie jest pełną symulacją protokołu ani wszystkich reguł export | [Understanding Routing Policy Tests](https://www.juniper.net/documentation/us/en/software/junos/routing-policy/topics/concept/policy-testing-routing-policies.html) | 8.14, warsztat |
| Hardware counters | Filter configured nie oznacza bezbłędnej instalacji w ASIC; statystyki zależą od platformy | [Installation Status](https://www.juniper.net/documentation/us/en/software/junos/routing-policy/topics/concept/display_firewall_filter_installation_status.html) | 8.11, warsztat |
| Routing BGP policy jako whitelist | Dwustronnie jawny końcowy reject pozwala wymagać testu negatywnego i ogranicza leak | [Day One Configuring Routing Policies](https://www.juniper.net/documentation/en_US/day-one-books/topics/topic-map/configuring-routing-policies.html) | 8.3–8.6, 8.14 |

## Kwerenda community jako sygnał problemu (nie autorytet)

Wykorzystano publiczne zagadnienia o zachowaniu `test policy`, kolejności termów i problemach podczas route-filter evaluation jako pomysł na pytania aktywizujące, a następnie porównano je z powyższą dokumentacją producenta.

- https://community.juniper.net/discussion/cli-moving-policy-terms-around
- https://community.juniper.net/discussion/policy-subroutines
- https://community.juniper.net/discussion/verifing-route-filters

Wnioski z forum nie są w podręczniku przedstawiane jako oficjalne pytania egzaminacyjne.

## Prace, których **nie** wykonano

- Brak uruchomionych vSRX, EVE-NG i hostów Linux dla labu 8.14.
- Brak automatycznej walidacji składni na konkretnym Junos release.
- Brak dowodów z `show route advertising-protocol`, `show firewall filter` oraz `show security flow session` po realnym uruchomieniu.
- Nie weryfikowano capacity/ASIC dla dowolnego EX/QFX ani supportu konkretnego SKU i Junos.
- Nie wykonywano commitów na urządzeniach, żadnych awarii ani capture ruchu.

Przykładowy status testu labu to **NOT TESTED**, nie PASS. Konfiguracje wymagają testów na wybranym obrazie, szczególnie mapowanie ge-*/fxp*, instalacja filtrów i przetwarzanie pakietów SRX.

## Zalecana kontrola przed uznaniem RUNTIME VALIDATED

1. Zapisz wersję obrazu vSRX i faktyczne mapowanie portów w EVE.
2. Ustal baseline i prowadź oddzielną konsolę do obu vSRX.
3. Uruchom `commit check` dla fragmentów, zastosuj `commit confirmed` i potwierdzaj po postcheck.
4. Potwierdź eBGP Established + reklamowane/pobrane prefixes; test dodatni i ujemny whitelisty.
5. Potwierdź widoczność odrzuconego TEST-ONLY jako hidden (jeśli właściwa retencja), a nie aktywnego route.
6. Zweryfikuj PC-A↔PC-B: output `show security flow session` i route return, nie tylko ping z RE.
7. Wyślij kontrolowane pakiety z testowego źródła, zmierz delta counter; sprawdź IPv4 i odrębnie IPv6 jeśli włączone.
8. Zapisz dowody, PASS/FAIL/NOT TESTED dla każdego etapu i wykonaj rollback do określonego baseline.

**Redakcyjna zasada granic:** szczegółowy OSPF, IS-IS i BGP należą do rozdziałów 9–11, RIB groups i FBF do rozdziału 7, L2 security do rozdziału 16. Rozdział 8 uczy logiki polityk i filtrów oraz ich diagnostycznego użycia w tych technologiach.

# CH07 — Protocol-Independent Routing: rejestr źródeł, gotchas i zakres weryfikacji

**Data badań:** 18 listopada 2026.  
**Blueprint:** [7.1–7.8 [S]](../JUNOS_JNCIA_JNCIS_JNCIP_unified_blueprint.md).  
**Rozdział ucznia:** [Protocol-Independent Routing](../chapters/JUNOS_Rozdzial_07_Protocol_Independent_Routing.md).

Rejestr jest **materiałem dla autora/Codexa**, nie osobnym bankiem pytań. Pułapki zostały **wplecione w właściwe sekcje rozdziału**, gdzie służą wyjaśnieniu przyczynowemu, Cisco ↔ Junos i testom CLI. Nie ujawniamy ani nie kopiujemy exam dumps.

## 21 istotnych pułapek z kwerendy

| Nr | Miejsce w wykładzie | Gotcha / konsekwencja | Źródło |
|---:|---|---|---|
| 1 | 7.1 | `aggregate` wymaga co najmniej jednej active more-specific contributing route; samo poprawne `commit check` nie tworzy aktywnego summary. | [Aggregation](https://www.juniper.net/documentation/us/en/software/junos/static-routing/topics/topic-map/config-route-aggregation.html) |
| 2 | 7.1 | Aggregate ma domyślnie `reject` next-hop; pakiet do nieobsługiwanego podprefiksu summary może otrzymać ICMP unreachable, zamiast iść default. | [Aggregation](https://www.juniper.net/documentation/us/en/software/junos/static-routing/topics/topic-map/config-route-aggregation.html) |
| 3 | 7.1 | Aggregate na routerze źródłowym nie zmniejsza jego własnego RIB; redukcja u sąsiadów następuje przy **export policy** ogłaszającej summary zamiast specifics. | [Aggregation](https://www.juniper.net/documentation/us/en/software/junos/static-routing/topics/topic-map/config-route-aggregation.html) |
| 4 | 7.1 | Generated route również potrzebuje contributor, ale jej next-hop może pochodzić od **primary contributor**; różni się tym od bezpiecznego aggregate reject. | [Generated Routes](https://www.juniper.net/documentation/us/en/software/junos/static-routing/topics/concept/policy-generated-routes.html) |
| 5 | 7.1 | Aggregate i generated mają domyślnie preference 130 i obie mogą być wyświetlane w `show route` jako `[Aggregate/130]`. | [preference](https://www.juniper.net/documentation/us/en/software/junos/cli-reference/topics/ref/statement/preference-edit-routing-options.html), [Community](https://community.juniper.net/discussion/aggregate-vs-generate-routes) |
| 6 | 7.1 | Contributor to **active more-specific**. Osiągalność tylko jednego podprefiksu nie dowodzi osiągalności wszystkich /24 w summary. | [Aggregation](https://www.juniper.net/documentation/us/en/software/junos/static-routing/topics/topic-map/config-route-aggregation.html) |
| 7 | 7.2 | `show route martians` zawiera `0.0.0.0/0 exact allowed`; **default /0 NIE jest zablokowana**, mimo że `0.0.0.0/8 orlonger` jest disallowed. | [show martians](https://www.juniper.net/documentation/us/en/software/junos/cli-reference/topics/ref/command/show-route-martians.html), [Community](https://community.juniper.net/discussion/why-is-the-default-route-in-the-default-martian-table) |
| 8 | 7.2 | RFC1918 (10/8, 172.16/12, 192.168/16) nie jest automatycznie martian; martian recognition nie jest packet filter/uRPF ani BGP import policy. | [Martian Recognition](https://www.juniper.net/documentation/us/en/software/junos/static-routing/topics/topic-map/recognize-martian-addr-routing.html) |
| 9 | 7.2 | `exact` vs `orlonger` zmienia zakres prefixów; `0/0 exact allow` nie oznacza przyjęcia wszystkich more-specific. | [martians](https://www.juniper.net/documentation/us/en/software/junos/cli-reference/topics/ref/statement/martians-edit-routing-options.html) |
| 10 | 7.3 | Routing instance `virtual-router` ≠ typ `vrf` L3VPN ≠ `forwarding` dla FBF; pakiety w FBF lookup mogą wymagać connected routes w secondary RIB. | [Routing Instances](https://www.juniper.net/documentation/us/en/software/junos/routing-overview/topics/concept/routing-instances-overview.html), [FBF Example](https://www.juniper.net/documentation/us/en/software/junos/routing-policy/topics/example/filter-based-forwarding-example.html) |
| 11 | 7.4 | RIB group definicja bez przypięcia do źródła (\`interface-routes rib-group\` itd.) nie kopiuje tras automatycznie. | [RIB Groups](https://www.juniper.net/documentation/us/en/software/junos/cli-reference/topics/ref/statement/rib-groups-edit-routing-options.html) |
| 12 | 7.4 | Pierwsza `import-rib` to **primary**, kolejne secondary; usunięcie primary route może usunąć jej secondary copy. | [import-rib](https://www.juniper.net/documentation/us/en/software/junos/cli-reference/topics/ref/statement/import-rib-edit-routing-options.html) |
| 13 | 7.4 | Na EX Series dokumentacja zawiera ograniczenia importu route types (w szczególności tylko dynamicznie learned w określonych kontekstach); nie przenosić bez testu labu vSRX/MX na EX. | [import-rib](https://www.juniper.net/documentation/us/en/software/junos/cli-reference/topics/ref/statement/import-rib-edit-routing-options.html) |
| 14 | 7.4 | RIB group może udostępnić więcej connected routes niż zamierzono, zmieniając forwarding/security boundaries; import-policy wymaga testów także na primary. | [Juniper RIB Groups Technical Note](https://www.juniper.net/documentation/us/en/software/nce/rg-understanding-tn/rg-understanding-tn.pdf) |
| 15 | 7.5 | W Junos legacy `load-balance per-packet` w częstych wariantach to **per-flow hash**, nie packet-by-packet spray. Istnieją inne tryby spray. | [Per-Packet Load Balancing](https://www.juniper.net/documentation/us/en/software/junos/sampling-forwarding-monitoring/topics/concept/policy-configuring-per-packet-load-balancing.html) |
| 16 | 7.5 | Multiple equal-cost next hops w RIB nie gwarantuje automatycznie multiple next hops w FIB; policy `forwarding-table export` i platforma mają znaczenie. | [Per-Flow Load Balancing](https://www.juniper.net/documentation/us/en/software/junos/sampling-forwarding-monitoring/topics/concept/policy-configuring-per-flow-load-balancing.html) |
| 17 | 7.5 | `forwarding-table export policy` ≠ BGP export policy. Policy nieaktywowana odpowiednim attachment nic nie zmienia. | [Per-Flow LB](https://www.juniper.net/documentation/us/en/software/junos/sampling-forwarding-monitoring/topics/concept/policy-configuring-per-flow-load-balancing.html) |
| 18 | 7.5 | Jeden ogromny TCP flow i mała różnorodność hash fields mogą przeciążyć jeden link mimo poprawnego ECMP; `hash-key` jest platform-dependent. | [hash-key](https://www.juniper.net/documentation/us/en/software/junos/cli-reference/topics/ref/statement/hash-key-edit-forwarding-options.html) |
| 19 | 7.6 | FBF = stateless firewall filter na ingress + action `then routing-instance` + instance typu `forwarding` + resolved next-hop (często przez RIB group). | [FBF Example](https://www.juniper.net/documentation/us/en/software/junos/routing-policy/topics/example/filter-based-forwarding-example.html) |
| 20 | 7.6 | Brak końcowego `then accept` dla ruchu niepasującego do FBF może skutkować implicit discard. `accept` stateless filter to nie SRX stateful permit. | [FBF Overview](https://www.juniper.net/documentation/us/en/software/junos/routing-policy/topics/concept/firewall-filter-option-filter-based-forwarding-overview.html) |
| 21 | 7.6 | Kombinacje `then routing-instance` z `count/log/policer` oraz IPv6 L4 FBF restrictions różnią się per platform; nie stosować ACX-specific ograniczeń jako ogólnych dla EX/MX/SRX. | [ACX FBF limitations](https://www.juniper.net/documentation/us/en/software/junos/routing-policy/topics/concept/filter-based-forwarding-acx-series.html), [Feature Explorer](https://apps.juniper.net/feature-explorer/) |

## Prawdziwe dyskusje vs autorskie pytania

**Weryfikowalne, publiczne wątki:**
- [Aggregate VS Generate routes](https://community.juniper.net/discussion/aggregate-vs-generate-routes) — pytanie o semantykę tych typów i prezentację w CLI.
- [Aggregate Routes vs Generated Routes](https://community.juniper.net/discussion/aggregate-routes-vs-generated-routes?hlmlt=VT) — opis podobnych labeli `Aggregate/130`.
- [Why is the default route in the default Martian table?](https://community.juniper.net/discussion/why-is-the-default-route-in-the-default-martian-table) — naprawdę zadane pytanie o `0.0.0.0/0`.
- [Filter Based Forwarding on SRX](https://community.juniper.net/discussion/filter-based-forwarding-on-srx) — pytania o SRX/PBR, *inspiracja*, nie dowód wsparcia we wszystkich release.

**Pytania z podręcznika poza tymi referencjami:** autorskie scenariusze bazujące na dokumentacji producenta; nie rościmy sobie dostępu do prawdziwych pytań egzaminacyjnych ani poufnych baz.

## Trudne granice techniczne, które trzeba zachować

- **Nie uniwersalizuj EX/QFX/MX/SRX.** Wsparcie FBF, rib-group, hashing, liczba routing instances i action combinations może się zmieniać z modelem i Junos release.
- **Nie traktuj `show route forwarding-table` jako raw ASIC proof.** W razie podejrzenia problemy na PFE użyj platformowych narzędzi i wsparcia JTAC. [show route forwarding-table](https://www.juniper.net/documentation/us/en/software/junos/cli-reference/topics/ref/command/show-route-forwarding-table.html).
- **RIB group import-policy** jest narzędziem ingerującym w import tras. Ograniczanie connected w main/secondary wymaga realnego testu, by przypadkowo nie usunąć routes z primary.
- **FBF na vSRX** może mieć inne punkty przetwarzania niż na MX/QFX; sprawdź Feature Explorer, release, flow/SRX policies i sesje. Nie zakładaj zero-impact.
- **Martians** nie zastępują ingress ACL/uRPF, a `allow` dla eksperymentalnych prefixów nie stanowi rekomendacji produkcyjnej.
- **ECMP** nie oznacza dokładnego 50/50 przy jednym flow; per-packet i packet-spray są różne.
- **LAB CH07:** nie mamy realnego `.unl` ani wykonanych testów. Expected observations ≠ PASS.

## Weryfikacja laboratorium

Topologia planowana: 1×vSRX (R1, 3 data NIC), 2×Linux clients, 2×Linux upstream gateways; odizolowane segments OFFICE, WAN-A, WAN-B; dwa upstreams mają wspólny, **testowy loopback anycast 203.0.113.10/32** dla obserwacji FBF i ECMP.

- Sprawdzić QEMU NIC → Junos ge/fxp mapping.
- Upewnić się, że nie ma inherited factory NAT i że SRX policies dopuszczają testowe ICMP do obu WAN zones.
- Testować baseline dwóch klientów przez WAN-A, potem aggregate/generate, RIB groups i FBF w oddzielnych krokach.
- Zbierać `show route`, `show route table FBF-ALT.inet.0`, FIB, ARP, SRX sessions oraz tcpdump po obu stronach.
- Fault injections: brak contributorów, usunięte interface-routes RIB group, brak final filter accept, generated forwarding do nieistniejącego hosta, osobny ECMP wariant.
- Stosować `show | compare` i `commit check` **przed** `commit confirmed`. Nie wykonywać `commit check` podczas pending timer.
- Nie publikować haseł, obrazów, tokenów, danych klienta ani raw full configurations z sekretami.
- Status na dziś: **PLANNED / NOT RUNTIME VALIDATED**.

**Na przyszłość:** przy pisaniu rozdziałów 8+ kontynuuj wyszukiwanie gotchas i realnych pytań community, ale przede wszystkim **wyjaśniaj je w miejscach mechanizmów, a nie dopisuj osobny bezkontekstowy bank zadań**.

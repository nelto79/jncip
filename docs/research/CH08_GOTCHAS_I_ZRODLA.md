# Rozdział 8 — rejestr źródeł, wyjątków i weryfikacji redakcyjnej

**Data:** 2026-11-19  
**Rozdział:** `docs/chapters/JUNOS_Rozdzial_08_Routing_Policy_and_Firewall_Filters.md`  
**Status:** SOURCE REVIEWED / STATIC EDITORIAL REVIEW / NOT RUNTIME VALIDATED. Nie wykonano testów na urządzeniach ani w EVE-NG.  
**Cel:** rejestr dla autorów, nie osobny fragment materiału ucznia. Haczyki egzaminacyjne są wyjaśnione w toku właściwych lekcji.

| Wniosek wymagający uwagi | Jak został pokazany w rozdziale | Źródło Juniper | Dalsza walidacja |
|---|---|---|---|
| BGP default export nie oznacza automatycznej redystrybucji connected/static; reguły domyślne zależą od protokołu, kierunku i platformy | 8.1: R1 ma direct, BGP Established, brak reklamy | [Default Routing Policies](https://www.juniper.net/documentation/us/en/software/junos/routing-policy/topics/concept/policy-routing-policies-actions-defaults.html) | Sprawdzić wydanie/platformę, zwłaszcza PTX |
| BGP import, hidden, inactive, active RIB i FIB to nie to samo | 8.2, 8.4, LAB 8.14: separacja dowodu nadawcy i odbiorcy | [Understanding Routing Policies](https://www.juniper.net/documentation/us/en/software/junos/routing-policy/topics/concept/policy-routing-policies-overview.html), [Basic BGP Policies](https://www.juniper.net/documentation/us/en/software/junos/bgp/topics/topic-map/basic-routing-policies.html) | Zrzuty live `receive-protocol hidden` / `advertising-protocol` |
| Accept/reject kończą ocenę, next term/next policy kierują dalszą analizą | 8.3: dwuetapowe oznaczenie community i whitelista | [Evaluation Method](https://www.juniper.net/documentation/us/en/software/junos/routing-policy/topics/concept/policy-routing-policies-evaluation-method.html), [Policy Chain](https://www.juniper.net/documentation/us/en/software/junos/routing-policy/topics/concept/policy-routing-policies-chain-evaluation-method.html) | `test policy` i testy BGP |
| `route-filter` używa najbardziej szczegółowego wzorca; /16 exact może przesłonić /8 orlonger w tym samym termie | 8.6: konkretny prefix /24 i drzewo wzorców | [Walkup Route Filter](https://www.juniper.net/documentation/us/en/software/junos/routing-policy/topics/concept/walkup-route-filter-policy-option-overview.html) | Test bez walkup i z walkup na wspieranym release |
| `test policy` nie jest pełnym symulatorem eksportu BGP i ma własny default-accept | 8.6 i 8.11: porównanie z `advertising-protocol` | [Understanding Routing Policy Tests](https://www.juniper.net/documentation/us/en/software/junos/routing-policy/topics/concept/policy-testing-routing-policies.html) | Weryfikacja outputów i kolejności w release |
| Stateless firewall filter ma implicit discard na końcu; pasujący term z samym `count` może zakończyć się implicit accept | 8.8: przypadek shadowing i domknięcie whitelisty | [Firewall Filter Components](https://www.juniper.net/documentation/us/en/software/junos/routing-policy/topics/concept/firewall-filter-components.html) | Liczniki i różnice EX/QFX/MX/vSRX |
| SRX security policy jest stateful i nie jest odpowiednikiem routing policy ani stateless filter | 8.7–8.9 i LAB faza 6 | [Security Policies](https://www.juniper.net/documentation/us/en/software/junos/security-policies/topics/concept/policy-overview.html) | Nowe sesje i filtry na konkretnym vSRX |
| Strict uRPF może odrzucać legalny ruch przy asymetrycznej ścieżce powrotnej | 8.10: dwa ISP i reverse path do źródła | [Unicast RPF](https://www.juniper.net/documentation/us/en/software/junos/security-services/topics/topic-map/interfaces-configuring-unicast-rpf.html) | Model ASIC/PFE, FBF/ECMP/VRF i `rpf-check` |
| OSPF import reject nie usuwa dowolnej nonexternal route z LSDB/RIB | 8.4, 8.12: odróżnienie link-state od redystrybucji | [OSPF Routing Policy](https://www.juniper.net/documentation/us/en/software/junos/ospf/topics/topic-map/configuring-ospf-routing-policy.html) | Sprawdzić external vs. internal na obrazie użytym w labie |
| IS-IS import istnieje w nowoczesnym Junos, lecz reject nonexternal nie jest uniwersalnym filtrem link-state | 8.12: przykład eksportu static i wyjaśnienie import | [IS-IS import CLI Reference](https://www.juniper.net/documentation/us/en/software/junos/cli-reference/topics/ref/statement/import-edit-protocols-isis.html), [Route Prioritization](https://www.juniper.net/documentation/us/en/software/junos/is-is/topics/example/example-configuring-routing-policy-to-prioritize-isis-routes.html) | Obsługa w konkretnej wersji Junos |
| LOCAL_PREF, route preference, MED i AS-PATH nie są zamiennymi sposobami sterowania ruchem | 8.13: ISP-A/ISP-B i kierunek własnego vs. przychodzącego ruchu | [BGP Policies](https://www.juniper.net/documentation/us/en/software/junos/bgp/topics/topic-map/basic-routing-policies.html), [Route Preferences](https://www.juniper.net/documentation/us/en/software/junos/routing-overview/topics/concept/routing-protocols-route-preferences-overview.html) | Lab z dwoma eBGP peers i realnym best path |

## Audyt po przebudowie

- Zakres **8.1–8.14** zachowany w nowym pliku.
- Przebudowana narracja opiera się na jednej firmowej topologii, stopniowanym problemie operacyjnym, a następnie czterech kontrolowanych usterkach; oddzielne dodatkowe scenariusze dla OSPF/IS-IS i dual ISP są jawnie oznaczone.
- Każdy blok poleceń ma bezpośrednio przed sobą nazwę urządzenia i tryb.
- Mermaid to autorskie diagramy tekstowe, bez cudzych obrazów i bez licencji grafiki wymagającej przeniesienia.
- Wszystkie wyniki urządzeń to **przewidywania**, a nie dowody uzyskanego runtime.
- Testy do wykonania: walidacja mapowania NIC vSRX, rzeczywistych poleceń CLI na użytym release, eBGP import/export, `hidden` route, stateless firewall counter i SRX stateful flow, bezpieczny cleanup.
- Nie traktować źródeł publicznych jako dowodu działania każdego polecenia na każdej platformie.

## Stan

**Editorial check completed** — logiczny przegląd zakresu, struktury, nagłówków CLI i równowagi bloków Markdown. **Runtime testing outstanding**. Nie nadpisano poprzedniego rozdziału `JUNOS_Rozdzial_08_Routing_Policy_and_Firewall_Filters_Agent.md`.

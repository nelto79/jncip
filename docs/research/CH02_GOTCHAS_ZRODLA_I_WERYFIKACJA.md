# Chapter 2 — gotchas: źródła, kontekst i weryfikacja (notatka redakcyjna)

**Kwerenda:** 2026-11-16. **Cel:** wspierać pełny, narracyjny podręcznik [Junos OS Fundamentals](../chapters/JUNOS_Rozdzial_02_Architektura_Junos_OS.md).

**To nie jest osobny zestaw materiałów do wkucia.** Właściwe wyjaśnienia znajdują się już wewnątrz wykładu: przed mechanizmem, który sprawia trudność, lub obok niego. Ten rejestr pozostaje do kontroli technicznej i weryfikacji przez Codexa.

## Udokumentowane gotchas

1. **Cisco eBGP vs OSPF i Junos BGP vs OSPF (2.3):** IOS/IOS XE domyślne eBGP AD 20, OSPF AD 110; Junos OSPF internal preference 10, OSPF external 150, BGP preference 170. Dla *tego samego prefiksu* mogą wybrać różne protokoły. Rozróżniać route preference od BGP LOCAL_PREF. Źródło producenta: https://www.juniper.net/documentation/us/en/software/junos/bgp/topics/topic-map/local-preference.html
2. **Hidden vs inactive vs niezaimportowana trasa (2.3):** nie zawsze jest to ten sam stan. Hidden route bywa nieużywalna z powodu next-hop lub policy; niektóre route są odrzucane wcześniej niż etap tworzenia hidden. Źródła: https://www.juniper.net/documentation/us/en/software/junos/bgp/topics/topic-map/troubleshooting-bgp-sessions.html , https://www.juniper.net/documentation/us/en/software/junos/cli-reference/topics/ref/command/show-route-hidden.html
3. **Master forwarding table na RE vs PFE (2.4):** wpis w \`show route forwarding-table\` i \`sent to PFE\` nie są automatycznie potwierdzeniem przekazywania konkretnego pakietu. Źródło: https://www.juniper.net/documentation/us/en/software/junos/cli-reference/topics/ref/command/show-route-forwarding-table.html
4. **Stały \`default\` next-hop / \`perm rjct\` bez użytkowej trasy 0/0 (2.4):** wpis permanentny w tabeli forwarding nie dowodzi domyślnego routingu do Internetu. Źródło: https://www.juniper.net/documentation/en_US/day-one-books/junos-beginners-guide.pdf
5. **\`reject\` vs \`discard\` (2.4):** oba odrzucają pakiety, lecz reject wysyła informację ICMP unreachable, a discard nie. Źródło: https://www.juniper.net/documentation/us/en/software/junos/cli-reference/topics/ref/statement/static-edit-routing-options.html
6. **SRX intra-zone i factory defaults (2.5):** ruch między interfejsami tej samej strefy podlega ocenie policy, ale fabryczne urządzenie z preinstalowaną konfiguracją trust/untrust może już mieć dozwolone przepływy. Nie uogólniaj domyślnej konfiguracji modelu na całą rodzinę. Źródła: https://www.juniper.net/documentation/us/en/software/junos/security-policies/topics/concept/policy-overview.html , https://www.juniper.net/documentation/us/en/guided-setup/branch-srx-gs/topics/topic-map/step-1-p1-verify_defaults.html
7. **Interface host-inbound ma pierwszeństwo przed zone host-inbound (2.6); system-services vs protocols (2.6):** \`system-services ping\` to nie \`protocols bgp\`. Źródła: https://supportportal.juniper.net/s/article/SRX-Getting-Started-Configure-Management-Access , https://www.juniper.net/documentation/us/en/software/junos/bgp/topics/topic-map/bgp-overview.html
8. **Dedykowany management port fxp0 nie oznacza automatycznej osobnej VRF (2.6):** default routing table może być współdzielona, jeśli nie skonfigurowano dedykowanej instancji zarządzania. Źródło: https://www.juniper.net/documentation/us/en/software/junos/junos-getting-started/topics/topic-map/management-interface-in-non-default-instance.html
9. **Zmiana policy, deaktywacja i istniejące sesje na SRX (2.LAB):** \`policy-rematch\` jest domyślnie wyłączony, ale polityka usunięta/przemianowana/dezaktywowana ma inne skutki dla sesji niż edycja jej warunków. Podręcznik koryguje wcześniejszy skrót w LAB-12. Źródło: https://www.juniper.net/documentation/us/en/software/junos/cli-reference/topics/ref/statement/security-edit-policy-rematch.html

## Jak interpretować status źródeł

- Powyższe punkty mają **źródła w dokumentacji Juniper/HPE**, a nie wyłącznie w anegdotach z forów.
- Takie różnice **mogą być sprawdzane** na certyfikacji, ale **brak dowodu**, że konkretne zdanie pochodzi z rzeczywistego lub poufnego egzaminu. Nie przypisuj go oficjalnemu question bank.
- Część tematów (SRX stateful security, system management, BGP OSPF preference) wychodzi poza literalną treść celu Associate 2.1–2.6; są to wyraźnie dydaktyczne rozszerzenia pod Specialist/Professional, a nie nowe cele JNCIA.
- Komendy i dokładne efekty testuj na wersji obrazu używanej w EVE-NG. Nie wprowadzaj konfiguracji z tego rejestru do produkcji.
- Zasada przygotowania materiału: przykład Cisco → błędna intuicja → zachowanie Junos → dowód CLI → skutek praktyczny → źródło. Umieszczaj bezpośrednio w lekcji, nie w „zrzucie” listy pułapek na końcu.

## Zalecana kwerenda dla każdego kolejnego rozdziału

Przed redakcją: przeszukaj aktualną oficjalną dokumentację Juniper/HPE, CLI Reference, Day One i odpowiednie KB; sprawdź historyczne oraz nowsze dyskusje community. Szukaj zwłaszcza: **default vs factory-default**, **hidden vs inactive**, **ROUTING vs FORWARDING**, **protocol-dependent vs platform-dependent**, **preferencje i tie-break**, **session / established state**, **dowód CLI a rzeczywisty efekt**, **różnice IOS/IOS XE vs Junos**. Każdy wybrany haczyk zweryfikuj w dokumentacji producenta.

Stan: research checked against cited official public documentation; lab runtime validation not performed.

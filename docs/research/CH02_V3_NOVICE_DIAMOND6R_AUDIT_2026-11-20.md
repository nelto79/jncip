# Rozdział II v3 — audyt dydaktyczny i status walidacji

**Data:** 2026-11-20  
**Tekst kanoniczny:** `docs/chapters/JUNOS_Rozdzial_02_Architektura_Junos_OS.md`  
**Archiwum:** `docs/archive/JUNOS_Rozdzial_02_pre_novice_rewrite_2026-11-20.md`  
**Odbiorca:** Network Engineer ~1 rok doświadczenia, znajomość wyłącznie podstaw IP, VLAN, switch, router, gateway. Nie zakłada się Cisco/CCNA/JNCIA.

## Pokrycie blueprintu

| Pozycja | Miejsce w nowym rozdziale | Główna kompetencja |
|---|---|---|
| 2.1 Architektura Junos | §2.1 | Rozumienie RE/PFE i procesów rpd, mgd, chassisd, dcd |
| 2.2 Control/forwarding | §2.2 | Oddzielenie przygotowania tras od przekazywania pakietów |
| 2.3 Routing Engine | §2.3 | Direct, Local, Static, RIB, aktywna trasa, FIB |
| 2.4 PFE | §2.4 | Ethernet/ARP, next-hop, LPM, TTL, forwarding |
| 2.5 Transit | §2.5 | R5→R1→R2; source, destination i return path |
| 2.6 Exception | §2.6 | Host-bound, locally originated, transit, punt, host-inbound |
| Rozszerzenie LAB | §2.LAB | 2 vSRX → 3 vSRX → DIAMOND-6R, awarie i rollback |
| Rozszerzenie troubleshooting | §2.TS | Stopniowe lokalizowanie punktu awarii |
| Rozszerzenie pytania | §2.TEST | 10 pytań z odpowiedziami i wyjaśnieniami |

## Kontrola statyczna

- **13 294 słowa**, około 100 tys. znaków, 7 diagramów Mermaid, 66 bloków CLI.
- Zachowano wszystkie 6 głównych punktów blueprintu oraz dołożono laboratorium, troubleshooting i pytania z odpowiedziami.
- Każdy blok `~~~junos` ma bezpośrednio nad nim hostname R1–R6 i odpowiedni tryb.
- Brak niezakończonych bloków kodu i wyświetlanych dosłownie escape'owanych znaczników Markdown.
- Adresacja core 10.12.0.0/30, 10.13.0.0/30, 10.24.0.0/30, 10.34.0.0/30 oraz LAN-A / LAN-B zgodna z referencyjnym DIAMOND-6R.
- Podział poziomów laboratorium chroni początkującego przed nadmierną liczbą pojęć w pierwszej lekcji.
- **Nie testowano w EVE-NG ani na vSRX.** Nie przyznano statusu RUNTIME VALIDATED.

## Weryfikacja wymagająca urządzeń

1. Zidentyfikować release Junos oraz rzeczywiste mappingi ge-0/0/x w EVE-NG, osobno od management.
2. Potwierdzić składnię `commit confirmed`, host-inbound-traffic i stref po inspekcji istniejącej konfiguracji.
3. Sprawdzić efekt ruchu host-bound i transit na R1–R2, na nowych sesjach SRX.
4. Dla loopbacków z sieciami /24 potwierdzić zgodność release. Ogólna dokumentacja Junos dopuszcza subnet routes na lo0; dokumentacja SRX preferuje /32. Jeżeli potrzebny wariant /32 plus static discard /24, oznaczyć go jawnie w labie i sprawdzić wpływ na routing/polityki.
5. Sprawdzić oba kierunki statycznego routingu przy R5–R6 oraz security flow na R1, R2/R3, R4 i host-inbound R6.
6. Przeprowadzić i udokumentować controlled fault injection i automatyczne rollback, bez modyfikowania innych laboratoriów.

## Źródła

- [Junos Software Architecture](https://www.juniper.net/documentation/us/en/software/junos/junos-overview/topics/concept/junos-software-architecture.html)
- [Routing Engine Components](https://www.juniper.net/documentation/us/en/software/junos/junos-overview/topics/concept/junos-software-components-and-processes.html)
- [Routing Databases](https://www.juniper.net/documentation/us/en/software/junos/routing-overview/topics/concept/routing-protocols-routing-databases-overview.html)
- [show route forwarding-table](https://www.juniper.net/documentation/us/en/software/junos/cli-reference/topics/ref/command/show-route-forwarding-table.html)
- [SRX Traffic Processing](https://www.juniper.net/documentation/us/en/software/junos/flow-packet-processing/topics/topic-map/security-srx-devices-processing-overview.html)
- [Junos Loopback Interfaces](https://www.juniper.net/documentation/us/en/software/junos/interfaces-fundamentals/topics/topic-map/loopback-interfaces.html)
- [SRX Loopback Interfaces](https://www.juniper.net/documentation/us/en/software/junos/interfaces-security-devices/topics/topic-map/security-interface-config-loopback-interfaces.html)

**Wniosek:** redakcja gotowa do czytania; laboratorium gotowe do próby, ale jego rzeczywiste wyniki pozostają niezweryfikowane.

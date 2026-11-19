# Audyt migracji podręcznika JNCIA/JNCIS/JNCIP — DIAMOND 4×vSRX + 2×VPCS

**Data:** 2026-11-19. **Status:** CONTENT UPDATED / STATIC ADDRESS REVIEW — **NIE RUNTIME VALIDATED**. Dokument odnotowuje zakończoną redakcję ośmiu *głównych* rozdziałów. Nie dowodzi działania konfiguracji na konkretnej instancji vSRX.

## Jedyna referencyjna topologia

- [Mermaid: pełny DIAMOND — IP/CIDR obu stron, 6 segmentów i 4 loopbacki](../../labs/reference-diamond-4vsrx-2vpcs/DIAMOND_TOPOLOGY.md).
- [Kanoniczna tabela urządzeń/interfejsów, zasady tras statycznych i failover](../../labs/reference-diamond-4vsrx-2vpcs/README.md).
- PC-A: `192.0.2.10/24`, GW R1 `192.0.2.1/24`; PC-B: `198.51.100.10/24`, GW R4 `198.51.100.1/24`.
- R1–R2: `10.12.0.0/30`; R1–R3: `10.13.0.0/30`; R2–R4: `10.24.0.0/30`; R3–R4: `10.34.0.0/30`.
- `lo0.0` / Router ID: R1 `10.255.255.1/32`, R2 `.2/32`, R3 `.3/32`, R4 `.4/32`.

Nie powstają dodatkowe fizyczne łącza R2–R3 lub R1–R4. Baseline statyczny preferuje R2, alternatywa prowadzi przez R3. Pojedyncza awaria R1–R2 nie zapewnia automatycznej konwergencji w obu kierunkach; do testu pełnego failover statycznego trzeba świadomie utracić obydwa górne segmenty albo użyć protokołu/detekcji obejmującej całą ścieżkę. SRX security flow wpływa na wynik.

## Matryca redakcyjna ośmiu rozdziałów

| Rozdział i kanoniczny plik | Jak wykorzystuje diament | Uzasadnione wyjątki |
|---|---|---|
| [1 — Networking Fundamentals](../chapters/JUNOS_Rozdzial_01_Networking_Fundamentals_i_Enterprise_Portfolio.md) | pełny Mermaid, subnetting /30, bramy /24 i loopbacki | architektura switching/PoE/EX/QFX, hardware portfolio i fizyczny Greenfield/BOM to inne technologie |
| [2 — Junos OS Architecture](../chapters/JUNOS_Rozdzial_02_Architektura_Junos_OS.md) | L2 hop-by-hop przez R1–R2–R4, R3 backup, CLI/FIB i cztery lo0 | diagramy RE/PFE/procesów pozostają schematami mechanizmów, a nie nowymi sieciami |
| [3 — Junos CLI](../chapters/JUNOS_Rozdzial_03_Junos_User_Interfaces_CLI.md) | R1–R2 na ge-0/0/0.0, R1–R3 na ge-0/0/1.0, workflow candidate/commit na R1 | logika shared/private/commit nie potrzebuje dodatkowych routerów |
| [4 — Initial Configuration & Services](../chapters/JUNOS_Rozdzial_04_Initial_Configuration_and_System_Services.md) | R1 w bazowym DIAMOND | odizolowany **OOB/services** `172.31.240.0/24` dla NTP/AAA/SNMP/syslog/SCP jest *oddzielnym labem usług*, nie częścią sześciowęzłowej topologii |
| [5 — Monitoring & Maintenance](../chapters/JUNOS_Rozdzial_05_Operational_Monitoring_and_Maintenance.md) | laboratorium 5.9 **i specyfikacja labs/ch05** przebudowane na 4 vSRX, 2 VPCS, ścieżka zwrotna i R3 alternate | DOM optyczny, Virtual Chassis i fizyczne wymiany sprzętu wymagają urządzeń rzeczywistych |
| [6 — Routing Fundamentals](../chapters/JUNOS_Rozdzial_06_Routing_Fundamentals.md) | 4-hop path + obie drogi, static/qualified-next-hop, nowe 6.11, problemy ARP/FIB i reverse route | **dual-stack IPv6** na tych samych linkach w osobnym wariancie; hosty Linux tylko jeśli VPCS IPv6 nieobsługiwane; scenariusz overlapping VRF to osobna *logiczna* ilustracja |
| [7 — Protocol-Independent Routing](../chapters/JUNOS_Rozdzial_07_Protocol_Independent_Routing.md) | aggregate/generated na sztucznych trasach via R2/R3; RIB group/FBF, ECMP i lab 7.8 na DIAMOND | syntetyczne `10.60.0.0/16` / `10.70.0.0/16` są wpisami RIB, **nie nowymi LAN-ami**; FBF `then routing-instance` zależy od wsparcia platformy vSRX |
| [8 — Routing Policy & Firewall Filters](../chapters/JUNOS_Rozdzial_08_Routing_Policy_and_Firewall_Filters.md) | cztery eBGP peerings po istniejących odcinkach; R1 LAN-A, R4 LAN-B; export/import, LocalPref, stateless vs stateful i fault lab 8.14 | dodatkowe syntetyczne prefiksy route-filter/uRPF nie stanowią siódmego segmentu; wariant BGP musi usunąć konfliktujące statics na *izolowanym klonie* |

## Archiwum i wersje niekanoniczne

- [Poprzedni lab CH05 (dwa vSRX, błędne podłączenie PC-B do R2)](../archive/CH05_LEGACY_TWO_VSRX_MONITORING_LAB.md) — **ARCHIVE ONLY**.
- [Poprzedni lab 6.11 (dwa routery, osobne IPv6)](../archive/CH06_ROUTING_FUNDAMENTALS_LEGACY_TWO_ROUTER_LAB.md) — **ARCHIVE ONLY**.
- [Poprzedni lab 7.8 (R1 i Linux upstream)](../archive/CH07_LEGACY_LINUX_FBF_LAB.md) — **ARCHIVE ONLY**.
- [Poprzedni lab 8.14 (dwa routery eBGP)](../archive/CH08_LEGACY_TWO_ROUTER_BGP_LAB.md) — **ARCHIVE ONLY**.
- `JUNOS_Rozdzial_07_Protocol_Independent_Routing_v1_ARCHIVE.md` to wcześniejszy draft, nie aktualizuj jako wersji produkcyjnej.
- `JUNOS_Rozdzial_08_Routing_Policy_and_Firewall_Filters_Agent.md` to alternatywny draft, nie kanoniczny Rozdział 8.

## Zasady dla kolejnego agenta szkoleniowego

1. Używaj powyższych ośmiu **głównych plików**, a nie archiwów/draftów. W rozdziałach linkuj **ten sam** diagram Mermaid.
2. Zanim podasz komendę Junos, napisz hostname i tryb CLI bezpośrednio nad blokiem. Nigdy nie używaj starego R1–R2 LAN-B ani fictitious PC-B `192.0.2.11`.
3. Nie twórz połączeń do drugiej fizycznej sieci na R2. Nie myl syntetycznych contributors z prawdziwymi sieciami EVE; nie obiecuj ping do nieistniejących usług.
4. Na podstawowym vSRX nadal obowiązują zone policy i flow; host-inbound ping i host-inbound BGP to osobne zagadnienia.
5. Rozdział 7 rozróżnia source-based PBR z DWOMA hostami od rzeczywiście wykonalnej **destination-based FBF** z PC-A i PC-B na przeciwnych końcach diamentu.
6. Rozdział 8 korzysta z **czterech AS** na czterech linkach P2P. R1 i R4 ogłaszają swoje LAN-y; R2 i R3 są transit. Niedopuszczalne jest stwierdzenie, że LAN-B jest connected na R2.
7. Zachowaj głębokość narracyjnego wykładu; sam Mermaid ani tabela CLI nie zastępują analizy pakietu, drogi powrotnej, awarii i naprawy.
8. Nie wpisuj w repo haseł, rzeczywistych IP OOB, kluczy GCP, obrazów Junos/QCOW2. Publikowane przykłady nie są wynikami testów runtime.

## Walidacja i ograniczenia

**Przeprowadzona:** przegląd spójności adresów i sześciu połączeń diagramu referencyjnego, ujednolicenie głównych ośmiu rozdziałów i wymiana starodwu-routerowych laboratoryjnych topologii. Dodatkowe przykłady techniczne zostały oznaczone jako warianty lub abstrakcyjne wpisy RIB.

**Nie przeprowadzono:** uruchomienie EVE-NG na GCP, walidacja faktycznej kolejności QEMU vNIC, commit check na wybranym obrazie Junos, testy BGP/FBF/ECMP, ping/traceroute i sesje flow runtime. Konkretna dostępność FBF w vSRX wymaga potwierdzenia. Rzeczywiste statusy scenariuszy: **PLANNED / NOT TESTED**.

**Uwaga redakcyjna:** niniejszy audyt nie zastępuje testów technicznych ani review każdego zdania wszystkich rozdziałów przez inżyniera. Warianty protokołowe powinny być przy uruchamianiu testowane oddzielnie.

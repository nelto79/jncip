# JNCIP-ENT — JUNOS Enterprise Routing & Switching

Własny podręcznik i repozytorium laboratoriów do nauki **JNCIA-Junos (JN0-106) → JNCIS-ENT (JN0-352) → JNCIP-ENT (JN0-650)**. Materiał prowadzi od fundamentów Junos do poziomu Professional, z myślą o inżynierze Cisco i laboratoryjnym EVE-NG.

## Zacznij tutaj

1. [Wspólny blueprint — wszystkie trzy poziomy](docs/JUNOS_JNCIA_JNCIS_JNCIP_unified_blueprint.md)
2. [Rozdział 1 — Networking Fundamentals oraz Enterprise Portfolio, Greenfield i BOM](docs/chapters/JUNOS_Rozdzial_01_Networking_Fundamentals_i_Enterprise_Portfolio.md)
3. [Rozdział 2 — Junos OS Fundamentals (rozbudowany wykład)](docs/chapters/JUNOS_Rozdzial_02_Architektura_Junos_OS.md)
4. [Rozdział 3 — Junos User Interfaces: CLI, candidate/active, commit, rollback, J-Web](docs/chapters/JUNOS_Rozdzial_03_Junos_User_Interfaces_CLI.md)
5. [Rozdział 4 — Initial Configuration and System Services: management, AAA, NTP, SNMPv3, syslog, backup i rescue](docs/chapters/JUNOS_Rozdzial_04_Initial_Configuration_and_System_Services.md)
6. [Rozdział 5 — Operational Monitoring and Maintenance: show/monitor, interfejsy, narzędzia sieciowe, upgrade, reboot, recovery i diagnostyka](docs/chapters/JUNOS_Rozdzial_05_Operational_Monitoring_and_Maintenance.md)
7. **[Kanoniczna topologia DIAMOND — 4×vSRX, 2×VPCS, CIDR i loopbacki](labs/reference-diamond-4vsrx-2vpcs/README.md)** oraz [prompt dla Codexa: wdrożenie EVE-NG na GCP](labs/reference-diamond-4vsrx-2vpcs/CODEX_GCP_EVE_NG_PROMPT.md)
8. [Lab rozdziału 2 — 4×vSRX i 2×VPCS](labs/ch02-four-vsrx/README.md)
9. [Lab rozdziału 3 — Shared/Private Candidate, Load, Commit Confirmed](labs/ch03-cli-change-management/README.md)
10. [Lab rozdziału 4 — pełny onboarding Junos z bezpiecznym recovery](labs/ch04-initial-config/README.md)
11. [Lab rozdziału 5 — 2×vSRX: interface fault, missing route i policy deny](labs/ch05-operational-monitoring/README.md)
12. [Instrukcje dla Codexa](AGENTS.md)
13. [Źródła modeli, wersji Junos i ograniczeń BOM — research 2026-11-16](docs/research/CH01_ENTERPRISE_PORTFOLIO_ZRODLA_2026-11-16.md)

## Projekt — zasady

- Dokumenty źródłowe są w **Markdown (.md)**: bez skrótów, zachowujemy kompletną treść i numerację.
- Materiały edukacyjne i skrypty/laby są oddzielone: docs/ i labs/.
- JNCIA i JNCIS stanowią podstawę; wspólne rozdziały rozwijają temat do JNCIP.
- Rozdział 2 (wersja wizualna, 16 listopada 2026) zawiera liczne analogie Cisco IOS XE ↔ Junos, diagramy Mermaid i grafiki JPG/PNG z legalnymi licencjami, obok lekcji RE/PFE, laboratoriów 4×vSRX i troubleshootingu.
- [Źródła grafik, autorzy i licencje](docs/visuals/IMAGE_SOURCES.md) — osobna dokumentacja mediów.
- [Pochodzenie pytań i kwerenda oficjalnych źródeł JNCIA/Junos](docs/research/CH02_OFICJALNE_PYTANIA_I_POCHODZENIE_ZRODEL.md) — notatka dla autora i Codexa; wyjaśnienia zawarto wewnątrz rozdziału 2.
- [Haczyki i gotchas rozdziału 2 — zweryfikowane źródła Juniper/HPE](docs/research/CH02_GOTCHAS_ZRODLA_I_WERYFIKACJA.md) — 9 nieoczywistych zachowań, objaśnionych w tekście podręcznika.
- [Rozdział 3 — techniczne gotchas i źródła dokumentacji](docs/research/CH03_JUNOS_CLI_GOTCHAS_I_ZRODLA_2026-11-16.md) — 14 pułapek z objaśnieniami w lekcji, m.in. confirm po commit check.
- [Rozdział 4 — badanie 15 gotchas i źródła Juniper/HPE](docs/research/CH04_GOTCHAS_ZRODLA_I_WERYFIKACJA.md) — factory defaults, AAA, management, groups, NTP, SNMP, syslog, archival i rescue.
- [Rozdział 5 — 18 gotchas i zweryfikowane źródła](docs/research/CH05_MONITORING_MAINTENANCE_GOTCHAS_I_ZRODLA_2026-11-17.md) — RE capture, counters/DOM, source-aware ping, software upgrades, reboot i root recovery.
- [Archiwum oryginalnego rozdziału 2 v1](docs/archive/JUNOS_Rozdzial_02_Architektura_Junos_OS_v1_2026-11-15.md) pozostaje dostępne do porównania.
- Topologia kanoniczna: **4 × vSRX + 2 × VPCS**, cztery połączenia tranzytowe tworzące romb + dwa połączenia do hostów (6 segmentów L2); CIDR i loopbacki /32 są na stałe zdefiniowane w `labs/reference-diamond-4vsrx-2vpcs/README.md`.
- Najpierw statyczny routing i analiza ścieżki; protokoły dynamiczne w kolejnych lekcjach.
- EVE-NG, dostęp do hosta, dostępne obrazy i realną numerację portów należy zweryfikować przed generowaniem i uruchamianiem labu.
- Repozytorium nie zawiera komercyjnych obrazów QCOW2, haseł, kluczy ani danych środowiska produkcyjnego.
- Gotowość zawartości repozytorium **nie oznacza**, że laboratorium zostało wdrożone i zweryfikowane runtime w EVE-NG.

## Dla Codexa

Otwórz ten katalog jako workspace. Przeczytaj AGENTS.md, blueprint oraz rozdział 2. Następnie zrealizuj specyfikację z labs/ch02-four-vsrx/README.md: przygotuj topologię EVE-NG (.unl), osobne konfiguracje startowe i scenariusze walidacji; rzeczywiste wdrożenie tylko po weryfikacji środowiska.

## Źródła

- [Juniper / HPE Learning](https://learningportal.juniper.net/)
- [Juniper documentation](https://www.juniper.net/documentation/)
- [Juniper Software Downloads](https://support.juniper.net/support/downloads/)

Stan dokumentacji: 2026-11-17. Dane o sugerowanych wydaniach Junos dla rozdziału 1 mają datę odniesienia 2026-11-04 (JTAC KB21476); weryfikuj je przed każdą ofertą lub zmianą.

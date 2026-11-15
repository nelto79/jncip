# JNCIP-ENT — JUNOS Enterprise Routing & Switching

Własny podręcznik i repozytorium laboratoriów do nauki **JNCIA-Junos (JN0-106) → JNCIS-ENT (JN0-352) → JNCIP-ENT (JN0-650)**. Materiał prowadzi od fundamentów Junos do poziomu Professional, z myślą o inżynierze Cisco i laboratoryjnym EVE-NG.

## Zacznij tutaj

1. [Wspólny blueprint — spis treści wszystkich trzech poziomów](docs/JUNOS_JNCIA_JNCIS_JNCIP_unified_blueprint.md)
2. [Rozdział 2 — Junos OS Fundamentals: pełny podręcznik, wersja korepetycyjna](docs/chapters/JUNOS_Rozdzial_02_Architektura_Junos_OS.md)
3. [Lab rozdziału 2 — cztery vSRX i dwa hosty](labs/ch02-four-vsrx/README.md)
4. [Instrukcje dla Codexa](AGENTS.md)

## Projekt — zasady

- Dokumenty źródłowe są w **Markdown (.md)**: bez skrótów, zachowujemy kompletną treść i numerację.
- Materiały edukacyjne i skrypty/laby są oddzielone: docs/ i labs/.
- JNCIA i JNCIS stanowią podstawę; wspólne rozdziały rozwijają temat do JNCIP.
- Rozdział 2 (v2, ~11 tys. słów) obejmuje RE, PFE, control/data plane, transit/exception traffic, laboratorium 4×vSRX, interpretację CLI, scenariusze i troubleshooting; napisany jako prowadzona lekcja, nie encyklopedia.
- [Archiwum oryginalnego rozdziału 2 v1](docs/archive/JUNOS_Rozdzial_02_Architektura_Junos_OS_v1_2026-11-15.md) pozostaje dostępne do porównania.
- Pierwsza topologia: **4 × vSRX + 2 × VPCS**, cztery połączenia tworzące romb (dwie trasy do sieci zdalnej).
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

Stan projektu: 2026-11-15.

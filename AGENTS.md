# AGENTS.md — instrukcje dla Codexa / innych agentów

## Cel

To repozytorium dotyczy nauki JUNOS i przygotowania do JNCIA-Junos (JN0-106), JNCIS-ENT (JN0-352), JNCIP-ENT (JN0-650), włącznie z ćwiczeniami EVE-NG. Użytkownik jest doświadczonym inżynierem Cisco, dlatego ważne są porównania Cisco IOS/IOS XE ↔ Junos, zachowanie protokołów i praktyczny troubleshooting.

## Canonical source of truth

1. docs/JUNOS_JNCIA_JNCIS_JNCIP_unified_blueprint.md — obowiązujący, zunifikowany spis treści; zachowaj zakres i numerację.
2. docs/chapters/JUNOS_Rozdzial_02_Architektura_Junos_OS.md — pełny rozdział 2 (nie skracaj, nie nadpisuj streszczeniem).
3. labs/ch02-four-vsrx/README.md — specyfikacja pierwszego większego labu.
4. labs/ch02-four-vsrx/CODEX_TASK.md — kryteria implementacji i walidacji.

## Zasady dokumentacyjne

- Podręczniki i blueprint są plikami Markdown (.md). Zawsze zachowuj pełny tekst źródłowy.
- Do istniejących rozdziałów dodawaj zmiany punktowo, bez redukowania zakresu.
- Każdy blok CLI poprzedzaj nagłówkiem z nazwą urządzenia i trybem operacyjnym/konfiguracyjnym.
- Rozróżniaj polecenia potwierdzone dla wersji obrazu od przykładów wymagających walidacji.
- Każde ćwiczenie opisuj: cel, diagram, sprzęt, adresacja, konfiguracja bazowa, procedura, verification, expected output, fault injection, repair i cleanup.
- Nie sugeruj, że laboratorium działa, dopóki nie zostało wykonane oraz zweryfikowane w EVE-NG.
- Oznaczaj jawnie status: PLANNED / GENERATED / STATIC VALIDATED / RUNTIME VALIDATED.

## Bezpieczeństwo i zmiany w EVE-NG

- Najpierw odczyt stanu (read-only): dostęp SSH, aktualne katalogi, wersja EVE, obrazy, wolne zasoby, obsługiwane nazwy interfejsów, istniejące laby.
- Utrzymuj nowy katalog laboratorium osobno od istniejących Cisco 300-410; nie modyfikuj ich.
- Nie nadpisuj istniejących plików .unl, obrazów QEMU ani konfiguracji; twórz nowe pliki z własnym UUID.
- Nie zmieniaj globalnych ustawień EVE-NG, routingu hosta, fizycznych bridge ani istniejącego labu bez potwierdzonej potrzeby.
- Domyślnie projektuj lab bez połączenia do sieci firmowej, Internetu ani management cloud; używaj izolowanych segmentów.
- Nie commituj obrazów QCOW2/OVA, backupów, haseł, kluczy, adresów produkcyjnych, tokenów, zrzutów sesji.
- Zastosuj commit check oraz bezpieczne commit confirmed w konfiguracjach testowych, jeśli konsola jest dostępna.
- Nie uruchamiaj wszystkich czterech vSRX na niewystarczającym hoście; przeprowadź capacity check.
- Weryfikuj, że mapowanie interfejsów EVE rzeczywiście odpowiada fizycznym interfejsom danych Junos. vSRX może mieć oddzielny port zarządzający.

## Pierwszy lab

Na podstawie labs/ch02-four-vsrx/README.md przygotuj pliki .unl, przykładowe konfiguracje .set i instrukcje uruchomienia. Nie deklaruj runtime PASS bez rzeczywistych wyników ping/trace/route/session. Statyczna poprawność XML nie dowodzi działającego forwardingu.

## Aktualizacja postępu

Po wykonanym zadaniu zapisz rzeczywisty status w README laboratorium, wraz z listą utworzonych plików, dowodem walidacji, ograniczeniami oraz następnym krokiem. Unikaj zbędnych zapisów do historii.

## Źródła techniczne

W razie rozbieżności sięgaj do oficjalnej dokumentacji Juniper Junos/vSRX, dokumentacji wersji obrazu oraz dokumentacji EVE-NG. Rozróżniaj klasyczne routery/switche Junos (tradycyjny PFE) i vSRX (stateful flow processing).

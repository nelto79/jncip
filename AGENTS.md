# AGENTS.md — instrukcje dla Codexa / innych agentów

## Cel

To repozytorium dotyczy nauki JUNOS i przygotowania do JNCIA-Junos (JN0-106), JNCIS-ENT (JN0-352), JNCIP-ENT (JN0-650), włącznie z ćwiczeniami EVE-NG. Użytkownik jest doświadczonym inżynierem Cisco, dlatego ważne są porównania Cisco IOS/IOS XE ↔ Junos, zachowanie protokołów i praktyczny troubleshooting.

## Canonical source of truth

**Rozdział 1**: docs/chapters/JUNOS_Rozdzial_01_Networking_Fundamentals_i_Enterprise_Portfolio.md — fundamenty + [EXT] portfolio Campus LAN, HLD/BOM i wersje Junos. Portfolio jest zmienne w czasie; sprawdzaj datasheets, supported models, Juniper JTAC KB21476 i release notes, zwłaszcza przed projektowaniem ofert.

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

## Obowiązkowy styl dydaktyczny podręcznika (od rozdziału 2 v2)

Użytkownik **nie chce encyklopedycznego materiału ani zestawu tabel do wykucia**. Pisz po polsku, jak doświadczony instruktor prowadzący żywe korepetycje dla inżyniera Cisco:

1. Zacznij od życiowego lub operatorskiego problemu: czego nie dało się zrobić, po co powstał mechanizm, jakie były alternatywy i kompromisy.
2. Użyj trafnej analogii, ale jasno pokaż, gdzie analogia się kończy; nie infantylizuj ucznia.
3. Prowadź narrację **przyczyna → mechanizm → skutek → obserwacja w CLI → awaria → naprawa**.
4. Zanim pokażesz konfigurację, wyjaśnij **po co** ją wpisujemy i jaki wynik przewidujemy; po CLI naucz interpretować wynik i wyjątki.
5. Zanim pokażesz tabelę, opowiedz, jaki problem porządkuje; preferuj opowieść, scenariusze, diagnozę i pytania aktywizujące przed tabelami.
6. Wyraźnie rozróżniaj: **na pamięć**, **do zrozumienia**, **do przećwiczenia**; nie wymagaj zapamiętywania całych outputów show.
7. Podawaj łączniki Cisco IOS/IOS XE ↔ Junos oraz typowe błędne intuicje inżyniera Cisco, ale nie zamieniaj rozdziału w dwukolumnowy słownik.
8. Łącz każde objective z rzeczywistym symptomem i rozwiązaniem (lab, fault injection, troubleshooting).
9. Utrzymuj pełną techniczną precyzję i zakres blueprintu. Nie rezygnuj z głębi, by brzmieć bardziej swobodnie.
10. Nie kopiuj specyficznego języka konkretnych instruktorów; wzoruj się na wysokiej jakości wyjaśniania, energii, przykładach i stopniowaniu trudności.
11. Najważniejszy punkt odniesienia stylistycznego: docs/chapters/JUNOS_Rozdzial_02_Architektura_Junos_OS.md (v2).

## Cisco ↔ Junos: transfer wiedzy oraz ilustracje

- **Każdy rozdział ma regularnie wykorzystywać wiedzę o Cisco IOS/IOS XE**: co użytkownik już zna (RIB/FIB, CEF, C9500 SVI, CoPP, ACL, BGP/OSPF, HA), jaki problem rozwiązuje Junos, jakie są podobieństwa i **gdzie kończy się analogia**. W co najmniej kluczowych sekcjach rozdziału umieszczaj **Most z Cisco** w toku lekcji, nie tylko końcową tabelę.
- Dla CLI przedstaw **nazwę urządzenia bezpośrednio nad każdym blokiem komend**, osobno Cisco i Junos. Najpierw explain why, potem komendy i expected interpretation. Nie używaj mechanicznego tłumaczenia pojedynczych słów Cisco na Junos.
- **Ilustracje są wymaganym elementem dydaktyki, kiedy pomagają zobaczyć proces, topologię lub ruch pakietu.** Dodawaj diagramy Mermaid bezpośrednio do Markdown, a odpowiednie JPG/PNG osadzaj z pewnych źródeł z jasno określoną licencją do ponownego wykorzystania.
- Każda grafika ma mieć (a) związek z dokładnie omawianym problemem, (b) wyjaśnienie przed i po, (c) tekst alternatywny, (d) właściwą atrybucję i licencję. Zero przypadkowych obrazów dekoracyjnych.
- Utrzymuj dokument źródeł grafik: docs/visuals/IMAGE_SOURCES.md. Rozróżniaj grafiki zewnętrzne hotlinkowane z Wikimedia od obrazów przechowywanych lokalnie. Nie twierdź, że pobrano plik do repo, jeżeli jest tylko link zewnętrzny.
- Odróżniaj sprzętowe platformy Cisco (Catalyst/QFP/ASIC), MX/EX/QFX i wirtualny vSRX. Cisco CEF i Junos PFE mają analogiczne cele, lecz nie identyczną implementację.

## Integracja oficjalnych pytań i problemów z materiałami lekcyjnymi

- Gdy użytkownik prosi o pytania egzaminacyjne, **najpierw zbadaj jawne źródła**: oficjalne Juniper/HPE practice exams (z dostępem tylko tam, gdzie faktycznie możliwy), pytania po webcastach, dokumentację Day One, Juniper CLI Reference, FAQ/KB oraz dyskusje inżynierów.
- Oddziel trzy kategorie: **(1) pytanie rzeczywiście opublikowane w źródle**, **(2) temat/diagram producenta z pytaniem sformułowanym przez autora**, **(3) własne ćwiczenie laboratoryjne**. Nie przedstawiaj przykładów własnych jako prawdziwych pytań egzaminacyjnych.
- **Nie kopiuj całych banków pytań ani wielu zadań z komercyjnych i nieautoryzowanych źródeł.** Oficjalny practice exam może wymagać interaktywnego uruchomienia; nie twierdź, że odczytałeś pytania, jeśli widoczna była tylko strona opisowa.
- **Wyjaśnienie odpowiedzi ma być wplecione w wykład** w sekcji omawiającej daną mechanikę, a nie dodane jako osobny spis źródłowych Q&A na końcu. Buduj z tego mini-scenariusz: problem → przewidywanie → przyczyna → mechanizm → skutek → weryfikacja CLI → odniesienie do Cisco.
- Zostaw link do pierwotnego materiału przy właściwym wyjaśnieniu. Pochodzenie kwerendy i wersji dokumentuj osobno dla autorów, np. docs/research/CH02_OFICJALNE_PYTANIA_I_POCHODZENIE_ZRODEL.md.
- Pytania z forów są wskazówkami dydaktycznymi, **nie autorytatywną odpowiedzią techniczną**; rozwiązania konfrontuj z dokumentacją Juniper i odpowiednią wersją platformy.

## Obowiązkowa kwerenda gotchas przy każdym rozdziale

**Przed napisaniem lub rozbudową każdego rozdziału wykonaj aktywne wyszukiwanie podchwytliwych zachowań Junos** w oficjalnej dokumentacji Juniper/HPE, CLI Reference, Day One, KB i dyskusjach społeczności. Szukaj zwłaszcza różnic z Cisco IOS/IOS XE, nieintuicyjnych wartości domyślnych, wyjątków platformowych, stanów hidden/inactive, osobnego RE/PFE, sesji SRX, priorytetu konfiguracji, fałszywych wniosków z pojedynczej komendy i zagadnień typowych dla troubleshootingu.

Każdy wartościowy haczyk **wplataj organicznie do właściwej lekcji**, nie zbieraj wszystkich na końcu:
1. **Znana intuicja Cisco / pytanie egzaminacyjne** — skąd bierze się pomyłka?
2. **Kontrprzykład Junos** — na której platformie/release i z jakimi ustawieniami?
3. **Przyczyna i skutek** — co robi pakiet, control plane, PFE, security policy lub sesja?
4. **CLI i praktyczny dowód** — oznacz hostname nad każdym blokiem komend; porównaj polecenia Cisco ↔ Junos.
5. **Co zapamiętać** — niewielka reguła/wyjątek, nie kolejne wielostronicowe definicje; odnośnik do zweryfikowanego źródła.
6. **Scenariusz laboratoryjny**, jeśli da się go bezpiecznie odtworzyć bez modyfikowania produkcyjnych systemów.

**Nie nazywaj własnych scenariuszy prawdziwymi pytaniami z egzaminu.** Pytanie z forum nie jest certyfikowanym twierdzeniem — porównaj je z oficjalnym producentem. Różnice zależne od wydania/jednostki zaznaczaj jawnie. Nie powielaj mitów „default permit/deny” bez rozróżnienia factory-default config i default software behavior.

Dla rozdziału 2 odpowiedni rejestr źródeł: \`docs/research/CH02_GOTCHAS_ZRODLA_I_WERYFIKACJA.md\`. Dla kolejnych rozdziałów prowadź analogiczny krótki rejestr redakcyjny w \`docs/research/\`; finalny uczeń ma spotkać wyjaśnienia **wewnątrz wykładu**, nie osobno na końcu.

## Portfolio producenta i projektowanie greenfield — obowiązkowe na przyszłość

- W rozdziale 1.12 [EXT] oraz przyszłych rozdziałach doboru sprzętu rozróżniaj **access / distribution / core / wireless / security / WAN**, ale nigdy nie upraszczaj do „jedna rodzina = tylko jedna warstwa”. Dobieraj konkretne modele i pełne SKU per role.
- Zawsze porównuj do Cisco (Catalyst 9200/9300/9500, ASA/FTD, Catalyst 9800, ISR/ASR, Catalyst Center) z wyjaśnieniem ograniczeń analogii. Mist AP ≠ Junos i SSR ≠ Junos.
- Podczas podawania wersji oprogramowania sprawdzaj **konkretną rodzinę i model, Suggested Releases JTAC KB21476, Hardware/Feature Explorer, Release Notes, EoE/EoS**. Nigdy nie sugeruj jednego release dla wszystkich EX/QFX/SRX ani nie udawaj, że „-Sx” to dokładny obraz do instalacji. Snapshot zawsze datuj.
- Przy bill of materials: licz niezależnie porty 1G i mGig/PoE, uplink speeds, optykę, zasilacze i redundancję, licencje Mist/security, support/SLA, potrzeby przyszłego rozwoju oraz kompatybilność z NAC/AAA i Wi-Fi. Nie wymyślaj cen, lead times ani pełnych orderable part numbers.
- Do repo trafiają tylko bezpieczne, publiczne, ogólne szablony, nie produkcyjne IP, numery ofert, dane klientów ani klucze.
- Jeżeli nie można potwierdzić 2026 modelu/release, pozostaw status **TO VERIFY**, zamiast zgadywać.

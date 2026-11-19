# AGENTS.md — instrukcje dla Codexa / innych agentów

## Cel

To repozytorium dotyczy nauki JUNOS i przygotowania do JNCIA-Junos (JN0-106), JNCIS-ENT (JN0-352), JNCIP-ENT (JN0-650), włącznie z ćwiczeniami EVE-NG. Użytkownik jest doświadczonym inżynierem Cisco, dlatego ważne są porównania Cisco IOS/IOS XE ↔ Junos, zachowanie protokołów i praktyczny troubleshooting.

## Canonical source of truth

**Rozdział 1**: docs/chapters/JUNOS_Rozdzial_01_Networking_Fundamentals_i_Enterprise_Portfolio.md — fundamenty + [EXT] portfolio Campus LAN, HLD/BOM i wersje Junos. Portfolio jest zmienne w czasie; sprawdzaj datasheets, supported models, Juniper JTAC KB21476 i release notes, zwłaszcza przed projektowaniem ofert.

1. docs/JUNOS_JNCIA_JNCIS_JNCIP_unified_blueprint.md — obowiązujący, zunifikowany spis treści; zachowaj zakres i numerację.
2. docs/chapters/JUNOS_Rozdzial_02_Architektura_Junos_OS.md — pełny rozdział 2 (nie skracaj, nie nadpisuj streszczeniem).
2a. docs/chapters/JUNOS_Rozdzial_03_Junos_User_Interfaces_CLI.md — pełny rozdział 3 (3.1–3.11), warstwa CLI/candidate/commit/load/rollback/J-Web. Nie zastępuj streszczeniem i nie przenoś wszystkich gotchas na koniec.
2b. docs/chapters/JUNOS_Rozdzial_04_Initial_Configuration_and_System_Services.md — pełny rozdział 4 (4.1–4.14 oraz 4.15 LAB), Cisco ↔ Junos, AAA, NTP, SNMPv3, syslog, archival, traceoptions, rescue.
2c. docs/chapters/JUNOS_Rozdzial_05_Operational_Monitoring_and_Maintenance.md — pełny rozdział 5 (5.1–5.8 oraz 5.9 LAB), workflow diagnostyki i maintenance, interface counters/DOM, show/monitor, ping/source/VRF, upgrade, reboot/halt i root recovery.
3. **labs/reference-diamond-4vsrx-2vpcs/README.md — KANONICZNA topologia diamentowa: 4 vSRX + 2 VPCS, wszystkie interfejsy, CIDR i loopbacki /32. Nie zmieniaj ich w kolejnych rozdziałach bez jawnego wariantu.**
3a. **labs/reference-diamond-4vsrx-2vpcs/DIAMOND_TOPOLOGY.md — diagram Mermaid do osadzania i linkowania w podręczniku (sześć łączy, oba IP/CIDR każdego segmentu, wszystkie loopbacki).** W rozdziałach wykorzystuj ten sam model, a w razie korekty diagramu aktualizuj również jego kopię w referencyjnym README.
3a. labs/reference-diamond-4vsrx-2vpcs/CODEX_GCP_EVE_NG_PROMPT.md — instrukcja wdrożenia na EVE-NG/GCP, bez ingerencji w istniejące laby.
3b. labs/ch02-four-vsrx/README.md — specyfikacja pierwszego większego labu.
4. labs/ch02-four-vsrx/CODEX_TASK.md — kryteria implementacji i walidacji.
5. labs/ch03-cli-change-management/README.md — lab z dwiema sesjami administratorów, konsolą i kontrolowanym commit confirmed; status PLANNED, dopóki nie ma rzeczywistych testów.
6. labs/ch04-initial-config/README.md — nowy izolowany lab z vSRX, Linux bastion i services (NTP, syslog, SNMPv3, SCP, AAA opcjonalnie), status PLANNED.
7. labs/ch05-operational-monitoring/README.md — nowy izolowany 2×vSRX+2×VPCS lab fault injection, status PLANNED.

## Obowiązkowa topologia DIAMOND w rozdziałach JNCIP

**Po migracji 2026-11-19** obowiązuje [audyt CH01–CH08](docs/research/DIAMOND_CH01_CH08_MIGRATION_AUDIT_2026-11-19.md). Osiem *głównych* rozdziałów jest w `docs/chapters/JUNOS_Rozdzial_01_...md` do `..._08_...md`; draft `..._08_..._Agent.md` i `..._07_..._v1_ARCHIVE.md` nie są źródłem prawdy. Nie przywracaj dawnych sieci PC-B `192.0.2.11`, bezpośredniego LAN-B na R2, WAN uplink `198.51.100.0/30`, ani labów z Linux UP-A/UP-B w roli urządzeń diamentu.



- Przy wykładach wykorzystujących kilka routerów (routing statyczny, OSPF, BGP, routing policy, ECMP, troubleshooting) używaj [kanonicznego diagramu Mermaid](labs/reference-diamond-4vsrx-2vpcs/DIAMOND_TOPOLOGY.md) i [tabeli adresacji](labs/reference-diamond-4vsrx-2vpcs/README.md) zamiast tworzyć od zera sprzeczne sieci.
- Identyfikatory w całym kursie: **R1-vSRX, R2-vSRX, R3-vSRX, R4-vSRX, PC-A/VPC1, PC-B/VPC2**. Bazowe segmenty, IP/CIDR, lo0.0 /32 i bramy VPCS są stałe; dla konkretnego ćwiczenia zmieniaj jedynie omawiany routing/policy, a każdy odmienny plan oznacz jako osobny wariant.
- W tekście przed diagramem postaw problem inżynierski, a po nim opowiedz, którędy płynie pakiet, jak wraca i co zmienia awaria; diagram nie zastępuje dydaktyki.
- W scenariuszach statycznych nie obiecuj pełnej automatycznej konwergencji po awarii tylko R1–R2; loopbacki innych routerów nie są reachable bez jawnego routingu. Na vSRX uwzględnij security policies oraz stan sesji.
- Nie nazywaj labu uruchomionym/zweryfikowanym bez dowodów z EVE-NG. Interfejsy ge-0/0/x z diagramu są logicznym celem do sprawdzenia na rzeczywistym obrazie.

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
- Pułapka failoveru: pojedyncza awaria R1–R2 przy statycznym routingu nie gwarantuje poprawnej drogi zwrotnej z R4; do testu działającego failoveru w baseline izoluj cały górny korytarz R1–R2 i R2–R4. Zdalne loopbacki nie są automatycznie osiągalne bez routingu.
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


## Specjalne zasady CH03 — konfiguracja i automatyczny rollback

- Standardowy `configure` używa shared candidate; w `configure private` sprawdzaj stale candidate i konflikty `update`. `configure exclusive` wymaga koordynacji.
- **`commit check` może potwierdzić oczekujący `commit confirmed`**. Nie dawaj w labie dodatkowego `commit check` już po warunkowej aktywacji.
- `load override` nie jest alternatywą dla `load set` z krótkim fragmentem. Wskazuj zakres replacement i podawaj diff.
- `rollback 0` odrzuca niezacommitowane zmiany; `rollback 1` ładuje poprzednią konfigurację do candidate. Historia numeryczna zmienia się po commit.
- Rozdział 3 trenuj najpierw na bezpiecznym login banner, potem na shared/private, bez kasowania management. Nie stosuj fault injection bez rzeczywistego OOB/console.
- Na EX4400 zarządzanie OOB zwykle jest przez `me0`, podczas gdy vSRX może mieć `fxp0`; nie zgaduj mapowania.
- Dokumentacja: docs/research/CH03_JUNOS_CLI_GOTCHAS_I_ZRODLA_2026-11-16.md. Status labu dokumentuj jawnie jako PLANNED / STATIC VALIDATED / RUNTIME VALIDATED.

## Specjalne zasady CH04 — bezpieczny onboarding i system services

- Chapter 4 canonical: `docs/chapters/JUNOS_Rozdzial_04_Initial_Configuration_and_System_Services.md`, punkty 4.1–4.14 oraz 4.15 [LAB], **nie redukuj do listy poleceń**.
- Dobór factory defaults **per platform i wydanie**; EX4400 `me0`, SRX/vSRX może mieć `fxp0`, a `mgmt_junos` nie jest automatycznie obecny po użyciu portu OOB.
- Najpierw lokalne konto, test oddzielnej sesji SSH, potem ograniczanie root SSH i centralne AAA; porównaj **AAA timeout vs remote reject** dla różnych `authentication-order`. Zachowaj niezależny console/OOB, aby nie spowodować lockoutu.
- Nigdy nie commituj haseł, shared secrets, SNMPv3 auth/privacy keys, prywatnych SSH keys, faktycznych adresów management ani pełnego configu klienta. Używaj interaktywnego prompt/vault, a w dokumentacji jawnie oznacz szablon jako **niekompletny**.
- Dla config groups sprawdzaj `display inheritance`, lokalne precedence i kolejność `apply-groups`.
- Udowadniaj **NTP synchronization** przez status/association, **SNMPv3** przez authPriv GET, **syslog** przez odbiornik, **archival** przez zdalny plik i test restore. Sam `commit complete` niczego z tych rzeczy nie potwierdza.
- W rescue lab **oddziel auto-rollback po `commit confirmed` od ręcznego `rollback rescue`**; nie uruchamiaj `commit check` podczas oczekującego confirmed commit.
- Wzorzec labu: `labs/ch04-initial-config/README.md`; dowody i wynik per usługa jako PASS / FAIL / NOT TESTED, status całego labu **PLANNED**, dopóki nie uruchomiono go w EVE.
- Rejestr źródeł i 15 haczyków: `docs/research/CH04_GOTCHAS_ZRODLA_I_WERYFIKACJA.md`; wyjaśnienia mają być w tekście wykładu, nie w suchym banku pytań.

## Specjalne zasady CH05 — monitoring i maintenance

- Kanoniczny rozdział: `docs/chapters/JUNOS_Rozdzial_05_Operational_Monitoring_and_Maintenance.md`; zachowaj 5.1–5.9 i pełny wykład narracyjny.
- Rób **dwa pomiary counters z interwałem**, nie oceniaj awarii tylko po wartościach skumulowanych; `show interfaces terse up/up` nie dowodzi braku CRC, drops i problemów optyki.
- DOM `show interfaces diagnostics optics` wymaga wspieranej **fizycznej wkładki**, nie jest sensownym testem „laser Rx power” na vSRX. Rx local to sygnał pochodzący z TX peer; interpretuj wartości względem specyfikacji modułu.
- `monitor traffic` zwykle obserwuje z perspektywy RE i **nie jest PFE-SPAN**. Ruch host-bound i transit są różne; na SRX używaj flow sessions/policies.
- `ping`/traceroute testuj z właściwym `source`, `routing-instance`, z klienta, w obu kierunkach. Ping do gateway nie wystarcza do potwierdzenia aplikacji klienta.
- Software maintenance wymaga exact model/Junos release/image, Suggested Releases JTAC, upgrade matrix, backup, OOB console i pre/post. `no-validate` tylko na udokumentowanych wyjątkach; `rollback 1` config ≠ software rollback; snapshot może nadpisać poprzedni image.
- `request system reboot/halt`, `restart routing`, `clear interfaces statistics` i root recovery mogą być destrukcyjne. Nie uruchamiaj ich na produkcji bez zatwierdzonej procedury, impact analysis i konsoli. Root recovery zależy od FreeBSD/Junos Evolved/security hardening.
- Laboratorium: `labs/ch05-operational-monitoring/README.md`, oddzielny 2×vSRX/2×VPCS; test admin disable, missing static route, policy deny; `commit confirmed` i automatyczny rollback, bez dodatkowego `commit check` podczas pending.
- Źródła i 18 gotchas: `docs/research/CH05_MONITORING_MAINTENANCE_GOTCHAS_I_ZRODLA_2026-11-17.md`. Nie przedstawiaj własnych case studies jako pytań certyfikacyjnych ani runtime PASS bez dowodów.

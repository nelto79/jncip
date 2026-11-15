# CODEX TASK — EVE-NG LAB CH02, 4 × vSRX

## Zlecenie

Na podstawie materiałów z tego repozytorium przygotuj działające laboratorium Chapter 2 JUNOS Architecture w EVE-NG. Nie edytuj ani nie streszczaj oryginalnego blueprintu i podręcznika. Pracuj w osobnym katalogu labs/ch02-four-vsrx/.

## Konieczna kolejność

1. Przeczytaj ../../AGENTS.md (w katalogu głównym), ../../docs/JUNOS_JNCIA_JNCIS_JNCIP_unified_blueprint.md, ../../docs/chapters/JUNOS_Rozdzial_02_Architektura_Junos_OS.md oraz README.md z bieżącego folderu.
2. Sprawdź lokalne środowisko robocze (pwd, git status, gałąź, wersja Git).
3. Sprawdź **read-only** dostęp do EVE-NG, jego wersję, dostępność obrazu vSRX, liczbę vCPU/RAM hosta, istniejącą strukturę labs, przykładowy prawidłowy .unl do odtworzenia schematu.
4. Jeśli nie ma obrazu legalnego vSRX KVM/QCOW2, nie pobieraj go z nieoficjalnych serwisów. Przygotuj topologię statycznie, oznacz blokadę i przekaż użytkownikowi wymagany format/nazwę obrazu.
5. Wygeneruj nowy, niezależny .unl z 4 vSRX (R1-R4), 2 VPCS (PC-A/PC-B), 6 izolowanymi segmentami według tabeli adresów w README. Nie nadpisuj istniejących labów.
6. Wygeneruj osobne konfiguracje .set dla każdego vSRX i pliki konfiguracji VPCS. Użyj właściwego dla obrazu mapowania interfejsów (w tym ewentualnego fxp0).
7. Dla bazowej konfiguracji dodaj działające statyczne trasy z trasą zapasową, strefy bezpieczeństwa SRX, host-inbound ping i jawnie opisane reguły umożliwiające ruch PC-A ↔ PC-B. **Nie wyłączaj security na całym urządzeniu bez potrzeby.**
8. Zapisz VERIFY.md z checklistą i konkretnymi expected outputs oraz TROUBLESHOOTING.md z kontrolowanymi awariami i procedurą odtworzenia stanu.
9. Waliduj XML, unikalność ID/UUID, mapowanie segmentów do interfejsów, odczyt startup-config oraz ew. realny forwarding. Jeśli EVE niedostępny, powiedz to wprost: STATIC VALIDATED ≠ RUNTIME VALIDATED.
10. W raporcie końcowym podaj dokładne pliki, gałąź/commit, status walidacji i blokery. Jeśli wykonałeś realny deployment, podaj również dokładną ścieżkę .unl na EVE-NG.

## Oczekiwane operacyjne wyniki

- Cztery aktywne instancje vSRX, o ile zasoby hosta i dostępny obraz na to pozwalają.
- PC-A ma połączenie ze swoim gateway R1; PC-B z R4.
- R1 i R4 widzą oczekiwane wpisy route/FIB; każda para sąsiadów widzi poprawną adresację/ARP.
- PC-A i PC-B komunikują się w obie strony przy włączonych potrzebnych security policies.
- Traceroute idzie przez R2 w baseline (z uwzględnieniem reakcji i ograniczeń SRX/ICMP).
- Po świadomym zerwaniu R1–R2 lub fizycznego primary linka ruch powinien przejść przez R3, po wygaśnięciu ewentualnych sesji/stanu. Powrót następuje po odtworzeniu połączenia.
- Ruch host-bound (ping na R1) można zablokować osobno od ruchu transit.
- Ruch transit można zablokować osobno od host-bound.
- Po fault injection wszystkie zmiany są przywrócone i ponownie zweryfikowane.

## Ważne uwagi

- Wersja vSRX może wpływać na nazwy interfejsów, obsługiwane komendy i wymagania licencyjne.
- Dla static failover nie obiecuj detekcji awarii linku R2–R4 z poziomu R1, jeśli R1–R2 nadal jest up.
- Nie wpisuj haseł i danych dostępu do repo.
- Nie podłączaj hostów laboratoryjnych do zewnętrznych segmentów ani produkcyjnego networku.
- W przypadku konfliktu przykładowej konfiguracji z rzeczywistą platformą, pierwszeństwo ma zweryfikowane zachowanie danej wersji vSRX i oficjalna dokumentacja.

## Gotowy prompt uruchomieniowy

Zrealizuj zadanie opisane w labs/ch02-four-vsrx/CODEX_TASK.md, korzystając z AGENTS.md i pełnego podręcznika Chapter 2. Najpierw przeprowadź bezpieczny audyt read-only mojego EVE-NG. Następnie przygotuj cztery vSRX w topologii diamond plus dwa VPCS, poprawne startup-config i komplet testów z README. Nie ingeruj w istniejące laboratoria Cisco 300-410. Rozróżnij walidację statyczną od runtime.

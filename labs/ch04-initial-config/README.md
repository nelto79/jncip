# LAB CH04 — Enterprise Junos: initial configuration, AAA, system services & rescue

**Status: PLANNED / NOT RUNTIME VALIDATED.** Nie ma wygenerowanego `.unl`, nie uruchamiano wymienionych usług i nie ma potwierdzonych wyników. To specyfikacja do późniejszego wdrożenia.

**Podręcznik:** [Rozdział 4](../../docs/chapters/JUNOS_Rozdzial_04_Initial_Configuration_and_System_Services.md), sekcja **4.15**.  
**Blueprint:** [4.1–4.15](../../docs/JUNOS_JNCIA_JNCIS_JNCIP_unified_blueprint.md).  
**Research/gotchas:** [CH04 research](../../docs/research/CH04_GOTCHAS_ZRODLA_I_WERYFIKACJA.md).

## Cel szkolenia

Zamiast tylko wkleić hostname, udowodnij, że urządzenie Junos po pierwszym uruchomieniu jest **zarządzalne, bezpieczne, obserwowalne i możliwe do odtworzenia**. Skonfiguruj konta i RBAC, management, AAA (opcjonalne), NTP, syslog, SNMPv3, archival SCP i rescue. Testuj **po obu stronach** połączenia, nie tylko `commit complete`.

## Topologia (izolowana EVE-NG)

~~~mermaid
flowchart LR
  A["LAB-ADMIN\nLinux 192.0.2.20/24"] --- S["NET-CH04-MGMT\nizolowany vSwitch"]
  S --- J["LAB-SRX01\nvSRX fxp0.0 192.0.2.10/24"]
  S --- B["LAB-SERVICES\nLinux 192.0.2.30/24"]
  C["Niezależna konsola EVE"] -. "OOB" .-> J
~~~

**Adresy TEST-NET nie są routowane publicznie.** Użyj wyłącznie izolowanego połączenia EVE, bez bridge do produkcji. Zanim podłączysz interfejs, sprawdź, czy konkretny vSRX udostępnia `fxp0` i które NIC w EVE/QEMU go reprezentuje; nie zakładaj kolejności.

## Przygotowanie i bezpieczeństwo

1. Odczytaj istniejące obrazy, wersję EVE, zasoby, obecne laby i mapę połączeń. Żadnych zmian w istniejącym labie Cisco 300-410.
2. Zapewnij konsolę niezależną od SSH i sprawdź, że działa.
3. Użyj nowego ID i nazwy labu, nie nadpisuj `.unl`, obrazów ani bazowych konfiguracji.
4. Po pierwszym logowaniu wykonaj **read-only**: `show version`, `show chassis hardware`, `show interfaces terse`, `show system commit`, `show configuration`.
5. Zapisz baseline root/local accounts/login message/management, **bez sekretów**.
6. Nie używaj globalnego `load factory-default` ani `zeroize` na istniejącym urządzeniu bez odrębnej, zatwierdzonej procedury.
7. Ustawiaj hasła i shared secrets **interaktywnie lub z vault**. Nie zapisuj ich w `.set`, GitHub, logach uruchomienia ani w pliku `VERIFY.md`.
8. Za każdym razem: `show | compare` → `commit check` → `commit confirmed` → realne testy → `commit`. **Nie wykonuj `commit check` po uruchomieniu pending confirmed commit**.
9. Zabezpiecz management (SSH z LAB-ADMIN; brak niekontrolowanego root SSH) dopiero po działającym indywidualnym koncie.

## Sekwencja ćwiczeń

| ID | Cel i związek z blueprintem | Wymagany dowód |
|---|---|---|
| CH04-A | Factory default / hardware/mgmt mapping (4.1/4.2) | PID, Junos, actual NICs, oddzielna konsola |
| CH04-B | Ustaw hostname, root pass, user `labadmin`, SSH (4.2/4.3) | Nowe SSH z LAB-ADMIN do LAB-SRX01 |
| CH04-C | Login classes `LAB-NOC`, konto `nocview` (4.4) | `show cli authorization` na 2 kontach |
| CH04-D | `apply-groups`, lokalny override i `display inheritance` (4.7) | Wykazana kolejność pierwszeństwa |
| CH04-E | Opcjonalnie AAA accept/timeout/reject (4.5) | Log Junos + log serwera TACACS+/RADIUS |
| CH04-F | NTP (4.8) | NTP association selected + status synchronized |
| CH04-G | Syslog (4.10) | Zdarzenie lokalne i ten sam event na LAB-SERVICES |
| CH04-H | SNMPv3 USM/VACM (4.9) | Zdalny SNMPv3 authPriv GET `sysUpTime.0` |
| CH04-I | SCP archival (4.11) | Zdalny plik na LAB-SERVICES + ewentualnie test restore |
| CH04-J | Traceoptions i cleanup w oddzielnym OSPF labie (4.12) | Konfiguracja/wyłączenie tracingu i log |
| CH04-K | Rescue save i **dwa rozdzielone testy**: auto-rollback i manual rescue (4.13) | Historia commit, read-only test, potwierdzony restore |
| CH04-L | Kompletny commissioning check (4.14) | PASS/FAIL/NOT TESTED dla wszystkich usług i cleanup |

**Trzeba faktycznie uruchomić** Linux NTP, syslog receiver, SSH/SCP i klienta SNMPv3. To nie jest automatycznie dostarczone przez ten opis. Jeśli usługa nie jest przygotowana, wpisz **NOT TESTED**.

## Testy kontrolowanych błędów

- AAA timeout versus aktywny reject, ale **tylko z niezależną konsolą** i działającym lokalnym break-glass.
- Zły NTP source/VRF (symuluj na izolowanym LAB, nie zmieniaj czasu podczas pending commit).
- Blokada syslog do LAB-SERVICES — porównaj zachowanie local i remote.
- Błędny VACM read-view / SNMPv3 username — odróżnij authentication failure od reachability.
- Archival SCP z niepoprawnym fingerprintem klucza lub brakiem dostępu do katalogu — nie akceptuj ślepo key.
- Grupy lokalne i inherited values — obserwuj `display inheritance`.
- Rescue przywracaj **dopiero gdy nie ma oczekującego confirmed commit**.

## Pliki do przygotowania w implementacji

- `topology/CH04_INITIAL_CONFIG.unl` — nowy, izolowany lab.
- `configs/CH04_safe_baseline.set` — **bez haseł**, tylko zweryfikowane bezpieczne statementy.
- `VERIFY.md` — statusy testów, realny Junos version, outputy i godziny, ale bez sekretów.
- `TROUBLESHOOTING.md` — symptom, hipoteza, dowód CLI, root cause, fix, ponowny test.
- `baseline/README.md` — sposób bezpiecznego dokumentowania poprzedniego stanu.

## Statusy

- **PLANNED:** opis, bez plików `.unl`.
- **GENERATED:** pliki stworzone, lecz nieuruchomione.
- **STATIC VALIDATED:** poprawny XML/struktura; nie dowodzi usług.
- **RUNTIME VALIDATED:** wykonano realne testy z dowodami dla danej wersji obrazu.

Wynik ostateczny oznaczaj per ćwiczenie PASS/FAIL/NOT TESTED. Nie twierdź, że cała topologia działa, gdy brakuje NTP, SCP albo NMS. Żadne hasło, token ani realna konfiguracja klienta nie mogą trafić do publicznego repozytorium.

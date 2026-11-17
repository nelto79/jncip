# LAB CH05 — Operational Monitoring, Maintenance & Fault Diagnosis on Junos

**Status: PLANNED / NOT RUNTIME VALIDATED.** To gotowa specyfikacja szkoleniowa, ale **nie wygenerowano nowego pliku EVE-NG .unl, nie uruchomiono vSRX ani nie przeprowadzono testów**. Nie używaj „PASS”, dopóki nie masz realnych dowodów.

**Podręcznik:** [Rozdział 5](../../docs/chapters/JUNOS_Rozdzial_05_Operational_Monitoring_and_Maintenance.md), sekcja **5.9**.  
**Blueprint:** [5.1–5.9](../../docs/JUNOS_JNCIA_JNCIS_JNCIP_unified_blueprint.md).  
**Źródła/gotchas:** [CH05 Research](../../docs/research/CH05_MONITORING_MAINTENANCE_GOTCHAS_I_ZRODLA_2026-11-17.md).

## Cel

Nauka **diagnozy na podstawie przyczyn i dowodów**, nie mechaniczne wykonywanie `show`. Uruchom jedno poprawne połączenie przez dwa vSRX, potem osobno symuluj:
1. **administracyjnie wyłączony transit link**;
2. **brak statycznej trasy**;
3. **poprawną składniowo, lecz blokującą security policy**;
4. dodatkowo **zły prefiks IPv4 /33** odrzucony przez parser.

Każdy test: precheck → hipoteza → fault → odczyt rzeczywistego stanu → test klienta → auto-rollback → postcheck.

## Nowa, izolowana topologia

~~~mermaid
flowchart LR
  A["PC-A 192.0.2.10/24"] --- R1["R1-vSRX\nge-0/0/1 .1/24\nge-0/0/0 198.51.100.1/30"]
  R1 --- R2["R2-vSRX\nge-0/0/0 198.51.100.2/30\nge-0/0/1 203.0.113.1/24"]
  R2 --- B["PC-B 203.0.113.10/24"]
  CON["Independent EVE consoles"] -.-> R1
  CON -.-> R2
~~~

Adresy **RFC 5737 TEST-NET**. Management i konsola muszą być **niezależne** od `ge-0/0/0`. Nazwy `ge` są założeniem do sprawdzenia w `show interfaces terse`, NIE faktem o aktualnej instancji.

## Zakres bazowy

### R1
- `ge-0/0/1.0 = 192.0.2.1/24`, zone `CLIENTS`;
- `ge-0/0/0.0 = 198.51.100.1/30`, zone `TRANSIT`;
- static route `203.0.113.0/24 -> 198.51.100.2`;
- security policy `CLIENTS → TRANSIT` dopuszczająca `junos-ping`;
- host-inbound ping tylko na testowanych adresach.

### R2
- `ge-0/0/0.0 = 198.51.100.2/30`, zone `TRANSIT`;
- `ge-0/0/1.0 = 203.0.113.1/24`, zone `SERVERS`;
- static route `192.0.2.0/24 -> 198.51.100.1`;
- security policy `TRANSIT → SERVERS` dopuszczająca `junos-ping`.

**Dokładne linie `set` znajdują się w sekcjach 5.9.4–5.9.5 podręcznika.** Nie commituj ich, jeśli urządzenie ma istniejące factory-default NAT, overlapping addresses albo inne zones. Preflight i `show | compare` obowiązkowe.

## Preflight read-only

1. Sprawdź wersję EVE, obrazy, RAM, CPU, istniejące labs i port mappings.
2. Utwórz osobną kopię/nowy lab z nowym UUID; nie zmieniaj CH02 diamond, CH03, CH04 ani Cisco 300-410.
3. Zweryfikuj `show version`, `show chassis hardware`, `show interfaces terse` obu SRX.
4. Potwierdź console independent of SSH i brak sieci zewnętrznej.
5. Zapisz pełny baseline **bez sekretów**.
6. Uruchom politykę ICMP i static routes, dopiero potem potwierdź PC-A → PC-B ping.
7. **Wstrzymaj fault**, jeśli bazowy ping nie działa. Inaczej nie da się interpretować wyniku.

## Scenariusze testowe

| ID | Fault injection | Oczekiwany stan i test | Recovery |
|---|---|---|---|
| CH05-A | R1 `set interfaces ge-0/0/0 disable` | Transit admin down, brak komunikacji PC-A→PC-B | `commit confirmed 2` bez potwierdzenia, auto-rollback |
| CH05-B | R1 `delete routing-options static route 203.0.113.0/24` | Port transit up, brak drogi do PC-B (o ile nie istnieje inna) | Auto-rollback, verify route |
| CH05-C | R2: permit→deny w policy `CH05-ALLOW-PING` | Nowe ICMP sessions blokowane, choć route i porty up; stare sesje mogą się utrzymać | Auto-rollback i test nowej sesji |
| CH05-D | `set ... address 192.0.2.1/33` | Parser powinien odrzucić niedozwolony prefix | Nie commitować, zweryfikować diff |
| CH05-E | `monitor start messages`, `monitor interface` | Widzisz moment zmiany i zdarzenia | Monitor stop; bez usuwania logów |
| CH05-F | `monitor traffic` host-bound vs transit | Zrozumienie ograniczeń RE capture | Nie utożsamiaj brak pakietów z brakiem forwarding |
| CH05-G | Optional upgrade/read-only planning | Version, supported image, storage, rollback path | Nie instaluj software bez osobnego approved lab |

## Bezpieczeństwo

- Fault inject na **oddzielnym** testowym interfejsie transit, NIE na interfejsie management.
- W czasie `commit confirmed` **nie wykonuj dodatkowego `commit check`**, bo może potwierdzić zmianę i wyłączyć automatyczny timer.
- Nie wykonuj bez zgody `clear security flow session all`, `request system reboot`, `load override`, `zeroize` ani `request system storage cleanup`.
- Nie publikuj haseł, prywatnych kluczy, tokenów, obrazów QEMU, rzeczywistych adresów ani full config klienta.
- Nie przypisuj `show interfaces diagnostics optics` do vSRX jako sprawdzanego lasera — wymaga prawdziwej obsługiwanej wkładki sprzętowej.

## Dokumentowanie dowodów

W `VERIFY.md` utwórz tabelę: ID, date/time, Junos release, source command, raw excerpt (bez sekretów), expected, actual, conclusion, PASS/FAIL/NOT TESTED, cleanup. Dla każdego fault co najmniej:
- precheck z CLI R1/R2;
- potwierdzenie działającego ICMP PC-A → PC-B;
- dokładna komenda fault i diff;
- read-only wynik RIB/FIB, interface, SRX flow, log oraz klient;
- wynik auto-rollback z `show system commit`;
- końcowe potwierdzenie przywrócenia ruchu.

Osobno zapisz `TROUBLESHOOTING.md` — root cause i sekwencję weryfikacyjną; `baseline/README.md` — jak zapisano konfigurację bez sekretów.

## Etapy gotowości

- **PLANNED:** tylko dokumenty. **Obecny status.**
- **GENERATED:** nowy plik `.unl`, bez realnego uruchomienia.
- **STATIC VALIDATED:** potwierdzona składnia `.unl` i pliki; nie stanowi runtime PASS.
- **RUNTIME VALIDATED:** uruchomiono EVE i zapisano dowody realnych ping, commit confirmed, interface/route/policy events oraz cleanup.

**Warunek zaliczenia:** uczący się potrafi odróżnić L1 down, brak routing i blokadę policy oraz wykazać to odczytami, a nie zgadywać.

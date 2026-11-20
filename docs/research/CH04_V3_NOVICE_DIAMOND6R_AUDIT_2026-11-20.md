# CH04 v3 — audyt redakcyjny, blueprint i status walidacji

**Data:** 2026-11-20  
**Rozdział kanoniczny:** `docs/chapters/JUNOS_Rozdzial_04_Initial_Configuration_and_System_Services.md`  
**Archiwum v2:** `docs/archive/JUNOS_Rozdzial_04_pre_beginner_rewrite_2026-11-20.md`  
**Odbiorca:** Network Engineer z około rocznym doświadczeniem, bez wymogu znajomości Cisco, CCNA, JNCIA ani JNCIS.  
**Status:** **SOURCE REVIEWED / STATIC EDITORIAL REVIEW / NOT RUNTIME VALIDATED**.

## Kompletność blueprintu

| Punkt | Miejsce w tekście | Uczeń potrafi wyjaśnić |
|---|---|---|
| 4.1 | §4.1 | Factory-default nie jest uniwersalnym pustym config; ZTP a zeroize |
| 4.2 | §4.2 | Hostname, DNS, OOB, management i SSH |
| 4.3 | §4.3 | Root/local users, interaktywne hasła, klucze SSH i root SSH |
| 4.4 | §4.4 | Klasy logowania, authentication vs authorization, ograniczenia praw |
| 4.5 | §4.5 | TACACS+/RADIUS, timeout vs reject, source/VRF i local fallback |
| 4.6 | §4.6 | Fizyczny/logiczny port, family inet i różne interfejsy management |
| 4.7 | §4.7 | Definicja group, apply-groups, inheritance i lokalne wyjątki |
| 4.8 | §4.8 | NTP association, synchronized vs skonfigurowany NTP |
| 4.9 | §4.9 | SNMPv3 USM/VACM, authPriv, polling vs trap/inform, engine ID |
| 4.10 | §4.10 | Lokalny syslog, kolektor, source-address i poprawny timestamp |
| 4.11 | §4.11 | Off-device archival, kontrola transferu i restoration |
| 4.12 | §4.12 | Zastosowanie/ryzyko traceoptions i usuwanie po diagnostyce |
| 4.13 | §4.13 | Rescue save i rollback rescue vs commit confirmed |
| 4.14 | §4.14 | Spójny proces odbioru i dowody end-to-end |
| 4.15 | §4.15 | 1 vSRX → usługi administracyjne → sześć routerów DIAMOND-6R |

## Przeprowadzona kontrola statyczna

- **15 183 słów**, około **116 771 znaków**, **94 bloki Junos CLI**, **2 diagramy Mermaid**.
- Każdy blok `~~~junos` ma bezpośrednio nad nim hostname R1–R6 i jawny Operation/Configuration Mode.
- Wszystkie bloki źródłowe Markdown są zamknięte; nie pozostały dosłowne sekwencje escape `\\``.
- Wszystkie 15 punktów blueprintu są przedstawione jako lekcje, a dodatkowo §4.TEST zawiera **25 pytań z odpowiedziami i wyjaśnieniami**, §4.REF mapę usług.
- R5/R6 to routery vSRX, nie PC-A/PC-B VPCS. Wszystkie połączenia i interfejsy danych odpowiadają DIAMOND-6R; management `203.0.113.0/24` to **oddzielny, opcjonalny segment labowy**, nie zmiana istniejących łączy.
- Nie umieszczono rzeczywistych haseł użytkowników, TACACS+/RADIUS, SNMPv3 ani prywatnych kluczy SSH. W przykładach SNMPv3 i SCP jawnie opisano brakujące elementy bezpiecznego wdrożenia.
- Nie wygenerowano `.unl`, nie uruchamiano usług w EVE-NG ani nie wykonano faktycznych `commit check` na vSRX.

## Techniczne warunki, które muszą zostać sprawdzone w runtime

1. **vSRX/OOB:** `show version`, faktyczne mappingi `fxp0`/innego NIC, istniejące strefy management, stan `mgmt_junos` oraz routing do `LAB-SERVICES`. Nie zakładaj, że OOB 203.0.113.10 istnieje automatycznie.
2. **First commit:** potwierdź wymóg `system root-authentication` przy pierwszym zatwierdzeniu nowej konfiguracji.
3. **Local SSH:** dwa niezależne logowania po zmianie, nie tylko otwarta sesja; root SSH ograniczaj dopiero po działającym `labadmin`.
4. **AAA:** prawdziwy serwer i logi po obu stronach. Osobno test timeout oraz reject przy `authentication-order tacplus` i `[ tacplus password ]`. Szczegóły zależą od login class i release.
5. **Groups:** `display inheritance` i lokalne precedence; definicja bez apply-groups nie wystarcza.
6. **NTP:** `show ntp associations` oraz `show ntp status` z wybraną synchronizacją, nie tylko ICMP.
7. **SNMPv3:** kompletny rzeczywisty USM + VACM + zgodne auth/privacy; odczyt authPriv `sysUpTime.0` z NMS. Sam przedstawiony w podręczniku fragment VACM **nie** oznacza poprawnego SNMPv3.
8. **Syslog:** to samo zdarzenie w `show log messages` i na serwerze kolektora z prawidłowym czasem.
9. **SCP archive:** bezpieczne host-key trust, właściwa autoryzacja, katalog i rzeczywisty plik na serwerze; ewentualny test odtworzenia kopii.
10. **Traceoptions:** wyłącznie w świadomie uruchomionym osobnym wariancie OSPF, limitowany rozmiar i cleanup.
11. **Rescue:** rescue istnieje, jest właściwa dla stanu zaakceptowanego, nie myl ręcznego `rollback rescue` z automatycznym cofnięciem `commit confirmed`.
12. **Safe commit:** `commit check` wykonujemy **przed** warunkową aktywacją; podczas oczekiwania po `commit confirmed` nie wykonujemy `commit check`, które mogłoby potwierdzić warunkową zmianę.
13. **Six-router management:** na każdym R1–R6 osobna candidate, commit, pre/post-check i status usług. Nie uruchamiaj kilku timerów jednocześnie bez specjalnego planu.

## Powiązane dokumenty

- [Unified Blueprint](../JUNOS_JNCIA_JNCIS_JNCIP_unified_blueprint.md)
- [DIAMOND-6R](../../labs/reference-diamond-6vsrx/README.md)
- [LAB CH04](../../labs/ch04-initial-config/README.md)
- [Pierwotne gotchas i źródła](CH04_GOTCHAS_ZRODLA_I_WERYFIKACJA.md)
- [Oficjalne reguły authentication order](https://www.juniper.net/documentation/us/en/software/junos/user-access/topics/topic-map/junos-os-authentication-order.html)
- [Oficjalna dokumentacja SNMPv3](https://www.juniper.net/documentation/us/en/software/junos/network-mgmt/topics/topic-map/configure-snmpv3.html)
- [Oficjalna dokumentacja archival](https://www.juniper.net/documentation/us/en/software/junos/junos-overview/cli/topics/task/junos-software-system-management-router-configuration-archiving.html)

**Wniosek:** Rozdział jest gotowy do czytania i oceny dydaktycznej, natomiast testy implementacyjne pozostają zaplanowane; nie deklarujemy działania usług bez dowodów z EVE-NG i serwerów.

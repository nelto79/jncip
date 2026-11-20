# CH03 v3 — audyt przebudowy dla początkującego

**Data:** 2026-11-20  
**Rozdział kanoniczny:** `docs/chapters/JUNOS_Rozdzial_03_Junos_User_Interfaces_CLI.md`  
**Archiwum:** `docs/archive/JUNOS_Rozdzial_03_pre_beginner_rewrite_2026-11-20.md`  
**Odbiorca:** Network Engineer ~1 rok doświadczenia, znajomość podstaw adresowania, routera, switcha i VLAN; brak wymogu wiedzy Cisco/CCNA/JNCIA.

## Pokrycie blueprintu

| Cel | Sekcja |
|---|---|
| 3.1 Funkcje CLI i zasady interakcji | 3.1 |
| 3.2 Operational vs Configuration | 3.2 |
| 3.3 Hierarchia i nawigacja | 3.3 |
| 3.4 Pomoc kontekstowa i autocomplete | 3.4 |
| 3.5 Filtrowanie wyjścia | 3.5 |
| 3.6 Active/candidate, shared/private/exclusive | 3.6 |
| 3.7 set/delete/activate/deactivate/copy/replace/save | 3.7 |
| 3.8 Przeglądanie, porównywanie i load | 3.8 |
| 3.9 Rollback i rescue | 3.9 |
| 3.10 J-Web | 3.10 |
| 3.11 Procedura bezpiecznego change | 3.11 |
| Rozszerzenie praktyczne | 3.LAB, 3.TS, 3.TEST, 3.REF |

## Audyt statyczny

- Około **11 038 słów**, 84 343 znaki, 3 diagramy Mermaid, 90 bloków poleceń Junos.
- Każdy blok Junos ma bezpośrednio nad nim hostname R1–R6 i Operation/Configuration Mode.
- Bloki `~~~` zamknięte poprawnie; brak przypadkowego tekstu escape w składni `inline code`.
- Nie pozostały stare role VPCS/PC-A/PC-B ani adresacja inna niż DIAMOND-6R.
- Ćwiczenia prowadzą od pojedynczego vSRX (candidate/active) przez współpracę administratorów po workflow na całym DIAMOND-6R.
- Brak testów EVE-NG. **PLANNED / NOT RUNTIME VALIDATED**. Poprawność składni jest przewidywana na podstawie dokumentacji, ale nie została potwierdzona na konkretnym obrazie.

## Krytyczne niuanse zachowania

1. `commit check` **przed** `commit confirmed` pomaga sprawdzić konfigurację; `commit check` **po** `commit confirmed` może **potwierdzić** warunkowy commit, zamiast zachować rollback timer.
2. `configure` jest współdzieloną candidate; `configure private` przechowuje osobną candidate, ale nie blokuje zmian active przez innych administratorów.
3. `update` w private mode i `load update` są różnymi mechanizmami.
4. `load replace` wymaga prawidłowego znacznika `replace:`, aby zastąpić oznaczone gałęzie; `load override` zastępuje całą candidate.
5. `rollback 0` odrzuca niezacommitowane zmiany candidate; `rollback 1` ładuje poprzednią konfigurację do candidate, nie aktywuje jej samodzielnie.
6. `save` tworzy plik konfiguracyjny, lecz nie jest commitem ani niezależnym backupem.
7. J-Web jest zależny od modelu i wersji; nie udajemy, że jeden interfejs graficzny i ścieżka menu pasują do wszystkich SRX/EX.
8. W labie z wieloma routerami unikamy jednoczesnego uruchamiania wielu `commit confirmed` bez osobnego planu i konsoli.

## Warunki przyszłej walidacji runtime

- `show version` i rzeczywiste mappingi interfejsów dla użytego obrazu vSRX.
- Dwie równoległe sesje administracyjne, wynik shared/private oraz zachowanie `update` po zmianie active.
- Wyniki `show | compare`, `commit check`, `commit confirmed`, świadomego potwierdzenia i automatycznego cofnięcia wraz z timestampami.
- Test importu `load set terminal`/scoped merge oraz kontrola diffu.
- Odczyt historii rollback, weryfikacja odpowiedniej rescue i brak wpływu na zarządzanie.
- Read-only status wszystkich sześciu routerów DIAMOND-6R po testowych zmianach opisów R1/R4.

## Źródła referencyjne

- [Overview of configure](https://www.juniper.net/documentation/us/en/software/junos/cli/topics/topic-map/configure-command.html)
- [Commit the Configuration](https://www.juniper.net/documentation/us/en/software/junos/cli/topics/topic-map/junos-configuration-commit.html)
- [Loading Configuration Files](https://www.juniper.net/documentation/us/en/software/junos/cli/topics/topic-map/junos-config-files-loading.html)
- [show system rollback](https://www.juniper.net/documentation/us/en/software/junos/cli-reference/topics/ref/command/show-system-rollback.html)
- [J-Web Documentation](https://www.juniper.net/documentation/product/us/en/j-web/)

**Status końcowy:** redakcja zakończona; walidacja runtime pozostaje do wykonania.

# LAB CH03 — Junos CLI Change Management: Candidate, Commits, Private Sessions, Rollback

**Status: PLANNED — opis ćwiczeń; jeszcze nie wygenerowano .unl i nie wykonano runtime w EVE-NG.**

Powiązania:
- [Pełny rozdział 3](../../docs/chapters/JUNOS_Rozdzial_03_Junos_User_Interfaces_CLI.md) — od początku 3.1 do 3.END;
- [Unified Blueprint 3.1–3.11](../../docs/JUNOS_JNCIA_JNCIS_JNCIP_unified_blueprint.md);
- [AGENTS.md](../../AGENTS.md).

## Dlaczego jedna instancja, nie cztery?

Cele CH03 są inne od CH02: chcemy obserwować **transakcje konfiguracji, nie trasy przez wielu sąsiadów**. Wystarczy R1-vSRX, dwie niezależne sesje administracyjne i konsola EVE jako recovery. Istniejącą topologię 4×vSRX z CH02 można wykorzystać ponownie, lecz **nie jest to wymagane**.

~~~mermaid
flowchart LR
 A["Admin A — SSH"] --> R1["R1-vSRX — Junos"]
 B["Admin B — SSH"] --> R1
 C["Console EVE — recovery"] --> R1
~~~

## Bezpieczeństwo / preflight

1. Przeprowadź **read-only** inspekcję hosta EVE, wersji, obrazów vSRX, zasobów i bieżącego labu.
2. Zidentyfikuj management interface oraz trasę SSH, nie zgaduj, że to ge-0/0/1 albo fxp0.
3. Potwierdź realny dostęp do konsoli poza SSH.
4. Sprawdź `show system users`, `show system commit`, `show configuration` dla testowanego statementu oraz `show interfaces terse`.
5. Nigdy nie rób `rollback 0` na shared candidate zawierającej zmiany innych osób.
6. W tym labie nie wykonuj `load override` na całej konfiguracji ani nie odcinaj management. Komendy destrukcyjne omawiać jedynie jako hipotezy.
7. Przed testami zapisz baseline dla login message, description wybranego portu danych, stanu systemu i historii commit.
8. Nie commituj sekretów, rzeczywistego IP management ani pełnego zrzutu konfiguracji użytkownika do GitHuba.

## Pomiary i testy

| ID | Eksperyment | Warunek sukcesu |
|---|---|---|
| CH03-A | Własny private candidate bez commit | candidate diff jest; active bez zmiany |
| CH03-B | Dwie sesje shared `configure` | obie widzą wspólne poprawki |
| CH03-C | Dwie private candidates, jeden wcześniejszy commit | obsłużono stale candidate przez `update`; konflikt opisany, jeśli wystąpił |
| CH03-D | `load set terminal` i `load merge terminal` | różnica zgodna z zamierzeniem, nic nie skasowano |
| CH03-E | `commit confirmed 2` bez potwierdzenia | rzeczywisty automatyczny rollback i wpis w historii |
| CH03-F | `show system rollback` i rescue | historia zinterpretowana prawidłowo, rescue zapisana jedynie po zgodzie |
| CH03-G | Opcjonalny loss-of-SSH (tylko po OOB check) | recovery wykonano i udokumentowano, nigdy na istniejącym site |

**Ważna pułapka:** `commit check` po `commit confirmed` może **potwierdzić** zmianę. W CH03-E nie wolno ponownie go wykonywać podczas obserwacji timera. Weryfikuj `show system commit` i aktywny stan przez operational `show` lub `run`, a nie kolejny commit.

## Artefakty do stworzenia przez Codexa

- `topology/CH03_CONFIG_MGMT.unl` — opcjonalny, nowy, izolowany EVE lab; jeśli korzystasz z CH02, tylko jeśli celowo tworzysz kopię, **bez nadpisywania** istniejącego .unl.
- `VERIFY.md` — realne kroki, outputy, czasy i status PASS/FAIL per scenariusz.
- `TROUBLESHOOTING.md` — co się dzieje przy konflikcie, pending commit, błędnym load, po rollback.
- `baseline/README.md` — wyłącznie opis sposobu dokumentowania baseline, bez danych uwierzytelniających.
- `configs/CH03_safe_changes.set` — tylko bezpieczny login banner i opis testowego portu po potwierdzeniu realnego mapowania interfejsów.

## Warunki oznaczenia statusu

- **PLANNED:** masz instrukcję, nie masz jeszcze wygenerowanego laba.
- **GENERATED:** istnieje .unl i pliki konfiguracyjne; brak rzeczywistego uruchomienia.
- **STATIC VALIDATED:** struktura XML i plików sprawdzona; nadal brak dowodu działania Junos.
- **RUNTIME VALIDATED:** udokumentowano rzeczywisty `commit`, `show | compare`, `commit confirmed` i auto-rollback z czasami oraz wynikami, w konkretnej wersji vSRX.

**Nie oznaczaj runtime PASS na podstawie wygenerowanego przewidywanego outputu.**

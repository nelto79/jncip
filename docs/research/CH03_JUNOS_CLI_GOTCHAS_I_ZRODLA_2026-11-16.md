# Rozdział 3 — gotchas, źródła i uwagi do walidacji

**Data przeglądu:** 2026-11-16. **Rozdział ucznia:** [Junos User Interfaces — CLI](../chapters/JUNOS_Rozdzial_03_Junos_User_Interfaces_CLI.md). **Blueprint:** 3.1–3.11.

To jest plik redakcyjny dla autora, audytora i Codexa — **nie osobny bank pytań do wkuwania**. Pytania, scenariusze Cisco → Junos i konsekwencje techniczne są już wplecione do właściwych sekcji rozdziału 3.

## Najważniejsze, zweryfikowane pułapki

1. **Operational mode `>` nie jest trybem tylko do odczytu.** Zmiany w systemie można wywołać poleceniami operacyjnymi (`clear`, `request`, reboot), mimo że normalne `set` wykonuje się w configuration mode. Źródło: [CLI Overview](https://www.juniper.net/documentation/us/en/software/junos/cli/topics/topic-map/cli-overview.html).
2. **`configure` współdzieli candidate.** Commit jednego administratora może zatwierdzić zmiany drugiego. `configure private` izoluje edycję, lecz aktywna konfiguracja może zmienić się w tle; `update` próbuje połączyć aktualny active z prywatnymi zmianami. `configure exclusive` zakłada lock, a jego zachowanie po automatycznym rollbacku jest szczególne. Źródło: [Configure Command Overview](https://www.juniper.net/documentation/us/en/software/junos/cli/topics/topic-map/configure-command.html).
3. **`commit check` po oczekującym `commit confirmed` MOŻE potwierdzić commit.** Krytyczne dla nocnego change management. Nie wykonuj ponownego `commit check` „dla bezpieczeństwa”, gdy oczekujesz na potencjalny rollback. Źródło: [commit CLI Reference](https://www.juniper.net/documentation/us/en/software/junos/cli-reference/topics/ref/command/commit.html).
4. **`commit check` nie testuje sieci end-to-end.** Poprawny candidate może nadal odciąć SSH lub usługi. Źródło: [Commit the Configuration](https://www.juniper.net/documentation/us/en/software/junos/cli/topics/topic-map/junos-configuration-commit.html).
5. **`rollback 0` vs `rollback 1`.** 0 odrzuca niezacommitowane zmiany candidate; 1 ładuje poprzednią zatwierdzoną konfigurację do candidate. Sam `rollback` nie musi aktywować zmiany. Domyślnie wersje 0–49 obejmują aktualną + 49 wcześniejszych. Źródło: [rollback](https://www.juniper.net/documentation/us/en/software/junos/cli-reference/topics/ref/command/rollback.html).
6. **`show route | count` liczy linie, nie trasy.** `match`, `find`, `except` mają odmienną semantykę; filtracja może ukryć kontekst. Źródło: [pipe CLI Reference](https://www.juniper.net/documentation/us/en/software/junos/cli-reference/topics/ref/command/pipe.html).
7. **`show | display set` dotyczy konfiguracji, nie dowolnego operational output.** Źródło: [show | display](https://www.juniper.net/documentation/us/en/software/junos/cli-reference/topics/ref/command/show-pipe-display.html).
8. **`load override` zastępuje całą candidate.** `load replace` szuka znacznika `replace:`. `load set` wczytuje komendy, a `load update` nie jest tym samym co `update` private candidate. Źródła: [load CLI Reference](https://www.juniper.net/documentation/us/en/software/junos/cli-reference/topics/ref/statement/load.html), [Loading Configuration Files](https://www.juniper.net/documentation/us/en/software/junos/cli/topics/topic-map/junos-config-files-loading.html).
9. **`delete` całej gałęzi vs usunięcie jednego leaf.** `deactivate` zachowuje statement, ale oznacza go inactive; może odciąć ruch, kiedy następuje commit. Źródło: [Configuration Mode Overview](https://www.juniper.net/documentation/us/en/software/junos/cli/topics/topic-map/cli-configuration.html).
10. **`replace pattern` używa regex i może wpłynąć na wiele identyfikatorów.** `upto n` liczy obiekty, nie wystąpienia w tekście. Źródło: [replace](https://www.juniper.net/documentation/us/en/software/junos/cli-reference/topics/ref/command/replace.html).
11. **`save /var/tmp/...` to zapis tekstu, nie commit.** Ponadto lokalny backup nie jest backupem off-device. Źródło: [save](https://www.juniper.net/documentation/us/en/software/junos/cli-reference/topics/ref/statement/save-cli-config.html).
12. **`show system rollback` ma ograniczenia użycia przez `run`.** Nie wszystkie operational commands działają jako `run` w configuration mode. Źródło: [show system rollback](https://www.juniper.net/documentation/us/en/software/junos/cli-reference/topics/ref/command/show-system-rollback.html).
13. **Nie wszystkie systemy Junos mają ten sam shell.** Klasyczny Junos jest powiązany z FreeBSD, Junos OS Evolved z Linux; logowanie do shell to nie „wejście do jądra”. Źródło: [Junos OS Evolved CLI](https://www.juniper.net/documentation/us/en/software/junos/cli-evo/cli-evo.pdf).
14. **J-Web jest platform/release dependent.** Wspierany EX/SRX może uruchamiać J-Web przez HTTPS, ale certyfikaty, interfejsy, FIPS i konkretne pakiety GUI mają ograniczenia. Na EX4400 port management to **me0** (`me0.0`), natomiast na wielu routerach/SRX jest **fxp0**. Źródła: [J-Web](https://www.juniper.net/documentation/product/us/en/j-web/), [EX4400 Quick Start](https://www.juniper.net/documentation/us/en/quick-start/hardware/qsg/ex4400/topics/topic-map/step-2-up-and-running.html).

## Rzetelność i rodzaje źródeł

- Materiał oparto na **publicznej dokumentacji producenta** i oryginalnych przykładach operatora.
- Nie deklarujemy dostępu do aktualnych pytań z interaktywnego oficjalnego JNCIA Practice Exam.
- Nie korzystamy z exam dumps. Zadania w 3.TEST to **autorskie scenariusze**, nie rzekomo autentyczne pytania certyfikacyjne.
- Polecenia nie były jeszcze wykonywane na konkretnym obrazie vSRX; faktyczne różnice platformowe i wsparcie opcji należy zweryfikować na EVE-NG przed oznaczeniem labu jako RUNTIME VALIDATED.
- Dla publicznej dokumentacji i screenshotów J-Web: **własne Mermaid** w podręczniku, a do screenshotów producenta linki, bez rehostingu cudzych plików graficznych bez licencji.

## Checklist na przyszły review

- Czy zostały zachowane punkty blueprintu 3.1–3.11?
- Czy polecenia są poprzedzone nazwą urządzenia i trybem?
- Czy `commit check` po pending commit zostało opisane poprawnie?
- Czy nie zrównano `save` z `commit` albo `load replace` z `load override`?
- Czy `configure private` uwzględnia stale candidate i `update`?
- Czy J-Web używa poprawnego `me0.0` dla EX4400 i bezpiecznego HTTPS?
- Czy w przykładach nie usuwamy aktywnego interfejsu management bez konsoli i izolacji?
- Czy dokument ucznia zawiera wyjaśnienia w tekście wykładu, a nie dopisek „Top 20 gotchas” na końcu?


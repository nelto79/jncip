# Rozdział 3. Junos User Interfaces — CLI, konfiguracja, bezpieczny change i J-Web

**Podręcznik:** JUNOS Enterprise Routing & Switching · JNCIA-Junos → JNCIS-ENT → JNCIP-ENT  
**Zakres Unified Blueprint:** **3.1–3.10 [A]** oraz **3.11 [EXT]**  
**Redakcja:** 16 listopada 2026 · materiał wykładowy, Cisco IOS/IOS XE → Junos OS  
**Platforma labowa:** vSRX w EVE-NG; odpowiedniki Cisco IOS XE oraz uwagi dla EX/QFX/MX i Junos OS Evolved  
**Status weryfikacji:** dokumentacja producenta sprawdzona; przykłady konfiguracji i laboratorium są **PLANNED / NOT RUNTIME VALIDATED** (nie wykonano ich na żywym vSRX).  
**Następna lekcja:** Rozdział 4 — initial configuration, użytkownicy, AAA, usługi i rescue configuration.

> **Umowa z uczniem:** nie będziemy zapamiętywać przypadkowych komend. Każda komenda odpowiada na konkretne pytanie: **co chcę zmienić, co system przygotował, co rzeczywiście działa, co może się nie udać i jak odzyskać dostęp?** Masz doświadczenie Cisco, więc wykorzystamy je jak trampolinę. Nie będziemy od nowa uczyć się routingu, tylko innego sposobu prowadzenia zmiany.

## 03:07 — incydent, od którego zaczyna się ta lekcja

W środku nocnego maintenance window masz przygotować nowe peeringi BGP, dopisać trzy prefiksy i zmodyfikować filtr wejściowy. Na Cisco IOS XE wydajesz kolejne polecenia \`configure terminal\`. Większość z nich działa od razu na bieżącej konfiguracji. Ktoś pomylił kolejność, zastosował błędny filtr lub niechcący odciął SSH. Teraz dowiadujesz się, jak dobra była Twoja ścieżka OOB i czy uruchomiłeś mechanizm bezpiecznego cofania zmian.

Na Junos także możesz odciąć sobie sieć. **To nie jest system magicznie wolny od błędów operatora.** Ale filozofia pracy zmienia się radykalnie: konfiguracja przygotowywana jest jako **candidate**, a dopiero commit aktywuje ją jako nową **active configuration**. Zanim naciśniesz przycisk, możesz zobaczyć różnicę, zweryfikować poprawność i ustawić automatyczne przywrócenie poprzedniej wersji.

Wyobraź sobie pilota samolotu. Pilot nie zmienia ustawień nawigacji całego lotniska za każdym razem, gdy edytuje plan lotu na ekranie. Najpierw przygotowuje i sprawdza plan, potem go aktywuje. **Candidate** jest planem przygotowanym; **active** — planem, według którego system aktualnie pracuje. Ale ważne: nawet plan zatwierdzony przez komputer może poprowadzić Cię na niewłaściwe lotnisko, jeśli podałeś niewłaściwy cel. To ograniczenie \`commit check\`.

### Zanim przejdziemy do CLI — trzy rodzaje wiedzy

**Na pamięć:** znaki promptu \`>\` i \`#\`; operational vs configuration; \`set/delete/activate/deactivate\`; \`show | compare\`; \`commit\`, \`commit confirmed\`, \`rollback 0/1\`; \`configure private\`; różnice \`load merge/replace/override/set\`.

**Do zrozumienia:** shared vs private candidate; dlaczego \`commit check\` nie jest testem reachability; co oznacza „zapisano plik” vs „zatwierdzono konfigurację”; czemu utrata sesji po \`commit confirmed\` nie zawsze oznacza, że SSH samo odżyje.

**Do przećwiczenia:** wejście do hierarchii, szybkie \`?\` i \`Tab\`, filtrowanie outputu, porównanie dwóch konfiguracji, bezpieczny import fragmentu, dwa równoległe logowania administracyjne, celowy rollback i weryfikacja historii zmian.

~~~mermaid
flowchart LR
  subgraph IOS["Cisco IOS/IOS XE — typowy ręczny workflow"]
    A1["conf t"] --> A2["polecenie 1, 2, 3..."]
    A2 --> A3["zmiany zwykle od razu w running-config"]
    A3 --> A4["test / ewentualnie rollback mechanizmami Cisco"]
  end
  subgraph JUN["Junos OS — workflow transakcyjny"]
    B1["configure"] --> B2["candidate: set/delete/load"]
    B2 --> B3["show | compare"]
    B3 --> B4["commit check"]
    B4 --> B5["commit confirmed"]
    B5 --> B6["test produkcyjny"]
    B6 --> B7["commit lub automatyczny rollback"]
  end
~~~

*Diagram dydaktyczny: na wybranych platformach IOS XE istnieją również archiwizacja, configuration replace i confirmed rollback. Porównanie przedstawia typowy styl pracy CLI, a nie twierdzenie, że Cisco nie ma mechanizmów transakcyjnych.*

---

## 3.1. Funkcje CLI i zasady interakcji z systemem [A]

### 3.1.1. Dlaczego Junos CLI nie przypomina „samego routera z komendami”

W poprzednim rozdziale poznaliśmy RE i daemony. CLI nie jest osobną magiczną instancją routingu: jest interfejsem człowieka do stanu urządzenia i systemu konfiguracji, powiązanym z usługami zarządzania (m.in. \`mgd\`). Możesz zarządzać urządzeniem przez konsolę, SSH, protokoły automatyzacyjne, a na wspieranych platformach także J-Web.

Przy pierwszym logowaniu do Junos zwykle widzisz prompt podobny do \`user@R1>\`. Pierwsza ważna różnica względem Cisco: **nie używasz polecenia \`enable\`**, by podnieść uprawnienia. Klasa użytkownika / autoryzacja określa, jakie polecenia możesz wykonywać. Sam prompt \`>\` **nie oznacza**, że masz uprawnienia superuser.

**Most z Cisco:** w IOS/IOS XE przyzwyczaiłeś się do user EXEC, privileged EXEC i global configuration. W Junos rozdział opiera się na **trybie operational i configuration**, a poziom uprawnień wynika z konta/klasy oraz konfiguracji uwierzytelniania i autoryzacji. Pełne login classes i TACACS/RADIUS pojawią się w rozdziale 4.

### 3.1.2. Trzy środowiska, z których nie wszystkie są „trybem CLI”

**Operational CLI** to interfejs diagnostyczny/zarządczy. **Configuration CLI** to edytor hierarchii. **System shell** to środowisko leżącego pod Junos systemu operacyjnego: zwykle powłoka FreeBSD w klasycznym Junos, a na Junos OS Evolved — środowisko Linux.

Nie nazywaj każdego promptu z \`%\` „trybem jądra”. To powłoka **użytkownika systemu operacyjnego**, nie jądro. I nie zakładaj, że na każdej platformie shell musi wyglądać identycznie.

**Urządzenie: R1-vSRX — Operational Mode (przykładowa sesja, ilustracyjna)**

~~~text
user@R1-vSRX> show version
user@R1-vSRX> show cli
user@R1-vSRX> show system users
~~~

\`show cli\` odpowiada na pytanie o ustawienia sesji (terminal, screen length/width, completion). \`show system users\` pomaga zobaczyć, kto jeszcze jest zalogowany. Właśnie od tego zaczyna się dobry precheck do nocnego change'u.

**Pytanie kontrolne:** czy \`>\` oznacza „bezpieczny read-only”? **Nie.** W operational mode możesz także wykonać \`clear\`, \`request\`, wyłączenie urządzenia i inne działania zmieniające stan systemu, zależnie od uprawnień. Polecenia **nie edytują candidate configuration**, ale to nie znaczy, że nie mogą wywołać przestoju.

### 3.1.3. Shell i gotcha z bezpieczeństwem

**Urządzenie: R1-vSRX — Operational Mode, tylko w LAB z odpowiednimi uprawnieniami**

~~~junos
start shell
~~~

**Urządzenie: R1-vSRX — shell systemowy, przykład klasycznego Junos**

~~~sh
pwd
exit
~~~

Nie używaj shella do rutynowego monitoringu, skoro istnieją komendy CLI. Nie edytuj \`juniper.conf\` bezpośrednio edytorem systemowym ani nie traktuj dostępu do shella jako odpowiednika \`configure\`. Ewentualne problemy z plikami i recovery wymagają jawnej procedury oraz zgodności z wersją.

**Gotcha Cisco:** \`start shell\` nie jest „bezpieczniejszym enable”. To przejście do innej warstwy systemu, gdzie błędy nie są objęte zwykłą transakcją \`candidate/commit\`.

---

## 3.2. Operational mode vs configuration mode [A]

### 3.2.1. Co naprawdę zmienia znak promptu?

W Junos zwykle zobaczysz:

- \`user@host>\`: **operational** — obserwacja i działania operacyjne.
- \`[edit]\` nad \`user@host#\`: **configuration** — budowanie/zmiana candidate.

Bardzo ważne: na Cisco \`#\` kojarzy Ci się z uprzywilejowanym EXEC. Na Junos jest znakiem promptu konfiguracji. Nie próbuj bezmyślnie kopiować poleceń Cisco z \`#\` do Junos \`#\`.

**Urządzenie: R1-vSRX — Operational → Configuration Mode**

~~~junos
configure
~~~

**Urządzenie: R1-vSRX — Configuration Mode**

~~~junos
show
top
run show version
exit
~~~

Co właśnie zrobiłeś? **\`show\` w configuration mode** pokazuje kontekst konfiguracji, a **\`run show version\`** pozwala wykonać polecenie operacyjne bez opuszczania edytora. \`top\` przenosi do korzenia hierarchii. \`exit\` na głównym poziomie wraca do operational; jeśli w candidate pozostały niezatwierdzone zmiany, system może poprosić o potwierdzenie opuszczenia trybu.

**Most z Cisco:** w \`(config)#\` do komend operacyjnych często używasz \`do show ...\`. W Junos najbliższym funkcjonalnym odpowiednikiem jest \`run show ...\`.

### 3.2.2. Istotny wyjątek: nie wszystkie polecenia operational wolno poprzedzić run

Przykładowo dokumentacja polecenia \`show system rollback\` zaznacza, że **nie działa ono przez \`run\` z configuration mode**. Dlatego zasada „wystarczy dać \`run\` przed dowolnym operational show” byłaby zbyt prosta.

**Urządzenie: R1-vSRX — Operational Mode**

~~~junos
show system rollback 1
show system commit
~~~

**Zapamiętaj:** \`run\` rozwiązuje większość codziennych potrzeb, ale CLI ma ograniczenia poleceń. W razie problemu sprawdź dokumentację konkretnej komendy, zamiast szukać „błędu Junos”.

### 3.2.3. Jak nie pomylić edycji z natychmiastową zmianą sieci

Wpisanie \`set interfaces ...\` w configuration mode zazwyczaj zmienia **candidate**, nie bieżący forwarding. Dopóki nie wykonasz skutecznego commit, Active pozostaje bez zmian. To nie znaczy, że wszystkie polecenia w configuration mode są nieszkodliwe: **\`commit\`, \`load override\`, \`rollback\` i polecenia masowej edycji mogą mieć poważne skutki po zatwierdzeniu**.

Urządzenie i jego wersja mogą także posiadać mechanizmy dynamicznej/ephemeral configuration. Na poziomie tego rozdziału uczymy się **standardowej bazy konfiguracji Junos**, z którą najczęściej pracujesz w CLI.

### 3.2.4. Prosta próba z własną hipotezą

**Urządzenie: R1-vSRX — Configuration Mode, wyłącznie izolowany LAB**

~~~junos
set system login message "LAB CH03 - test candidate"
show | compare
rollback 0
show | compare
~~~

Przewidywanie: po \`set\` diff będzie zawierał dodany login message; po \`rollback 0\` różnica z active zniknie (zakładając brak innych zmian). **Żaden commit nie został wykonany**, więc aktywna konfiguracja nie została zmieniona.

Jeśli nie zniknie — sprawdź, czy edytujesz wspólną konfigurację równocześnie z inną sesją, czy pozostajesz w innym edit level, czy w międzyczasie ktoś wykonał commit.


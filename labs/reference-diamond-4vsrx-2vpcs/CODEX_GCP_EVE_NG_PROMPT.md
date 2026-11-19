# PROMPT DLA CODEXA — zbudowanie referencyjnego diamentu JUNOS w EVE-NG na GCP

Wklej poniższą sekcję jako zadanie w Codex z otwartym workspace repozytorium [nelto79/jncip](https://github.com/nelto79/jncip). **To jest instrukcja dla agenta, nie raport z wykonanego wdrożenia.**

## Zadanie do wykonania

Jesteś seniorem Juniper/EVE-NG i ostrożnym inżynierem automatyzacji. Masz **stworzyć, uruchomić i zweryfikować, jeśli środowisko i uprawnienia na to pozwalają**, nowy izolowany lab szkoleniowy DIAMOND: 4 × Juniper vSRX (R1–R4) + 2 × VPCS (PC-A, PC-B) na istniejącej instancji EVE-NG hostowanej jako Google Compute Engine VM.

### Kontekst i źródło prawdy

Repo: `https://github.com/nelto79/jncip`; branch `main`.

Przed pracą przeczytaj:
1. `AGENTS.md`;
2. `labs/reference-diamond-4vsrx-2vpcs/README.md` — **kanoniczne nazwy urządzeń, 6 połączeń, pełna adresacja i cztery loopbacki /32**; nie zmieniaj ich;
3. `labs/ch02-four-vsrx/README.md` i `labs/ch02-four-vsrx/CODEX_TASK.md`;
4. odpowiedni rozdział 2 z `docs/chapters/`.

Istniejąca maszyna GCP w projekcie szkoleniowym była opisywana jako `eve-ng-vm` w `us-central1-a`; historyczny rozmiar `n2-standard-8` (8 vCPU / 32 GiB). **Traktuj to jako wskazówkę do wyszukania, a nie dowód aktualnego stanu.** Project ID/credentials odczytaj wyłącznie z uwierzytelnionego środowiska operatora (np. aktywnej konfiguracji `gcloud`), **nie zapisuj ich w publicznym repo**. Nie zgaduj publicznego IP i nie wklejaj kluczy do logów/commitów.

Jeżeli nie masz dostępu do GCP, SSH, EVE GUI/API albo obrazu vSRX, przygotuj komplet artefaktów offline, wskaż konkretne blokery i oznacz status tylko jako GENERATED/STATIC VALIDATED. Nie przedstawiaj syntaktycznego XML jako działającego labu.

### Etap A — audyt read-only (obowiązkowy)

1. Sprawdź `pwd`, repo, branch, `git status`. Nie nadpisuj lokalnych zmian. Pracuj na osobnej gałęzi/PR, chyba że użytkownik wyraźnie wymaga bezpośredniego commitu.
2. Zbadaj aktualny stan `gcloud auth list`, `gcloud config list`, `gcloud compute instances describe` wyłącznie odczytowo. Gdy gcloud nie jest autoryzowany, nie próbuj interaktywnego obejścia i nie deklaruj dostępu.
3. Ustal: właściwy GCP projekt, zonę, nazwę/stan VM, wersję systemu EVE, dostępną pamięć/vCPU, wolne miejsce, stan `/dev/kvm`, nested virtualization, obecne obrazy QEMU i nazwy szablonów w EVE; udokumentuj wersję rzeczywistego Junos. Nie zakładaj, że starszy vJunos lub vSRX-ng można zamienić bez testu.
4. Zidentyfikuj istniejące laboratoria oraz autentyczny przykład zgodnego pliku `.unl`; nie kopiuj mechanicznie UUID ani ID istniejących obiektów.
5. Zweryfikuj realne mapowanie danych `ge-0/0/x`, obecność oddzielnego `fxp0` i podpięcia kart EVE; `ge-0/0/0` z tabeli referencyjnej jest **założeniem**, które trzeba potwierdzić.
6. Wykonaj capacity check. 4 vSRX × wstępnie 2 vCPU i 4 GiB RAM = 8 vCPU i 16 GiB RAM + zapas dla EVE/host/QEMU. Historyczna VM n2-standard-8 może być obciążona także innymi labami. **Nie uruchamiaj wszystkich naraz, jeśli preflight pokaże zbyt mały headroom**. Nigdy samodzielnie nie resize'uj VM, dysku ani nie zmieniaj GCP billing/sieci/firewall.

### Etap B — artefakty i projekt

Utwórz nowy namespace labu, np. `JNCIP-REFERENCE-DIAMOND-4vSRX`, tak by nie naruszać starych laboratoriów (zwłaszcza Cisco 300-410). Docelowo zapisz w repo:

- `labs/reference-diamond-4vsrx-2vpcs/topology/JNCIP_REFERENCE_DIAMOND_4vSRX_2VPCS.unl`
- `labs/reference-diamond-4vsrx-2vpcs/configs/R1.set`, analogicznie `R2.set`, `R3.set`, `R4.set`
- `labs/reference-diamond-4vsrx-2vpcs/configs/PC-A.txt` i `PC-B.txt`
- `labs/reference-diamond-4vsrx-2vpcs/VERIFY.md` — statyczne i runtime testy wraz z dowodami
- `labs/reference-diamond-4vsrx-2vpcs/TROUBLESHOOTING.md` — kontrolowana awaria → symptomy → naprawa → cleanup
- krótki `DEPLOYMENT_REPORT.md` z wersją EVE/vSRX, realnym mapowaniem interfejsów, statusami i blockerami. **Bez IP zarządzania, sekretów i identyfikatorów chmurowych w publicznym repo**.

W pliku `.unl` ma być dokładnie sześć urządzeń (R1-vSRX, R2-vSRX, R3-vSRX, R4-vSRX, PC-A, PC-B) oraz dokładnie sześć niezależnych segmentów: PC-A–R1, R1–R2, R1–R3, R2–R4, R3–R4, R4–PC-B. **Nie twórz R2–R3 ani R1–R4.** Wszystkie segmenty mają być wewnętrzne/izolowane: brak bridge do cloud management, GCP VPC, Internetu czy sieci firmowej. Zachowaj fizyczny układ rombu w GUI EVE-NG.

### Etap C — adresacja, loopbacki i konfiguracje

Wykonaj *dokładnie* adresację z dokumentu kanonicznego:

- PC-A `192.0.2.10/24`, gateway `192.0.2.1`; R1 LAN `192.0.2.1/24`;
- R1–R2 `10.12.0.0/30`: R1 `10.12.0.1`, R2 `10.12.0.2`;
- R1–R3 `10.13.0.0/30`: R1 `10.13.0.1`, R3 `10.13.0.2`;
- R2–R4 `10.24.0.0/30`: R2 `10.24.0.1`, R4 `10.24.0.2`;
- R3–R4 `10.34.0.0/30`: R3 `10.34.0.1`, R4 `10.34.0.2`;
- R4 LAN `198.51.100.1/24`; PC-B `198.51.100.10/24`, gateway `198.51.100.1`;
- loopback `lo0.0`: R1 `10.255.255.1/32`, R2 `10.255.255.2/32`, R3 `10.255.255.3/32`, R4 `10.255.255.4/32`. Ustaw odpowiadające `routing-options router-id`.

Przyporządkowanie interfejsów według tabeli z README jest celem, ale **najpierw skontroluj faktyczne NIC w EVE**. Jeśli mapping różni się, dokumentuj przekład mapowania i popraw pliki konfiguracji konkretnego środowiska bez zmiany logicznych adresów.

Zacznij od **baseline statycznego**. Trasa preferowana PC-A ↔ PC-B przez R2; dolna przez R3 jako floating backup na R1/R4; zwykłe trasy statyczne na R2 i R3. Skontroluj wersję/składnię `qualified-next-hop` i wartości route preference. Nie uruchamiaj OSPF/BGP w baseline — to późniejsze zadania.

**vSRX jest stateful**: skonfiguruj dedykowane LAB security zone, minimalne reguły transit (ICMP dla testów) i wymagane host-inbound ping, bez NAT. Nie przełączaj globalnie w packet-mode bez uzasadnienia. Pamiętaj o symetrii ścieżki i session state. W plikach wynikowych nie umieszczaj hasła root, tokenów ani kluczy; jeśli autostart config wymaga credentials, użyj bezpiecznej metody interaktywnej bez commitowania sekretu.

### Etap D — bezpieczny deployment

1. Waliduj składnię XML i strukturę ID/UUID, porównaj z formatem bieżącego EVE-NG.
2. Wdrożenie pliku `.unl` tylko do **nowego** katalogu/labu użytkownika w EVE-NG; nie nadpisuj istniejących plików. Użyj wspieranego GUI/API lub sprawdzonej procedury tej wersji EVE; wykonaj niezbędne permission fixes tylko w zakresie przewidzianym przez EVE.
3. Jeśli capacity check zaliczony, uruchamiaj węzły **etapami**, notuj RAM/CPU, awarie QEMU i boot log. W przeciwnym razie zatrzymaj się na statycznym przygotowaniu; nie modyfikuj rozmiaru VM.
4. Zbierz rzeczywiste `show version`, `show interfaces terse`, `show route` i wersję image. Gdy porty się nie zgadzają, nie wgrywaj w ciemno bazowego config.
5. Dla zmian Junos stosuj `commit check` przed `commit confirmed`, a następnie potwierdź wynik w bezpiecznej sesji (nie uruchamiaj `commit check` w trakcie oczekiwania na `commit confirmed`).

### Etap E — scenariusze akceptacyjne (nie pomijaj)

- A: PC-A ↔ własny gateway; PC-B ↔ własny gateway; ARP i status interfejsów.
- B: R1–R2, R1–R3, R2–R4, R3–R4 sąsiadują i pingują się na adresach łącza.
- C: wszystkie cztery `lo0.0 /32` istnieją lokalnie i Router ID jest poprawny; **nie oczekuj** zdalnej osiągalności loopback bez dodatkowego routingu.
- D: ping PC-A→PC-B i PC-B→PC-A w baseline; traceroute preferuje górną trasę R1–R2–R4 (przy właściwym ICMP host-inbound i flow).
- E: sesje i policies SRX pozwalają wyjaśnić transit vs host-bound; sprawdź counters i ewentualne deny.
- F: **pełna awaria górnego korytarza**: wyłącz R1–R2 i R2–R4, przetestuj przejście przez R3, przywróć i zweryfikuj bazę.
- G: **pojedyncza awaria R1–R2**: sprawdź, czy powstaje niesymetryczna ścieżka lub blackhole. To kontrolowany problem diagnostyczny; **nie zakładaj PASS** end-to-end przy statycznym routingu.
- H: przywróć wszystkie łącza, config i stan testowy; udokumentuj PASS / FAIL / NOT TESTED, skopiuj wyłącznie niesekretne dowody.

### Reguły bezpieczeństwa i raportowania

- Nie dotykaj istniejących labów Cisco, globalnych GCP firewall/VPC, polityk EVE management, komercyjnych obrazów lub konfiguracji produkcyjnych.
- Nie pobieraj obrazów Junos z nieoficjalnych źródeł. Nie eksportuj obrazów QCOW2 do publicznego GitHub.
- Jeśli brakuje uprawnień, poświadczeń, SSH lub obrazu: **nie zgaduj i nie obchodź ograniczeń**; opisz blocker i jednoznaczne polecenie, które użytkownik może wykonać.
- Statusy raportuj ściśle: PLANNED, GENERATED, STATIC VALIDATED, RUNTIME VALIDATED. Każde runtime PASS wymaga faktycznych dowodów.
- Na końcu podaj: pliki, commit/PR, nazwę i ścieżkę labu na EVE (jeśli naprawdę wdrożono), rzeczywiste image/port mapping, resource check, wyniki testów i ograniczenia.
- Wykonuj tyle, ile bezpiecznie umożliwiają narzędzia. Nie deklaruj, że operacja w GCP lub EVE miała miejsce, jeśli jedynie wygenerowano lokalne pliki.

**Nie zmieniaj kanonicznej adresacji i nie dodawaj urządzeń/połączeń „dla wygody”.**

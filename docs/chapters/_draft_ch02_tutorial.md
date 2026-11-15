# Rozdział 2. Junos OS Fundamentals — od pierwszego pakietu do architektury RE/PFE

**Podręcznik:** JUNOS Enterprise Routing & Switching — JNCIA-Junos → JNCIS-ENT → JNCIP-ENT (JN0-650)  
**Wersja redakcyjna:** 2.0 — lekcja prowadzona, 15 listopada 2026  
**Oficjalny zakres w naszym Unified Blueprint:** 2.1–2.6 [A]  
**Dla kogo:** inżynier Cisco IOS/IOS XE, który chce zrozumieć Junos, a nie tylko nauczyć się kolejnej listy komend  
**Platforma praktyczna:** vSRX; uwagi o routerach MX i switchach EX/QFX podane tam, gdzie mają znaczenie  
**Zasada:** najpierw „dlaczego?” i „co się stanie?”, dopiero później CLI, skrócona ściąga i pytania kontrolne.

> **Umowa na ten rozdział:** nie będę Cię zasypywać tabelami z definicjami. Zaczniemy od problemu, pokażemy, co naprawdę robi urządzenie, następnie przewidzisz wynik, sprawdzisz go w Junos i wreszcie sam wywołasz kontrolowaną awarię. Gdy pojawi się termin egzaminacyjny, od razu dowiesz się, czy trzeba go **znać na pamięć**, **rozumieć**, czy **umieć sprawdzić**.

## Zanim zaczniemy — pewien incydent o 02:17

Wyobraź sobie nocną zmianę w NOC. Na ekranie monitoringu wszystko wygląda spokojnie: Twój router w oddziale odpowiada na ping, SSH działa, a w tablicy routingu widzisz prawidłową trasę do centrum danych. W tym samym czasie użytkownicy krzyczą, że aplikacja ERP nie działa.

— „Przecież router żyje!” — mówi kolega.

To prawda. Ale co tak naprawdę udowodnił? Jedynie to, że **jakiś rodzaj ruchu do routera** dotarł i otrzymał odpowiedź. Nie udowodnił, że router potrafi przepuścić **ruch przez siebie**.

Godzinę później spotykasz odwrotną sytuację: klienci pracują normalnie, ale ping do bramy nie odpowiada. Czy router jest zepsuty? Niekoniecznie. Mógł przestać przyjmować ping do własnego adresu, a jednocześnie bezbłędnie przekazywać pakiety klientów.

Te dwie pozornie sprzeczne sytuacje są drzwiami do architektury Junos. Przez cały rozdział będziemy szukać odpowiedzi na trzy pytania:

1. **Kto wymyśla drogę dla pakietu?**
2. **Kto naprawdę przekazuje pakiet?**
3. **Co się dzieje, gdy pakiet chce rozmawiać z samym routerem, zamiast przejść przez router?**

Jeśli po rozdziale będziesz umiał rozdzielić te trzy pytania w czasie awarii, podstawowa architektura Junos przestanie być teorią egzaminacyjną — zacznie działać w Twojej głowie jak mapa diagnostyczna.

### Jak będziemy pracować

W tym rozdziale zachowujemy dokładnie sześć podpunktów blueprintu:

- **2.1.** Architektura oprogramowania Junos OS.
- **2.2.** Rozdzielenie control plane i forwarding plane.
- **2.3.** Routing Engine (RE) — rola i funkcje.
- **2.4.** Packet Forwarding Engine (PFE) — rola i funkcje.
- **2.5.** Przetwarzanie ruchu tranzytowego (transit traffic).
- **2.6.** Exception traffic i relacja do control plane.

Po nich przejdziesz większe laboratorium **4 × vSRX + 2 × VPCS**, przewidziane specjalnie po to, by można było obserwować ścieżkę podstawową, zapasową i skutki różnych usterek. Dodatkowe sekcje LAB, TROUBLESHOOTING i TEST są autorskimi rozszerzeniami dydaktycznymi, a nie osobnymi oficjalnymi objectives.

### Trzy rodzaje wiedzy, których będę od Ciebie wymagać

**Na pamięć:** co oznaczają RE, PFE, RIB i FIB; funkcje rpd oraz mgd; rozróżnienie transit, host-bound i locally originated.

**Musisz zrozumieć:** jak informacja od protokołu routingu trafia do mechanizmu forwarding; dlaczego prawidłowa trasa nie gwarantuje sukcesu aplikacji; dlaczego ICMP nie jest automatycznie „exception traffic”.

**Musisz umieć wykonać:** znaleźć wersję systemu, sprawdzić procesy, interfejsy, trasę, forwarding table i — na SRX — sesję bezpieczeństwa; opisać, na którym etapie pakiet przestał iść dalej.

---

## 2.1. Architektura oprogramowania Junos OS [A]

### 2.1.1. Problem sprzed rozdzielenia ról: nie każ każdemu samochodowi dzwonić do dyspozytora

Wyobraź sobie ogromną firmę kurierską. Z sortowni wyjeżdżają tysiące paczek na minutę. Mamy dwa zadania:

- Ustalić plan: gdzie są miasta, jakie drogi istnieją i która trasa będzie najlepsza.
- Dla każdej paczki błyskawicznie odczytać etykietę i skierować ją na właściwy taśmociąg.

Gdyby sorter musiał za każdym razem zadzwonić do centralnego biura i zapytać „czy paczka do Łodzi ma dziś jechać przez Poznań?”, firma natychmiast by się zatkała. Przy tysiącach paczek na sekundę nawet świetny dyspozytor nie nadąży.

**Sieci mają dokładnie ten sam problem.** Routing dynamiczny, konfiguracja, sąsiedztwa OSPF i BGP to działalność inteligentna, ale stosunkowo kosztowna obliczeniowo. Przekazanie kolejnego pakietu na znaną już trasę powinno być szybkie i przewidywalne.

Stąd logiczny podział Junos:

- **Routing Engine (RE)** — dyspozytornia. Poznaje sieć, przelicza trasy, utrzymuje konfigurację i odpowiada na polecenia operatora.
- **Packet Forwarding Engine (PFE)** — automatyczna sortownia. Patrzy na pakiet i realizuje wcześniej przygotowaną decyzję przekazywania.

To bardzo dobra analogia, ale pamiętaj o jej granicach: na fizycznym MX lub EX/QFX wiele operacji danych wykonują wyspecjalizowane układy, a na vSRX funkcje forwarding i kontroli sesji realizuje programowe środowisko wirtualne. Nie zawsze są to dwa oddzielne fizyczne procesory.

**Zatrzymaj się na dziesięć sekund.** Jeśli do routera wpadnie milion podobnych pakietów w ciągu sekundy, czy milion razy trzeba uruchomić OSPF, żeby znaleźć drogę? Nie. OSPF pomaga przygotować tablice, a forwarding korzysta z przygotowanego wyniku.

### 2.1.2. Podróż informacji: od BGP do kabla

Załóżmy, że R1 otrzymał od sąsiada BGP informację: sieć 10.50.0.0/16 jest osiągalna przez 10.12.0.2.

~~~text
  R2 informuje R1: "znam drogę do 10.50.0.0/16"
                 |
                 v
         rpd analizuje BGP
                 |
                 v
         RIB — tablica tras
          (np. inet.0)
                 |
                 v
       wybór aktywnej ścieżki
                 |
                 v
    master forwarding table na RE
                 |
                 v
     instalacja/synchronizacja
      tablicy forwarding do PFE
                 |
                 v
          PAKIET PRZYCHODZI
   destination IP = 10.50.8.17
                 |
                 v
       PFE wybiera next hop
        i interfejs wyjściowy
                 |
                 v
       Pakiet wychodzi z R1
~~~

Kiedy później na tym samym routerze zrobisz **show route**, sprawdzasz perspektywę routingu. Kiedy wykonasz **show route forwarding-table**, przyglądasz się master forwarding table po stronie RE. To nie jest automatycznie bezpośredni zrzut sprzętowej tablicy z PFE.

I właśnie tu rodzi się pierwszy egzaminacyjny haczyk: **informacja o istnieniu trasy to nie to samo co dowód, że pakiet został poprawnie wysłany**.

### 2.1.3. Junos to nie jeden ogromny proces

W Cisco spotkałeś różne procesy i podsystemy IOS XE. Junos również składa się z wyspecjalizowanych usług, często określanych jako **daemons**. Nie musisz znać wszystkich. Na początku poznaj cztery postacie naszej historii.

**rpd (routing protocol daemon)** jest analitykiem transportu. Utrzymuje informacje protokołów takich jak OSPF, BGP czy IS-IS i uczestniczy w budowaniu tablic routingu. Kiedy tracisz sąsiedztwo BGP, rpd należy do pierwszych elementów, których stan chcesz rozumieć.

**mgd (management daemon)** jest kierownikiem przyjmującym zmiany. Kiedy wpisujesz polecenia konfiguracyjne i wykonujesz commit, zarządzanie konfiguracją wiąże się z mgd. Jeśli występuje problem z zatwierdzaniem zmian, nie oznacza to automatycznie, że forwarding przestał działać.

**chassisd** dogląda platformy. Zależnie od modelu zna stan kart, modułów, alarmów i komponentów chassis.

**Komponenty obsługi interfejsów, np. dcd**, dbają o odpowiednie aspekty konfiguracji i stanu interfejsów. Dokładny zestaw procesów oraz ich nazwy mogą zależeć od rodziny Junos i wydania.

A gdzie w tym wszystkim kernel? To fundament systemu operacyjnego, zapewniający procesom i mechanizmom systemowym wspólne usługi. Nie jest „piątym protokołem routingu”. W dużym uproszczeniu procesy i kernel współpracują w aktualizacji stanu potrzebnego do forwarding.

### 2.1.4. Co się stanie, jeśli jeden z tych procesów będzie mieć problem?

Wyobraź sobie, że na chwilę zabrakło analityka tras. Sortownia wciąż ma wydrukowaną ostatnią mapę dróg, więc **część ruchu może przez pewien czas działać**. Jednak gdy zamkną autostradę, nie ma komu przeliczyć poprawnej drogi.

To tłumaczy, dlaczego:
- problem z rpd może uderzyć w sąsiedztwa lub aktualizację tras, choć część istniejących ścieżek nadal przekazuje pakiety;
- problem z mgd może przeszkodzić operatorowi w konfiguracji, lecz nie dowodzi natychmiastowej utraty forwarding;
- awaria krytycznego komponentu lub synchronizacji może jednak dotknąć obu płaszczyzn.

Nie wyciągaj z analogii zbyt daleko idącej gwarancji. **RE/PFE separation nie zastępuje mechanizmów HA** — GRES, NSR czy redundantne RE mają osobne wymagania.

### 2.1.5. Sprawdźmy, z czym naprawdę rozmawiamy

Nie zakładamy, że wirtualny SRX zachowuje się identycznie jak MX czy EX. Najpierw identyfikujemy platformę i wersję.

**Urządzenie: R1-vSRX — tryb operational**

~~~junos
show version
show system uptime
show chassis hardware
show system alarms
show chassis alarms
~~~

Co chcesz zobaczyć? Model i wersję, czas od ostatniego startu, wykryte elementy platformy i aktualne alarmy.

Potem zajrzyjmy „na zaplecze” i sprawdźmy procesy.

**Urządzenie: R1-vSRX — tryb operational**

~~~junos
show system processes extensive
show system processes extensive | match "rpd|mgd|chassisd|dcd"
~~~

Pierwsza komenda pokazuje pełny kontekst. Druga jest tylko ułatwiającym filtrem tekstowym. **Brak danej nazwy w przefiltrowanym wyjściu nie dowodzi automatycznie awarii**. Proces może nie występować w danej postaci, nie mieścić się w wyświetlanym fragmencie albo być inaczej opisany.

**Przewidź wynik, zanim sprawdzisz:** gdy SSH działa, ale operator nie może zatwierdzić żadnej zmiany, który obszar obejrzysz najpierw? Management/configuration — mgd, logi, stan config, zasoby. Nie zaczynasz od wyłączania interfejsów.

### 2.1.6. Klasyczny Junos, Junos OS Evolved i vSRX

Junos OS to wspólna rodzina systemowa, ale **nie wszystkie urządzenia mają identyczne wnętrze**. Klasyczny Junos wywodzi się z FreeBSD. Junos OS Evolved używa jądra Linux i ma odmienną architekturę wdrożeniową. vSRX to wirtualna platforma security, na której dodatkowo istotna jest obsługa **flow-based**.

Dla poziomu Associate zapamiętaj logikę RE/PFE. Dla poziomów Specialist i Professional dodaj nawyk pytania: „**na jakiej platformie i wersji działa ten przykład?**”.

Jeśli kiedyś zobaczysz komendę działającą na MX, a niedostępną na vSRX, nie zakładaj błędu ucznia. Możesz po prostu mierzyć inną implementację.

### Mała powtórka po 2.1

Wyjaśnij własnymi słowami: dlaczego zwykły pakiet tranzytowy nie powinien pytać procesu rpd o trasę? Jeśli odpowiadasz „bo rpd przygotował informacje wcześniej, a forwarding korzysta z gotowej tablicy”, możemy iść dalej.

---


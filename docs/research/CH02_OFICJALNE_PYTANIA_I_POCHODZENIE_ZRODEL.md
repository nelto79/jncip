# Notatki badawcze — pytania źródłowe do Rozdziału 2 (Junos OS Fundamentals)

**Data kwerendy:** 2026-11-16  
**Repozytorium:** nelto79/jncip  
**Materiał docelowy:** docs/chapters/JUNOS_Rozdzial_02_Architektura_Junos_OS.md  
**Cel:** zachować pochodzenie inspiracji dydaktycznych. Ten plik jest NOTATKĄ DLA AUTORA I CODEXA, **nie osobnym bankiem pytań i odpowiedzi dla ucznia**. Wyjaśnienia odpowiedzi umieszczono organicznie we właściwych punktach wykładu.

## Zasady weryfikacji i prawa autorskie

- **Oficjalny JNCIA Practice Exam:** https://learningportal.juniper.net/juniper/user_activity_info.aspx?id=EXAM-JNCIA-JUNOS — strona publiczna potwierdza darmowy test, wynik, próg ćwiczeniowy 70%, nielimitowane próby i wyjaśnienia. Samych pytań **nie dało się zobaczyć bez uruchomienia interaktywnego assessment**. Żadne z niżej wykorzystanych pytań **nie jest prezentowane jako skopiowane z tego testu**. Jeśli użytkownik wykona assessment i udostępni własne notatki ze sprawdzianu, można analizować je osobno.
- **Juniper Community / Elevate:** zawiera pytania prawdziwych uczestników webcastów lub dyskusji technicznych, ale **nie jest równoznaczne z oficjalnym bankiem pytań egzaminacyjnych**. Pytania parafrazowano w języku polskim, a samodzielne wyjaśnienia oparto także na dokumentacji technicznej.
- **Juniper Day One:** zawiera wykłady, ilustracje i pytania prowokujące do myślenia; korzystamy z mechanizmu i scenariuszy, **bez kopiowania większych fragmentów książki**.
- **Juniper KB i CLI Reference:** źródła poparcia dla konkretnych mechanizmów; nie nazywaj własnego pytania autentycznym pytaniem testowym, jeżeli dokumentacja sama nie zadaje go w tej formie.
- Nie korzystaliśmy z witryn oferujących *exam dumps* ani nie wklejaliśmy z nich pytań.

## Rejestr pochodzenia, nie lista pytań do wkuwania

| Kod | Źródło | Typ źródła / status | Jaki problem badamy | Wpleciony fragment |
|---|---|---|---|---|
| SRC-01 | [Juniper JNCIA webcast Q&A](https://community.juniper.net/discussion/do-you-have-questions-about-the-jncia-junos-exam), post 2 lutego 2022 | Prawdziwe pytanie z oficjalnego webcastu, udostępnione w społeczności Junipera | RE a PFE, skąd biorą się tablice | 2.1 |
| SRC-02 | [Juniper Elevate: Routing Engine and Forwarding Engine](https://community.juniper.net/discussion/routing-engine-and-forwarding-engine) | Prawdziwe pytania inżyniera na forum, NIE bank egzaminacyjny | Dlaczego 2 płaszczyzny, brak forwarding entry | 2.2 |
| SRC-03 | [Juniper Support FAQ on EX Virtual Chassis GRES](https://supportportal.juniper.net/s/article/EOE-EOL-GRES-FAQ-for-EX-Virtual-Chassis) + [GRES current documentation](https://www.juniper.net/documentation/us/en/software/junos/high-availability/topics/topic-map/gres-understanding.html) | FAQ producenta / techniczna weryfikacja | Czy backup RE automatycznie zapewnia HA | 2.3, rozszerzenie pod 17 |
| SRC-04 | [Juniper Community: Hold entry in show route forwarding-table](https://community.juniper.net/discussion/hold-entry-in-show-route-forwarding-table) + [CLI Reference](https://www.juniper.net/documentation/us/en/software/junos/cli-reference/topics/ref/command/show-route-forwarding-table.html) | Prawdziwe pytanie operatora i dokumentacja | Nierozwiązany ARP/next-hop typu hold | 2.4 |
| SRC-05 | [Day One: Beginner's Guide to Learning Junos](https://www.juniper.net/documentation/en_US/day-one-books/junos-beginners-guide.pdf), rozdział o routing policy | Mechanizm z książki, pytanie kontrolne zredagowane przez autora | Wiele kandydatów w routingu vs używane next-hopy | 2.4 |
| SRC-06 | [Day One: Beginner's Guide to Learning Junos](https://www.juniper.net/documentation/en_US/day-one-books/junos-beginners-guide.pdf), Figure 1.12 | Opublikowany przez Juniper scenariusz + autorskie pytanie na podstawie ryciny | Ten sam SSH może być transit na jednym hopie i exception na innym | 2.5 |
| SRC-07 | [Juniper Community: Exception Traffic example](https://community.juniper.net/discussion/exception-traffic-example) | Prawdziwe pytania uczestników forum | SCP/SFTP do urządzenia vs przez urządzenie | 2.6 |
| SRC-08 | [Juniper Community: Routing Engine and Forwarding Engine](https://community.juniper.net/discussion/routing-engine-and-forwarding-engine) + [Juniper IJOS training PDF](https://www.juniper.net/content/dam/www/assets/eguides/jp/ja/introduction-to-the-junos-operating-system.pdf) | Forum i starszy oficjalny materiał szkoleniowy | TTL expired: kto wykonuje specjalne przetwarzanie | 2.6 |
| SRC-09 | [Juniper Community: MX104 capture question](https://community.juniper.net/discussion/is-it-possible-do-packet-capture-on-mx104-without-use-analyzer) + [monitor traffic documentation](https://www.juniper.net/documentation/us/en/software/junos/cli-reference/topics/ref/command/monitor-traffic.html) | Prawdziwe pytanie techniczne + CLI Reference | Dlaczego monitor traffic nie widzi całego ruchu transit | 2.6 |
| SRC-10 | [Juniper JNCIA webcast Q&A](https://community.juniper.net/discussion/do-you-have-questions-about-the-jncia-junos-exam), post 2 lutego 2022 | Prawdziwe pytanie uczestnika szkolenia | Czy ćwiczyć vSRX / vLabs | 2.LAB |

## Warunki i ograniczenia wnioskowania

1. Posty społecznościowe, nawet opublikowane na stronie producenta, mogą zawierać uproszczenia lub nieścisłości. Kiedy forum sugeruje, że PFE jest „częścią RE”, podręcznik jawnie koryguje tę analogię na podstawie architektury producenta.
2. Uogólnienia związane z punt do RE, ICMP TTL expiry oraz obsługą PFE różnią się między platformami. Nie traktować jednego scenariusza z MX/EX jako gwarancji zachowania konkretnego vSRX.
3. Starsze Day One i materiały IJOS mogą opisywać wersje starsze niż współczesny lab — przy konfiguracjach weryfikować zachowanie na konkretnym obrazie.
4. Linki do dawnych wątków community.juniper.net mogą przekierowywać do nowej społeczności HPE. Podczas kwerendy wyszukiwarka indeksowała pierwotną treść; w przypadku przeniesienia nie deklarować, że źródło jest dostępne w oryginalnej postaci.
5. Test JNCIA-Junos jest **assessmentem do samodzielnego uruchomienia**. Nie kopiować jego pytań z nieautoryzowanych witryn.

## Dalsze materiały

- [JNCIA-Junos Certification Overview — official blueprint](https://learningportal.juniper.net/juniper/user_activity_info.aspx?id=14354).
- [JNCIA-Junos Certification Practice Exam](https://learningportal.juniper.net/juniper/user_activity_info.aspx?id=EXAM-JNCIA-JUNOS).
- [Day One Beginner's Guide](https://www.juniper.net/documentation/en_US/day-one-books/junos-beginners-guide.pdf).

**Wniosek redakcyjny:** materiał egzaminacyjny wnosi wartość tylko wtedy, gdy zmienia sposób **myślenia o pakiecie i awarii**. Dlatego nie wklejać powyższej tabeli do rozdziału 2 i nie tworzyć w nim nowej sekcji z pytaniami źródłowymi. Połączenia źródło ↔ rozdział są dokumentowane tu na potrzeby weryfikowalności.

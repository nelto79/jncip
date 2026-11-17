# Źródła ilustracji i zasady wizualne — podręcznik JNCIP

## Dlaczego rysujemy?

Grafika jest elementem lekcji, a nie ozdobą. Każda ilustracja w podręczniku ma odpowiedzieć na konkretne pytanie: **którędy idzie pakiet, co podejmuje decyzję, co widać w CLI i co się stanie przy awarii?** Diagram powinien być poprzedzony kontekstem, a po nim powinno być pytanie kontrolne. Unikamy dekoracyjnych zdjęć urządzeń bez wartości szkoleniowej.

## Zewnętrzne grafiki z internetowych repozytoriów, użyte w rozdziale 2

1. **Network Routing Illustration Diagram.jpg**, autor: **Jan Bellon**, bez zmian. Źródło: https://commons.wikimedia.org/wiki/File:Network_Routing_Illustration_Diagram.jpg . Link do osadzonego pliku: https://upload.wikimedia.org/wikipedia/commons/1/1f/Network_Routing_Illustration_Diagram.jpg . Licencja: **Creative Commons Attribution-ShareAlike 4.0 International (CC BY-SA 4.0)**, https://creativecommons.org/licenses/by-sa/4.0/ . Ilustracja ogólnego routingu i alternatywnych ścieżek; nie jest schematem dokładnego labu Juniper.
2. **ARP und Routing.png**, autor oryginału: **Hubi**, bez zmian. Źródło: https://commons.wikimedia.org/wiki/File:ARP_und_Routing.png . Link do osadzonego pliku: https://upload.wikimedia.org/wikipedia/commons/f/fd/ARP_und_Routing.png . Licencja: **Creative Commons Attribution-ShareAlike 3.0 Unported (CC BY-SA 3.0)**, https://creativecommons.org/licenses/by-sa/3.0/ . Diagram ogólnej kolejności decyzji routing i ARP, z oryginalnymi niemieckimi opisami.

**Sposób osadzenia:** obrazy są osadzone w Markdown bezpośrednimi URL do infrastruktury Wikimedia, więc GitHub może je wyświetlić bez lokalnego przechowywania plików binarnych. Nie jest to kopia pliku w repozytorium. Przy offline render potrzebny byłby osobny proces pobrania i dodania obrazów zgodnie z warunkami licencji.

**Licencje materiałów graficznych nie oznaczają automatycznie objęcia tekstu całego podręcznika licencją konkretnej grafiki.** Nie przerabiamy tych obrazów i podajemy credit przy każdym z nich.

## Fotografia źródłowa wykorzystana w rozdziale 4

- **Juniper Networks QFX5100 ethernet switch.jpg** — autor: **Juniper Networks**; źródło i atrybucja: https://commons.wikimedia.org/wiki/File:Juniper_Networks_QFX5100_ethernet_switch.jpg ; licencja **CC BY-SA 3.0**: https://creativecommons.org/licenses/by-sa/3.0/ ; bez modyfikacji. Bezpośredni URL osadzenia: https://upload.wikimedia.org/wikipedia/commons/9/99/Juniper_Networks_QFX5100_ethernet_switch.jpg . Zdjęcie historycznego przełącznika pokazuje fizyczne porty i uplinki w punkcie 4.6.2. **Nie przedstawia sprzętu używanego w labie ani współczesnej rekomendacji zakupowej**. Hotlink Wikimedia, bez kopiowania pliku do repo.

## Fotografia źródłowa rozdziału 5 — Tx/Rx modułu SFP

- **SFP-front-RX-TX.jpg** — fotografia bazowa **Adamantios**, oznaczenia Tx/Rx dodane przez **MichaelFrey** (w oryginalnym pliku źródłowym), [Wikimedia Commons](https://commons.wikimedia.org/wiki/File:SFP-front-RX-TX.jpg), licencja **CC BY-SA 3.0 Unported**: https://creativecommons.org/licenses/by-sa/3.0/ ; obraz hotlinkowany bez dalszych modyfikacji: https://upload.wikimedia.org/wikipedia/commons/a/a3/SFP-front-RX-TX.jpg . Wykorzystanie w sekcji 5.3.5a: związanie pomiaru DOM Rx/Tx z fizycznym kierunkiem światłowodu. **Nie jest to model Juniper ani fotografia wkładki z konkretnego labu.** Pliku JPEG nie pobierano do repo.

## Diagramy stworzone w repozytorium

W rozdziale 2 znajdują się diagramy **Mermaid**, renderowane jako grafika na GitHubie:

- Cisco IOS XE kontra Junos: RE/RP, RIB, CEF FIB, PFE;
- Mac rewrite na kolejnych hopach;
- rozdzielenie host-bound i transit;
- topologia diamond 4×vSRX + 2×VPCS.

Ich źródło jest bezpośrednio w pliku `docs/chapters/JUNOS_Rozdzial_02_Architektura_Junos_OS.md`, dzięki czemu Codex może je aktualizować razem z topologią i adresacją.

## Zasady dla kolejnych rozdziałów

1. Najpierw cel nauczania, potem diagram, na końcu pytanie „co wynika z rysunku?”.
2. Tam, gdzie pomocne, porównuj na jednej ilustracji odpowiednie procesy w Cisco i Junos — bez sugerowania, że są implementacyjnie identyczne.
3. Używaj wyłącznie grafik na jasnej licencji do ponownego wykorzystania albo własnych diagramów. Nie kopiuj przypadkowych obrazów z wyszukiwarki.
4. Dla zdjęć i grafik online: autor, tytuł, link do oryginału, licencja, wskazanie modyfikacji, jeśli istnieją.
5. Nie przedstawiaj grafiki ogólnego routing jako rzeczywistego PFE lub topologii Juniper.
6. Jeśli zmienią się węzły, porty lub adresy w .unl, zaktualizuj też diagram laboratorium.
7. Nie buduj całego wykładu z obrazów: tekst musi tłumaczyć powód, mechanizm i konsekwencje.

Stan: 2026-11-17 (uzupełniono o rozdział 5).

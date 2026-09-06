# GreenEvo / GREENSTRAT — notatka do planowania architektury Django

Data sporządzenia: 2026-09-03
Źródła: `Specyfikacja wymagań systemu.pdf` (68 str.), `Matryce oceny ekoinnowacji - 08.07.2026` (docx, dr Beata Paliwoda), `(graf Andrzeja) wymagania_wnioski_app_ext.svg` (diagram BPMN TO-BE), `Przydatne linki.docx` (link do aktualnej wersji specyfikacji na OneDrive — nie pobrany, może zawierać nowszą wersję niż PDF).

---

## 1. Kontekst projektu

- Projekt **GREENSTRAT**: opracowanie i przetestowanie narzędzi wspierających administrację publiczną w ocenie, wdrażaniu i monitorowaniu ekoinnowacji.
- Jeden z rezultatów: **narzędzie cyfrowe wspierające ocenę wniosków** w programie **GreenEvo** (Akcelerator Zielonych Technologii, Ministerstwo Klimatu i Środowiska — MKiŚ), potencjalnie do rozszerzenia na inne instrumenty wsparcia.
- System ma być **funkcjonalnym prototypem (MVP) na poziomie TRL 7** — nie pełnym wdrożeniem produkcyjnym (chyba że Zamawiający odrębnie to potwierdzi).
- Cel narzędzia: **nie zastąpienie człowieka w decyzji**, tylko uporządkowanie procesu, wsparcie analizy, wskazywanie niespójności wymagających weryfikacji eksperckiej.
- Harmonogram: **Zadanie 7** (prototypowanie/testowanie MVP) VIII–X 2026 (3 miesiące), **Zadanie 8** (testy przedwdrożeniowe w warunkach zbliżonych do administracji) XI 2026–I 2027.
- Realizacja ma być **iteracyjna/zwinna** (iteracje tygodniowe/dwutygodniowe, przeglądy, prezentacje przyrostów). Po stronie Zamawiającego kluczowa rola: **Product Manager**.
- Ja (Wojciech) jestem w roli **Django backend developera** — mam zaproponować architekturę backendu (i prawdopodobnie całościową, skoro dokument nie wskazuje osobno frontendu).

---

## 2. Aktorzy / role systemowe

| Rola | Ma konto w systemie? | Zakres |
|---|---|---|
| **Wnioskodawca** | Tak (samodzielna rejestracja, gdy nabór aktywny i publiczny) | Wypełnia i składa wniosek online, widzi status, może kontynuować wersję roboczą |
| **Administrator** | Tak (nadawany przez innego admina; pierwsze konto zakładane na starcie) | Zarządza naborem, użytkownikami, przypisaniami do ekspertów, statusami, generuje podsumowania/zestawienia dla Kapituły, wprowadza finalne decyzje |
| **Ekspert branżowy** | Tak (nadawany przez admina) | Widzi tylko przypisane wnioski, weryfikuje/koryguje ocenę systemu, wprowadza raport z wizytacji, zatwierdza wniosek do dalszego etapu |
| **Kapituła Konkursu** | **NIE ma kont w systemie** (świadoma decyzja) | Otrzymuje wygenerowane podsumowania/zestawienia (eksport, np. Excel) poza systemem; decyzja Kapituły może zapadać poza systemem, Admin wprowadza wynik ręcznie |
| **Kierownictwo MKiŚ** | Poza systemem / rola biznesowa Admina | Zatwierdza i ogłasza wyniki — w TO-BE zakłada się, że Administrator = osoba z Kierownictwa MKiŚ |
| **System / moduł AI** | — | Automatyczna ocena formalna, wsparcie oceny merytorycznej, wskazywanie niespójności, generowanie podsumowań |

Ważne: **role uprzywilejowane (Admin, Ekspert) NIE mogą być nadawane przez samodzielną rejestrację** — tylko przez Administratora. To ma wpływ na projekt modelu User/Roles w Django (custom User + role/grupy, a nie np. allauth self-signup dla wszystkich ról).

---

## 3. Model oceny — matryce A/B/C (merytoryczne serce systemu)

Dokument "Matryce oceny ekoinnowacji" dr Beaty Paliwody to **specyfikacja treści formularza wniosku i logiki scoringu** — kluczowy input do modelu danych.

### 3.1 Struktura trzypoziomowa

1. **Matryca A — potencjał rozwiązania/technologii** (8 kryteriów: A1–A8)
2. **Matryca B — potencjał organizacji/ekoinnowatora** (7 kryteriów: B1–B7)
3. **Matryca C — potencjał wdrożeniowy/rynkowy/skalowalności** (9 kryteriów: C1–C9)

Łącznie **24 kryteria oceny**. Każde kryterium to:
- pytanie oceny,
- **5 poziomów odpowiedzi (1–5)**, każdy z opisem,
- instrukcja dot. wymaganych dowodów/informacji,
- pole opisowe (uzasadnienie + dowody) — limit znaków (FUN-025: **3000 znaków dla wniosku**, więcej dla raportu z wizytacji).

### 3.2 Scoring

- Każde kryterium ma **maksymalną liczbę punktów** (wagę), różną per kryterium (np. A1 = 300 pkt, A5 = 100 pkt, B4 = 100 pkt, C1 = 200 pkt...).
- Przelicznik oceny 1–5 → mnożnik: 1→0.05, 2→0.25, 3→0.50, 4→0.75, 5→1.00.
- **Punkty = max_punktów_kryterium × przelicznik(ocena)**.
- MAX: Matryca A = 1600, B = 1000, C = 1400.
- **Udział matryc w ocenie ogólnej: A=40%, B=25%, C=35%** (wskazane jako wartość **przykładowa/wstępna** — patrz niżej, wagi mają być ustalone metodą ekspercką).

### 3.3 Wagi są tymczasowe i będą aktualizowane metodą ABCD Suzuki

- Wagi (zarówno kryteriów, jak i udział matryc A/B/C) mają zostać ustalone/zwalidowane w **badaniu eksperckim metodą ABCD Suzuki** (ranking kryteriów przez ekspertów, agregacja rang → wagi).
- **Konsekwencja architektoniczna: model oceny (kryteria, pytania, odpowiedzi, wagi, reguły) MUSI być danymi konfigurowalnymi, a nie zahardkodowaną logiką.** Wagi się zmienią po badaniu eksperckim, a system ma dodatkowo (poza zakresem MVP, ale architektura ma to umożliwiać w przyszłości) obsługiwać różne modele oceny, różne nabory/programy.
- W zakresie MVP: **zaawansowany edytor modelu oceny dla Administratora jest POZA zakresem** — model oceny "zostanie dostarczony Wykonawcy" i ma być **odwzorowany** (czyli raczej zaimportowany/zaszyty jako dane konfiguracyjne/fixture niż edytowalny przez UI). Architektura powinna mimo to trzymać go jako dane (tabele: Kryterium, PoziomOdpowiedzi, Waga, RegułaFormalna, RegułaDyskwalifikująca), żeby aktualizacja po badaniu ABCD Suzuki nie wymagała zmian w kodzie.

### 3.4 Pytania dyskwalifikujące vs rankingujące

- Osobna kategoria: **pytania dyskwalifikujące (0/1)** w sekcji "Informacje o Wnioskodawcy" i "Informacje o Ekoinnowacji" (np. czy podmiot ma prawo sprzedaży technologii, czy wdrożona u ≥1 klienta w pełnej skali, czy nie jest w likwidacji, czy nie zalega w US/ZUS, czy kwalifikuje się do pomocy de minimis).
- W matrycach też są kryteria, gdzie **niska ocena = potencjalna dyskwalifikacja** (np. A5=1 "prototyp/demonstrator" jest traktowane jak nie spełniające wymogu formalnego — FUN-046: "A5 jest dyskwalifikujące tak jak formalne").
- **Reguły formalne** (0/1, blokujące) i **reguły dyskwalifikujące na poziomie merytorycznym** to dwie osobne kategorie logiki w modelu oceny.

### 3.5 Sekcje formularza wniosku (niezależne od matryc)

1. Informacje o Wnioskodawcy (forma prawna, NIP, KRS/CEIDG/REGON, dane kontaktowe, obroty, % eksportu, liczba pracowników, pytania dyskwalifikujące).
2. Informacje o Ekoinnowacji (nazwa PL/EN, opis PL/EN, dominujący obszar środowiskowy — 1 z listy ESRS E1–E5 + obszar przekrojowy, obszary szczegółowe — wielokrotny wybór, bardzo rozbudowana lista ~80 checkboxów pogrupowanych wg ESRS, SDGs — wielokrotny wybór z 17, pytania dyskwalifikujące, % obrotów ze sprzedaży technologii, rok opracowania/wdrożenia, nagrody, info o dofinansowaniu).
3. Matryca A (8 kryteriów).
4. Matryca B (7 kryteriów).
5. Matryca C (9 kryteriów).

Uwaga architektoniczna: pola "Dominujący obszar środowiskowy", "Obszary szczegółowe", "SDGs" są **auto-uzupełniane** w kryterium A2 z sekcji "Informacje o Ekoinnowacji" — czyli formularz ma zależności międzysekcyjne (nie jest to zbiór niezależnych stron).

---

## 4. Proces AS-IS vs TO-BE

### AS-IS (obecny, w większości manualny)
Złożenie wniosku (papier/mail, kolejność wpływu, limit) → ocena formalna (Sekretariat, 0/1, uzupełnienia w 5 dni) → kwalifikacja do oceny merytorycznej → ocena merytoryczna I stopnia (Zespół ekspertów) → wizytacja (Ekspert branżowy) → ocena merytoryczna II stopnia → lista rekomendowanych technologii → decyzja Kapituły Konkursu → zatwierdzenie i ogłoszenie przez Kierownictwo MKiŚ → informacja do wnioskodawców + możliwość odwołania.

Kluczowe ograniczenie: **brak naboru ciągłego** (wymaga angażowania zewnętrznych ekspertów w konkretnych oknach czasowych).

### TO-BE (docelowy, wspierany przez system)
Nabór **ciągły** (z możliwością administracyjnego wstrzymania) → rejestracja/logowanie Wnioskodawcy → wypełnienie cyfrowego formularza (autozapis) → złożenie + **podpis elektroniczny** → **automatyczna ocena formalna** (reguły z modelu oceny) → **automatyczna ocena merytoryczna wspierana przez system/AI** (analiza uzasadnień vs wybrana odpowiedź, wskazanie niespójności, potencjalnych zawyżeń, kryteriów dyskwalifikujących) → przypisanie do Eksperta branżowego (przez Admina) → **Ekspert weryfikuje/zatwierdza/koryguje** ocenę systemu (z uzasadnieniem korekty) → wizytacja → **raport z wizytacji** wprowadzony przez Eksperta → **automatyczne wygenerowanie podsumowania dla Kapituły** (trigger: wprowadzenie raportu) → Admin generuje **zbiorcze zestawienie** dla wybranej grupy wniosków → decyzja Kapituły (**poza systemem**) → Admin wprowadza finalny status/wynik → **automatyczny e-mail do Wnioskodawcy**.

System **zastępuje pracę Zespołu ekspertów** w I etapie oceny merytorycznej, ale **NIE zastępuje** Eksperta branżowego ani Kapituły — ich decyzja jest ostateczna.

### 4.1 Statusy wniosku (workflow — kluczowe dla projektu maszyny stanów)

`Wersja robocza` → `Złożony` → *(ocena formalna — status pośredni skreślony w dokumencie, tzn. system NIE dopuszcza złożenia wniosku niespełniającego wymogów formalnych, więc nie ma osobnego stanu "w ocenie formalnej")* → `Do uzupełnienia` (pętla powrotna, jeśli braki) → `W automatycznej ocenie merytorycznej` → `Do przypisania eksperta` → `W weryfikacji eksperta` → `Do wizytacji` → `Po wizytacji` → `Gotowy dla Kapituły` → *(decyzja poza systemem)* → `Rekomendowany` / `Nierekomendowany` / `Odrzucony / zdyskwalifikowany`.

Uwaga: kilka FUN-ów oznaczonych jako **przekreślone/duplikaty w dokumencie** (FUN-037, FUN-062, FUN-065, FUN-068, FUN-087, FUN-091, FUN-094, FUN-106, FUN-112, FUN-115) — to ważne, bo numeracja rejestru wymagań ma luki i wersja z linku w "Przydatne linki.docx" może być zaktualizowana/doczyszczona. **Trzeba zweryfikować najnowszą wersję specyfikacji przed finalizacją architektury** (link OneDrive w `Przydatne linki.docx`, niedostępny offline).

System musi przechowywać **historię zmian statusu** + kto/co (użytkownik czy mechanizm automatyczny) dokonało zmiany — pełna audytowalność.

---

## 5. Zakres MVP — co jest w środku, co poza

### W zakresie (must-have wg specyfikacji):
- Aplikacja webowa (przeglądarka), nabór ciągły z wstrzymaniem,
- rejestracja/logowanie/reset hasła Wnioskodawcy, logowanie Admina i Eksperta (konta nadawane ręcznie),
- cyfrowy formularz wniosku wg modelu oceny, autozapis (co przejście zakładki / wyjście / co 15 min), limit 3 rozpoczęte+złożone wnioski na Wnioskodawcę,
- walidacja kompletności, podpis elektroniczny (wariant techniczny **do potwierdzenia z Ministerstwem**),
- automatyczna ocena formalna (reguły 0/1),
- automatyczna ocena merytoryczna wspierana przez system + moduł AI (analiza uzasadnień, wskazywanie niespójności, wsparcie identyfikacji kryteriów dyskwalifikujących),
- panel Eksperta: przegląd oceny systemu, zatwierdzenie/korekta z uzasadnieniem, raport z wizytacji,
- panel Administratora: nabór, użytkownicy, przypisania, statusy, podsumowania, zbiorcze zestawienie, finalne decyzje, eksport, historia działań,
- powiadomienia e-mail (4 zdarzenia minimum: rejestracja, złożenie wniosku, przypisanie eksperta, wynik końcowy),
- eksport podsumowań pojedynczych i zbiorczych (**format: Excel** — priorytet; PDF do doprecyzowania),
- pełna historia/log działań i statusów (audytowalność),
- środowisko testowe/pilotażowe + dokumentacja (techniczna, administracyjna/serwisowa, instrukcja użytkownika).

### Jawnie POZA zakresem MVP:
1. Pełne produkcyjne wdrożenie.
2. Pełny moduł pracy Kapituły w systemie (konta, poprawki, komentarze, głosowanie, protokół) — Kapituła działa **poza systemem**.
3. Integracja z zewnętrznymi rejestrami/systemami Ministerstwa (np. brak automatycznego pobierania danych z KRS/CEIDG mimo że formularz sugeruje "autouzupełnianie" nazwy/adresu z NIP — **to jest niespójność do wyjaśnienia**, patrz sekcja 9).
4. Publiczne API systemu.
5. Zaawansowany edytor modelu oceny dla Administratora (model oceny dostarczony przez Wykonawcę/Zamawiającego, "odwzorowany" w systemie).
6. Zaawansowane moduły analityczne poza: podsumowania, statusy, scoring, podstawowe zestawienia.
7. Pełna obsługa procesu odwołań (AS-IS miał odwołania; TO-BE tego nie precyzuje jako funkcji systemu).

---

## 6. Moduł AI — kluczowy i wrażliwy element

### 6.1 Dwa zastosowania
1. **Wsparcie ewaluacji wniosku**: analiza odpowiedzi + uzasadnień opisowych Wnioskodawcy względem modelu oceny → weryfikacja czy uzasadnienie wspiera wybraną odpowiedź → wskazanie przypadków, gdzie bardziej adekwatna byłaby niższa ocena → wsparcie identyfikacji kryteriów dyskwalifikujących → wynik czytelny dla Eksperta.
2. **Wsparcie przygotowania podsumowania dla Kapituły**: synteza najważniejszych info z wniosku + wynik oceny merytorycznej + raport eksperta z wizytacji, z **wyraźnym rozróżnieniem źródeł** (dane Wnioskodawcy / analiza AI / decyzje-korekty eksperta / raport eksperta).

### 6.2 Zasada nadrzędna
**AI nigdy nie jest finalną decyzją.** Wskazania AI muszą być odtwarzalne później (przechowywane jako dane, powiązane z wersją wniosku/oceny widoczną dla eksperta na danym etapie) — to sugeruje **wersjonowanie/snapshot** wyników analizy AI, nie tylko "aktualny wynik".

### 6.3 Model wdrożenia AI (rekomendacja dokumentu, do potwierdzenia)
- Korzystanie z modelu **przez API** (zewnętrzny dostawca lub infrastruktura wskazana przez Ministerstwo) — **NIE własny hosting** (nieproporcjonalne koszty przy skali "kilka wniosków miesięcznie").
- Dane **nie powinny opuszczać EOG**, chyba że Ministerstwo dopuści inaczej.
- **Brak wykorzystania danych z wniosków do trenowania modelu dostawcy.**
- Zgodność z RODO / privacy by design / security by design.

### 6.4 Bezpieczeństwo AI — istotne dla architektury promptów
- Ryzyko **prompt injection** — treść wniosku musi być traktowana jako **dane wejściowe, nigdy jako instrukcja** dla modelu. Wnioskodawca nie może wpływać na instrukcje systemowe/kryteria przez pola opisowe.
- **Rozdzielenie instrukcji systemowych od treści przekazywanych przez Wnioskodawcę** (architektonicznie: osobne warstwy promptu, sanityzacja/ escaping treści usera przed wstrzyknięciem do promptu, minimalizacja zakresu danych przekazywanych do AI do niezbędnych).
- Logowanie podstawowych informacji pozwalających odtworzyć wynik analizy AI (np. request/response, model, wersja promptu — do ustalenia poziom szczegółowości względem retencji/kosztów).

### 6.5 Otwarte decyzje wymagające potwierdzenia z Ministerstwem (kluczowe dla architektury integracji AI)
1. Czy dopuszczalne jest korzystanie z modelu przez API (zewnętrzny dostawca)?
2. Czy Ministerstwo wskazuje dopuszczonych dostawców / infrastrukturę?
3. Czy tylko modele open-source, czy też zamknięte?
4. Czy istnieje państwowe API do modelu AI?
5. Zasady retencji danych i logowania zapytań do AI.
6. Zasady finansowania/limitów/kontroli kosztów wykorzystania AI (kto płaci w testach, jakie limity, czy system blokuje po przekroczeniu limitu).

**Wniosek architektoniczny:** warstwa integracji z LLM powinna być **abstrakcyjna/wymienna** (interfejs "AI provider" niezależny od konkretnego dostawcy — OpenAI/Azure OpenAI/Anthropic/self-hosted open-source), bo decyzja o dostawcy zapadnie później i może się zmienić między MVP a produkcją.

---

## 7. Raporty, podsumowania, eksporty

| Element | Kto tworzy | Zawartość |
|---|---|---|
| **Wyniki analizy** | System/AI, widoczne dla Eksperta | dane z wniosku, odpowiedzi, uzasadnienia, wstępna ocena systemu, wskazania AI; Ekspert koryguje i zatwierdza |
| **Raport z wizytacji** | Ekspert branżowy, po wizytacji | powiązany z konkretnym wnioskiem, treść wg wcześniej ustalonych wytycznych (limit znaków > niż dla wniosku) |
| **Podsumowanie (dla Kapituły, per wniosek)** | System (auto po wprowadzeniu raportu eksperta), Admin generuje/wybiera | dane wniosku, wynik oceny merytorycznej, punktacja, wskazania AI + niespójności, decyzje/korekty eksperta, raport eksperta, rekomendacja eksperta, **jasne rozróżnienie źródeł informacji** |
| **Zbiorcze zestawienie (dla Kapituły)** | Admin generuje dla wybranej grupy wniosków (status "Gotowy dla Kapituły") | lista podsumowań + zbiorcza punktacja wszystkich wniosków, eksport (Excel priorytetowo) |

Format eksportu: **plik Excel** (jednoznacznie wskazany jako potrzebny do posiedzenia Kapituły), PDF opcjonalnie/do doprecyzowania.

---

## 8. Wymagania niefunkcjonalne (skrót)

- Dostępność przez przeglądarkę, w ramach środowiska pilotażowego.
- Wydajność: **niska skala** — kilka wniosków miesięcznie, kilku równoczesnych użytkowników. To istotnie obniża próg wymagań skalowalności na etapie MVP (ale architektura ma umożliwiać rozwój bez przebudowy).
- Kompatybilność: aktualne wersje Chrome, Edge, Firefox, Safari.
- Audytowalność / logowanie zdarzeń: złożenie wniosku, zmiany statusów, przypisania ekspertów, korekty ocen, generowanie podsumowań.
- Backup i odtwarzanie danych.
- Utrzymywalność — kod/konfiguracja/dokumentacja mają umożliwić przejęcie utrzymania przez Ministerstwo lub wskazany podmiot **po zakończeniu prac Wykonawcy**.
- Komunikaty błędów czytelne, bez ujawniania szczegółów technicznych (stack trace, struktura bazy itd.).

---

## 9. Bezpieczeństwo, RODO, zgodność prawna

- **Privacy by design / security by design** od początku architektury (nie "domalowane" na końcu).
- Minimalizacja danych, ograniczenie celu, kontrola dostępu wg roli, rozliczalność (logi), retencja wg zasad Ministerstwa (**nieustalone jeszcze konkretne okresy** — TBD), integralność i poufność danych.
- Kontrola dostępu: Wnioskodawca widzi wyłącznie własne wnioski, Ekspert wyłącznie przypisane.
- Standardowe zabezpieczenia OWASP: nieuprawniony dostęp, obejście autoryzacji, wstrzykiwanie danych wejściowych, błędy walidacji formularzy, przejęcie sesji, nadużycia związane z modułem AI.
- Bezpieczeństwo danych i dokumentów: szyfrowanie/ochrona w bazie i repozytorium plików, kontrola dostępu, zabezpieczenie transmisji (TLS), backupy, zabezpieczone eksporty.
- **Dane nie powinny opuszczać EOG** (dotyczy też ew. hostingu samej aplikacji, nie tylko AI — do potwierdzenia zakres).
- Retencja danych: okresy TBD przez Ministerstwo.

---

## 10. Infrastruktura i integracje

- **Warstwy minimalne**: Frontend (SPA/webapp w przeglądarce), Backend (logika formularzy, użytkowników, wniosków, statusów, workflow, scoring, AI, raportowanie), Baza danych, Repozytorium plików (załączniki, eksporty), moduł uwierzytelniania/autoryzacji, mechanizm powiadomień e-mail, moduł AI (integracja z LLM), logi/historia działań, backup, środowisko testowe/pilotażowe.
- **Model dostępu — dwa warianty do wyboru z Ministerstwem**:
  - **Dostęp publiczny przez Internet** (Wnioskodawcy spoza sieci MKiŚ) — właściwy, jeśli MVP ma obsługiwać realne wnioski.
  - **Dostęp ograniczony** (VPN / sieć Ministerstwa / kontrolowane środowisko) — właściwy dla testów/symulacji.
  - **To jest fundamentalna decyzja architektoniczna** (wpływa na hosting, uwierzytelnianie, zgodność, WAF/DDoS, publiczny rejestr kont) — **nierozstrzygnięta w dokumencie**, oznaczona jako "decyzja wymagająca potwierdzenia".
- Środowisko uruchomieniowe: 4 warianty możliwe (infrastruktura wskazana przez Ministerstwo / infrastruktura dostawcy-wykonawcy na testy / chmura zaakceptowana przez Ministerstwo / inne uzgodnione) — **nierozstrzygnięte**.
- Uniwersalność: architektura nie powinna blokować w przyszłości: obsługi różnych modeli oceny, konfigurowalnych kryteriów/pytań/odpowiedzi/punktacji, różnych naborów/programów, API do wymiany danych, przekształcenia w e-usługę publiczną, wykorzystania przez inne instytucje.
- Utrzymanie **docelowo po stronie Ministerstwa** (lub wskazanego podmiotu) — Wykonawca dostarcza dokumentację umożliwiającą przejęcie.

---

## 11. UX/UI

- Wykonawca (czyli częściowo ja / zespół) odpowiada za projekt interfejsu.
- Przed właściwym programowaniem: **makiety kluczowych widoków** (formularz wniosku, panel Wnioskodawcy, panel Administratora, panel Eksperta, widok wyników analizy, raport z wizytacji, podsumowanie dla Kapituły) — do akceptacji przez Zamawiającego przed implementacją.
- Interfejs ma jasno prezentować statusy, komunikaty systemowe, wskazania AI, informacje wymagające decyzji/działania.

---

## 12. Kryteria akceptacji (odbiór systemu) — 12 punktów, potraktować jako checklistę E2E testów

1. Wnioskodawca może założyć konto, uzupełnić, zapisać i złożyć wniosek online.
2. System obsługuje nabór, statusy i workflow wniosku.
3. Automatyczna ocena formalna wg reguł.
4. Automatyczna ocena merytoryczna wg modelu oceny.
5. Moduł AI wskazuje niespójności i elementy wymagające weryfikacji eksperta.
6. Administrator może przypisać wniosek do eksperta.
7. Ekspert może zweryfikować ocenę, skorygować ją i wprowadzić raport z wizytacji.
8. System generuje podsumowanie dla Kapituły oraz zbiorcze zestawienie.
9. Administrator może wprowadzić finalny status/wynik.
10. Odtworzenie historii działań i zmian statusów.
11. Zgodność z wymaganiami bezpieczeństwa i RODO wg dokumentu.
12. System przetestowany, bez błędów blokujących.

---

## 13. Ryzyka (zidentyfikowane w dokumencie)

- Moduł AI może nie osiągnąć oczekiwanej jakości analizy → mitygacja: testy na realistycznych przykładach, AI jako wsparcie a nie decyzja, możliwość korekty eksperckiej.
- Ryzyka bezpieczeństwa danych w AI (lokalizacja, retencja, poufność) → potwierdzenie modelu z Ministerstwem, minimalizacja danych do AI.
- Koszty AI/infrastruktury zależne od wybranego wariantu i skali → uzgodnienie zasad finansowania, limitów, monitoringu.
- Rozjazd MVP vs oczekiwania produkcyjne Ministerstwa → wczesne obustronne potwierdzenie zakresu.
- Niepotwierdzone podstawy prawne RODO (zakres danych, retencja, obowiązki informacyjne) → potwierdzić przed implementacją kluczowych elementów.
- Model oceny może być zbyt złożony/nieintuicyjny → testowanie z użytkownikami, iteracyjne upraszczanie.
- Niska dostępność interesariuszy Ministerstwa/użytkowników końcowych → ryzyko formalnej zgodności ale niskiej użyteczności.
- Zakres/budżet/harmonogram → priorytetyzacja MVP, backlog, szybkie decyzje zakresowe.
- Jakość/stabilność systemu → testy funkcjonalne/użytecznościowe/bezpieczeństwa/wydajności, przeglądy iteracyjne.
- Utrzymanie po zakończeniu prac → pełna dokumentacja przekazania.

---

## 14. Kluczowe otwarte pytania / decyzje (zebrane z całego dokumentu) — priorytet do rozmowy przed projektowaniem architektury

**Blokujące dla architektury backendu:**
1. **Model dostępu**: publiczny internet czy ograniczony do sieci/VPN Ministerstwa? (wpływa na auth, hosting, zgodność).
2. **Środowisko uruchomieniowe / hosting**: kto je zapewnia i gdzie (Ministerstwo / dostawca / chmura zaakceptowana)?
3. **Model AI**: czy przez API zewnętrznego dostawcy jest dopuszczalny? Jaki dostawca/model? Czy jest wymagane rozwiązanie rządowe/EOG-only? Otwarte vs zamknięte modele?
4. **Retencja danych** (ile, dla jakich kategorii) — Ministerstwo ma to określić.
5. **Podpis elektroniczny wniosku** — jaki wariant techniczny (np. profil zaufany, e-podpis kwalifikowany, prostszy mechanizm na etapie MVP)?
6. **Finansowanie/limity kosztów AI**.

**Merytoryczne (mniej blokujące technicznie, ale wpływają na model danych):**
7. Ostateczne wagi kryteriów i udział matryc A/B/C (po badaniu ABCD Suzuki) — architektura MUSI zakładać, że to się zmieni.
8. Format i szczegółowość eksportów PDF (Excel jest pewny).
9. Niespójność: formularz sugeruje "autouzupełnianie" nazwy/adresu z NIP (sekcja 8 wniosku) — ale integracje z zewnętrznymi rejestrami są **poza zakresem MVP**. Trzeba wyjaśnić, czy to pole zwykłego tekstowego wpisania, czy faktycznie ma być zintegrowane (wtedy jednak przeczy punktowi "Elementy poza zakresem" #3).
10. Najnowsza wersja specyfikacji — link w `Przydatne linki.docx` (OneDrive) może zawierać zaktualizowaną wersję różną od PDF-a przeanalizowanego tutaj (dokument ma ślady redakcji — przekreślone/zduplikowane wymagania FUN-037, 062, 065, 068, 087, 091, 094, 106, 112, 115 — sugeruje to trwającą pracę redakcyjną nad rejestrem).

---

## 15. Wstępne obserwacje pod kątem architektury Django (do przedyskutowania, NIE finalna decyzja)

Poniżej tylko surowe obserwacje wynikające z materiałów — punkt wyjścia do rozmowy, nie rekomendacja:

- **Model oceny jako dane konfiguracyjne**, nie kod: potrzebna jest struktura danych reprezentująca Matryca → Kryterium → PoziomOdpowiedzi(1–5) → Waga, oraz osobno RegułyFormalne i RegułyDyskwalifikujące, tak by aktualizacja wag (po ABCD Suzuki) czy przyszła obsługa "różnych modeli oceny/naborów" (wskazana explicite jako pożądany kierunek rozwoju) nie wymagała zmian w kodzie/migracjach.
- **Workflow wniosku jako jawna maszyna stanów** (wiele statusów, część automatycznych, część ręcznych/eksperckich) — dobry kandydat na bibliotekę FSM lub jawnie zamodelowany model History + Status z walidacją przejść, plus pełny log (django model changes / custom audit log), bo audytowalność jest wymaganiem twardym (kryterium akceptacji #10 i #11).
- **Custom User model z rolami** (Wnioskodawca / Administrator / Ekspert branżowy), rejestracja publiczna tylko dla Wnioskodawcy, pozostałe role nadawane ręcznie — klasyczny wzorzec Django (grupy/permissions albo prosty enum roli + object-level permissions dla "widzi tylko przypisane/własne wnioski").
- **Warstwa integracji AI jako abstrakcja** (provider-agnostic), z twardym rozdzieleniem system prompt / user content, logowaniem zapytań i wyników z możliwością odtworzenia (wersjonowanie wyniku per etap procesu) — to wprost wymaganie (FUN-052, sekcja 8.4, 9.3).
- **Formularz wniosku wielosekcyjny z zależnościami międzysekcyjnymi** (autouzupełnianie A2 z sekcji "Informacje o Ekoinnowacji") i autozapisem — sugeruje potrzebę jasnego modelu "wersja robocza" (draft) vs "złożony" (immutable snapshot po złożeniu), żeby ocena i AI zawsze operowały na zamrożonej treści.
- Skala ruchu jest bardzo niska (kilka wniosków/miesiąc) — nie ma potrzeby projektować pod duże obciążenie na MVP, ale trzeba zostawić "drzwi" do skalowania (wymaganie wprost: architektura nie może blokować dalszego rozwoju).
- Eksport do Excel jako wymaganie twarde (openpyxl/xlsxwriter), PDF opcjonalny.
- E-mail (min. 4 zdarzenia) — prosty async task (np. Celery/Django-Q/RQ czy nawet zadania synchroniczne na start, do decyzji wg wolumenu).

---

## 16. Rekomendacja co do dalszych kroków

1. **Zdobyć aktualną wersję specyfikacji** z linku OneDrive (obecna analiza oparta o PDF, który ma ślady bieżącej redakcji rejestru wymagań).
2. Rozmowa z Product Managerem / Ministerstwem w sprawie punktów z sekcji 14 (zwłaszcza #1–#6) — to determinuje architekturę hostingu, auth i integracji AI.
3. Na tej podstawie — wspólna rozmowa o architekturze Django (moduły appek, model danych pod matryce, warstwa AI, workflow engine, wybór narzędzi do PDF/Excel, kolejek zadań, itd.).

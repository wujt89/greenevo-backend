# GreenEvo / GREENSTRAT — backend

Repozytorium robocze dla backendu narzędzia wspierającego ocenę wniosków w programie GreenEvo (projekt GREENSTRAT, Ministerstwo Klimatu i Środowiska). Na razie zawiera materiały analityczne i decyzje architektoniczne — kod aplikacji dopiero powstaje.

## Kluczowe dokumenty

- [`NOTATKA_ARCHITEKTURA.md`](./NOTATKA_ARCHITEKTURA.md) — research z materiałów źródłowych (specyfikacja wymagań, matryce oceny, diagram procesu).
- [`ARCHITEKTURA_DECYZJE.md`](./ARCHITEKTURA_DECYZJE.md) — **żywy dokument**, wszystkie decyzje architektoniczne (stack, model danych, maszyna stanów, API, prompty AI, struktura repo). Aktualizować przy każdej kolejnej decyzji.
- [`NAUKA_STACK.md`](./NAUKA_STACK.md) — plan nauki stacku technicznego z priorytetami.
- `GreenEvo_Architektura_prezentacja.pdf`, `GreenEvo_Nauka_Stack.pdf` — wersje prezentacyjne powyższych, do spotkań.

## Materiały źródłowe

- `Specyfikacja wymagań systemu.pdf` — specyfikacja funkcjonalna od Zamawiającego.
- `Matryce oceny ekoinnowacji - 08.07.2026` — metodyka oceny (matryce A/B/C, kryteria, punktacja).
- `(graf Andrzeja) wymagania_wnioski_app_ext.svg` — diagram BPMN procesu TO-BE.
- `Przydatne linki.docx` — linki do materiałów zewnętrznych.

## Stack (skrót — pełne uzasadnienie w `ARCHITEKTURA_DECYZJE.md`)

Django + Django Ninja Extra (API) + django-ninja-jwt (auth) + PostgreSQL + django-fsm-2 (workflow) + Django-Q2 (async) + pydantic (walidacja/schematy) + React (osobny frontend, poza tym repo).

## Backlog

Zobacz [Issues](../../issues) — rozbite wg aplikacji (`accounts`, `evaluation_model`, `applications`, `ai_assist`, `reporting`, `infra`).

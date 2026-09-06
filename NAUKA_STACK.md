# GreenEvo — co ogarnąć przed spotkaniem (stack techniczny)

Kolejność wg priorytetu — jeśli masz mało czasu, rób **Tier 1** w całości, **Tier 2** przynajmniej "żeby umieć wytłumaczyć i napisać prosty przykład", **Tier 3** wystarczy przeczytać raz (to głównie konfiguracja, nie głęboka wiedza).

Każda pozycja: co to jest → gdzie dokładnie tego używamy w GreenEvo → konkretne rzeczy do przećwiczenia → na co uważać / co mogą Cię zapytać.

---

## TIER 1 — musisz to ogarniać płynnie

### 1. Django (odświeżenie pod kątem projektu)

**Co to jest**: wiesz już, ale skup się na częściach, które będą kluczowe tutaj, nie na ogólnym odświeżaniu wszystkiego.

**Gdzie w GreenEvo**: custom User model (`accounts`), migracje przy każdej zmianie modelu oceny, management command do ładowania fixture'ów, signals (używane z django-fsm-2).

**Do przećwiczenia:**
- Custom User model od zera: `AbstractBaseUser` + `PermissionsMixin` + `BaseUserManager`, z `USERNAME_FIELD = "email"` (nie `username`). To się różni od domyślnego Django User i trzeba to zrobić **przed pierwszą migracją** — jeśli zaczniesz projekt bez custom usera i zechcesz go dodać później, to boli (wymaga resetu migracji albo skomplikowanej migracji danych). Zrób to jako pierwszy krok w projekcie.
- `BaseCommand` — napisz prosty management command (`add_arguments`, `handle()`), bo `load_evaluation_model` będzie właśnie tym.
- Signals: `post_save`, a przede wszystkim `pre_transition`/`post_transition` z django-fsm-2 (patrz punkt 4).

**Na co uważać**: różnica między `AbstractUser` (dziedziczysz domyślne pola username/first_name/last_name) a `AbstractBaseUser` (budujesz od zera, więcej kontroli, ale musisz sam zdefiniować manager). Dla nas: `AbstractBaseUser` + `PermissionsMixin`, bo pola takie jak `username` nam niepotrzebne.

---

### 2. Django Ninja Extra

**Co to jest**: warstwa API REST na Django w stylu FastAPI — schematy danych to zwykłe modele **pydantic** (nie osobny system serializerów jak w DRF), routing przez klasowe kontrolery (`api_controller`) z dekoratorami `@route`, OpenAPI generowane automatycznie z type hintów.

**Zmiana decyzji**: pierwotnie planowany był DRF, zmieniliśmy na Ninja Extra już po zaplanowaniu architektury (patrz `ARCHITEKTURA_DECYZJE.md` rozdz. 2, sekcja uzasadnienia) — głównie z powodu spójności z pydantic (i tak potrzebnym do walidacji odpowiedzi AI) i automatycznego OpenAPI dla naszych licznych custom-action endpointów (przejścia FSM).

**Gdzie w GreenEvo**: dosłownie cały nasz backend — każdy endpoint z `ARCHITEKTURA_DECYZJE.md` (rozdz. 7) to jakiś `api_controller` + metoda z `@route`.

**Do przećwiczenia:**
- `Schema` (pydantic) jako input/output — w tym **nested schematy** dla `PATCH /applications/{id}/` (zagnieżdżone `applicant_info`, `eco_innovation_info`, lista `criterion_answers` w jednym requeście). Ważna różnica względem DRF: `Schema` to tylko kontrakt danych, **nie ma automatycznego zapisu do bazy** — logikę zapisu (w tym zagnieżdżonych obiektów) zawsze piszesz sam w ciele metody kontrolera. To większa swoboda, ale też więcej kodu do napisania niż przy prostym CRUD w DRF.
- `@api_controller("/applications")` + `@route.get(...)`/`@route.post(...)` — dokładnie tak zrobimy przejścia FSM (`submit`, `assign-expert` itd.) jako osobne metody kontrolera.
- Permission classes Ninja Extra (`permissions.BasePermission`) — dla per-obiektowych uprawnień (`IsOwnerApplicant`/`IsAssignedExpert`) to mniej ubita ścieżka niż w DRF; często prościej zrobić zwykły `if not (obj.applicant == request.user): raise PermissionDenied` wprost w metodzie kontrolera.
- Filtrowanie querysetu wg roli w ciele metody (nie ma automatycznego `get_queryset()` jak w DRF ViewSet — filtrujesz explicite).
- `@api.exception_handler(...)` zarejestrowany na poziomie `NinjaExtraAPI` — to jak zrobimy nasz spójny format błędów z kodami (`RECRUITMENT_CLOSED` itd.).
- Paginacja (`ninja_extra.pagination`, `@paginate`) — do listy userów/wniosków.

**Na co uważać**: skoro nie ma automatycznego zapisu z `Schema` do modelu (w przeciwieństwie do `ModelSerializer.save()`), musisz pamiętać, żeby logika biznesowa (np. "czy nabór aktywny") była sprawdzana explicite w kontrolerze przed zapisem — nie ma tu żadnego ukrytego miejsca jak `serializer.validate()`, gdzie mogłaby się "sama" wykonać.

---

### 3. PostgreSQL — konkretnie `JSONField`

**Co to jest**: nie musisz się uczyć Postgresa od zera (SQL już znasz), skup się na tym jak Django go używa, szczególnie `JSONField`.

**Gdzie w GreenEvo**: `Application.submitted_snapshot`, `AIAnalysisResult.raw_output`, `ApplicationSummary.content` — wszystko to `django.db.models.JSONField`, mapowane na typ `jsonb` w Postgresie.

**Do przećwiczenia:**
- Zapis/odczyt `JSONField` w Django ORM — to jest trywialne (zwykły dict/list w Pythonie).
- Filtrowanie po kluczach wewnątrz JSON (`Model.objects.filter(content__some_key="value")`) — może się przydać przy raportowaniu, warto wiedzieć że to istnieje, nie musisz być ekspertem.

**Na co uważać**: `jsonb` w Postgresie jest zaindeksowywalny (GIN index), ale na MVP przy tej skali (kilka wniosków miesięcznie) to zupełnie nieistotne — nie trzeba tego teraz ogarniać.

---

### 4. django-fsm-2

**Co to jest**: biblioteka do maszyny stanów na modelach Django. To jest **najmniej standardowa** biblioteka z całego stacku, więc warto jej poświęcić realny czas — reszta zespołu (i Ministerstwo, jeśli będą pytać) może nie znać tej biblioteki, więc dobrze żebyś Ty to ogarniał na pamięć.

**Gdzie w GreenEvo**: cały `Application.status` i 12 przejść z `ARCHITEKTURA_DECYZJE.md` (rozdz. 6).

**Do przećwiczenia (zrób dosłownie mały toy-projekt, 15-20 minut):**
```python
from django_fsm import FSMField, transition

class Order(models.Model):
    status = FSMField(default="new")

    @transition(field=status, source="new", target="paid")
    def pay(self):
        pass

    @transition(field=status, source="paid", target="shipped",
                conditions=[lambda self: self.has_stock()])
    def ship(self):
        pass
```
- Sprawdź, co się dzieje, gdy wywołasz `order.ship()` z niewłaściwego stanu — złap wyjątek `django_fsm.TransitionNotAllowed`, zobacz jego atrybuty (`object`, `method_name`) — to jest dokładnie to, czego użyjemy w naszym exception handlerze do budowania czytelnego komunikatu błędu.
- Napisz receiver na sygnał `django_fsm.signals.post_transition` — to jest miejsce, gdzie u nas będzie się zapisywał `StatusTransition` i lecieć powiadomienia.
- `conditions=[...]` parametr dekoratora — pozwala wpiąć warunek biznesowy (np. "czy nabór aktywny") bezpośrednio w definicję przejścia, zamiast sprawdzać to ręcznie w widoku. Rozważ, czy chcemy to tak zrobić dla `submit()`.
- `protected=True` na `FSMField` — sprawdź co się stanie, jak spróbujesz zrobić `obj.status = "x"` bezpośrednio (powinno rzucić błąd).

**Na co uważać**: `source` może być listą statusów (`source=[Status.DRAFT, Status.TO_COMPLETE]`) — to dokładnie nasz przypadek dla `submit()`. `source="*"` oznacza "z dowolnego stanu" — możesz się na to natknąć w dokumentacji, ale my tego nie używamy (chcemy jawnych przejść).

---

### 5. django-ninja-jwt

**Co to jest**: JWT auth dla Django Ninja (odpowiednik `simplejwt`, ten sam koncept: access token krótki + refresh token dłuższy). Zmiana z `djangorestframework-simplejwt` wynika wprost ze zmiany frameworka API (rozdz. 2 `ARCHITEKTURA_DECYZJE.md`) — API tego samego rodzaju, inna biblioteka pod spodem.

**Gdzie w GreenEvo**: `POST /api/auth/token/`, `/token/refresh/`, cała autoryzacja API.

**Do przećwiczenia:**
- Domyślny flow zwraca `{access, refresh}` w body JSON, tak samo jak w simplejwt. **U nas refresh token ma iść do httpOnly cookie** (bezpieczniejsze niż localStorage po stronie React) — to wymaga własnej obsługi w kontrolerze logowania (ustawiasz `response.set_cookie(...)` ręcznie, usuwasz refresh z body). To **niezależne od wyboru biblioteki** — ten sam wzorzec trzeba by napisać ręcznie zarówno z `simplejwt`, jak i z `ninja-jwt`, bo to niestandardowe rozszerzenie w obu przypadkach.
- Config tokenów (odpowiednik `SIMPLE_JWT` w settings): czas życia access/refresh tokena, rotacja refresh tokenów.
- Mechanizm blacklisty tokenów (jeśli dostępny w tej bibliotece — sprawdź w dokumentacji) — potrzebny, żeby faktyczne "wylogowanie" unieważniało refresh token (bez tego JWT jest bezstanowy i nie da się go "unieważnić" przed wygaśnięciem).

**Na co uważać**: JWT jest **bezstanowy** — jeśli ktoś zapyta "jak dezaktywujecie konto w trakcie aktywnej sesji", odpowiedź to: `is_active=False` na Userze zablokuje nowe logowania, ale już wydany access token będzie ważny do wygaśnięcia (stąd krótki czas życia access tokena, np. 15 min, jest ważny). To nie zależy od tego, czy używamy `simplejwt` czy `ninja-jwt` — sama natura JWT.

---

## TIER 2 — poznaj na tyle, żeby napisać prosty przykład i wytłumaczyć decyzję

### 6. django-simple-history

**Co to jest**: automatyczne wersjonowanie zmian na modelu — dodajesz `history = HistoricalRecords()` do modelu, biblioteka sama tworzy tabelę `Historical<Model>` i zapisuje snapshot przy każdym `save()`.

**Gdzie w GreenEvo**: `Application`, `CriterionAnswer`, `ExpertReview` — audytowalność zmian.

**Do przećwiczenia**: dodaj `HistoricalRecords()` do jednego modelu w toy-projekcie, zrób kilka zmian, sprawdź `instance.history.all()` i `instance.history.as_of(jakas_data)`.

**Na co uważać**: to jest **inny mechanizm niż nasz `StatusTransition`** — `django-simple-history` śledzi zmiany *pól* (kto zmienił co i kiedy), `StatusTransition` to nasz własny, celowy log *biznesowych przejść workflow*. Oba działają równolegle, nie zastępują się nawzajem — warto umieć to wytłumaczyć, jeśli ktoś zapyta "po co dwa mechanizmy audytu".

---

### 7. Django-Q2

**Co to jest**: kolejka zadań asynchronicznych działająca na Postgres (bez Redis/RabbitMQ).

**Gdzie w GreenEvo**: wysyłka maili, wywołania AI, generowanie Excela, **łańcuch zadań po zatwierdzeniu raportu z wizytacji** (rozdz. 7a w `ARCHITEKTURA_DECYZJE.md`).

**Do przećwiczenia:**
```python
from django_q.tasks import async_task

async_task("myapp.tasks.send_email", user_id, hook="myapp.tasks.on_email_sent")
```
- Uruchom lokalnie `python manage.py qcluster` (to jest worker) i sprawdź, że zadanie faktycznie się wykonuje w tle.
- Zrozum wzorzec `hook` — funkcja wywoływana po zakończeniu taska, dostaje obiekt `Task` z wynikiem. To jest właśnie mechanizm do **łańcuchowania**: task 1 (analiza AI) w hooku odpala task 2 (budowa podsumowania), który w swoim hooku wywołuje przejście FSM.

**Na co uważać**: Django-Q2 zapisuje wyniki zadań w bazie (`django_q.models.Success`/`Failure`) — to się przyda też do debugowania/audytu "czy zadanie AI się wykonało".

---

### 8. pydantic

**Co to jest**: walidacja danych przez modele z typami (`BaseModel`, pola z typami Pythona, automatyczna walidacja).

**Gdzie w GreenEvo**: walidacja JSON-a zwracanego przez model AI (`CriterionEvaluationResult`, `SummarySupportResult` z `ARCHITEKTURA_DECYZJE.md` rozdz. 8a).

**Do przećwiczenia:**
```python
from pydantic import BaseModel, Field, ValidationError

class Result(BaseModel):
    suggested_level: int | None = Field(ge=1, le=5)
    is_consistent: bool

try:
    r = Result.model_validate_json(raw_text_from_llm)
except ValidationError as e:
    print(e.errors())  # lista konkretnych błędów pól
```
- Zrozum różnicę: `model_validate()` (z dicta) vs `model_validate_json()` (z surowego stringa JSON — dokładnie to dostaniemy z odpowiedzi LLM-a).
- Popatrz na strukturę `ValidationError.errors()` — to Ci pomoże zbudować sensowny komunikat przy "jednej próbie korekty" (rozdz. 8a).

**Na co uważać**: to ta sama biblioteka, na której opiera się `Schema` w Django Ninja Extra (patrz punkt 2) — ucząc się pydantic tutaj, uczysz się jednocześnie fundamentu całego API, nie tylko warstwy AI. To jeden z głównych powodów, dla których wybraliśmy Ninja zamiast DRF.

---

## TIER 3 — przeczytaj raz, to głównie konfiguracja

### 9. django-storages
Ustawiasz `STORAGES` w settings (Django 4.2+ nowy format, dict z kluczami `default`/`staticfiles`), wskazujesz backend (`FileSystemStorage` lokalnie, `storages.backends.s3.S3Storage` w chmurze). `FileField`/`ImageField` używają tego transparentnie — kod modelu się nie zmienia między środowiskami, zmienia się tylko config.

### 10. openpyxl
```python
from openpyxl import Workbook
wb = Workbook()
ws = wb.active
ws.append(["Nazwa wniosku", "Punktacja A", "Punktacja B", "Punktacja C"])
for app in applications:
    ws.append([app.name, app.score_a, app.score_b, app.score_c])
wb.save("export.xlsx")  # albo do BytesIO na odpowiedź HTTP
```
Tyle właściwie wystarczy do MVP — nie trzeba znać zaawansowanego formatowania.

### 11. django-environ
```python
import environ
env = environ.Env()
DEBUG = env.bool("DEBUG", default=False)
DATABASES = {"default": env.db("DATABASE_URL")}
```
Jedno zadanie: nie trzymać sekretów w kodzie.

### 12. pytest-django + factory_boy
```python
@pytest.mark.django_db
def test_submit_application(applicant_user):
    app = ApplicationFactory(applicant=applicant_user, status="draft")
    app.submit()
    assert app.status == "submitted"
```
`ApplicationFactory(factory.django.DjangoModelFactory)` z polami wypełnianymi automatycznie (`factory.Faker(...)`) — oszczędza pisanie boilerplate'u w każdym teście.

### 13. ~~drf-spectacular~~ — niepotrzebne
Po zmianie na Django Ninja Extra dokumentacja OpenAPI (`/api/docs/`) generuje się **automatycznie** z type hintów kontrolerów i schematów pydantic — nie trzeba osobnej biblioteki ani ręcznych adnotacji na custom endpointach. Jeden punkt mniej do nauczenia się.

---

## Jeśli będą pytać "dlaczego nie X" na spotkaniu

- **Dlaczego nie Celery zamiast Django-Q2?** — bo przy skali kilku wniosków miesięcznie osobny broker (Redis/RabbitMQ) to zbędna infrastruktura do utrzymania; Django-Q2 działa na tej samej bazie Postgres. Migracja do Celery jest możliwa później.
- **Dlaczego nie ręczny słownik przejść zamiast django-fsm-2?** — bo sam pilnuje niedozwolonych przejść i jest czytelniejszy (metody na modelu zamiast rozproszonej logiki w widokach).
- **Dlaczego JWT a nie sesje?** — bo frontend (React) i backend to rozdzielone aplikacje, JWT jest bezstanowym standardem dla takiej architektury.
- **Dlaczego PostgreSQL a nie coś innego?** — Twoja decyzja "100%", dobrze pasuje do potrzeb audytowalności i JSONField.
- **Dlaczego Django Ninja Extra a nie Django REST Framework?** — spójność: i tak potrzebujemy pydantic do walidacji odpowiedzi AI, więc Ninja daje jeden mechanizm walidacji w całym projekcie zamiast dwóch. Do tego automatyczny OpenAPI bez ręcznych adnotacji — istotne, bo znaczna część naszego API to custom-action endpointy (przejścia FSM), nie zwykły CRUD. Świadomie akceptujemy mniejszy ekosystem/społeczność niż DRF.
- **Dlaczego django-fsm-2 a nie django-viewflow?** — Viewflow to pełny silnik BPM zaprojektowany pod server-rendered UI (widoki zadań, Django admin) — nie pasuje do architektury z osobnym frontendem React. Nasz workflow (12 statusów, liniowy z jedną pętlą i jednym rozgałęzieniem) nie potrzebuje pełnego BPM engine z równoległymi gałęziami/join — fsm-2 jest proporcjonalny do rzeczywistej złożoności.

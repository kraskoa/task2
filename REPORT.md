# Sprawozdanie – Task 2 (wariant Python, Flask_Book_Library)

## Zadanie 1 – Wyciek danych wrażliwych w logach

### Znalezione wycieki
1. **`Customer.__init__`** (`project/customers/models.py`) wypisuje `str(self)`, a `__repr__` zawiera PESEL, ulicę i nr mieszkania:
   ```
   Getting: Customer(ID: None, Name: Jan Kowalski, City: Warszawa, Age: 30, Pesel: 90010112345, Street: Marszalkowska 1, AppNo: 12)
   ```
2. **Komunikat wyjątku SQLAlchemy** – dodanie klienta o istniejącej nazwie (`name` jest `unique`) powoduje `IntegrityError`, którego treść zawiera parametry zapytania INSERT. Trafia on do logu oraz do odpowiedzi HTTP 500:
   ```
   Error creating customer: (sqlite3.IntegrityError) UNIQUE constraint failed: customers.name
   [SQL: INSERT INTO customers (name, city, age, pesel, street, "appNo") VALUES (?, ?, ?, ?, ?, ?)]
   [parameters: ('Jan Kowalski', 'Krakow', '40', '85050554321', 'Florianska 5', '3')]
   ```

### Poprawka
- `project/customers/models.py` – funkcja `mask()` zamienia każdy znak na `*`; `__repr__` maskuje `pesel`, `street`, `appNo`. Format logu bez zmian.
- `project/__init__.py` – `SQLALCHEMY_ENGINE_OPTIONS = {'hide_parameters': True}`, dzięki czemu SQLAlchemy nie umieszcza wartości parametrów w komunikatach błędów (log i odpowiedź HTTP).

### Wynik po poprawce
```
Getting: Customer(ID: None, Name: Jan Kowalski, City: Warszawa, Age: 30, Pesel: ***********, Street: ***************, AppNo: **)
Error creating customer: (sqlite3.IntegrityError) UNIQUE constraint failed: customers.name
[SQL: INSERT INTO customers (name, city, age, pesel, street, "appNo") VALUES (?, ?, ?, ?, ?, ?)]
[SQL parameters hidden due to hide_parameters=True]
```

## Zadanie 2 – Wyciek sekretów (gitleaks)

```shell
docker run --rm --user $(id -u):$(id -g) -v "$PWD":/path zricethezav/gitleaks:latest detect --source="/path" -v --redact
```
(`--user` zapobiega błędowi git `dubious ownership`, `--redact` ukrywa wartości sekretów w wyniku.)

### Wyniki – 3 znaleziska, wszystkie `private-key`

| Plik | Commit | Weryfikacja | Ocena |
|---|---|---|---|
| `deployment2.key` | `de9d7b8` | `openssl pkey` – poprawny klucz RSA 512 bit | **True positive** |
| `deployment.key` | `bc17b7d` | `openssl pkey` – poprawny klucz RSA 512 bit | **True positive** |
| `awscredentials.json` (`private_key`, l. 5) | `bc17b7d` | plik poświadczeń konta serwisowego; treść klucza nie parsuje się jako poprawny klucz | **True positive** (plik z poświadczeniami w repo; wartość najpewniej testowa) |

Pliki zostały usunięte w commicie `fd67400 remove artifacts`, ale nadal są w historii (i publicznie dostępne na GitHubie pod linkiem do commita). Usunięcie pliku nie usuwa wycieku.

**Fałszywe pozytywy: brak.** Miejsca, w których słowo `password`/`token` występuje jako zmienna bez wartości (`StudentController.java: String password = ""`, `mvnw: $MVNW_PASSWORD`, `loans.js: csrf_token`), nie zostały zgłoszone.

### Zalecenia
1. Unieważnić/zrotować wyciekłe klucze – są skompromitowane niezależnie od dalszych kroków.
2. Usunąć pliki z historii (za pomocą polecenia `git filter-repo`) i wykonać force push.
3. Sekrety przekazywać przez zmienne środowiskowe / secret manager zamiast w kodzie i `Dockerfile`; potencjalnie dodać gitleaks jako hook pre-commit / krok CI.

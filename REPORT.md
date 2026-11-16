# Sprawozdanie - Task 2 (wariant Python, Flask_Book_Library)

## Zadanie 1 - Wyciek danych wrażliwych w logach

### Znalezione wycieki
1. **`Customer.__init__`** (`project/customers/models.py`) wypisuje `str(self)`, a `__repr__` zawiera PESEL, ulicę i nr mieszkania:
   ```
   Getting: Customer(ID: None, Name: Jan Kowalski, City: Warszawa, Age: 30, Pesel: 90010112345, Street: Marszalkowska 1, AppNo: 12)
   ```
2. **Komunikat wyjątku SQLAlchemy** - dodanie klienta o istniejącej nazwie (`name` jest `unique`) powoduje `IntegrityError`, którego treść zawiera parametry zapytania INSERT. Trafia on do logu oraz do odpowiedzi HTTP 500:
   ```
   Error creating customer: (sqlite3.IntegrityError) UNIQUE constraint failed: customers.name
   [SQL: INSERT INTO customers (name, city, age, pesel, street, "appNo") VALUES (?, ?, ?, ?, ?, ?)]
   [parameters: ('Jan Kowalski', 'Krakow', '40', '85050554321', 'Florianska 5', '3')]
   ```

### Poprawka
- `project/customers/models.py` - funkcja `mask()` zamienia każdy znak na `*`; `__repr__` maskuje `pesel`, `street`, `appNo`. Format logu bez zmian.
- `project/__init__.py` - `SQLALCHEMY_ENGINE_OPTIONS = {'hide_parameters': True}`, dzięki czemu SQLAlchemy nie umieszcza wartości parametrów w komunikatach błędów (log i odpowiedź HTTP).

### Wynik po poprawce
```
Getting: Customer(ID: None, Name: Jan Kowalski, City: Warszawa, Age: 30, Pesel: ***********, Street: ***************, AppNo: **)
Error creating customer: (sqlite3.IntegrityError) UNIQUE constraint failed: customers.name
[SQL: INSERT INTO customers (name, city, age, pesel, street, "appNo") VALUES (?, ?, ?, ?, ?, ?)]
[SQL parameters hidden due to hide_parameters=True]
```

## Zadanie 2 - Wyciek sekretów (gitleaks)

```shell
docker run --rm --user $(id -u):$(id -g) -v "$PWD":/path zricethezav/gitleaks:latest detect --source="/path" -v --redact
```
(`--user` zapobiega błędowi git `dubious ownership`, `--redact` ukrywa wartości sekretów w wyniku.)

### Wyniki - 3 znaleziska, wszystkie `private-key`

| Plik | Commit | Weryfikacja | Ocena |
|---|---|---|---|
| `deployment2.key` | `de9d7b8` | `openssl pkey` - poprawny klucz RSA 512 bit | **True positive** |
| `deployment.key` | `bc17b7d` | `openssl pkey` - poprawny klucz RSA 512 bit | **True positive** |
| `awscredentials.json` (`private_key`, l. 5) | `bc17b7d` | plik poświadczeń konta serwisowego; treść klucza nie parsuje się jako poprawny klucz | **True positive** (plik z poświadczeniami w repo; wartość najpewniej testowa) |

Pliki zostały usunięte w commicie `fd67400 remove artifacts`, ale nadal są w historii (i publicznie dostępne na GitHubie pod linkiem do commita). Usunięcie pliku nie usuwa wycieku.

**Fałszywe pozytywy: brak.** Miejsca, w których słowo `password`/`token` występuje jako zmienna bez wartości (`StudentController.java: String password = ""`, `mvnw: $MVNW_PASSWORD`, `loans.js: csrf_token`), nie zostały zgłoszone.

### Zalecenia
1. Unieważnić/zrotować wyciekłe klucze - są skompromitowane niezależnie od dalszych kroków.
2. Usunąć pliki z historii (za pomocą polecenia `git filter-repo`) i wykonać force push.
3. Sekrety przekazywać przez zmienne środowiskowe / secret manager zamiast w kodzie i `Dockerfile`; potencjalnie dodać gitleaks jako hook pre-commit / krok CI.

## Zadanie 3 - Podatności w bibliotekach OpenSource (safety)

```shell
docker run --rm -v "$PWD":/sources pyupio/safety safety check -r /sources/requirements-task.txt --full-report
```

### Wyniki
Sprawdzono 18 pakietów, wykryto 17 podatności w 5 pakietach:

| Pakiet | Wersja | Liczba podatności | Przykłady |
|---|---|---|---|
| werkzeug | 2.3.7 | 8 | 71594 - RCE przez debugger, 62019 - DoS przy parsowaniu multipart |
| jinja2 | 3.1.2 | 5 | 64227 - XSS przez filtr `xmlattr`, ucieczki z sandboxa |
| mako | 1.2.4 | 2 | |
| flask | 2.3.3 | 1 | 86909 - brak nagłówka `Vary: Cookie` (ujawnienie informacji) |
| healpy | 1.8.0 | 1 | 61774 - podatny dołączony `libcurl` |

### Analiza: healpy 1.8.0 - ID 61774 (CVE-2023-38545)
**Podatność:** healpy ≤ 1.16.6 dołącza `libcurl` w wersji podatnej na CVE-2023-38545 - przepełnienie bufora na stercie w obsłudze proxy SOCKS5 (curl: *High*, NVD CVSS 9.8 *Critical*). To najpoważniejsza z wykrytych podatności, bo potencjalnie umożliwia wykonanie kodu (RCE).

**Warunki wykorzystania:** biblioteka musi pobierać zasób przez `libcurl` (w healpy np. odczyt pliku FITS z URL) przez proxy SOCKS5 z rozwiązywaniem nazw po stronie proxy (`socks5h://`), a atakujący musi przekazać nazwę hosta dłuższą niż 255 bajtów (np. przez przekierowanie z kontrolowanego serwera) przy wolnym handshake'u proxy.

**Ocena dla aplikacji:**
- `healpy` nie jest importowany nigdzie w kodzie (tylko wpis w `requirements-task.txt`),
- nie ma go w `requirements.txt`, z którego budowany jest obraz Docker - `pip show healpy` w kontenerze: *Package(s) not found*,
- aplikacja nie wykonuje żadnych połączeń wychodzących ani nie korzysta z proxy SOCKS5.

Podatny kod nie jest obecny w środowisku uruchomieniowym i nie jest osiągalny z aplikacji - **prawdopodobieństwo wykorzystania jest pomijalne**. Zalecenie: usunąć zbędną zależność `healpy` z `requirements-task.txt`.

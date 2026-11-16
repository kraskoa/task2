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

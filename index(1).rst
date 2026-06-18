=====================================================
Rozdział 4: Implementacja bazy danych i import danych
=====================================================

W ramach czwartego rozdziału zrealizowano dwa etapy prac: utworzenie fizycznej struktury bazy danych oraz przygotowanie mechanizmu importu danych testowych do środowisk SQLite i PostgreSQL. Celem było nie tylko zdefiniowanie tabel i relacji, ale również sprawdzenie poprawności działania bazy przez jej wypełnienie przykładowymi rekordami oraz automatyzację masowego ładowania danych z pliku CSV.

4.1. Utworzenie struktury bazy danych
-------------------------------------

W pierwszym etapie utworzono plikową bazę danych ``wypozyczalnia.db`` z użyciem silnika SQLite. Za implementację odpowiada wbudowany moduł ``sqlite3`` języka Python. Po nawiązaniu połączenia wykonano skrypt DDL definiujący encje ``Kategorie``, ``Klienci``, ``Samochody`` oraz ``Wypozyczenia``. 

Zgodnie z najlepszymi praktykami, jawnie włączono obsługę więzów referencyjnych dla silnika SQLite używając instrukcji ``PRAGMA foreign_keys = ON;``, która domyślnie jest wyłączona.

Poniższy kod realizuje zadanie utworzenia schematu fizycznego:

.. code-block:: python

    import sqlite3

    # Nawiązanie połączenia z bazą SQLite
    conn_sqlite = sqlite3.connect('wypozyczalnia.db')
    cursor_sqlite = conn_sqlite.cursor()

    # Wymuszenie kontroli kluczy obcych (wymagane w SQLite)
    cursor_sqlite.execute("PRAGMA foreign_keys = ON;")

    # Inicjalizacja schematu bazy danych
    cursor_sqlite.executescript("""
    CREATE TABLE IF NOT EXISTS Kategorie (
        ID_Kategorii INTEGER PRIMARY KEY,
        Nazwa TEXT NOT NULL,
        Cena_Za_Dzien REAL NOT NULL
    );

    CREATE TABLE IF NOT EXISTS Klienci (
        ID_Klienta INTEGER PRIMARY KEY,
        Imie TEXT NOT NULL,
        Nazwisko TEXT NOT NULL,
        PESEL TEXT UNIQUE NOT NULL,
        Nr_Prawa_Jazdy TEXT UNIQUE,
        Telefon TEXT,
        Email TEXT
    );

    CREATE TABLE IF NOT EXISTS Samochody (
        ID_Samochodu INTEGER PRIMARY KEY,
        Marka TEXT NOT NULL,
        Model TEXT NOT NULL,
        Nr_Rejestracyjny TEXT UNIQUE NOT NULL,
        Rok_Produkcji INTEGER,
        ID_Kategorii INTEGER,
        FOREIGN KEY (ID_Kategorii) REFERENCES Kategorie(ID_Kategorii)
    );

    CREATE TABLE IF NOT EXISTS Wypozyczenia (
        ID_Wypozyczenia INTEGER PRIMARY KEY,
        ID_Klienta INTEGER,
        ID_Samochodu INTEGER,
        Data_Od TEXT NOT NULL,
        Data_Do TEXT NOT NULL,
        Status TEXT NOT NULL,
        FOREIGN KEY (ID_Klienta) REFERENCES Klienci(ID_Klienta),
        FOREIGN KEY (ID_Samochodu) REFERENCES Samochody(ID_Samochodu)
    );
    """)

    conn_sqlite.commit()
    cursor_sqlite.close()
    conn_sqlite.close()


4.2. Wprowadzenie danych demonstracyjnych (DML)
-----------------------------------------------

Aby umożliwić testowanie logiki zapytań (DQL), schemat uzupełniono przykładowymi danymi w ramach zapytań z rodziny DML. Rekordy pozwalają na symulację wypożyczeń samochodów przez klientów w określonych kategoriach cenowych.

.. code-block:: python

    import sqlite3

    conn_sqlite = sqlite3.connect('wypozyczalnia.db')
    cursor_sqlite = conn_sqlite.cursor()
    cursor_sqlite.execute("PRAGMA foreign_keys = ON;")

    cursor_sqlite.executescript("""
    INSERT INTO Kategorie (Nazwa, Cena_Za_Dzien) VALUES 
    ('Kompakt', 150.00), ('SUV', 250.00), ('Premium', 400.00);

    INSERT INTO Klienci (Imie, Nazwisko, PESEL, Nr_Prawa_Jazdy, Telefon, Email) VALUES 
    ('Jan', 'Kowalski', '90051412345', '12345/67/8901', '+48123456789', 'jan.kowalski@email.com'),
    ('Anna', 'Nowak', '85112298765', '98765/43/2109', '+48987654321', 'anna.nowak@email.com');

    INSERT INTO Samochody (Marka, Model, Nr_Rejestracyjny, Rok_Produkcji, ID_Kategorii) VALUES 
    ('Toyota', 'Corolla', 'WA12345', 2022, 1),
    ('Kia', 'Sportage', 'KR54321', 2023, 2),
    ('BMW', 'Seria 5', 'PO98765', 2024, 3);

    INSERT INTO Wypozyczenia (ID_Klienta, ID_Samochodu, Data_Od, Data_Do, Status) VALUES 
    (1, 1, '2026-06-01', '2026-06-05', 'Zakończone'),
    (2, 2, '2026-06-10', '2026-06-15', 'W_trakcie'),
    (1, 3, '2026-07-01', '2026-07-03', 'Zarezerwowane');
    """)

    conn_sqlite.commit()
    cursor_sqlite.close()
    conn_sqlite.close()


4.3. Generowanie zbioru testowego CSV
-------------------------------------

Z myślą o późniejszym zasilaniu silników relacyjnych stworzono skrypt w języku Python generujący losowe i unikalne dane o klientach do formatu wyjściowego CSV. 

.. code-block:: python

    import csv
    import random

    imiona = ["Jan", "Anna", "Piotr", "Katarzyna", "Tomasz"]
    nazwiska = ["Kowalski", "Nowak", "Wiśniewski", "Kamiński"]

    dane_do_csv = []
    for i in range(10):
        imie = random.choice(imiona)
        nazwisko = random.choice(nazwiska)
        pesel = f"{random.randint(50, 99):02d}{random.randint(1, 12):02d}{random.randint(1, 28):02d}{random.randint(10000, 99999)}"
        nr_pj = f"{random.randint(10000, 99999)}/{random.randint(10, 99)}/{random.randint(1000, 9999)}"
        telefon = f"+48{random.randint(100000000, 999999999)}"
        email = f"{imie.lower()}.{nazwisko.lower()}{random.randint(1,100)}@email.com"
        dane_do_csv.append([imie, nazwisko, pesel, nr_pj, telefon, email])

    with open('klienci.csv', 'w', newline='', encoding='utf-8') as plik_csv:
        writer = csv.writer(plik_csv)
        writer.writerow(['Imie', 'Nazwisko', 'PESEL', 'Nr_Prawa_Jazdy', 'Telefon', 'Email'])
        writer.writerows(dane_do_csv)


4.4. Masowy import danych do PostgreSQL i SQLite
------------------------------------------------

Do importu danych przygotowano ustandaryzowaną procedurę ETL opartą o dwa odmienne mechanizmy ładowania, optymalne z punktu widzenia systemów docelowych. PostgreSQL wykorzystuje potok binarny tworzony dyrektywą ``COPY``, będącą branżowym standardem bardzo wydajnego przenoszenia danych między stacją roboczą a serwerem bazy danych. W przypadku lekkiego silnika SQLite wykorzystano mechanikę transakcyjnego wprowadzania pakietów danych poprzez metodę ``executemany()`` – co minimalizuje narzut operacji IO w bazach jednoplikowych.

.. code-block:: python

    import csv
    import sqlite3
    import psycopg
    import simplejson

    def importuj_dane(sciezka_csv):
        with open(sciezka_csv, 'r', encoding='utf-8') as plik_csv:
            reader = csv.reader(plik_csv)
            next(reader)
            dane_z_csv = [row for row in reader]

        # Wersja z użyciem zoptymalizowanego COPY (PostgreSQL)
        try:
            with open("database_creds.json") as db_con_file:
                creds = simplejson.loads(db_con_file.read())

            conn_pg = psycopg.connect(
                host=creds['host_name'], user=creds['user_name'],
                dbname=creds['db_name'], password=creds['password'], port=creds['port_number']
            )

            with conn_pg.cursor() as cursor_pg:
                cursor_pg.execute("TRUNCATE TABLE Klienci CASCADE;")
                
                # Zastosowanie strumieniowania COPY
                with cursor_pg.copy(
                    "COPY Klienci (Imie, Nazwisko, PESEL, Nr_Prawa_Jazdy, Telefon, Email) FROM STDIN"
                ) as copy_operation:
                    for row in dane_z_csv:
                        copy_operation.write_row(row)

            conn_pg.commit()
            conn_pg.close()
        except Exception as e:
            print(f"Błąd PostgreSQL: {e}")

        # Wersja z użyciem transakcyjnego executemany (SQLite)
        try:
            conn_sl = sqlite3.connect('wypozyczalnia.db')
            cursor_sl = conn_sl.cursor()
            cursor_sl.execute("PRAGMA foreign_keys = ON;")
            cursor_sl.execute("DELETE FROM Klienci;")

            # Zastosowanie optymalnego wsadowego INSERT (batch)
            cursor_sl.executemany("""
                INSERT INTO Klienci (Imie, Nazwisko, PESEL, Nr_Prawa_Jazdy, Telefon, Email)
                VALUES (?, ?, ?, ?, ?, ?)
            """, dane_z_csv)

            conn_sl.commit()
            cursor_sl.close()
            conn_sl.close()
        except Exception as e:
            print(f"Błąd SQLite: {e}")

    if __name__ == "__main__":
        importuj_dane("klienci.csv")
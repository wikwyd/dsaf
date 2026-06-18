====================================================================
Rozdział 3. Dokumentacja modelu bazy danych wypożyczalni samochodów
====================================================================

Wybór zagadnienia i opis procesów
=================================

Zagadnienie i jego opis
-----------------------
System zarządzania flotą i wynajmem w wypożyczalni samochodów.
Projekt obejmuje ewidencję klientów, dostępnych pojazdów z podziałem na kategorie cenowe, a także pełen proces wypożyczeń aut przez klientów, z określeniem czasu wynajmu i statusu operacji.

Opis procesów
-------------
Głównym procesem w systemie jest obieg samochodu między wypożyczalnią a klientem.

* **Rejestracja klienta:** Klient jest dodawany do bazy, system wymaga podania danych osobowych i kontaktowych. Wymagana jest unikalność numeru PESEL oraz Numeru Prawa Jazdy.
* **Katalogowanie floty:** Samochody posiadają unikalne numery rejestracyjne i są przypisywane do grup cenowych (Kategorii).
* **Wypożyczenie:** Rejestracja faktu wynajmu auta, który łączy konkretnego klienta z pojazdem w zdefiniowanym przedziale czasu (od-do). Proces ten otrzymuje status (np. Zakończone, W_trakcie, Zarezerwowane).
* **Więzy integralności:** Usunięcie lub modyfikacja samochodu/klienta powiązanego z rezerwacją musi być kontrolowana przez system (np. poprzez klucze obce), aby zapobiegać niespójności bazy.

Wykaz gromadzonych danych
-------------------------
* **Dane klienta:** Imie, Nazwisko, PESEL, Nr_Prawa_Jazdy, Telefon, Email.
* **Dane auta:** Marka, Model, Rok_Produkcji, Nr_Rejestracyjny.
* **Dane kategorii:** Nazwa kategorii, Cena za dzień.
* **Dane transakcyjne:** Data_Od, Data_Do, Status wypożyczenia.

Prototypowy JSON/CSV
====================

Aby zweryfikować kompletność wprowadzanych danych, przygotowano pliki zawierające próbkowe dane.

Prototyp CSV
------------
.. code-block:: text

    Imie,Nazwisko,PESEL,Nr_Prawa_Jazdy,Telefon,Email
    Piotr,Wiśniewski,59081618205,84679/96/1949,+48706147330,piotr.wiśniewski8@email.com

Prototyp JSON
-------------
.. code-block:: json

    {
      "klienci": [
        {
          "Imie": "Piotr",
          "Nazwisko": "Wiśniewski",
          "PESEL": "59081618205",
          "Nr_Prawa_Jazdy": "84679/96/1949",
          "Telefon": "+48706147330",
          "Email": "piotr.wiśniewski8@email.com"
        }
      ]
    }

Model koncepcyjny bazy danych
=============================

Zidentyfikowane encje
---------------------
* **Klient**
* **Samochód**
* **Kategoria**
* **Wypożyczenie**

Zdefiniowane atrybuty
---------------------
* **Klient:** Imie, Nazwisko, PESEL, Nr_Prawa_Jazdy, Telefon, Email
* **Samochód:** Marka, Model, Rok_Produkcji, Nr_Rejestracyjny
* **Kategoria:** Nazwa, Cena_Za_Dzien
* **Wypożyczenie:** Data_Od, Data_Do, Status

Opis związków
-------------
* **Kategoria – Samochód (1:N):** Do jednej kategorii może być przypisanych wiele samochodów, ale samochód należy tylko do jednej kategorii.
* **Klient – Wypożyczenie (1:N):** Pojedynczy klient ma możliwość wypożyczania aut wielokrotnie.
* **Samochód – Wypożyczenie (1:N):** Samochód może być w historii swojej eksploatacji wypożyczony przez wielu różnych klientów.

Identyfikacja encji słabych
---------------------------
Ze względu na występowanie relacji wiele-do-wielu (M:N) pomiędzy encjami ``Klient`` i ``Samochód``, utworzono encję asocjacyjną (słabą) o nazwie **Wypożyczenia**. Rozwiązuje ona pułapkę połączeń (wiatrak) i ewidencjonuje każdą transakcję.

Model w notacji Chena
---------------------
.. image:: model_koncepcyjny_chen.png
   :width: 80%
   :alt: Model konceptualny bazy danych wypożyczalni samochodów w notacji Chena.

Model logiczny i proces normalizacji
====================================

Proces normalizacji
-------------------
Zastosowano proces normalizacji, doprowadzając relacje do 3 Postaci Normalnej (3NF). Każda tabela posiada klucz główny (PK). Aby uniknąć redundancji i anomalii modyfikacji, atrybuty określające stawkę cenową przeniesiono do słownikowej tabeli ``Kategorie``. Utworzono relacyjne klucze obce (FK), gwarantujące integralność referencyjną.

Ostateczna struktura tabel (3NF)
--------------------------------
* **Kategorie:** ID_Kategorii (PK), Nazwa, Cena_Za_Dzien
* **Klienci:** ID_Klienta (PK), Imie, Nazwisko, PESEL, Nr_Prawa_Jazdy, Telefon, Email
* **Samochody:** ID_Samochodu (PK), ID_Kategorii (FK), Marka, Model, Nr_Rejestracyjny, Rok_Produkcji
* **Wypożyczenia:** ID_Wypozyczenia (PK), ID_Klienta (FK), ID_Samochodu (FK), Data_Od, Data_Do, Status

Schemat ERD w notacji Barkera
-----------------------------
.. image:: model_logiczny_barker.png
   :width: 80%
   :alt: Diagram logiczny ERD w notacji Barkera (Postać 3NF).

Model fizyczny bazy danych
==========================

Model fizyczny dla środowiska SQLite
------------------------------------
SQLite korzysta z podstawowych typów ``TEXT``, ``INTEGER`` i ``REAL``. Brak natywnych typów daty wymaga zapisywania ich jako ciągi znaków (``TEXT``). Do kluczy głównych zastosowano automatyczną sekwencję ``INTEGER PRIMARY KEY``.

* **Kategorie:** ID_Kategorii : INTEGER PRIMARY KEY, Nazwa : TEXT, Cena_Za_Dzien : REAL
* **Klienci:** ID_Klienta : INTEGER PRIMARY KEY, Imie : TEXT, Nazwisko : TEXT, PESEL : TEXT UNIQUE, Nr_Prawa_Jazdy : TEXT UNIQUE, Telefon : TEXT, Email : TEXT
* **Samochody:** ID_Samochodu : INTEGER PRIMARY KEY, Marka : TEXT, Model : TEXT, Nr_Rejestracyjny : TEXT UNIQUE, Rok_Produkcji : INTEGER, ID_Kategorii : INTEGER REFERENCES Kategorie
* **Wypożyczenia:** ID_Wypozyczenia : INTEGER PRIMARY KEY, ID_Klienta : INTEGER REFERENCES Klienci, ID_Samochodu : INTEGER REFERENCES Samochody, Data_Od : TEXT, Data_Do : TEXT, Status : TEXT

Model fizyczny dla środowiska PostgreSQL
----------------------------------------
Baza PostgreSQL umożliwiła implementację ścisłych typów natywnych, optymalizujących miejsce i wydajność silnika z wykorzystaniem ``VARCHAR``, ``DATE`` i formatowania liczbowego ``NUMERIC``. Zastosowano typ ``SERIAL`` do obsługi sekwencji kluczy głównych.

* **Kategorie:** ID_Kategorii : SERIAL PRIMARY KEY, Nazwa : VARCHAR(50), Cena_Za_Dzien : NUMERIC(10,2)
* **Klienci:** ID_Klienta : SERIAL PRIMARY KEY, Imie : VARCHAR(50), Nazwisko : VARCHAR(50), PESEL : VARCHAR(11) UNIQUE, Nr_Prawa_Jazdy : VARCHAR(20) UNIQUE, Telefon : VARCHAR(15), Email : VARCHAR(100)
* **Samochody:** ID_Samochodu : SERIAL PRIMARY KEY, Marka : VARCHAR(50), Model : VARCHAR(50), Nr_Rejestracyjny : VARCHAR(20) UNIQUE, Rok_Produkcji : INT, ID_Kategorii : INT REFERENCES Kategorie
* **Wypożyczenia:** ID_Wypozyczenia : SERIAL PRIMARY KEY, ID_Klienta : INT REFERENCES Klienci, ID_Samochodu : INT REFERENCES Samochody, Data_Od : DATE, Data_Do : DATE, Status : VARCHAR(20)
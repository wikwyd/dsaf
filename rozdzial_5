Rozdział 5: Zapytania do bazy danych
====================================

Poniższa sekcja stanowi dokumentację funkcji analitycznych przygotowanych do obsługi relacyjnej bazy danych (PostgreSQL).

Moduł: Zapytania
----------------

.. py:function:: pobierz_czas_wypozyczen(conn)

   Pobiera listę zakończonych wypożyczeń wraz z obliczonym czasem ich trwania.

   **Zadanie/cel funkcji:** Zastosowanie selekcji danych (klauzula WHERE) oraz funkcji wierszowych (operacje arytmetyczne na datach i konkatenacja ciągów znaków).

   :param conn: Aktywny obiekt połączenia z bazą danych.
   :return: Lista krotek zawierających ID wypożyczenia, pełną nazwę auta oraz liczbę dni.

   **Opis działania:**
   Wykonuje zapytanie obliczające różnicę między ``data_do`` a ``data_od`` dla rekordów ze statusem ``Zakonczone``. Złącza nazwy aut z tabeli samochodów.


.. py:function:: raport_przychodow_kategorii(conn)

   Generuje raport łącznych przychodów i liczby wypożyczeń dla każdej kategorii aut.

   **Zadanie/cel funkcji:** Wykorzystanie funkcji agregujących (COUNT, SUM) oraz grupowania rekordów (GROUP BY).

   :param conn: Aktywny obiekt połączenia z bazą danych.
   :return: Lista krotek (nazwa kategorii, liczba wypożyczeń, łączny szacowany przychód).

   **Opis działania:**
   Zapytanie łączy wielokrotnie tabele ``kategorie``, ``samochody`` i ``wypozyczenia``. Grupuje wyniki po nazwie kategorii, sumując iloczyn dni wypożyczenia i ceny za dzień.


.. py:function:: sprawdz_historie_aut(conn)

   Zwraca pełną listę samochodów włączając w to auta, które nigdy nie były wypożyczone.

   **Zadanie/cel funkcji:** Zastosowanie złączeń zewnętrznych (LEFT JOIN).

   :param conn: Aktywny obiekt połączenia z bazą danych.
   :return: Lista krotek (marka, model, nr rejestracyjny, data_od).

   **Opis działania:**
   Używa LEFT JOIN między tabelą ``samochody`` a ``wypozyczenia``. Dzięki temu samochody bez historii wypożyczeń pojawią się w wyniku z wartością NULL w dacie.


.. py:function:: dostepne_samochody(conn)

   Wyszukuje samochody, które aktualnie stoją na placu (nie są wypożyczone).

   **Zadanie/cel funkcji:** Wykorzystanie operatorów zbiorowych (EXCEPT - różnica zbiorów).

   :param conn: Aktywny obiekt połączenia z bazą danych.
   :return: Lista krotek (ID samochodu, marka, model).

   **Opis działania:**
   Zapytanie pobiera zbiór wszystkich istniejących samochodów i odejmuje od niego zbiór tych aut, które mają aktywny status ``W trakcie`` w wypożyczeniach.


.. py:function:: klienci_najdrozszych_aut(conn)

   Pobiera dane kontaktowe klientów, którzy wypożyczyli najdroższe auta z floty.

   **Zadanie/cel funkcji:** Zastosowanie podzapytań (subqueries).

   :param conn: Aktywny obiekt połączenia z bazą danych.
   :return: Lista krotek (imię, nazwisko, email, telefon).

   **Opis działania:**
   Wykorzystuje zagnieżdżone podzapytanie do określenia ID kategorii o maksymalnej cenie za dzień (``ORDER BY DESC LIMIT 1``), a następnie po tym ID filtruje klientów.

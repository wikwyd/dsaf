=====================================================
Implementacja bazy danych i import danych
=====================================================

W ramach czwartego rozdziału zrealizowano dwa kluczowe etapy prac wdrożeniowych: utworzenie struktur tabelarycznych zdefiniowanych w modelu fizycznym oraz wdrożenie zoptymalizowanych mechanizmów importu danych testowych. Prace przeprowadzono w sposób równoległy dla środowiska lokalnego opartego na silniku SQLite oraz serwerowego wykorzystującego system PostgreSQL.

4.1. Definicja i inicjalizacja struktur danych
==============================================

Pierwszym etapem wdrażania systemu bazodanowego było przeniesienie założeń modelu fizycznego do wytypowanych systemów zarządzania bazami danych (DBMS). Proces ten wymagał adaptacji skryptów DDL (Data Definition Language) do specyficznych dialektów SQL obsługiwanych przez wybrane środowiska.

W przypadku **PostgreSQL**, definicje tabel utworzono wykorzystując zaawansowane typy danych (np. ``NUMERIC`` dla walut) oraz współczesne metody autoinkrementacji kluczy głównych poprzez klauzulę ``GENERATED ALWAYS AS IDENTITY``. Implementacja na serwerze objęła również deklarację kluczy obcych (``FOREIGN KEY``) oraz więzów sprawdzających (``CHECK``), co gwarantuje integralność danych na poziomie silnika relacyjnego, m.in. poprzez walidację chronologii dat wypożyczeń.

W środowisku **SQLite**, będącym lekką bazą wbudowaną o ograniczonej kontroli typowania (tzw. *manifest typing*), wykorzystano uproszczone mapowanie typów ograniczające się do ``INTEGER``, ``TEXT`` oraz ``REAL``. Proces inicjalizacji wymagał w tym przypadku wywołania procedury aktywującej wsparcie dla więzów referencyjnych za pomocą dyrektywy ``PRAGMA foreign_keys = ON;``, która w konfiguracji domyślnej jest dezaktywowana w celu optymalizacji wydajności. W obu przypadkach zaimplementowany schemat ściśle odzwierciedla relacje zachodzące między encjami: Kategoriami, Klientami, Samochodami oraz Wypożyczeniami.

4.2. Zasilenie bazy danymi demonstracyjnymi
===========================================

Na etapie testowania architektury modelu fizycznego, puste relacje zostały zasilone zbiorem danych demonstracyjnych wykorzystując klasyczne polecenia DML z rodziny ``INSERT``. Procedura ta miała na celu weryfikację poprawności działania nałożonych ograniczeń integralnościowych – na przykład weryfikację unikalności numerów PESEL i numerów rejestracyjnych pojazdów (więzy ``UNIQUE``) oraz zgodność typów asocjacyjnych. Pomyślne zainicjowanie procesu DML potwierdziło poprawność logiki izolacji poszczególnych encji (Trzecia Postać Normalna - 3NF).

4.3. Koncepcja mechanizmów masowego importu (ETL)
=================================================

Dla zasymulowania warunków produkcyjnych i obciążenia systemów większym wolumenem rekordów, zaprojektowano dedykowany generator danych. Utworzył on standaryzowany plik formatu CSV (Comma-Separated Values) reprezentujący logikę płaskich rekordów z informacjami o potencjalnych klientach.

Mając na uwadze fundamentalne różnice architektoniczne między serwerem bazodanowym a bazą jednoplikową, zaimplementowano dwie zróżnicowane strategie masowego ładowania danych:

Mechanizm COPY w PostgreSQL
---------------------------
Ze względu na architekturę klient-serwer, transfer ogromnej ilości pojedynczych zapytań ``INSERT`` jest nieefektywny ze względu na opóźnienia sieciowe (network latency) oraz koszt parsowania zapytań przez optymalizator. Wdrożono natywną funkcję ``COPY`` w trybie ładowania ze strumienia (``STDIN``). Pozwala to na binarny, ciągły przesył paczek danych bezpośrednio do pliku tabeli, omijając standardowy narzut kontroli transakcyjnej dla pojedynczych rzędów. Jest to referencyjne podejście zalecane do zadań typu Data Ingestion.

Mechanizm Batch Insert w SQLite
-------------------------------
SQLite zapisuje wszystkie informacje w jednym pliku na dysku. Przy tradycyjnym podejściu, każda operacja ``INSERT`` wymaga otwarcia transakcji, zablokowania pliku, modyfikacji struktury i wywołania operacji I/O (fsync). Aby zminimalizować ten destrukcyjny dla wydajności narzut, import przeprowadzono metodą operacji wsadowych (Batch Insert) wykorzystując funkcję środowiskową ``executemany()``. Pozwoliło to zgrupować cały import w ramach jednej, ciągłej transakcji aplikacyjnej, radykalnie skracając czas trwania procesu.

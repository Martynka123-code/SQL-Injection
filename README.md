# Laboratorium: SQL-Injection

**Środowisko:** Damn Vulnerable Web Application (DVWA) - Docker  
**Cel:** Praktyczna identyfikacja, wykorzystanie podatności SQL Injection (In-band, Blind) oraz analiza metod obrony.

**ZASADY:** Zabrania się używania automatycznych skanerów (np. `sqlmap`). Wszystkie ataki należy przeprowadzić manualnie lub przy wsparciu narzędzi deweloperskich/proxy (Burp Suite).

### Konfiguracja Startowa

1. Uruchom kontener: `docker run --rm -it -p 8080:80 vulnerables/web-dvwa`
    
2. Zaloguj się (`admin`/`password`).
    
3. Przejdź do **Setup / Reset DB** $\rightarrow$ kliknij **Create / Reset Database**.
    
4. Przejdź do **DVWA Security** $\rightarrow$ Ustaw poziom na **Low** $\rightarrow$ **Submit**.

---

### Zadanie 1: Rozpoznanie (Error-Based Recon)

**Cel:** Identyfikacja środowiska bazy danych poprzez wywołanie błędu.

**Lokalizacja:** Menu SQL Injection.

1. Wprowadź znak lub ciąg znaków, który celowo **zaburzy oryginalną składnię zapytania SQL**, generując błąd bazy danych.
    
2. **Obserwacja:** Przeanalizuj komunikat błędu wyświetlony przez aplikację.
    
    - **Weryfikacja:** Ustal, jaki silnik bazy danych (np. MySQL, PostgreSQL) został ujawniony w komunikacie.

<details> 
<summary>Pytanie do refleksji: </summary>
Dlaczego ujawnianie tak szczegółowych komunikatów o błędach jest poważnym błędem bezpieczeństwa? 
</details>


### Zadanie 2: Enumeracja Struktury (Union-Based)

**Cel:** Wydobycie struktury bazy danych (nazw tabel i kolumn) za pomocą operatora `UNION`.

**Lokalizacja:** Menu SQL Injection.

1. **Ustalenie liczby kolumn:** Metodą prób i błędów (np. używając `ORDER BY` lub `UNION SELECT NULL, ...`) określ, ile kolumn jest zwracanych przez oryginalne zapytanie.
    
2. **Eksploatacja `information_schema`:** Skonstruuj payload wykorzystujący `UNION`, który wyświetli nazwy tabel znajdujące się w bieżącej bazie danych (`dvwa`). Użyj systemowej tabeli **`information_schema.tables`**.
    
3. **Weryfikacja:** Odnotuj nazwy dwóch kluczowych tabel znalezionych w bazie `dvwa`.
    
<details> 
<summary>Pytanie do refleksji: </summary>
Jakie uprawnienia musiałyby być odebrane użytkownikowi bazy danych, aby dostęp do <code>information_schema</code> był niemożliwy?
</details>


### Zadanie 3: Kradzież Danych (Data Dump)

**Cel:** Wydobycie loginów i haszy haseł.

**Lokalizacja:** Menu SQL Injection.

1. **Skonstruowanie zapytania:** Znając już nazwy tabel, stwórz zapytanie `UNION`, które wyświetli zawartość kolumn `user` i `password` z tabeli użytkowników.
    
2. **Formatowanie Danych:** Użyj funkcji **`CONCAT()`** (lub jej odpowiednika), aby połączyć dane z obu kolumn w jednym, czytelnym ciągu znaków.
    
3. **Weryfikacja:** Zlokalizuj hasło (hash) dla użytkownika o ID "1337".
    
<details> 
<summary>Pytanie do refleksji: </summary>
Dlaczego haszowanie (np. MD5) nie chroni haseł, gdy atakujący ma dostęp do całej bazy danych i jakie techniki można zastosować, aby utrudnić ich złamanie?
</details>


### Zadanie 4: Poziom Medium (Omijanie filtrów)

**Cel:** Przeprowadzenie ataku SQL Injection mimo podstawowej sanityzacji.

1. Przejdź do **DVWA Security** i zmień poziom na **Medium**.
    
2. Przechwyć żądanie HTTP (za pomocą Burp Suite lub Narzędzi Deweloperskich) i zmodyfikuj parametr `id` w żądaniu POST.
    
3. **Wyzwanie (Brak Apostrofów/Cudzysłowów):** Skonstruuj działający payload (np. wydobywający wersję bazy danych – `@@version`), **nie używając ani jednego znaku apostrofu (`'`) ani cudzysłowu (`"`).**
    
4. **Analiza Powodzenia:**
    
    - Wklej zmodyfikowany parametr `id` (Payload) wysłany w żądaniu POST.
        
<details> 
<summary>Pytanie do refleksji: </summary>
Zastanów się (opierając się na różnicy w kodzie PHP między poziomem Low a Medium), dlaczego ten atak zadziałał mimo użycia funkcji zabezpieczającej. Odpowiedź powinna dotyczyć sposobu, w jaki kod traktuje wartość <code>$id</code> (czy jest ona w cudzysłowach, czy jest traktowana jako liczba).
</details>


### Zadanie 5: Blind SQL Injection (Time-Based) & Logika Ataku

**Cel:** Eksploatacja, gdy baza nie zwraca **żadnych** danych ani błędów, opierając się na czasie odpowiedzi serwera.

**Lokalizacja:** SQL Injection (Blind). (Upewnij się, że Security Level to **Low**).

#### Część A: Proof of Concept

1. Skonstruuj zapytanie logiczne (`AND`), które uśpi bazę danych na zdefiniowany czas (np. 3 sekundy), tylko jeśli spełniony jest **określony warunek** (np. jeśli wersja bazy danych zaczyna się od '5').
    
2. **Weryfikacja:** Zaobserwuj, czy serwer odpowiada z opóźnieniem, co potwierdza wykonanie polecenia `sleep()`.
    
    - **Wymagane Funkcje MySQL:** `AND IF(WARUNEK, sleep(czas), 0)` oraz `SUBSTRING()` lub `LENGTH()`.
        

#### Część B: Algorytmika Wydobycia

<details> 
<summary>Pytanie do refleksji: </summary>
Zastanów się nad <code>logiką algorytmu</code>, który pozwoliłby automatowi (np. skryptowi) odgadnąć hasło administratora znak po znaku (np. 32-znakowy hash).
</details>

1. **Kluczowe Kroki:** W opisie uwzględnij:
    
    - Jak ustalić długość hasła.
        
    - Jakie pętle są potrzebne (zewnętrzna po znakach, wewnętrzna do odgadnięcia znaku).
        
    - Jak zoptymalizować proces (np. ASCII).
        

### Zadanie 6: Analiza kodu i mechanizmy obronne (Impossible Level)

**Cel:** Zrozumienie działania zapytań parametryzowanych (Prepared Statements).

1. Przejdź do **DVWA Security** i zmień poziom na **Impossible**.
    
2. Kliknij przycisk **View Source** (na dole strony SQL Injection) i przeanalizuj kod PHP (zwróć uwagę na wykorzystanie obiektu PDO).
    
<details> 
<summary>Pytania do refleksji: </summary>
  
- Jaka jest fundamentalna różnica w sposobie konstruowania zapytania SQL między poziomem Low (konkatenacja stringów), a poziomem Impossible (z PDO)? 
  
- W jaki sposób silnik bazy danych traktuje strukturę zapytania SQL, a w jaki dane wejściowe (parametry) w podejściu Prepared Statements?

- Dlaczego wpisanie złośliwego kodu (np. <code>' OR 1=1 #</code>) w pole formularza nie jest w stanie zmienić logiki wykonywanego zapytania, lecz jest traktowane jedynie jako niegroźna wartość (np. ID użytkownika)?
</details>

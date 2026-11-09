https://github.com/prochotwik-pv/repozytorium  
https://github.com/prochotwik-pv/repozytorium/blob/main/lab1/sesje/transcript_full.jsonl  
Antigravity · Gemini 4 Argon · MEDIUM<br>  


1. Jaki wynik zlecono, jakie zmiany były dozwolone i o jakie informacje do odbioru poproszono agenta?<br>   
Zlecono wygenerowanie pliku - raportu z pliku wejściowego data/exercise.csv do result/exercise-summary.json
Zastosowano ścisłe granice tzn. całkowity zakaz zmiany danych, kodu i testów, a także wymóg zatwierdzania i wyjaśniania swoich decyzji  
przed wykonaniem jakiejś komendy. POproszono agenta, by do odbioru zaprezentował lokalizacje i zawartość raportu,  
wywoływanego polecenia i rzeczywisty wynik wykonania procesu i rezultat przygotowanego testu

2. Jak powstał raport? Wskaż wejście, polecenie, narzędzie, środowisko wykonania i wynik procesu. Dodaj  link do końcowego raportu i wyjaśnij jego liczebności.<br>  
Wejście - plik data/exercise.csv komenda py -3 describe_data.py --data data/exercise.csv --out results/exercise-summary.json  
Program: describe_data.py
Narzędzie - narzędzie uruchamiającego proces powłoki z ominięciem piaskownicy (BypassSandbox: true);   
Środowisko - skrypt przetworzony na lokalnym systemie Windows przez zainstalowany interpreter Python 3.15.1  
Wynik  - pomyślne wykonannie skryptu (kod wyjścia 0) wypisanie krótkiego komunikatu w konsoli i wygenerowanie raportu końcowego  
Raport - https://github.com/prochotwik-pv/repozytorium/blob/main/lab1/project/results/exercise-summary.json   
Raport powstał przez odczytanie danych CSV i zliczenie wiadomości według przypisanych im etykiet.  
Wyjaśnienie liczebności - 8 (total) parser potraktował każdy wieloliniowy e-mail jako pojedynczy rekord, nie gubiąc znaków na nowych liniach;  
 5(ham) i 3 (spam) < -- gotowe wartości z kolumny label dla każdego z 8 rekordów tzn maila, nie analizując ich treści
3. Który fragment dokumentacji lub kodu wyjaśniłeś? Podaj plik i miejsce, własne wyjaśnienie oraz mały
 przypadek: przewidywanie, sposób sprawdzenia i wniosek.<br>  
 Wybrałam plik describe_data.py (14-38), gdzie wczytywano csv i zliczano etykiety.  
 Parser csv.DictReader odczytuje plik, grupując wieloliniowe wiadomości w cudzysłowach w pojedyncze rekordy. Następnie pętla sumuje wyłącznie wartości  
 pobrane z kolumny label, całkowicie ignorując treść maila.  
 Przykład zmienionego przypadku - plik z 4 wiadomościami spam celowo w polu text wiadomości zostało oznaczone jako ham. Etykiety w kolumnie label 2 x ham, 2 x spam
 Przewidywanie - Przykład z 4 wiadomościami - program wskaże łącznie 4 wiadomości (2 spam ; 2 ham)   
                 Zmiana etykiety w zbiorze ćwiczeniowym - liczba ham zmniejszy się o 1, liczba spam zwiększy się o 1, a łączna liczba wiadomości pozostanie bez zmian - 8.
 Sposób sprawdzenia - Weryfikacja krok po kroku w czacie – podałam swój wynik na podstawie analizy kodu, zanim agent ujawnił prawidłową odpowiedź. - przeanalizował działanie pętli,  
uruchomił skrypt describe_data.py na zmodyfikowanym pliku csv i odczytał wygenerowany raport 
 Wniosek - Przewidywanie się potwierdziło i udowodniło, że skrypt zlicza tylko wyłącznie jawne etykiety - wartości z kolumny label i nie klasyfikuje tekstu na podstawie  
 zawartych słów oraz nie liczy fizycznych wierszy pliku.  
 
4. Jak uruchomiono gotowy test, jaki był wynik i czego dotyczyło sprawdzenie?<br>  

5. Jaką etykietę zmieniono w etapie 2? Jakiej zmiany liczebności oczekiwałeś, jakie operacje wybrałeś i dlaczego? Porównaj wynik przed zmianą i po nowym wykonaniu.<br>  


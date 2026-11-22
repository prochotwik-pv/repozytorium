Repozytorium: https://github.com/prochotwik-pv/repozytorium <br>  
Aplikacja agenta - Antigravity<br>  
Model językowy - Gemini Neon 4 <br>  
Poziom rozumowania - High<br>  


---

# Od problemu do planu rozwiązania — klasyfikacja spamu

## 1. Cel i zakres projektu

Celem zadania jest rozpoznanie problemu **binarnej klasyfikacji wiadomości e-mail** (odróżniania niechcianej poczty `spam` od prawidłowej korespondencji `ham`) oraz przygotowanie kompletnego, uzasadnionego **planu budowy lokalnego prototypu filtru spamu**.

Zakres obecnego etapu obejmuje:
- analizę struktury projektu, dokumentacji oraz faktycznej zawartości plików danych;
- zaprojektowanie sposobu przygotowania tekstu, reprezentacji liczbowej, wyboru modelu głównego i alternatywnego oraz procedury uczenia, klasyfikacji nowych wiadomości i oceny;
- wyraźne oddzielenie faktów potwierdzonych przez dane od założeń metodologicznych oraz określenie ryzyk i planowanych korekt.

**Czego obecny etap nie obejmuje:** projekt nie zawiera jeszcze kodu źródłowego, a model **nie został wytrenowany ani oceniony**. Oryginalne pliki w katalogu `data/` oraz dostarczony podział na zbiór treningowy i walidacyjny pozostają całkowicie nienaruszone. Wszystkie przyszłe artefakty robocze (kod, zapisany potok modelu, raporty) powinny być zapisywane w osobnych folderach poza `data/`.

---

## 2. Charakterystyka danych

### Fakty potwierdzone w dokumentacji i bezpośredniej analizie plików CSV

Projekt zawiera główny opis zadania (`README.md`), dokumentację danych (`data/README.md`), rejestr pochodzenia i sum kontrolnych (`data/manifest.json`) oraz dwa pliki CSV w kodowaniu UTF-8: `data/train.csv` i `data/validation.csv`.

Jeden rekord CSV odpowiada **jednej osobnej wiadomości e-mail** opisanej trzema kolumnami:
- `message_id` — 16-znakowy unikalny identyfikator szesnastkowy (w 100% przypadków równy pierwszym 16 znakom sumy kontrolnej `raw_sha256` z `manifest.json`); służy do kontroli integralności, łączenia z metadanymi i analizy błędów, **nie jest cechą wejściową dla modelu**;
- `text` — odkodowany temat i pełna treść tekstowa wiadomości (wyłączne źródło cech dla modelu);
- `label` — etykieta klasy (`ham` albo `spam`), wykorzystywana tylko do uczenia i oceny.

| Obszar kontroli | `data/train.csv` | `data/validation.csv` | Między zbiorami (`train` ↔ `val`) |
|---|---:|---:|---|
| Liczba rekordów CSV (uszkodzone wiersze) | 180 (0) | 60 (0) | Łącznie 240 (podział 75% / 25%) |
| Fizyczne linie pliku (wieloliniowość) | 8625 (180/180 z `\r\n`) | 3024 (60/60 z `\r\n`) | Wymagany właściwy parser CSV |
| Rozkład etykiet `ham` / `spam` | 120 (66,7%) / 60 (33,3%) | 40 (66,7%) / 20 (33,3%) | Identyczna proporcja 2 : 1 |
| Puste wartości (`message_id`, `text`, `label`, temat, treść) | 0 | 0 | Brak braków danych |
| Nieprawidłowe / nietypowe etykiety | 0 | 0 | Wyłącznie `ham` i `spam` |
| Unikalność `message_id` | 180 / 180 | 60 / 60 | 0 wspólnych identyfikatorów |
| Dokładne duplikaty treści (także po normalizacji) | 0 | 0 | 0 dokładnych duplikatów |
| Bliskie duplikaty treści (*near-duplicates*) | 10 par (wszystkie `spam`) | 0 | **6 par (wszystkie `spam` — 30% spamu w `val`)** |
| Wspólne wątki tematyczne (bez `Re:` / `Fwd:`) | 6 wątków | 1 wątek | **5 wspólnych wątków (wszystkie `ham`)** |
| Długość tekstu w znakach (min / mediana / max) | `ham`: 185 / 779 / 20 451<br>`spam`: 290 / 1468 / 16 810 | `ham`: 173 / 1054 / 9255<br>`spam`: 534 / 1390 / 16 263 | Ponad 100-krotna różnica długości |

Wszystkie 240 identyfikatorów, przypisań do zbiorów i etykiet w obu plikach CSV jest w 100% zgodnych z `data/manifest.json` (ziarno losowania `2026`). Ponadto analiza treści potwierdziła następujące cechy:
- **Przykłady rzeczywistych wiadomości:**
  - typowe `ham` to m.in. techniczne dyskusje na listach mailingowych, np. `bac28e58c42fa5f6` z `train.csv` (*Subject: [Spambayes] can't write to CVS...* — prośba programisty o pomoc z dostępem do repozytorium) lub `00f2413a5421b49a` z `validation.csv` (*Subject: [Razor-users] Exit.status =13 ?*), a także krótkie wpisy blogowe/RSS w formacie `URL:` / `Date:`, np. `cf6049755d0fd6af` z `train.csv` (*Subject: Perl programmers are so cute*);
  - typowe `spam` to m.in. niezamówione oferty sprzedażowe, np. `c69499c749040caa` z `train.csv` (*Subject: Low Price Smokes* — oferta zakupu tanich papierosów i tytoniu) czy `6c95b20ac24fd89d` z `validation.csv` (*Subject: International calls for only 33 cents per minute...*), oraz wiadomości typu „nigeryjski szwindel”, np. `eec728149e9886d7` z `validation.csv` (*Subject: [ILUG] Seeking Your Partnership*).
- **100% wiadomości** zaczyna się od linii `Subject:`, jest wieloliniowych i w przytłaczającej większości zawiera przecinki (170/180 w `train.csv` i 60/60 w `validation.csv`).
- **Linki URL są częste w obu klasach** (`train.csv`: 93/120 w `ham` i 50/60 w `spam`; `validation.csv`: 33/40 w `ham` i 19/20 w `spam`), więc sama obecność linku nie oznacza spamu.
- **Szum techniczny jest niewielki, ale obecny:** nieliczne znaczniki HTML (3 w `train`, 3 w `val`), nieodkodowane encje HTML takie jak `&#8220;` (3 w `train`, 2 w `val`), 1 znak kontrolny w `train.csv` (`fbafb6868ba99464`), brak uszkodzeń kodowania UTF-8 (`\ufffd`), obecne znaki spoza ASCII (14 w `train`, 5 w `val`), ozdobniki ASCII (`#####`, `-----`) oraz cytaty odpowiedzi (`>`). Wszystkie 3 wiadomości w `train.csv` o udziale wielkich liter powyżej 40% należą do `spam`.
- **Wyraźne artefakty źródła danych:** format wpisów z liniami `URL:` i `Date:` występuje wyłącznie w `ham` (33/120 w `train`, 8/40 w `val`). Specyficzne tagi list dyskusyjnych w temacie są silnie skorelowane z klasami (np. `[Spambayes]` i `[SAtalk]` występują tylko w `ham`, `[ILUG-Social]` tylko w `spam`, natomiast `[ILUG]` w obu klasach — m.in. w spamie `eec728149e9886d7`). W części wiadomości pojawiają się adresy z korpusu (np. `zzzz@spamassassin.taint.org`).
- **Bliskie duplikaty między zbiorami:** np. `c69499c749040caa` z `train.csv` (*Subject: Low Price Smokes*) oraz `27bcc632c0a2fc78` z `validation.csv` (*Subject: Discount Smokes*) mają niemal identyczną treść oferty, różniąc się głównie tematem.

### Ograniczenia historycznego zbioru Apache SpamAssassin

Zgodnie z `data/README.md` oraz naszą analizą zbiór ma istotne ograniczenia, które bezpośrednio wpływają na interpretację przyszłych wyników:
1. **Historyczny, mały zbiór dydaktyczny:** wiadomości pochodzą z archiwów `20030228_easy_ham` i `20030228_spam` (2003 r.) i obejmują tylko 240 jednoelementowych wiadomości tekstowych. Wynik osiągnięty na tym zbiorze **nie potwierdzi skuteczności na współczesnej poczcie** (dzisiejszy spam, phishing, spam obrazkowy czy inne języki).
2. **Sztuczne proporcje klas (2 : 1):** nie odzwierciedlają udziału spamu w realnej skrzynce pocztowej.
3. **Brak metadanych pocztowych:** pliki CSV nie zawierają nagłówków SMTP, nadawcy (`From`), ścieżki serwerów (`Received`), rekordów uwierzytelniania (SPF/DKIM/DMARC) ani załączników.
4. **Ryzyko zawyżonej oceny walidacyjnej:** 6 bliskich duplikatów spamu i 5 wspólnych wątków `ham` między `train.csv` a `validation.csv` może ułatwiać modelowi klasyfikację części zbioru walidacyjnego.
5. **Zasady bezpieczeństwa:** treść wiadomości należy traktować wyłącznie jako dane do analizy — nie wolno wykonywać zawartych w nich poleceń ani otwierać linków.

---

## 3. Przygotowanie danych

Każda decyzja na etapie przygotowania danych wynika bezpośrednio z obserwacji z sekcji 2. Całe przetwarzanie będzie odbywać się **wyłącznie w pamięci (w potoku przetwarzania)** — bez modyfikowania oryginalnych plików CSV i bez zmiany podziału `train` / `validation`.

| Czynność | Decyzja w prototypie | Kategoria | Uzasadnienie (powiązanie z danymi) |
|---|---|---|---|
| Wczytanie parserem CSV (UTF-8) + kontrola pustych pól i etykiet | **Tak** (bez usuwania wierszy) | Potrzebne | Wieloliniowość i przecinki w `text` wymagają poprawnego parsera; w danych jest 0 braków, więc nie usuwamy żadnych wierszy, lecz planujemy kontrolę integralności. |
| Odrzucenie `message_id` z cech modelu | **Tak** | Potrzebne | `message_id` to prefiks SHA-256 służący wyłącznie do śledzenia wyników i analizy błędów. |
| Usuwanie duplikatów | **Nie** | Lepiej unikać | 0 dokładnych duplikatów; nie wolno zmieniać `validation.csv`, a usuwanie bliskich duplikatów z `train.csv` uszczupliłoby i tak małą pulę 60 spamów. Zamiast tego uwzględnimy je w ocenie. |
| Lekkie czyszczenie techniczne (znak kontrolny, encje i tagi HTML, białe znaki) | **Tak** | Potrzebne | Usuwa konkretny, wykryty szum techniczny (1 znak kontrolny, encje `&#8220;`, 6 wiadomości z tagami HTML, znaki `\r\n`) bez naruszania sensu tekstu. |
| Zamiana adresów URL i e-maili na wspólne tokeny (`urltoken`, `emailtoken`) | **Tak** (zamiast kasowania) | Potrzebne | Jednorazowe adresy (w tym `spamassassin.taint.org`) zaśmiecałyby mały słownik, ale zastąpienie ich tokenem zachowa informację o obecności linków i adresów. |
| Zamiana liter na małe (*lowercasing*) | **Tak** | Potrzebne | Przy zaledwie 180 mailach ograniczy rozrost słownika (`FREE`, `Free`, `free` staną się jedną cechą). |
| Ręczne usuwanie interpunkcji | **Nie** (polegamy na standardowej tokenizacji słownej) | Opcjonalne / ostrożnie | Zwykła interpunkcja zostanie naturalnie pominięta przy dzieleniu na słowa, a brak agresywnego wycinania pozwoli w przyszłości łatwo dodać cechy symboli (`!`, `$`). |
| Usuwanie *stop words* | **Nie na start** | Opcjonalne | Wagi IDF w TF-IDF same wyciszą słowa obecne wszędzie, a wyrazy takie jak `you`, `your`, `our`, `now` mogą odróżniać marketingowy spam od technicznego `ham`. |
| Stemming / lematyzacja | **Nie** | Lepiej unikać w prototypie | Stemming zniekształca liczne w zbiorze terminy techniczne (`Spambayes`, `RedHat`, `CVS`, `freetype`), a lematyzacja dokłada ciężkie zależności przy niepewnym zysku na małym zbiorze. |

**Zależność z kolejnym krokiem:** czyszczenie techniczne jest deterministyczne (nie uczy się niczego z danych) i przygotuje jednolity, wolny od technicznego szumu tekst dla etapu TF-IDF, ograniczając rozmiar słownika przed zamianą słów na liczby.

---

## 4. Wybrana reprezentacja tekstu i jej uzasadnienie

Algorytmy uczenia maszynowego operują na liczbach, dlatego oczyszczoną kolumnę `text` trzeba zamienić na wektory liczbowe. Rozważono trzy podejścia: **Bag of Words (BoW)**, **TF-IDF** oraz **embeddingi tekstowe**.

### Wybór do prototypu
Do lokalnego prototypu wybrano **TF-IDF na pojedynczych słowach (unigramach)** działające na całym oczyszczonym polu `text` (temat `Subject:` + treść), z **normalizacją L2** oraz odcięciem słów jednorazowych (**`min_df=2`**), dopasowywane **wyłącznie na `data/train.csv`** (jako opcjonalny wariant do porównania po zbudowaniu wersji bazowej planujemy sprawdzić dodanie dwuwyrazowych fraz `ngram_range=(1, 2)`).

### Uzasadnienie wyboru (fakty z danych vs ogólna wiedza ML)
- **Z analizy naszych danych:**
  - *Ponad 100-krotne różnice długości wiadomości (173 – 20 451 znaków):* normalizacja L2 w TF-IDF wyrówna skalę krótkich i bardzo długich maili, co rozwiązuje główną wadę surowego Bag of Words.
  - *Stałe elementy w 100% wiadomości (`Subject:` w 180/180 i 60/60):* składnik IDF automatycznie obniży wagę słów występujących w niemal każdym mailu, bez ryzykownego usuwania pełnej listy *stop words*.
  - *Mały zbiór (180 wiadomości treningowych) i wykryte artefakty korpusu:* TF-IDF jest w pełni interpretowalne — w połączeniu z modelem liniowym pozwoli sprawdzić dokładne wagi konkretnych słów i zweryfikować, czy model nie polega wyłącznie na artefaktach (np. tagach `[Spambayes]` lub `URL:`/`Date:`).
  - *Dostępność wyłącznie kolumny `text`:* w prototypie będziemy analizować temat (`Subject:`) wraz z treścią, ponieważ nadawcy, nagłówków SMTP i załączników w plikach CSV nie ma (w rzeczywistym systemie produkcyjnym nagłówki, reputacja domeny i SPF/DKIM/DMARC byłyby kluczowym uzupełnieniem tekstu).
- **Z ogólnej wiedzy o ML i wymogu lokalnego działania:**
  - TF-IDF jest lekkie, deterministyczne, działa w ułamku sekundy na CPU bez GPU i bez pobierania ciężkich modeli embeddingowych ani wysyłania treści maili do zewnętrznych API (ochrona prywatności).
  - W przeciwieństwie do wielu modeli embeddingowych nie ucina długich wiadomości (w naszym zbiorze najdłuższe mają ok. 2800 słów).

### Jakie informacje wykorzystuje, a jakie pomija?
- **Wykorzystuje:** obecność i względną częstość słów (zarówno z tematu `Subject:`, jak i treści), ich rzadkość w zbiorze treningowym (IDF) oraz znormalizowane tokeny `urltoken` i `emailtoken`.
- **Pomija:** kolejność słów i szerszą składnię, synonimy i znaczenie semantyczne (np. nie powiąże `low price` z `discount`, jeśli nie współwystępowały w `train.csv`), słowa niewidziane w treningu (OOV) oraz cechy stylistyczne utracone po `lowercase` i tokenizacji (udział WIELKICH LITER, wielokrotne wykrzykniki, ramki ASCII).

**Zależność z kolejnym krokiem:** TF-IDF wyprodukuje rzadką macierz ciągłych, znormalizowanych liczb o dużej liczbie kolumn (słów) i małej liczbie wierszy (180 maili) — to bezpośrednio determinuje wybór prostego, zregularyzowanego klasyfikatora.

---

## 5. Wybrany model oraz porównanie z alternatywą

Rozważono trzy lekkie, lokalne modele klasyfikacji binarnej dobrze współpracujące z tekstem: **Naive Bayes**, **Logistic Regression** oraz **Linear SVM**.

### Model główny: **Logistic Regression (Regresja Logistyczna z regularyzacją L2)**
Przypisuje każdemu słowu ze słownika TF-IDF wagę liczbową (dodatnią w stronę `spam`, ujemną w stronę `ham`), sumuje je i zwraca **prawdopodobieństwo przynależności do klasy `spam`** z przedziału 0–100%.

### Model alternatywny do porównania: **Naive Bayes (`MultinomialNB`, z dodatkowym testem `ComplementNB`)**
Jako główny model porównawczy przyjmujemy klasyczny tekstowy **`MultinomialNB`** (oraz dodatkowo sprawdzimy odmianę **`ComplementNB`**, stworzoną z myślą o niezrównoważonych zbiorach tekstu). Działa on na odmiennej zasadzie probabilistycznej — wylicza częstości słów osobno dla `spam` i `ham` (przy założeniu niezależności słów) i wybiera klasę o wyższym prawdopodobieństwie. Jako najniższy punkt odniesienia dodamy również **klasyfikator naiwny (*dummy baseline*)** zawsze przewidujący klasę większościową `ham` (66,7% *accuracy*, 0% *F1* dla `spam`).

### Szczegółowe porównanie modelu głównego i alternatywy w kontekście zadania

| Kryterium | Logistic Regression (model główny) | Naive Bayes — `MultinomialNB` / `ComplementNB` (alternatywa) |
|---|---|---|
| **Współpraca z TF-IDF (norma L2)** | Naturalnie zaprojektowana dla ciągłych, znormalizowanych wag TF-IDF. | Pierwotnie projektowany pod surowe zliczenia (BoW); z TF-IDF działa dobrze, zwłaszcza w wariancie `ComplementNB`. |
| **10 par bliskich duplikatów spamu w `train.csv`** | Nie zakłada niezależności słów; regularyzacja L2 ogranicza dominację powtarzających się szablonów. | Założenie niezależności słów potrafi wielokrotnie „podbijać” pewność przy powtarzających się grupach słów. |
| **Ochrona ważnych maili `ham` i proporcja klas 2 : 1** | Zwraca lepiej skalibrowane prawdopodobieństwa — umożliwi precyzyjne **strojenie progu decyzji** (np. podniesienie z 0,50 na 0,65) oraz łatwe ważenie klas (`class_weight`). | Często zwraca skrajne prawdopodobieństwa (bliskie 0 lub 1), co utrudnia precyzyjną regulację progu; klasyczny `MultinomialNB` może faworyzować częstszą klasę `ham`. |
| **Interpretowalność pod kątem artefaktów korpusu** | Jedna czytelna waga na każde słowo (`+` spam / `-` ham) ułatwi inspekcję wpływu artefaktów. | Również interpretowalny (stosunek prawdopodobieństw słów), choć wagi liniowe są łatwiejsze do łącznej analizy. |
| **Przewaga alternatywy na małym zbiorze (180 próbek)** | Wrażliwsza na dobór siły regularyzacji (`C`). | Ekstremalnie prosta, ma mniej parametrów i bywa bardzo odporna na przeuczenie przy małej liczbie przykładów — dlatego jest cennym punktem porównania. |

*(Dlaczego nie Linear SVM jako model główny? Na TF-IDF daje zbliżoną skuteczność do Regresji Logistycznej, ale nie zwraca bezpośrednio prawdopodobieństw — bez dodatkowej kalibracji trudniej sterować progiem ochrony `ham`. Pozostaje opcją rezerwową).*

**Zależność z kolejnym krokiem:** wybór pary `TF-IDF + Logistic Regression` wymusza wyraźne rozdzielenie roli transformatora (`TF-IDF`, który nie korzysta z `label`) od roli klasyfikatora (`Logistic Regression`, który uczy się z `label`) oraz wspólne zapisanie obu elementów w jednym potoku.

---

## 6. Plan uczenia modelu

Planowany proces uczenia będzie odbywać się **wyłącznie na `data/train.csv`** — plik `data/validation.csv` pozostanie całkowicie poza wszystkimi poniższymi krokami:

1. **Wczytanie `data/train.csv`:** odczyt parserem CSV (UTF-8), kontrola integralności (180 wierszy, 0 braków, poprawne etykiety) oraz rozdzielenie na teksty (`X_train_raw`), zakodowane etykiety (`y_train`: `ham → 0`, `spam → 1`) i osobno odłożone `message_id`.
2. **Przygotowanie tekstu:** przepuszczenie 180 wiadomości przez deterministyczną funkcję lekkiego czyszczenia z sekcji 3 (`X_train_clean`).
3. **Przekształcenie TF-IDF (`fit_transform`):** bez zaglądania do `y_train` wektoryzator zbuduje słownik słów (`min_df=2`), wyliczy wagi rzadkości IDF i wygeneruje znormalizowaną macierz liczbową `X_train_tfidf` o wymiarze `180 × V` (gdzie `V` to liczba słów w słowniku) oraz **dopasowany obiekt `TF-IDF Vectorizer`**.
4. **Konfiguracja i przekazanie danych do klasyfikatora:** utworzenie obiektu `Logistic Regression` z regularyzacją L2 i ustalonym ziarnem losowości (oraz równolegle, w ścieżce porównawczej, `MultinomialNB` / `ComplementNB` i `DummyClassifier`), a następnie przekazanie sparowanych danych `X_train_tfidf` oraz `y_train`.
5. **Uczenie modelu (`fit`):** optymalizacja `V` wag słów i wyrazu wolnego (*bias*) tak, aby minimalizować błąd na `y_train` przy jednoczesnym ograniczaniu zbyt dużych wag przez regularyzację L2. *(Ewentualny dobór hiperparametrów, np. siły regularyzacji `C` lub progu decyzji, będzie odbywać się wyłącznie w walidacji krzyżowej wewnątrz `train.csv` — nigdy na `validation.csv`).*
6. **Zapisanie kompletnego pakietu artefaktów (poza folderem `data/`, np. w `artifacts/`):**
   - **jeden spójny potok (`Pipeline`)** zawierający dopasowany `TF-IDF Vectorizer` (słownik + wagi IDF + ustawienia) oraz wytrenowany klasyfikator (wagi + wyraz wolny), powiązany ze wspólną funkcją czyszczenia tekstu;
   - **plik metadanych (np. `metadata.json`)** zawierający: ustalony próg decyzji (domyślnie `0,50` lub dostrojony na `train.csv`), jawne mapowanie klas (`0 → ham`, `1 → spam`), opis reguł czyszczenia i kontraktu wejścia, hiperparametry, ziarno losowości, wersje Pythona/bibliotek oraz sumę kontrolną SHA-256 użytego pliku `train.csv`.

**Dlaczego nie wystarczy zapisać samego klasyfikatora?** Wytrenowana Regresja Logistyczna nie przechowuje nazw słów, lecz tablicę wag przypisaną do numerów kolumn. Bez dokładnie tego samego `TF-IDF Vectorizer` (słownika i wag IDF z `train.csv`) oraz identycznych reguł czyszczenia nowe wiadomości miałyby inną liczbę lub kolejność kolumn (błąd programu) albo byłyby inaczej oczyszczone (ciche ignorowanie ważnych słów).

---

## 7. Plan klasyfikowania nowych wiadomości

W przeciwieństwie do etapu uczenia (gdzie TF-IDF buduje słownik przez `fit_transform`, a klasyfikator dopasowuje wagi przez `fit` na podstawie `label`), klasyfikowanie nowej wiadomości (np. *„Congratulations! You have won a free prize. Click here now!”*) będzie działać wyłącznie w trybie **przekształcania i przewidywania** (`transform` + `predict_proba`) — **bez prawdziwej etykiety `label`** (którą chcemy dopiero przewidzieć) i **bez jakiejkolwiek zmiany** słownika TF-IDF czy wag klasyfikatora:

1. **Wejście programu:** program wczyta zapisany potok modelu i metadane z etapu 6 oraz przyjmie wyłącznie surowy tekst nowej wiadomości (UTF-8).
2. **Identyczne przygotowanie tekstu:** wiadomość przejdzie przez dokładnie tę samą funkcję czyszczenia (kontrola znaków/HTML, zamiana ewentualnych URL/e-maili na tokeny, zamiana na małe litery, ujednolicenie białych znaków).
3. **Przekształcenie przez zapisany TF-IDF (`transform`):** wektoryzator podzieli tekst na słowa i wpisze wartości `TF × zapamiętane IDF` wyłącznie do kolumn słów znanych ze słownika `train.csv` (słowa spoza słownika zostaną pominięte, pozostałe kolumny otrzymają wartość `0`), po czym znormalizuje wektor normą L2. Powstanie pojedynczy wektor o wymiarze `1 × V`.
4. **Decyzja klasyfikatora:** Regresja Logistyczna pomnoży wartości wektora przez wyuczone wagi, zsumuje wynik z wyrazem wolnym i obliczy prawdopodobieństwo `P(spam)`. Następnie porówna je z zapisanym progiem decyzji (np. `0,50` lub wyższym dla ochrony `ham`).
5. **Wynik zwracany użytkownikowi:** przewidziana etykieta (`spam` albo `ham` odczytana z zapisanego mapowania klas), obliczone prawdopodobieństwo `P(spam)` oraz opcjonalnie lista słów z tej wiadomości o największym wpływie na decyzję (dla przejrzystości prototypu).

---

## 8. Plan oceny modelu

### Zasady uczciwej oceny
- **Nie będziemy oceniać modelu na `train.csv` jako głównej mierze jakości**, ponieważ na danych znanych z treningu nawet przeuczony model może osiągać blisko 100% trafności.
- **Plik `data/validation.csv` (60 wiadomości: 40 `ham`, 20 `spam`) pozostanie poza uczeniem, dopasowaniem TF-IDF i doborem hiperparametrów** — nie będziemy go wielokrotnie „podglądać” podczas strojenia, aby nie przeuczyć decyzji pod te 60 przykładów. Dopiero na etapie końcowej oceny kolumna `label` zostanie odłożona na bok, zapisany potok z etapu 6 wygeneruje 60 predykcji wyłącznie z kolumny `text`, a następnie zestawimy je (po `message_id`) z prawdziwymi etykietami.

### Schemat planowanej macierzy pomyłek i koszt błędów (klasa pozytywna = `spam`)

*(Poniższa tabela przedstawia wyłącznie schemat klasyfikacji wyników do zastosowania po przyszłym treningu — nie zawiera wyników uruchomionego modelu).*

| Prawdziwa etykieta (`label`) \ Predykcja modelu | Model przewidzi: **`spam`** | Model przewidzi: **`ham`** |
|---|---|---|
| **Prawdziwy `spam` (20 wiadomości w `val`)** | **TP (*True Positive*)** — spam prawidłowo zatrzymany w folderze Spam | **FN (*False Negative*)** — spam przepuszczony do skrzynki odbiorczej |
| **Prawdziwy `ham` (40 wiadomości w `val`)** | **FP (*False Positive*)** — prawidłowa wiadomość błędnie wrzucona do spamu | **TN (*True Negative*)** — prawidłowa wiadomość bezpiecznie w skrzynce |

Z uwagi na proporcję klas `66,7% ham / 33,3% spam` sama *accuracy* nie wystarczy — naiwny klasyfikator zawsze przewidujący `ham` osiągnie 66,7% *accuracy* przy zerowym wykrywaniu spamu (a w skrzynce z 95% `ham` aż 95% *accuracy* przy 0% wykrytego spamu). Ponadto błędy mają dla użytkownika różny koszt:
- **False Positive (`FP`):** szczególnie groźny, bo użytkownik może przegapić ważną wiadomość (z pracy, uczelni, banku) i stracić zaufanie do filtru.
- **False Negative (`FN`):** zaśmieca skrzynkę, a w przypadku oszustw lub phishingu stanowi ryzyko bezpieczeństwa.

### Planowany zestaw miar oceny
1. **Macierz pomyłek (`TP`, `TN`, `FP`, `FN`)** — podstawa raportu; przy zaledwie 60 mailach walidacyjnych dokładna liczba błędów będzie najbardziej czytelna (1 pomyłka w `spam` zmienia *recall* o 5 p.p.).
2. **Precision dla `spam`** — główne kryterium bezpieczeństwa użytkownika (minimalizacja `FP`).
3. **Recall dla `spam`** — główne kryterium skuteczności filtra (kontrola, by wysoka precyzja nie wynikała z przepuszczania większości spamu).
4. **F1-score dla `spam`** — główna pojedyncza miara porównawcza łącząca *Precision* i *Recall* (służąca do uczciwego zestawienia `Logistic Regression`, `Naive Bayes` i `DummyClassifier`).
5. **Accuracy** — wyłącznie pomocniczo, zawsze zestawiana z punktem odniesienia klasy większościowej (`66,7%`).

### Jakościowa analiza błędów i kontrola ograniczeń zbioru
Poza metrykami ogólnymi plan przewiduje:
- wypisanie po `message_id` wszystkich wiadomości `FP` i `FN` oraz słów o największym wpływie na ich klasyfikację;
- **dodatkową kontrolę podgrup w `validation.csv`** (bez zmieniania oficjalnej oceny na wszystkich 60 wierszach): osobne sprawdzenie skuteczności na 6 spamach będących bliskimi duplikatami z `train.csv` względem **14 naprawdę nowych spamów**, aby ocenić realną zdolność do uogólniania.

---

## 9. Niewiadome i możliwe korekty

### Podsumowanie: fakty potwierdzone vs założenia ML
- **Potwierdzone przez dane i README:** liczebność 180/60, proporcja 2 : 1, brak pustych pól i błędnych etykiet, 0 dokładnych duplikatów, 6 bliskich duplikatów spamu i 5 wspólnych wątków `ham` między `train` i `val`, 10 par podobnego spamu w `train`, 100-krotna różnica długości, częste URL-e w obu klasach, artefakty archiwum z 2003 r. oraz brak nagłówków SMTP.
- **Założenia z ogólnej wiedzy ML (jeszcze niezweryfikowane empirycznie):** wystarczalność unigramów TF-IDF (`min_df=2`, bez *stop words* i bez lematyzacji), przewaga Regresji Logistycznej nad Naive Bayes, wystarczalność regularyzacji L2 przeciw przeuczeniu na 180 przykładach oraz możliwość bezpiecznego dobrania progu decyzji przy proporcji 2 : 1.

### Niewiadome do sprawdzenia w przyszłej implementacji i planowane korekty

Wszystkie ewentualne korekty będą najpierw weryfikowane w **walidacji krzyżowej na `train.csv`** (z uwzględnieniem grupowania podobnych wiadomości — *GroupKFold*), bez nadmiernego dopasowywania się do 60 wiadomości z `validation.csv`. Nie każdy słabszy wynik oznacza konieczność wymiany modelu — w pierwszej kolejności sprawdzimy błędy po `message_id`, próg decyzji, regularyzację i przygotowanie tekstu.

| Niewiadoma / potencjalny problem | Jak to sprawdzimy? | Co skłoni nas do korekty i jakie działanie podejmiemy? |
|---|---|---|
| **Wpływ artefaktów korpusu na decyzje modelu** | Inspekcja 15–20 słów o najwyższych dodatnich i ujemnych wagach w Regresji Logistycznej + test wrażliwości w kroswalidacji na `train.csv` z zamaskowanymi tagami `[...]`. | Jeśli czołowe wagi zdominują wyłącznie artefakty archiwum (`[Spambayes]`, `[ILUG-Social]`, `URL:`/`Date:`) → udokumentowanie skali tego wpływu jako ograniczenia zbioru oraz sprawdzenie wariantu diagnostycznego z usuniętymi tagami. |
| **Słabe wykrywanie spamu** (niski `Recall`, dużo `FN`) | Analiza `FN` po `message_id` + krzywa *Precision–Recall* w kroswalidacji na `train.csv`. | Przy `Recall < 75%`: włączenie `class_weight='balanced'` lub lekkie obniżenie progu decyzji; przetestowanie `min_df=1` i bigramów `ngram_range=(1, 2)` w TF-IDF lub osłabienie zbyt mocnej regularyzacji (`C`). |
| **Zbyt wiele fałszywych alarmów** (niska `Precision`, dużo `FP`) | Analiza `FP` po `message_id` i inspekcja najwyższych wag słów (np. wpływ terminów technicznych lub słowa `spam` na listach `[Spambayes]`). | Przy `FP ≥ 3`: **podniesienie progu decyzji** Regresji Logistycznej (np. na `0,65–0,75`), wzmocnienie regularyzacji L2 (mniejsze `C`) lub dodanie bigramów (dla rozróżnienia kontekstu, np. `"free software"` vs `"free membership"`). |
| **Wpływ niezrównoważenia klas (2 : 1 lub silniejszego w nowych danych)** | Porównanie domyślnych wag vs `class_weight='balanced'` i różnych progów w stratyfikowanej kroswalidacji. | Jeśli model będzie faworyzować `ham`: ważenie klas, strojenie progu lub wykorzystanie `ComplementNB` przy stałej ocenie przez `Precision`/`Recall`/`F1` zamiast samej `accuracy`. |
| **Nietypowe znaki, URL-e i sygnały stylu** | Przegląd tokenów słownika oraz błędów `FN` zawierających WIELKIE LITERY, symbole (`!`, `$`) lub resztki formatowania. | Jeśli szum obniży jakość: rozszerzenie czyszczenia (normalizacja Unicode, skracanie ramek ASCII, tokeny `urltoken`/`emailtoken` + liczba linków). Jeśli zgubimy sygnał stylu: dodanie n-gramów znakowych (`char_wb`) lub cech liczbowych (udział wielkich liter, liczba `!`). |
| **Niezadowalające wyniki TF-IDF** | Porównanie wariantów TF-IDF w kroswalidacji na `train.csv`: `(1,1)` vs `(1,2)`, `sublinear_tf=True`, `char_wb`, osobne TF-IDF dla `Subject:` i treści. | Jeśli dostrojone warianty TF-IDF i cechy stylu nadal nie wystarczą (np. przez wysoki odsetek synonimów i nieznanych słów OOV): rozważenie lokalnych embeddingów tekstowych jako wejścia dla Regresji Logistycznej. |
| **Przeuczenie (*overfitting*) i wpływ 6 bliskich duplikatów** | Porównanie wyników `train` vs kroswalidacja grupowa (*GroupKFold*) oraz osobny wgląd w 14 nowych spamów w `validation.csv`. | Duża luka `train` vs walidacja lub słaby wynik na 14 nowych spamach (np. `Recall < 60%`): silniejsza regularyzacja (mniejsze `C`), bardziej zwarty słownik (`min_df`, `max_features`) lub zmiana modelu na prostszy `Naive Bayes`. |
| **Model główny działa gorzej od alternatywy** | Porównanie `Logistic Regression` i `Naive Bayes` w kroswalidacji na `train.csv` (stabilność między foldami, `F1`, liczba `FP`) po dostrojeniu `C` i progu. | Jeśli po sprawdzeniu regularyzacji i progu `MultinomialNB` / `ComplementNB` konsekwentnie wygra stabilnością, `F1` i mniejszą liczbą `FP` → przyjęcie `Naive Bayes` jako modelu głównego (lub test rezerwowego `Linear SVM`). |
| **Nowe rodzaje spamu spoza korpusu z 2003 r.** | Pomiar odsetka nieznanych słów (OOV) w walidacji oraz dodatkowy test jakościowy na osobnym, małym zestawie współczesnych przykładów (poza `data/`). | Słaba skuteczność na współczesnych wzorcach potwierdzi granice historycznego zbioru — w realnym wdrożeniu konieczne byłyby świeższe dane, n-gramy znakowe lub embeddingi oraz metadane nagłówkowe (nadawca, SPF/DKIM/DMARC). |

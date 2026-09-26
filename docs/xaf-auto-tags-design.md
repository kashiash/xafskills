# Projekt automatycznego nadawania tagów w XAF

**Rekomendacja:** reguły powinny dodawać i zdejmować własne przypisania tagów. Rekord może nadal pokazywać jeden tag, nawet gdy ten sam tag pochodzi z kilku źródeł. Automat usuwa tylko źródło, którym zarządza.

Ten opis określa wspólne zachowanie automatycznego tagowania w XAF z EF Core i XPO. Mapowanie przypisań dopasuj do używanego ORM oraz reguł aplikacji.

## Cel

Automat oznacza rekordy według kryteriów XAF. Powtarza działanie w tle, nie tworzy duplikatów i nie usuwa tagu dodanego ręcznie ani przez inną regułę.

Przykład: reguła „Zaległa płatność” dodaje tag do klienta, który ma przeterminowaną należność. Gdy klient spłaci należność, reguła zdejmuje swoje przypisanie. Tag nadal będzie widoczny, jeśli użytkownik dodał go ręcznie albo inna reguła wciąż go wymaga.

## Model danych

### `Tag`

Katalog przechowuje nazwę i opis tagu. Według potrzeb może też przechowywać stan archiwalny, zakres typu obiektu, widoczność globalną, ostrzeżenie lub kolor plakietki. Kolor i ostrzeżenie nie sterują przypisaniem.

### `AutoTagRule`

Każda reguła określa:

- nazwę;
- typ obiektu, który implementuje kontrakt tagowalny;
- tag zgodny z zakresem tego typu;
- kryterium dodania;
- opcjonalne, niezależne kryterium usunięcia;
- stan aktywności.

Zapisuj nazwę typu w trwałej postaci i rozwiązuj ją przez metadane XAF. Waliduj typ oraz kryteria przy zapisie reguły. Sprawdź je ponownie podczas wykonania, bo typ lub właściwość mogły zmienić się po migracji modelu.

Kryterium usunięcia jest niezależne od kryterium dodania. Może więc wyrażać inną decyzję biznesową. Gdy go brak, reguła działa addytywnie: tag pozostaje przypisany, aż usunie go użytkownik lub inny jawny proces.

### Przypisanie i jego źródła

Modeluj widoczne przypisanie obiekt–tag oddzielnie od powodów, dla których ono istnieje:

- przypisanie łączy jeden rekord z jednym tagiem;
- źródło ręczne oznacza, że przypisał je użytkownik;
- źródło regułowe wskazuje `AutoTagRule`, która wymaga tego przypisania;
- opcjonalne źródło zdarzeniowe wskazuje operację biznesową, która dodała tag.

Jeśli ręczna akcja lub reguła dodaje tag, upewnij się, że istnieje przypisanie. Następnie zapisz jej źródło. Dla tego samego rekordu i tagu zachowaj jedno widoczne przypisanie oraz osobne informacje o źródłach. Unikalność przypisania zabezpiecz w bazie i sprawdź w kodzie.

W EF Core zastosuj projektowy wzorzec jawnych encji łączących. Przy relacjach przez wiele typów użyj relacji źródeł zgodnej z ich kluczami i zasadami migracji. Nie twórz jednej polimorficznej tabeli z kluczem obiektu bez prawdziwego klucza obcego, jeśli baza nie potrafi wymusić integralności.

W XPO użyj trwałej encji przypisania, gdy potrzebujesz metadanych źródeł. Podepnij ją relacjami do rekordu i tagu. Sama niejawna relacja wiele-do-wielu nie przechowa wielu źródeł przypisania.

## Przebieg reguły

Worker działa w granicach jednego tenanta i tworzy ObjectSpace dla jego bazy. Przetwarza aktywne reguły po kolei. Jeśli środowisko uruchamia wiele instancji workera, zabezpiecz je przed równoległym przetwarzaniem tej samej reguły i tenanta.

Dla każdej reguły:

1. Rozwiąż typ i potwierdź, że implementuje interfejs tagowalny.
2. Sprawdź, że reguła, jej tag i połączenie należą do bieżącego tenanta oraz zakres tagu pasuje do typu.
3. Odczytaj rekordy spełniające kryterium dodania. Dodaj źródło regułowe idempotentnie.
4. Jeśli istnieje kryterium usunięcia, odczytaj pasujące rekordy i usuń tylko źródło tej reguły.
5. Dla rekordów, które nie mają już żadnego źródła, usuń widoczne przypisanie.
6. Zatwierdź zmiany przypisania i źródeł razem.

Gdy rekord spełnia kryterium dodania i usunięcia w tym samym przebiegu, usunięcie tej reguły ma pierwszeństwo. Kryteria powinny jednak wyrażać rozłączne stany, jeśli reguła ma działać w sposób czytelny dla administratora.

Powtórne wykonanie tej samej reguły nie zmienia wyniku. Unikalne ograniczenia chronią przed wyścigiem między dwoma workerami; blokada albo dzierżawa ogranicza niepotrzebną pracę równoległą.

## Cykl życia reguły

- Wyłączenie reguły zatrzymuje dalsze przetwarzanie. Nie usuwa przypisań, które reguła już utworzyła.
- Ponowne włączenie kontynuuje tę samą regułę i korzysta z jej wcześniejszych źródeł.
- Usuwanie reguły z przypisaniami wymaga jawnej decyzji: zostawić tagi jako trwałe czy usunąć źródła tej reguły. Nie kasuj historii źródeł przez kaskadowy delete bez takiej decyzji.
- Zmiana tagu albo typu w regule powinna rozliczyć stare źródła i utworzyć nowe zgodnie z określoną operacją aktualizacji.

## Bezpieczeństwo i odporność

- Worker używa tożsamości serwisowej z minimalnymi uprawnieniami do odczytu reguł i aktualizacji przypisań.
- XAF security i granica tenanta obowiązują też w tle. Sam filtr po identyfikatorze tenanta nie zastępuje zabezpieczonego dostępu do bazy.
- Błędna reguła nie zatrzymuje innych reguł. Zapisz identyfikator reguły, tenant, etap i błąd bez logowania sekretów ani pełnych danych wrażliwych rekordu.
- Ogranicz ilość wczytywanych rekordów. Duże reguły przetwarzaj stronicami albo zapytaniami, które baza potrafi wykonać.
- Nie pobieraj rekordów do pamięci po to, by dopiero tam ocenić kryterium, jeśli CriteriaOperator da się wykonać przez provider.
- Weryfikuj kryteria przed aktywacją. Kryteria używane w zapytaniach muszą odwoływać się do właściwości trwałych i tłumaczonych przez provider.

## Testy akceptacyjne

1. Rekord spełnia kryterium dodania: tag pojawia się raz, a źródło reguły zostaje zapisane.
2. Ponowne wykonanie: liczba przypisań i źródeł pozostaje taka sama.
3. Kryterium usunięcia pasuje: znika źródło tej reguły.
4. Źródło ręczne albo druga reguła nadal istnieje: widoczne przypisanie pozostaje.
5. Nie ma innych źródeł: widoczne przypisanie znika.
6. Nieaktywna reguła nie dodaje ani nie usuwa źródeł.
7. Błędna reguła nie blokuje pozostałych reguł.
8. Ten sam worker obsługuje dwóch tenantów bez mieszania ich reguł i przypisań.
9. Równoległy przebieg nie tworzy duplikatów.
10. Niepasujący typ, tag archiwalny albo błędne kryterium nie powodują niejawnego przypisania.

## Zakres pierwszej wersji

Pierwsza wersja może działać wyłącznie addytywnie, jeśli produkt nie potrzebuje zdejmowania tagów. W takim wariancie opisz trwałość tagu i nie sugeruj, że automat sam go usunie. Jeśli potrzebne są odwracalne reguły, wdrażaj razem z nimi pochodzenie przypisań — bez niego usuwanie nie jest bezpieczne.

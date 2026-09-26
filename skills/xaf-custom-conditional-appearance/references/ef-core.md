# Reguły wyglądu z EF Core

Użyj tych wskazówek dopiero po potwierdzeniu, że moduł XAF korzysta z EF Core. Szczegóły mapowania dopasuj do konwencji encji i aktualizacji schematu w aplikacji docelowej.

## Encja reguły

Dla nowej implementacji użyj encji i tabeli `AdditionalAppearanceRule` oraz wspólnych nazw pól opisanych w głównym skillu. Zastosuj bazową klasę EF używaną przez aplikację, mapowane właściwości `virtual`, rejestrację w `DbContext` i istniejący proces migracji lub aktualizacji schematu.

Dane wyglądu — na przykład kolory, styl czcionki i widoczność — dodawaj tylko wtedy, gdy funkcja ich potrzebuje. Przechowuj wartości w formie odpowiedniej dla bazy. Właściwości UI, takie jak `TypeInfo` lub `Color`, mogą być niepersistowane i wyliczane z zapisanych wartości.

## ObjectSpace i pamięć podręczna

- Pobieraj encje przez zabezpieczony `IObjectSpace` i zachowuj izolację tenantów.
- Nie przechowuj encji EF poza czasem życia `ObjectSpace`.
- Jeśli aplikacja potrzebuje pamięci podręcznej, mapuj rekordy na niemutowalne snapshoty.
- Publikuj zmiany pamięci podręcznej dopiero po pomyślnym zatwierdzeniu transakcji.
- Unieważniaj zarówno wspólną pamięć podręczną, jak i reguły wybrane przez aktywne widoki.
- Przy wielu instancjach aplikacji określ, jak zatwierdzona zmiana dotrze do każdej instancji.

## Typ i widok

Zapisuj pełną, stabilną nazwę typu i rozwiązuj ją przez metadane XAF. Porównuj typy dokładnie. Jeśli reguła ma obejmować klasy pochodne, zbierz ich nazwy jawnie. Nie stosuj dopasowania `StartsWith` ani nazw klas po usuwaniu prefiksu proxy.

Ustal znaczenie pustego `ViewId`. Może oznaczać wszystkie widoki wybranego typu albo brak dopasowania. Utrzymuj tę samą zasadę w edytorze, selektorze i kontrolerze.

## Renderowanie platformowe

Standardowe reguły dodawaj przez `AppearanceController.CollectAppearanceRules` w module współdzielonym. Dodatkowe renderowanie, którego nie obsługuje standardowa funkcja Conditional Appearance, umieść w kontrolerze właściwym dla platformy i edytora.

## Co sprawdzić

- Rejestrację encji, migrację i uprawnienia XAF.
- Poprawność kryteriów względem wybranego typu.
- Zakres typu, widoku i tenanta.
- Zachowanie po zatwierdzeniu i wycofaniu transakcji.
- Odświeżenie otwartych widoków oraz pozostałych instancji aplikacji.
- Renderowanie każdej wspieranej platformy.

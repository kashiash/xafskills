# Porównanie rozszerzonych reguł wyglądu w XAF

## Zakres

Porównano śledzone pliki źródłowe w Fleetman, DataDrive, HIS i PathQ. Analiza dotyczy reguł, które można ustawiać podczas działania aplikacji, oraz sposobu dodawania ich do standardowego mechanizmu XAF `AppearanceController`.

## Wnioski

DataDrive ma najbardziej kompletny wzorzec jako punkt wyjścia: trwały rekord reguły, bezpieczny snapshot, dobór według typu i widoku, pamięć podręczną z kluczem tenanta i publikację zmian po zatwierdzeniu transakcji. HIS i PathQ potwierdzają wspólny model XAF (`CollectAppearanceRules` + `IAppearanceRuleProperties`), ale ich statyczne magazyny zachowują żywe obiekty z `ObjectSpace`. Fleetman pokazuje głównie reguły zapisane w atrybutach oraz mały adapter `AppearanceModel`; nie jest to kompletna implementacja reguł edytowanych w bazie.

## Projekty

| Projekt | Co znaleziono | Mocne strony | Ograniczenia |
|---|---|---|---|
| **Fleetman** | `[Appearance]` przy encjach; `Fleetman.Module/Utils/AppearanceModel.cs` implementuje `IAppearanceRuleProperties`; przykładowy wyspecjalizowany kontroler siatki: `Fleetman.Blazor.Server/XafControllers/OperationalListAppearanceController.cs`. | Atrybuty są proste i dobre dla stałych reguł biznesowych. `AppearanceModel` może opisać regułę wygenerowaną w kodzie. | Nie znaleziono wspólnego, trwałego edytora dodatkowych reguł. Domena korzysta z XPO, więc model EF z DataDrive nie nadaje się do skopiowania bez adaptacji. Kontroler siatki obsługuje konkretne widoki i prezentację, a nie uniwersalny zapis reguł. |
| **DataDrive** | `AdditionalAppearanceRule`; `AdditionalAppearanceRuleViewController`; `AdditionalAppearanceRuleSnapshot`; `AdditionalAppearanceRuleStorage`; `AdditionalAppearanceRuleSelector`; `AppearanceTypeMatching`; dodatkowe rysowanie obramowania w `AdditionalAppearanceRuleGridController`. | Reguły można edytować w UI. Widok dostaje niemutowalne snapshoty, a nie encje z zamkniętego `ObjectSpace`. Magazyn rozdziela tenantów. Zmiany trafiają do wspólnego storage po `Committed`. Dobór uwzględnia typ bazowy, `ViewId`, wyłączenie i priorytet. | Cache jest lokalny dla procesu. Przy wielu instancjach zmiana zapisana w jednej instancji nie unieważnia automatycznie cache pozostałych. Kontroler widoku przechowuje też własną selekcję `currentRules`; sama zmiana storage nie gwarantuje odświeżenia już otwartych widoków. Kryterium pozostaje tekstem zależnym od modelu, więc potrzebne są walidacja i logowanie starych kryteriów. Funkcja obramowania jest dodatkową, platformową warstwą Blazor. |
| **HIS** | Trwała encja `CustomApperance` implementuje `IAppearanceRuleProperties`; `CustomApperanceViewControler` dodaje reguły do zdarzenia; `CustomApperanceStorage` przechowuje je statycznie. Osobny kontroler generuje reguły dla widoku non-persistent. | Pełny przepływ edycji w UI, opcjonalny `ViewId`, kryterium i wybór pól. Reguły mogą też być tworzone z kodu dla obiektów non-persistent. | `OnSaving` aktualizuje pamięć procesu przed potwierdzeniem transakcji; wycofanie zapisu może rozjechać bazę i cache. `Type.GetType` wymaga zgodnej nazwy typu. Storage zawiera żywe encje i jest globalny dla procesu. |
| **PathQ** | Kopia wzorca `CustomApperance` z HIS, z tym samym statycznym storage i kontrolerem zdarzenia. Klasa nadal implementuje `IAppearanceRuleProperties`. | Zachowuje kryterium, wybór pól, `ViewId` i priorytet. Kontroler pomija `NonPersistentObjectSpace`, co oddziela go od generatora dynamicznych reguł HIS. | Ma te same problemy `OnSaving`, `Type.GetType` i żywych obiektów w pamięci. W porównaniu z HIS wersja PathQ nie ma `BorderColor` ani `CloneFrom`; ma też literówkę w caption `Syl czcionki`. |

## Fakty wspólne

- XAF udostępnia punkt rozszerzenia `AppearanceController.CollectAppearanceRules`.
- Regułę da się przekazać jako obiekt implementujący `IAppearanceRuleProperties`.
- Wspólne pola reguły to typ, kryterium, elementy docelowe, kontekst, priorytet, wygląd i opcjonalny identyfikator widoku.
- Atrybut `[Appearance]` i reguły z bazy mogą działać równolegle. Atrybuty pasują do niezmiennej logiki; baza służy do konfiguracji zmienianej przez uprawnionych użytkowników.
- HIS i PathQ mają niemal tę samą edytowalną encję i statyczny storage. Porównanie plików pokazuje, że PathQ skopiował wzorzec z HIS, ale usunął obsługę `BorderColor` i `CloneFrom`, dodał pominięcie `NonPersistentObjectSpace` oraz wprowadził literówkę w caption stylu czcionki.

## Rekomendacja

Użyć rozwiązania DataDrive jako bazy wzorca, ale nie przenosić go kopiuj-wklej. Zachować jego granice danych, selektor i snapshot; uzupełnić obsługę zmian między instancjami i jawne raportowanie reguł, których nie da się zastosować.

Wspólny rdzeń powinien zawierać:

1. Trwałą definicję reguły dobraną do ORM projektu: EF Core w DataDrive/HIS/PathQ, XPO w Fleetman.
2. Adapter lub niemutowalny snapshot `IAppearanceRuleProperties`, bez przechowywania encji z ObjectSpace w globalnej pamięci.
3. Jeden selektor reguł oparty na tenant scope, hierarchii typów, `ViewId`, statusie aktywności i priorytecie.
4. Kontroler podpinający i odpinający `CollectAppearanceRules` w prawidłowym cyklu życia widoku oraz odświeżający cache XAF.
5. Walidację kryterium i pól docelowych przy zapisie, a także bezpieczną obsługę starych, błędnych reguł przy odczycie.
6. Aktualizację cache dopiero po udanym commicie. Dla wdrożenia wieloinstancyjnego: sygnał unieważniający pozostałe procesy albo cache z wersją/krótkim TTL.
7. Osobną, opcjonalną warstwę platformową dla cech wykraczających poza Conditional Appearance, np. obramowania w DxGrid.

## Co warto przejąć i czego unikać

**Przejąć z DataDrive:** snapshoty; klucz cache per tenant; dopasowanie typu także do klas bazowych z ograniczeniem przed `BaseObject`; `ViewId`; status wyłączenia; kolejność po priorytecie; commit-hook wykonywany po zatwierdzeniu.

**Przejąć z PathQ/HIS:** prosty formularz reguły, kryterium i wybór pól przez `ICheckedListBoxItemsProvider`; obsługę reguł dla non-persistent view traktować jako wariant jawnie wspierany.

**Nie kopiować:** statycznej listy żywych encji z PathQ/HIS; publikowania zmiany w `OnSaving` przed commitem; polegania na `Type.GetType` lub `Type.Name` jako jedynym sposobie rozpoznania typu; cichego pomijania błędnych kryteriów bez logu.

## Granice wspólnego rozwiązania

- Nie próbować tworzyć jednego modelu trwałego dla EF Core i XPO. Wspólny ma być kontrakt działania, selektor i cykl życia; mapowanie encji pozostaje zgodne z ORM aplikacji.
- Nie zastępować wszystkich `[Appearance]` rekordami w bazie. Reguły wynikające z twardej logiki procesu zostają w kodzie.
- Nie zakładać, że nazwa tenanta i jego izolacja są takie same w czterech aplikacjach. W każdym projekcie trzeba sprawdzić rzeczywistą konfigurację.
- Nie łączyć Conditional Appearance z rendererami siatki w jeden kontroler. Renderer platformowy powinien konsumować ten sam zweryfikowany snapshot, jeśli potrzebuje niestandardowej prezentacji.

## Źródła w repozytoriach

- DataDrive: `CS/DataDrive.Module/BusinessObjects/AdditionalAppearanceRule.cs`; `CS/DataDrive.Module/Features/Appearance/*`; `CS/DataDrive.Blazor.Server/Controllers/AdditionalAppearanceRuleGridController.cs`; testy `CS/DataDrive.Module.Tests/Features/Appearance/*`.
- Fleetman: `Fleetman.Module/Utils/AppearanceModel.cs`; `Fleetman.Module/BusinessObjects/**` (atrybuty); `Fleetman.Blazor.Server/XafControllers/OperationalListAppearanceController.cs`.
- HIS: `HIS.Module/BusinessObjects/Helpers/CustomApperance.cs`; `HIS.Module/Controllers/HelperControllers/CustomApperanceViewControler.cs`; `HIS.Module/Storages/CustomApperanceStorage.cs`; `HIS.Module/Controllers/RegistrationFlow/RegistrationTimeSlotNPListViewController.cs`; `HIS.Module/BusinessObjects/NonPersistent/AppearanceModel.cs`.
- PathQ: `HIS.Module/BusinessObjects/Helpers/CustomApperance.cs`; `HIS.Module/Controllers/HelperControllers/CustomApperanceViewControler.cs`; `HIS.Module/Storages/CustomApperanceStorage.cs`.

Analiza obejmuje kod obecny w checkoutach w dniu 2026-09-25. Nie wykonywano buildów ani testów projektów; jest to porównanie statyczne.

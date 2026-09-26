# Tagi w aplikacjach XAF: Fleetman, DataDrive, HIS i PathQ

**Wniosek:** najlepszy wspólny kierunek to katalog tagów, jawne przypisania dopasowane do ORM, ręczne akcje działające w zabezpieczonym `ObjectSpace` oraz osobny, idempotentny mechanizm reguł. Trzeba też zapisywać, które przypisania utworzył automat, jeśli reguły mają tagi zdejmować.

Analiza opiera się na kodzie dostępnym 26 września 2026 r. Nie uruchamiałem aplikacji ani testów. Rozróżniam trwałe tagi `Tag` od zwykłych pól tekstowych, które tylko renderują się jako plakietki.

## Podsumowanie

| Projekt | ORM | Ręczne tagowanie | Tagi automatyczne | Trwałe przypisanie | Ważne ograniczenie |
|---|---|---|---|---|---|
| Fleetman | XPO | Dodaj, filtruj i usuń z listy; dodatkowo `DxTagBox` w edytorze. | Tak. Reguły mają kryterium dodania i usunięcia; jest też osobny handler zdarzeniowy. | Relacje XPO `[Association]`. | Ostrzeżenie jest kopiowane do pola obiektu. Automat nie zapisuje źródła przypisania. |
| DataDrive | EF Core | Masowe dodawanie, filtrowanie i usuwanie tagów. | Tak. Reguły nakłada cykliczny worker dla tenantów. | Jawne encje łączące dla każdego typu. | Obecny automat tylko dodaje. `Markers` duplikuje tekst ostrzeżeń. |
| HIS | EF Core | Masowe dodawanie, filtrowanie i usuwanie tagów. | Nie znaleziono silnika reguł tagowania. | Konfiguracja jawna dla `Patient`; interfejs i kontroler umożliwiają dalsze użycie. | W aktualnym kodzie `ITag` implementuje tylko `Patient`. |
| PathQ | EF Core | Masowe dodawanie, filtrowanie i usuwanie tagów. | Nie znaleziono silnika reguł tagowania. | Kolekcje wiele-do-wielu konfigurowane głównie przez konwencje EF. | Pięć typów implementuje `ITag`; wybór tagu nie odrzuca archiwalnych. |

Sama obecność klasy `Tag` nie oznacza, że użytkownik może tagować wszystkie encje. Sprawdź implementacje interfejsu, relacje w modelu, widoczność akcji i uprawnienia.

## Fleetman: XPO i reguły automatyczne

`Fleetman.Module/BusinessObjects/Common/Tags/Tag.cs` definiuje `Tag` jako `BaseObject`. Tag ma nazwę, `Global`, `TagType`, `Ostrzezenie` oraz typ obiektu. Model deklaruje osobne `[Association]` dla faktur, zleceń, najmów, zadań, osób, kontrahentów i pojazdów.

Te obiekty implementują `ITags`, który udostępnia `XPCollection<Tagi>` i tekstowe `Ostrzezenie`. `ITagsListViewController` ma akcje Dodaj, Filtruj i Usuń. Popup ogranicza tagi do typu bieżącego obiektu, jego bezpośredniego typu bazowego lub tagów globalnych. `TagBoxPropertyEditor` używa `DxTagBox` do edycji kolekcji.

`AutomatyczneTagi` przechowuje typ, tag, kryterium dodania, opcjonalne kryterium usunięcia i stan aktywności. `AddTagWorker` wykonuje reguły co 15 minut, między 06:00 a 24:00. Osobny handler może przypisać tag w konkretnej operacji biznesowej.

**Mocne strony:** obsługuje trzy sposoby nadawania tagu — użytkownik, reguła cykliczna i zdarzenie. Reguła może zdejmować tag po ustaniu warunku.

**Ryzyka:** przypisanie nie przechowuje reguły ani źródła, które je utworzyło. Kryterium usuwania może więc usunąć tag dodany ręcznie albo przez inną regułę. Ostrzeżenie jest dodatkowo kopiowane do tekstowego pola. Dodawanie i usuwanie elementów tej samej treści może skasować ostrzeżenie nadal potrzebne innemu tagowi. W kodzie widoczne jest też odświeżanie widoku po akcji tagowania; nowy projekt powinien używać właściwego ObjectSpace i odświeżania zgodnie z własnym cyklem życia XAF.

Szczegółowy projekt odwracalnych reguł, pochodzenia przypisań i ich cyklu życia jest w [opisie automatycznego tagowania](xaf-auto-tags-design.md).

## DataDrive: EF Core z jawnymi encjami łączącymi

`DataDrive.Module/BusinessObjects/Tags/Tag.cs` przechowuje nazwę, opis, ostrzeżenie, `ColorString`, zakres typu, `Global` i `Archival`. Typ obiektu jest zapisany jako `ObjectTypeName`, a rozwiązywany przez `XafTypesInfo`.

Pięć typów biznesowych implementuje `ITaggable`: `Customer`, `Vehicle`, `FinancialDocument`, `EmployeeTask` i `Party`. `ITaggable` zawiera relację `Tags` oraz tekst `Markers`. W `DataDriveDbContext` każda relacja ma jawną encję EF, np. `CustomerTag` lub `VehicleTag`. Osobna relacja `DiscountRuleTag` łączy regułę rabatową z tagami dostawców; sama nie oznacza, że `DiscountRule` jest obiektem tagowalnym.

`TaggableListViewController` dodaje i usuwa tagi na zaznaczonych obiektach, a filtrowanie ustawia kryterium dla tagu wybranego w popupie. Lista wyboru pomija tagi archiwalne i uwzględnia tagi globalne lub dokładny typ obiektu.

`AutoTagRule` ma typ, kryterium, tag i flagę aktywności. `AutoTagApplier` sprawdza typ i kryterium, pomija tag już obecny, po czym dodaje nowy. `AutoTagHostedService` uruchamia go per tenant co 15 minut.

**Mocne strony:** jawny model EF działa z konwencjami projektu, przypisania są idempotentne, typ tagowanego obiektu jest sprawdzany, a automat działa w granicach tenantów.

**Ryzyka:** reguła nie ma kryterium usunięcia. Nie zapisuje się też, która reguła utworzyła przypisanie. Jeśli wymagane będzie zdejmowanie tagów, obecny model nie odróżnia przypisań ręcznych od automatycznych. `Markers` przechowuje kopię ostrzeżeń oddzielnie od relacji. Usunięcie jednego tagu może usunąć taki sam tekst, którego potrzebuje drugi. `ColorString` istnieje w encji, lecz w przejrzanym kodzie serwera i modułu nie znalazłem miejsca, które go renderuje; jego użycie w UI wymaga osobnego potwierdzenia.

Starszy opis `DataDrive/CS/docs/system-tagow-porownanie.md` wymienia mniej encji tagowalnych niż obecny kod. Przy wdrażaniu zmian traktuj bieżący model i `OnModelCreating` jako źródło prawdy.

## HIS i PathQ: podobny interfejs, różny zasięg

Oba projekty mają `Tag` z nazwą, opisem, `Archival`, `Global` oraz `ObjectTypeFullName` i `ObjectTypeName`. `ITagController` oferuje akcje do filtrowania, dodawania i usuwania tagów. Filtr może dopasować którykolwiek z wybranych tagów. `PrepareTags` ogranicza listę do typu encji, bezpośredniego typu bazowego lub tagów globalnych.

W tych selektorach nie ma warunku `Archival = false`. Archiwalne tagi mogą więc nadal pojawić się na liście wyboru. To różni się od kontrolera DataDrive.

### HIS

W aktualnym kodzie XAF `Patient` jest jedyną encją implementującą `ITag`. `HISDbContext` jawnie mapuje relację `Patient.Tags` ↔ `Tag.Patients`. Import `ImportOldUkrainianTag` przenosi starsze dane, ale nie jest silnikiem reguł automatycznego przypisywania.

### PathQ

Pięć encji implementuje `ITag`: `Patient`, `Slide`, `BoxBlock`, `PathomorphologicalExamination` i `OrderPathomorphologyExamination`. Kolekcje używają `ObservableCollection<Tag>`. Kod modelu polega głównie na konwencjach EF dla relacji tych kolekcji.

### Plakietki tekstowe to inny mechanizm

`TagStringPropertyEditor` w HIS i PathQ edytuje `string`, dzieli jego treść po przecinku i wyświetla fragmenty jako plakietki. Nie czyta relacji `Patient.Tags` ani innej relacji `Tag`. Nazwa edytora może sugerować powiązanie, którego tu nie ma.

W `Tag.ObjectType` HIS i PathQ wywołują `Type.GetType` dla zapisanego `FullName`. To może nie rozwiązać typu z innego assembly. DataDrive korzysta z `XafTypesInfo`; w nowym kodzie używaj mechanizmu rozwiązywania typów, który działa dla wszystkich modułów aplikacji.

## Wspólny kierunek

Warto zachować rozdział na trzy części: definicje tagów, przypisania do rekordów i reguły zarządzające wybranymi przypisaniami.

1. **Katalog tagów.** Przechowuj stabilne identyfikatory, nazwę i opis. W razie potrzeby dodaj kolor plakietki, stan archiwalny, zakres typu i oznaczenie globalne. Tagi archiwalne domyślnie ukrywaj przy przypisywaniu.
2. **Zakres typów.** Zapisuj trwałą nazwę typu, a potem rozwiązuj ją przez metadane XAF. Ustal, czy typ bazowy obejmuje pochodne, i stosuj tę samą zasadę w selektorze, kontrolerze, walidacji i automacie.
3. **Przypisania zależne od ORM.** W EF Core użyj jawnej encji łączącej dla każdego typu tagowalnego zgodnie z regułami projektu. W XPO użyj `XPCollection` i spójnych `[Association]`. Nie kopiuj mapowania relacji między ORM-ami.
4. **Ręczne akcje.** Korzystaj z zabezpieczonego `ObjectSpace`, sprawdzaj duplikaty, zatwierdzaj zmianę w jednym miejscu i odświeżaj widok po zatwierdzeniu. Ustal, czy wiele wybranych tagów filtruje obiekty przez „dowolny” czy „wszystkie”.
5. **Automaty.** Oddziel ewaluację kryterium od kontrolera i workera. Sprawdzaj, czy typ reguły implementuje kontrakt tagowalny i czy wybrany tag pasuje do jej zakresu. Powtarzane uruchomienie ma być idempotentne.
6. **Pochodzenie przypisań.** Jeśli reguły mają zdejmować tagi, zapisuj źródło przypisania, np. ręczne albo konkretna reguła. Automat powinien usuwać tylko przypisania, którymi zarządza. Jeśli produkt potrzebuje jedynie jednorazowego oznaczania, jawnie ustal zachowanie wyłącznie addytywne.
7. **Wygląd.** Relacja tagu jest źródłem prawdy. Nie przechowuj kopii ostrzeżenia w drugim polu, jeśli da się ją bezpiecznie wyliczyć. Rozdziel kolor plakietki od reguł wyglądu wiersza.
8. **Testy i migracje.** Sprawdź zakres typu, tenantów, uprawnienia, duplikaty, dodawanie i usuwanie, filtr „dowolny/wszystkie”, reguły powtarzane i konflikt reguł z ręcznym tagiem. W EF Core dodaj migrację dla modelu oraz nowych relacji.

## Źródła w repozytoriach

- Fleetman: `Fleetman.Module/BusinessObjects/Common/Tags/Tag.cs`, `Fleetman.Module/Interfaces/ITags.cs`, `Fleetman.Module/Controllers/ListControllers/ITagsListViewController.cs`, `Fleetman.Module/BusinessObjects/Automaty/AutomatyczneTagi.cs`, `Fleetman.Module/HangfireJobs/Processor/AddTagWorker.cs`, `Fleetman.Blazor.Server/Editors/TagBoxPropertyEditor.cs`.
- DataDrive: `CS/DataDrive.Module/BusinessObjects/Tags/Tag.cs`, `TagJunctions.cs`, `AutoTagRule.cs`, `Interfaces/ITaggable.cs`, `BusinessObjects/DataDriveDbContext.cs`, `Features/Tags/AutoTagApplier.cs`, `CS/DataDrive.Blazor.Server/Controllers/TaggableListViewController.cs`, `HostedServices/AutoTagHostedService.cs`.
- HIS: `HIS.Module/BusinessObjects/Basic/Tag.cs`, `Interfaces/ITag.cs`, `Controllers/InterfaceControllers/ITagController.cs`, `BusinessObjects/HISDbContext.cs`, `HIS.Blazor.Server/Editors/TagStringPropertyEditor.cs`.
- PathQ: odpowiedniki HIS w `HIS.Module/`, `HIS.Blazor.Server/Editors/TagStringPropertyEditor.cs` oraz konfiguracja EF w `HIS.Module/BusinessObjects/HISDbContext.cs`.

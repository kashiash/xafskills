# Reguły wyglądu z XPO

Użyj tych wskazówek dopiero po potwierdzeniu, że moduł XAF korzysta z XPO. Zachowaj konwencje trwałych obiektów i sesji stosowane przez docelową aplikację.

## Encja reguły

Dla nowej implementacji zachowaj wspólny kontrakt `AdditionalAppearanceRule` oraz te same nazwy pól co w głównym skillu. Dostosuj klasę bazową, konstruktory, właściwości i mapowanie do XPO. Nie kopiuj encji EF z właściwościami mapowanymi przez `virtual`, `DbSet`, adnotacjami EF ani migracjami EF.

Zarejestruj typ reguły w modelu XAF i użyj właściwej procedury aktualizacji schematu. Zapisuj stabilny klucz typu, a obiekt UI rozwiązuj przez metadane XAF. Obsłuż brak typu lub zmienioną nazwę.

## Integracja z XAF

1. Pobieraj i zapisuj reguły przez zabezpieczony `IObjectSpace`.
2. Nie przechowuj obiektów XPO po zakończeniu sesji lub czasu życia `ObjectSpace`.
3. Jeśli pamięć podręczna jest potrzebna, twórz niemutowalne snapshoty.
4. Aktualizuj pamięć podręczną dopiero po zatwierdzeniu zmian.
5. Unieważniaj wspólną pamięć podręczną i wybór reguł w aktywnych widokach.
6. Określ obsługę zmian w wielu instancjach aplikacji.

Nie aktualizuj globalnej kolekcji z `OnSaving` przed zatwierdzeniem transakcji. Po wycofaniu transakcji pamięć mogłaby różnić się od bazy.

## Platformy i widoki

Dodawaj standardowe reguły przez `AppearanceController.CollectAppearanceRules` w module współdzielonym. Kod zależny od konkretnej siatki lub edytora umieść w module danej platformy. Nie odwołuj się do API Blazor lub DxGrid ze współdzielonego modułu XAF.

## Co sprawdzić

- Typ bazowy, konstruktory, rejestrację modelu i aktualizację schematu.
- Tworzenie, edycję i usuwanie przez zabezpieczony `ObjectSpace`.
- Wycofanie transakcji i czas życia snapshotów.
- Izolację tenantów oraz działanie w wielu instancjach.
- Osobno każdą wspieraną platformę UI.

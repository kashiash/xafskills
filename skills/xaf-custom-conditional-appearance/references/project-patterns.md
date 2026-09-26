# Wybór wzorca reguł wyglądu

Wybierz sposób przechowywania reguł według tego, kto je zmienia i jak długo mają obowiązywać.

| Potrzeba | Wzorzec |
|---|---|
| Stałe zachowanie wynikające z logiki domenowej | Atrybut `[Appearance]` lub reguła Application Model. |
| Reguła wyliczana z bieżącego widoku lub danych nietrwałych | Obiekt w pamięci implementujący `IAppearanceRuleProperties`, dodany przez `CollectAppearanceRules`. |
| Uprawnieni administratorzy zmieniają reguły bez wdrażania kodu | Encja `AdditionalAppearanceRule` z kontrolerem `CollectAppearanceRules`. |
| Stylowanie spoza Conditional Appearance | Osobny renderer lub kontroler zależny od platformy. |

## Wspólne granice

- Nie zapisuj reguł w bazie, jeśli są stałe i zmieniają się tylko razem z kodem.
- Nie używaj reguł wyglądu do kontroli dostępu do danych.
- Rozdziel encję trwałą, adapter lub snapshot XAF, selektor reguł i kod renderowania platformowego.
- Utrzymuj tę samą nazwę encji i wspólne pola w EF Core oraz XPO. Szczegóły mapowania mogą zależeć od ORM.
- Ustal zakres typu, widoku i tenanta przed implementacją.
- Po zapisaniu reguły zatwierdź zmiany, unieważnij pamięć podręczną i odśwież widoki.

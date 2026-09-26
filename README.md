# Umiejętności XAF dla Claude Code

![Przegląd umiejętności XAF](xafskills-overview.png)

Sprawdzone wskazówki z ponad 15 projektów DevExpress XAF. Zebraliśmy je w umiejętnościach dla [Claude Code](https://docs.anthropic.com/en/docs/claude-code).

Pomagają agentom AI unikać typowych pułapek podczas pracy z XAF i EF Core.

## Umiejętności

### Podstawy

| Umiejętność | Zakres |
|---|---|
| **xaf-efcore-entities** | Tworzenie encji: właściwości `virtual`, `BaseObjectInt`, `ObservableCollection`, precyzja `decimal`, pułapka `DateTime` w PostgreSQL, indeksy `GCRecord` i konwertery wartości. |
| **xaf-blazor-startup** | Konfiguracja `Startup.cs`: kolejność rejestracji usług, potok middleware, JWT, OData i Web API, cykl życia modułów oraz obiekty nietrwałe. |
| **xaf-security** | Uprawnienia do typów, obiektów i właściwości; eksport i import ról; `PermissionsReloadMode`; uwierzytelnianie zadań w tle; `CurrentUserIdOperator`. |
| **xaf-reporting** | ReportsV2: obiekty parametrów, pułapka `Visible=false`, `GetCriteria()` i `FilterString`, `PredefinedReportsUpdater`. |
| **devexpress-xaf-docker** | Konteneryzacja XAF Blazor: natywne zależności SkiaSharp, wersje DevExpress, PostgreSQL i MySQL oraz inicjalizacja schematu. |
| **xaf-tags** | Trwałe tagi w XAF dla EF Core i XPO: przypisywanie, filtrowanie, akcje zbiorcze, automatyzacja i wyświetlanie oznaczeń. |

### Wzorce

| Umiejętność | Zakres |
|---|---|
| **xaf-hangfire-jobs** | Zadania Command/Handler bez zależności od XAF, `HangfireJobDispatcher` i `DirectJobDispatcher`, encja `JobDefinition`, uwierzytelnianie konta technicznego oraz synchronizacja zadań przy starcie. |
| **xaf-search-panels** | Konfigurowalne okna wyszukiwania z DTO `[DomainComponent]` oraz ograniczenia kompilacji Roslyn podczas działania z `AddSecuredEFCore`. |
| **xaf-custom-conditional-appearance** | Wspólny kontrakt `AdditionalAppearanceRule`, reguły wbudowane, generowane w kodzie i zapisane w EF Core lub XPO. Opisuje też tworzenie reguły z aktywnego filtra listy. |
| **xaf-navigation-hub** | Startowy `DashboardView` z kafelkami, filtrowaniem według uprawnień i ulubionymi przypiętymi przez użytkownika. |
| **xaf-environment-auth** | Wybór SSO lub hasła na podstawie `ASPNETCORE_ENVIRONMENT`, wraz z wyjątkiem dla konta technicznego Hangfire. |
| **xaf-playwright-testing** | Testy E2E XAF Blazor w Playwright i NUnit: logowanie, odporne selektory, zrzuty ekranu po błędach i synchronizacja z siecią. |
| **xaf-easytest-authoring** | Testy funkcjonalne EasyTest dla WinForms i Blazor: konfiguracja projektów, selektory, API EasyTest oraz typowe pułapki lokalizacji i zagnieżdżonych siatek. |
| **xaf-saved-list-filters** | Zapisywanie filtrów list XAF z EF Core: filtry prywatne i publiczne, właściciel, zakres widoku i tenanta, bezpieczne użycie `CriteriaOperator` oraz czyszczenie kryteriów. |

## Materiały o wzorcach

- [Umiejętność zapisanych filtrów](skills/xaf-saved-list-filters/SKILL.md) — dodawanie zapisanych filtrów list w aplikacjach XAF z EF Core.
- [Porównanie reguł wyglądu](docs/appearance-rules-comparison.md) — wzorce Fleetman, DataDrive, HIS i PathQ oraz wspólny kontrakt.
- [Umiejętność reguł wyglądu](skills/xaf-custom-conditional-appearance/SKILL.md) — atrybuty XAF, reguły generowane w kodzie i konfiguracja EF Core/XPO.
- [Porównanie tagów](docs/xaf-tags-comparison.md) — rozwiązania Fleetman, DataDrive, HIS i PathQ.
- [Projekt automatycznego tagowania](docs/xaf-auto-tags-design.md) — reguły, praca na tenantach i bezpieczne usuwanie przypisań.
- [Umiejętność tagów](skills/xaf-tags/SKILL.md) — tagi w XAF dla EF Core i XPO.

## Instalacja

Umiejętności są dostępne jako wtyczka Claude Code `xaf-tools`. Zainstaluj ją z marketplace:

```text
/plugin marketplace add kashiash/xafskills
/plugin install xaf-tools@xafskills
```

Wtyczka instaluje wszystkie 14 umiejętności. Claude Code uruchamia je automatycznie przy pasujących zadaniach. Aby aktualizować wtyczkę automatycznie, ustaw `autoUpdate: true` dla marketplace w `~/.claude/settings.json` w sekcji `extraKnownMarketplaces`.

Możesz też zaktualizować ją ręcznie:

```text
/plugin marketplace update xafskills
```

### Instalacja ręczna

Możesz skopiować wybraną umiejętność z katalogu `skills/`:

```bash
cp -r skills/xaf-efcore-entities ~/.claude/skills/      # globalnie, dla wszystkich projektów
cp -r skills/xaf-efcore-entities /path/to/project/.claude/skills/   # tylko dla projektu
```

## Wymagania

- DevExpress XAF 25.2 lub nowszy z EF Core
- .NET 8.0 lub .NET 9.0

## Źródła wiedzy

Wskazówki pochodzą z projektów produkcyjnych. Obejmują dynamiczne ładowanie zestawów, tworzenie encji podczas działania aplikacji, Hangfire, Elsa, partycjonowanie PostgreSQL, zarządzanie rolami, nawigację, czat AI i raporty.

## Licencja

MIT

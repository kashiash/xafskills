# Umiejętności XAF dla Claude Code

![Przegląd umiejętności XAF](xafskills-overview.png)

Sprawdzone wskazówki z ponad 15 projektów DevExpress XAF. Zebraliśmy je w umiejętnościach dla [Claude Code](https://docs.anthropic.com/en/docs/claude-code).

Pomagają agentom AI unikać typowych pułapek podczas pracy z XAF i EF Core.

## Umiejętności

### Podstawy

| Umiejętność | Zakres |
|---|---|
| **[xaf-efcore-entities](skills/xaf-efcore-entities/SKILL.md)** | Tworzenie encji: właściwości `virtual`, `BaseObjectInt`, `ObservableCollection`, precyzja `decimal`, pułapka `DateTime` w PostgreSQL, indeksy `GCRecord` i konwertery wartości. |
| **[xaf-blazor-startup](skills/xaf-blazor-startup/SKILL.md)** | Konfiguracja `Startup.cs`: kolejność rejestracji usług, potok middleware, JWT, OData i Web API, cykl życia modułów oraz obiekty nietrwałe. |
| **[xaf-security](skills/xaf-security/SKILL.md)** | Uprawnienia do typów, obiektów i właściwości; eksport i import ról; `PermissionsReloadMode`; uwierzytelnianie zadań w tle; `CurrentUserIdOperator`. |
| **[xaf-reporting](skills/xaf-reporting/SKILL.md)** | ReportsV2: obiekty parametrów, pułapka `Visible=false`, `GetCriteria()` i `FilterString`, `PredefinedReportsUpdater`. |
| **[devexpress-xaf-docker](skills/devexpress-xaf-docker/SKILL.md)** | Konteneryzacja XAF Blazor: natywne zależności SkiaSharp, wersje DevExpress, PostgreSQL i MySQL oraz inicjalizacja schematu. |
| **[xaf-tags](skills/xaf-tags/SKILL.md)** | Trwałe tagi w XAF dla EF Core i XPO: przypisywanie, filtrowanie, akcje zbiorcze, automatyzacja i wyświetlanie oznaczeń. |

### Wzorce

| Umiejętność | Zakres |
|---|---|
| **[xaf-hangfire-jobs](skills/xaf-hangfire-jobs/SKILL.md)** | Zadania Command/Handler bez zależności od XAF, `HangfireJobDispatcher` i `DirectJobDispatcher`, encja `JobDefinition`, uwierzytelnianie konta technicznego oraz synchronizacja zadań przy starcie. |
| **[xaf-search-panels](skills/xaf-search-panels/SKILL.md)** | Konfigurowalne okna wyszukiwania z DTO `[DomainComponent]` oraz ograniczenia kompilacji Roslyn podczas działania z `AddSecuredEFCore`. |
| **[xaf-custom-conditional-appearance](skills/xaf-custom-conditional-appearance/SKILL.md)** | Wspólny kontrakt `AdditionalAppearanceRule`, reguły wbudowane, generowane w kodzie i zapisane w EF Core lub XPO. Opisuje też tworzenie reguły z aktywnego filtra listy. |
| **[xaf-view-layouts](skills/xaf-view-layouts/SKILL.md)** | Personalizacja i zapis układu ListView oraz DetailView przez wbudowane różnice modelu XAF, wraz z resetowaniem układu. |
| **[xaf-model-editor](skills/xaf-model-editor/SKILL.md)** | Edycja drzewa Application Model: wartości i węzły, lokalizacje, reset, zapis warstwy użytkownika i modelu współdzielonego. |
| **[xaf-navigation-hub](skills/xaf-navigation-hub/SKILL.md)** | Startowy `DashboardView` z kafelkami, filtrowaniem według uprawnień i ulubionymi przypiętymi przez użytkownika. |
| **[xaf-environment-auth](skills/xaf-environment-auth/SKILL.md)** | Wybór SSO lub hasła na podstawie `ASPNETCORE_ENVIRONMENT`, wraz z wyjątkiem dla konta technicznego Hangfire. |
| **[xaf-playwright-testing](skills/xaf-playwright-testing/SKILL.md)** | Testy E2E XAF Blazor w Playwright i NUnit: logowanie, odporne selektory, zrzuty ekranu po błędach i synchronizacja z siecią. |
| **[xaf-easytest-authoring](skills/xaf-easytest-authoring/SKILL.md)** | Testy funkcjonalne EasyTest dla WinForms i Blazor: konfiguracja projektów, selektory, API EasyTest oraz typowe pułapki lokalizacji i zagnieżdżonych siatek. |
| **[xaf-saved-list-filters](skills/xaf-saved-list-filters/SKILL.md)** | Zapisywanie filtrów list XAF z EF Core: filtry prywatne i publiczne, właściciel, zakres widoku i tenanta, bezpieczne użycie `CriteriaOperator` oraz czyszczenie kryteriów. |

## Materiały o wzorcach

- [Umiejętność zapisanych filtrów](skills/xaf-saved-list-filters/SKILL.md) — dodawanie zapisanych filtrów list w aplikacjach XAF z EF Core.
- [Umiejętność reguł wyglądu](skills/xaf-custom-conditional-appearance/SKILL.md) — atrybuty XAF, reguły generowane w kodzie i konfiguracja EF Core/XPO.
- [Umiejętność układów widoków](skills/xaf-view-layouts/SKILL.md) — personalizacja layoutu użytkownika oraz zapis różnic modelu XAF.
- [Umiejętność edytora modelu](skills/xaf-model-editor/SKILL.md) — edycja węzłów Application Model i zapis różnic użytkownika lub administratora.
- [Projekt automatycznego tagowania](docs/xaf-auto-tags-design.md) — reguły, praca na tenantach i bezpieczne usuwanie przypisań.
- [Umiejętność tagów](skills/xaf-tags/SKILL.md) — tagi w XAF dla EF Core i XPO.

## Instalacja

Umiejętności są dostępne jako wtyczka Claude Code `xaf-tools`. Zainstaluj ją z marketplace:

```text
/plugin marketplace add kashiash/xafskills
/plugin install xaf-tools@xafskills
```

Wtyczka instaluje wszystkie 16 umiejętności. Claude Code uruchamia je automatycznie przy pasujących zadaniach. Aby aktualizować wtyczkę automatycznie, ustaw `autoUpdate: true` dla marketplace w `~/.claude/settings.json` w sekcji `extraKnownMarketplaces`.

Możesz też zaktualizować ją ręcznie:

```text
/plugin marketplace update xafskills
```

### Instalacja ręczna

#### Codex

Umiejętności Codex odczytuje z katalogu użytkownika `~/.agents/skills/` albo z katalogu repozytorium `.agents/skills/`. Skopiuj cały folder umiejętności, razem z plikami pomocniczymi:

```bash
mkdir -p ~/.agents/skills
cp -R skills/xaf-efcore-entities ~/.agents/skills/
```

Aby udostępnić umiejętność tylko w bieżącym repozytorium:

```bash
mkdir -p .agents/skills
cp -R skills/xaf-efcore-entities .agents/skills/
```

Codex wykrywa zmiany umiejętności automatycznie. Jeśli nowa umiejętność się nie pojawi, uruchom Codex ponownie. Zobacz [dokumentację umiejętności Codex](https://developers.openai.com/codex/skills).

#### Claude Code

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

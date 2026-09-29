# Włączanie PostgreSQL w DataDrive, HIS i PathQ

Ten przewodnik opisuje, co trzeba ustawić, aby aplikacja XAF łączyła się z PostgreSQL. Stan kodu sprawdzono 26 września 2026 r. Nazwy projektów i ustawień odnoszą się do bieżących repozytoriów.

## Szybkie porównanie

| Projekt | Czy wybiera provider? | Konfiguracja EF Core | Ważne elementy PostgreSQL |
|---|---|---|---|
| DataDrive | Nie. Aplikacja używa PostgreSQL. | `UseNpgsql`, migracje tenantów i hosta. | `citext`; `vector` przez `UseVector()`; globalny przełącznik zgodności daty i czasu. |
| HIS | Nie. Sprawdzony kod XAF używa PostgreSQL bezwarunkowo. | `UseNpgsql` w fabrykach i konfiguracji XAF. | Pola tekstowe `citext`; w kodzie aplikacji nie znaleziono globalnego przełącznika czasu. |
| PathQ | Tak. | `Config.DbType` wybiera SQL Server albo PostgreSQL. | Osobny assembly migracji; `citext`; konwertery UTC i typ `timestamp without time zone`. |

Nie kopiuj ustawień między projektami bez sprawdzenia modelu. W szczególności sposób obsługi daty i czasu różni się między nimi.

## DataDrive

DataDrive jest skonfigurowany pod PostgreSQL. Nie ma przełącznika, który zmienia go na SQL Server.

### Połączenia

- Ustaw `ConnectionStrings:ConnectionString` dla bazy hosta.
- Ustaw `ConnectionStrings:EasyTestConnectionString` dla EasyTest.
- Hasła podawaj przez User Secrets albo zmienne środowiskowe. Kod odczytuje też hasła z `PostgreSql:PasswordEnvVar` i `PostgreSql:AdminPasswordEnvVar`.
- Klucze środowiskowe wymienione w przykładowym `appsettings.json` to `DATADRIVE_POSTGRES_PASSWORD` i `DATADRIVE_POSTGRES_ADMIN_PASSWORD`.
- `PostgreSql:ProvisionTenantDatabases` steruje tworzeniem baz tenantów. Domyślnie ma wartość `false`.
- Każda baza tenanta potrzebuje własnego poprawnego connection stringa. Provisioning może go budować z połączenia hosta.

Nie umieszczaj haseł w `appsettings*.json`, dokumentacji ani repozytorium.

### Pakiety i konfiguracja EF Core

Projekt modułu i serwera odwołują się do `Npgsql.EntityFrameworkCore.PostgreSQL` w wersji `10.0.1`. Konteksty biznesowe i audytowe używają `UseNpgsql`. Konfiguracja uwzględnia także ponawianie połączeń oraz `UseVector()`.

Przy dodawaniu nowego `DbContext` albo fabryki sprawdź, czy rejestracja providera również wywołuje `UseVector()`, jeśli kontekst mapuje typy wektorowe. Sam pakiet Npgsql nie rejestruje mapowania wektorów EF.

### Rozszerzenia PostgreSQL

- `citext` jest potrzebne, bo część pól tekstowych ma ten typ w modelu. Updater wykonuje `CREATE EXTENSION IF NOT EXISTS citext` dla bazy hosta i baz tenantów.
- Konto, którym updater łączy się z bazą, musi mieć uprawnienia do instalacji rozszerzenia. Można też zainstalować rozszerzenie wcześniej przez administratora PostgreSQL.
- Konfiguracja EF rejestruje `UseVector()`. To nie instaluje rozszerzenia serwerowego. Sprawdź wymagania aktualnych migracji i funkcji wektorowych, a następnie zapewnij dostępność rozszerzenia `vector` w każdej używanej bazie.
- `pg_trgm` jest potrzebne tylko wtedy, gdy używasz zapytań lub indeksów trigramowych. To osobne rozszerzenie, niezależne od `citext` i `vector`.

### Data i czas

`Program` ustawia `AppContext.SetSwitch("Npgsql.EnableLegacyTimestampBehavior", true)` w statycznym konstruktorze. Przełącznik musi zadziałać przed utworzeniem pierwszego połączenia Npgsql. Dzięki temu działa także w testach HTTP i narzędziach design-time, które mogą ominąć `Main`.

Nie przenoś tego przełącznika do `Main` ani po konfigurację hosta. Nie usuwaj go bez przeglądu wszystkich mapowań `DateTime`, zapisów seedujących i integracji.

## HIS

W repozytorium HIS konfiguracja XAF używa PostgreSQL bezwarunkowo. `Config.ConnectionString` składa adres z ustawień środowiskowych w wydaniu, a w buildzie `DEBUG` korzysta z ustawienia lokalnego. Nie ma widocznego przełącznika providera.

### Co ustawić

- W wydaniu ustaw `MXSHOST`, `MXSDATABASE`, `MXSUSERNAME` i `MXSPASSWORD`.
- W trybie `DEBUG` sprawdź lokalną konfigurację połączenia w `HIS.Module/Config.cs` i zastąp wartości sekretów ustawieniami lokalnymi. Nie kopiuj sekretów zapisanych w pliku źródłowym.
- Projekt `HIS.Module` zawiera `Npgsql.EntityFrameworkCore.PostgreSQL`; obiektowe i audytowe konteksty XAF są konfigurowane przez `UseNpgsql`.
- Wartości `DateTime` w migracjach i modelu używają `timestamp without time zone`. Migracje zawierają również kolumny `citext`.

Przed migracją przygotuj rozszerzenie `citext` w bazie docelowej, jeśli nie istnieje. Kod aplikacji nie wykonał jawnego `CREATE EXTENSION citext` w przeszukanych ścieżkach startowych. Użyj konta administracyjnego lub przygotuj rozszerzenie osobno.

W kodzie HIS nie znaleziono ustawienia `Npgsql.EnableLegacyTimestampBehavior`. Nie zakładaj, że DataDrive'owy przełącznik jest wymagany. Sprawdź zgodność wartości `DateTime` w modelu i dane seedujące HIS przed zmianą zachowania.

## PathQ

PathQ ma dwie gałęzie konfiguracji providera. `PathQ.Common.Config.DbType` wybiera provider, a `ProviderHelper` i konfiguracja XAF ustawiają odpowiednią fabrykę EF Core.

### Wybór providera i połączenie

- `MXSDBTYPE=1` wybiera SQL Server.
- `MXSDBTYPE=2` wybiera PostgreSQL.
- W `DEBUG` kod wybiera PostgreSQL bez sprawdzania `MXSDBTYPE`.
- W trybie wydania ustaw `MXSHOST`, `MXSDATABASE`, `MXSUSERNAME`, `MXSPASSWORD` oraz `MXSDBTYPE`. Wartości połączenia odszyfrowuje aplikacja.
- Zmienna `ConnectionString`, jeśli jest ustawiona, nadpisuje składanie connection stringa.
- Bez `MXSHOST` konfiguracja wraca do ustawień lokalnych i wybiera PostgreSQL. Nie uruchamiaj w ten sposób środowiska produkcyjnego.

Nazwy zmiennych są częścią obecnego kodu PathQ. Nie zmieniaj ich na nazwy HIS bez zmiany konfiguracji.

### Provider i migracje

- Wersja `Npgsql.EntityFrameworkCore.PostgreSQL` widoczna w `PathQ.PostgreSQL.csproj` to `9.0.4`.
- Dla PostgreSQL kod ustawia `UseNpgsql(..., x => x.MigrationsAssembly("PathQ.PostgreSQL"))`.
- Dla SQL Server kod ustawia `UseSqlServer(..., x => x.MigrationsAssembly("PathQ.SqlServer"))`.
- Te ustawienia występują w provider helperze, fabrykach design-time i konfiguracji startowej Blazor/WinForms. Nowa ścieżka EF musi zachować wybór assembly migracji.
- Hangfire również wybiera magazyn według `DbType`: `UsePostgreSqlStorage` dla PostgreSQL i `UseSqlServerStorage` dla SQL Server.

### `citext` i czas

- Dla PostgreSQL `HISEFCoreDbContext` przypisuje typ `citext` do właściwości tekstowych. Atrybut `IgnoreCiText` pozwala pominąć wybrane właściwości.
- Migracje PostgreSQL zawierają kolumny typu `citext`. Przygotuj rozszerzenie `citext` przed aktualizacją schematu, o ile migracja lub skrypt wdrożeniowy go wcześniej nie tworzy.
- Dla pól `DateTime` model ustawia `timestamp without time zone`. W `PathQ.Common.Helpers` istnieją też konwertery UTC dla `DateTime` i `DateTimeOffset`, ale przeszukany kod nie wywołuje tych metod. Nie zakładaj, że konwertery działają.
- Nie znaleziono globalnego przełącznika `Npgsql.EnableLegacyTimestampBehavior` w ścieżce HIS/PathQ. Nie dodawaj go jako zamiennika konwerterów bez ustalenia semantyki istniejących danych.

## Kontrola przed uruchomieniem

1. Potwierdź, że uruchamiasz właściwy projekt i tryb (`DEBUG` albo wydanie).
2. Ustaw connection string oraz wybór providera, jeśli projekt go obsługuje.
3. Sprawdź, czy baza jest osiągalna i czy konto ma wymagane uprawnienia.
4. Przygotuj `citext`, a w DataDrive także używane rozszerzenia `vector` lub `pg_trgm`.
5. Uruchom migracje właściwym assembly migracji.
6. Sprawdź typy kolumn czasu, mapowanie `DateTime` i dane seedujące.
7. Nie używaj haseł z plików źródłowych. Zastąp je sekretami środowiska.

## Gdzie sprawdzić szczegóły

- DataDrive: `CS/DataDrive.Blazor.Server/Program.cs`, `Startup.cs`, `appsettings.json`, `CS/DataDrive.Module/BusinessObjects/DataDriveDbContext.cs` i `DatabaseUpdate/Updater.cs`.
- HIS: `HIS.Module/Config.cs`, `HIS.Module/BusinessObjects/HISDbContext.cs`, `HIS.Blazor.Server/Startup.cs` i `HIS.Win/Startup.cs`.
- PathQ: `PathQ.Common/Config.cs`, `HIS.Module/Helpers/ProviderHelper.cs`, `HIS.Module/BusinessObjects/HISDbContext.cs`, `HIS.Blazor.Server/Startup.cs`, `HIS.Win/Startup.cs` oraz `HIS.PostgreSQL/`.

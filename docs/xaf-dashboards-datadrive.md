# Dashboardy DevExpress w XAF Blazor: studium przypadku DataDrive

**Wniosek:** dashboard w XAF Blazor da się zbudować w kodzie i dostarczyć na wiele tenantów, ale trzy rzeczy wyszły dopiero po wdrożeniu: przeglądarka ignoruje własne formaty liczb, element z filtrem na polu spoza wymiarów zwraca HTTP 500, a zaseedowana definicja nie aktualizuje się sama. Testy jednostkowe przechodziły przy wszystkich trzech. Dlatego skill [`xaf-dashboards`](../skills/xaf-dashboards/SKILL.md) wymaga weryfikacji na działającym środowisku i oznacza każdą tezę jako potwierdzoną, wynikającą z modelu albo hipotezę.

Analiza opiera się na pracy z 1 października 2026 r. (zgłoszenie #2749 i PR-y #2752–#2770 w DataDrive). Dane pochodzą ze środowiska testowego BBS i tenanta GITD. Nie sprawdzałem produkcji ani QS.

## Co zbudowano

Dashboardy mają być odpowiednikami analiz z menu „Analizy”: te same dane z tego samego loadera, żeby sumy dało się porównać. Firma nie zarabia na pojazdach, więc analiz przychodowych nie odwzorowano.

| Dashboard | Odpowiednik analizy | Wyróżniki |
|---|---|---|
| Koszty pojazdów — analityka floty i zestawienia szczegółowe | Analityka kosztów floty | Podział na dwa dashboardy, koło klas kosztu jako filtr, legendy |
| Koszty kartowe ORLEN per pojazd | Koszty kartowe ORLEN per pojazd | Rok do roku, top 10 pojazdów, koło rodzaju produktu |
| Prognoza płatności | Prognoza płatności | Zaległości jako osobny okres, saldo |
| Koszty tras — koszt na 100 km | Koszty tras | Ranking, pojazdy poniżej progu przebiegu |
| Zdarzenia flotowe i serwisy zaplanowane | Zdarzenia flotowe | Oś czasu, terminy z kartoteki pojazdu |

## Co się wydarzyło i jak to ustalono

| Objaw | Dowód | Przyczyna | Poprawka | Status |
|---|---|---|---|---|
| Brak pozycji menu Dashboardy | Filtr menu nie znajdował pozycji; kod aktualizatora usuwał wszystkie wygenerowane węzły | `NavigationItemsModelUpdater.UpdateNode` czyści menu i dodaje tylko wybrane pozycje | Jawny węzeł `Dashboards` w grupie Raporty, ten sam Id w `AllowedNavigation` | Potwierdzone. Podpis po polsku to „Pulpity nawigacyjne”, więc pierwsze sprawdzenie po angielskiej frazie dało fałszywy wniosek |
| Ranking zwraca 500 | `DashboardItemGetAction` dla jednego elementu: 500 w każdej próbie, reszta 200; w logu tylko `ERR … responded 500` | Ukryta miara użyta w Top N i w `VisibleDataFilterString` | Top N i sortowanie po widocznej mierze, bez filtra danych | Naprawa potwierdzona (200). Zdjęto dwie rzeczy naraz, więc nie wiadomo, która była winna |
| Karty KPI bez liczb | Odpowiedź `costKpi` zawierała wartości; na ekranie same tytuły | Element dostawał ok. 10% wysokości | Waga w układzie z 15 na 26 | Potwierdzone |
| „1,95M zł”, „0 zł” mimo formatu | `MeasureDescriptors[].Format` = `Currency/Auto`, a w zapisanej definicji `Custom` | Przeglądarka Blazor nie stosuje `CustomFormatString` | `Number` + `Unit=Ones`, zero jako `null` w danych | Potwierdzone na ekranie |
| Dashboard startuje przefiltrowany | W nagłówku widać wybrany kawałek koła | Tryb `Single` wymusza wybór elementu | Tryb `Multiple` | Zaobserwowane; poprawka wdrożona, nie sprawdzona po wdrożeniu |
| Siedem elementów z filtrem zwraca 500 | Wszystkie mają `FilterString` z polem, którego element nie używa jako wymiaru; działające filtry używają pól obecnych jako wymiary | Hipoteza: element nie ładuje pola użytego tylko w filtrze | `HiddenDimensions` dla pól filtra | **Hipoteza.** Poprawka wdrożona, wynik do potwierdzenia |
| Nowa definicja nie dociera na BBS | Zapisana definicja miała źródła i elementy innej wersji niż obie zaseedowane | Użytkownik zmienił rekord w projektancie, więc seeder go pominął | Zgodnie z projektem: edytowana definicja zostaje; decyzja należy do właściciela | Zaobserwowane |

Lokalny `DashboardExporter` policzył wszystkie elementy bez błędu, więc nie odtworzył żadnego z błędów 500. Nadaje się tylko jako test dymny.

## Decyzje projektowe

**Klasa kosztu jako wymiar.** Pierwsza wersja miała sześć kolumn kosztów jako miary. Takiego dashboardu nie da się filtrować kliknięciem w koło ani legendę, bo filtr główny działa na wymiarach. Nowe źródło ma jeden wiersz na pojazd × miesiąc × klasę kosztu i jedną miarę kwoty. Wiersz bez kosztu zachowuje pojazd i jego wycenę dla kart i siatki. Koło i wykresy wykluczają go filtrem `Not IsNull([CostClass])`.

**Aktualizacja zaseedowanej definicji przez hash.** Seeder tworzy brakujący rekord. Istniejący zastępuje tylko wtedy, gdy SHA-256 jego treści (po zamianie CRLF na LF) jest na liście znanych wcześniejszych wersji. Hash nowej wersji liczy się na starym kodzie, zanim zmieni się `Build()`. Mechanizm zadziałał dla kolejnych wersji na BBS. Rekord zmieniony ręcznie ma nieznany hash i zostaje.

**Źródła jako klasy nietrwałe z loaderem analizy.** Jeden wiersz odpowiada jednemu elementowi analizy, a testy porównują sumy projekcji z sumami loadera. Każdy element dashboardu woła loader osobno, więc ciężkie analizy (transakcje kartowe, trasy) obciążają bazę wielokrotnie. To argument za materializacją agregatów.

## Procedura weryfikacji po wdrożeniu

1. Zaloguj się rolą, jaką mają użytkownicy końcowi, nie tylko administratorem.
2. Otwórz dashboard z listy `DashboardData_ListView` i zapisz odpowiedzi `DashboardItemGetAction` oraz zapisaną definicję.
3. Sprawdź status każdego elementu, format miar, wartości w `Slices[].Data` i to, że definicja zawiera nowe nazwy elementów.
4. Zrób zrzut ekranu. Karty z tytułami bez liczb oznaczają za mały element, a nie brak danych.
5. Porównaj sumy z analizą.

Skrypt z dodatku robi kroki 1–4. Wymaga Node i pakietu `playwright`; hasło podaje się zmienną środowiskową.

## Co zostało otwarte

- Czy `HiddenDimensions` naprawiają siedem elementów z błędem 500 i czy nie rozbijają kart na grupy.
- Czy klik w sam element legendy filtruje; sprawdzono tylko klik w segment, i to nie na wdrożonym środowisku.
- Czy osie wykresów da się pozbawić skrótów „40K”.
- Jak ustawić szerokość kolumn siatki, żeby liczby nie były obcinane.

## Dodatek: skrypt przechwytywania odpowiedzi

```javascript
// Weryfikacja dashboardu DevExpress w działającej aplikacji XAF Blazor: login -> lista dashboardów -> otwarcie wskazanego ->
// zapis odpowiedzi DashboardItemGetAction (status + JSON per element), zapisanej definicji i zrzutu ekranu.
//
// Użycie (Node + playwright; własny Chromium, bez MCP):
//   BASE_URL=https://host LOGIN='Admin@tenant' PASSWORD="$(…)" TITLE='Koszty pojazdów — analityka floty' OUT_DIR=./out node capture-dashboard-items.js
// Hasło podawaj ze zmiennej środowiskowej; nic nie wypisujemy.
// Wynik: OUT_DIR/items/<itemId>.json, OUT_DIR/definition.json, OUT_DIR/dashboard.png i linie "ITEM <id> HTTP <status> len <n>" na stdout.
// Status 500 = element padł; format miar sprawdź w ItemData.MetaData.MeasureDescriptors[].Format, wartości w DataStorageDTO.Slices[].Data.
const fs = require('fs');
const path = require('path');
const { chromium } = require('playwright');

(async () => {
  const BASE = process.env.BASE_URL, LOGIN = process.env.LOGIN, PASSWORD = process.env.PASSWORD;
  const TITLE = process.env.TITLE, OUT = process.env.OUT_DIR || './out';
  if (!BASE || !LOGIN || !PASSWORD || !TITLE) { console.log('Ustaw BASE_URL, LOGIN, PASSWORD, TITLE'); process.exit(2); }
  fs.mkdirSync(path.join(OUT, 'items'), { recursive: true });

  const browser = await chromium.launch({ headless: true });
  const page = await (await browser.newContext({ viewport: { width: 1800, height: 1100 }, ignoreHTTPSErrors: true })).newPage();
  page.setDefaultTimeout(150000); // łącze do zdalnego środowiska bywa wolne: długie limity
  const seen = [];
  page.on('response', async response => {
    try {
      const url = response.url();
      if (/\/api\/dashboard\/dashboards\//.test(url)) fs.writeFileSync(path.join(OUT, 'definition.json'), await response.text());
      const m = url.match(/DashboardItemGetAction\?dashboardId=[^&]+&itemId=([A-Za-z0-9_]+)/);
      if (!m) return;
      const body = await response.text();
      fs.writeFileSync(path.join(OUT, 'items', m[1] + '.json'), body);
      seen.push({ item: m[1], status: response.status(), len: body.length });
    } catch { /* odpowiedź mogła zostać zamknięta przy nawigacji */ }
  });

  await page.goto(BASE, { waitUntil: 'domcontentloaded', timeout: 120000 });
  await page.waitForTimeout(5000);
  // Blazor gubi znaki przy fill(): pressSequentially.
  const user = page.getByLabel('Nazwa użytkownika'); await user.click(); await user.fill(''); await user.pressSequentially(LOGIN, { delay: 12 });
  const pass = page.getByLabel('Hasło'); await pass.click(); await pass.fill(''); await pass.pressSequentially(PASSWORD, { delay: 12 });
  await page.getByRole('button', { name: 'Zaloguj się' }).click();
  for (let i = 0; i < 90; i++) { // czekaj aż aplikacja się załaduje (zimny start potrafi trwać)
    if (!/\/Login/i.test(page.url())) { const t = await page.locator('body').innerText().catch(() => ''); if (t.length > 300) break; }
    await page.waitForTimeout(1000);
  }
  await page.goto(BASE + '/DashboardData_ListView', { waitUntil: 'domcontentloaded', timeout: 120000 });
  await page.waitForTimeout(8000);
  await page.getByText(TITLE, { exact: true }).first().click();
  for (let i = 0; i < 8; i++) { // dashboard ładuje elementy asynchronicznie
    await page.waitForTimeout(6000);
    if (/DashboardViewer_DetailView/.test(page.url()) && seen.length >= 3) break;
  }
  await page.waitForTimeout(4000);
  await page.screenshot({ path: path.join(OUT, 'dashboard.png'), fullPage: true });
  for (const s of seen) console.log(`ITEM ${s.item} HTTP ${s.status} len ${s.len}`);
  await browser.close();
})().catch(e => { console.log('BŁĄD', e.message.split('\n')[0]); process.exit(1); });
```

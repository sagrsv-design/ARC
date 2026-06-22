# Dziennik ARC — instrukcja uruchomienia chmury (Cloudflare)

Masz dwie części:

1. **Aplikacja** (folder `dziennik-arc/`) — `index.html`, `manifest.webmanifest`, `sw.js`, ikony.
   Hostujemy ją na **Cloudflare Pages** (darmowo, HTTPS, PWA).
2. **Backend / chmura** (folder `dziennik-arc/worker/`) — `worker.js`, `schema.sql`, `wrangler.toml`.
   To **Cloudflare Worker + baza D1**, który przechowuje konta i wpisy oraz synchronizuje je między telefonem a komputerem.

Bez backendu aplikacja i tak działa — tylko lokalnie na jednym urządzeniu. Chmura włącza synchronizację.

---

## Zanim zaczniesz (jednorazowo)

1. Załóż darmowe konto na <https://dash.cloudflare.com> (jeśli jeszcze nie masz).
2. Zainstaluj **Node.js** (LTS) z <https://nodejs.org>. Sprawdź w terminalu: `node -v`.
3. `wrangler` (narzędzie Cloudflare) będziemy uruchamiać przez `npx` — nie trzeba nic instalować ręcznie.

---

## CZĘŚĆ A — Backend (Worker + D1)

> Wszystkie polecenia uruchamiasz w terminalu, **w folderze `dziennik-arc/worker/`**.
> (W terminalu: `cd ścieżka/do/dziennik-arc/worker`)

### A1. Zaloguj się do Cloudflare
```bash
npx wrangler login
```
Otworzy się przeglądarka — kliknij **Allow**.

### A2. Utwórz bazę danych D1
```bash
npx wrangler d1 create dziennik-arc-db
```
W odpowiedzi dostaniesz blok podobny do:
```
[[d1_databases]]
binding = "DB"
database_name = "dziennik-arc-db"
database_id = "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
```
**Skopiuj wartość `database_id`** i wklej ją w pliku `wrangler.toml` w miejsce `WKLEJ_TUTAJ_DATABASE_ID`.

### A3. Utwórz tabele (schemat)
```bash
npx wrangler d1 execute dziennik-arc-db --file=./schema.sql --remote
```
(`--remote` = tworzy tabele w prawdziwej bazie w chmurze, nie tylko lokalnie.)

### A4. Ustaw sekret do podpisywania tokenów
```bash
npx wrangler secret put AUTH_SECRET
```
Poprosi o wartość — wklej **długi, losowy ciąg** (np. wygeneruj 40+ znaków). Zapamiętaj, że to klucz bezpieczeństwa — nie udostępniaj go nikomu.

> Szybki sposób na losowy sekret (w terminalu):
> `node -e "console.log(require('crypto').randomBytes(32).toString('hex'))"`

### A5. Wdróż Workera
```bash
npx wrangler deploy
```
Na końcu zobaczysz adres, np.:
```
https://dziennik-arc.twoja-nazwa.workers.dev
```
**To jest Twój „adres serwera" (Worker URL)** — zapisz go, wkleisz w aplikacji.

✅ Backend gotowy.

---

## CZĘŚĆ B — Aplikacja (Cloudflare Pages)

Masz dwie drogi. **B1 (przez przeglądarkę)** jest najprostsza.

### B1. Wgranie przez panel (drag & drop)
1. Wejdź na <https://dash.cloudflare.com> → po lewej **Workers & Pages** → **Create** → zakładka **Pages** → **Upload assets**.
2. Nadaj nazwę projektu, np. `dziennik-arc`.
3. Przeciągnij **zawartość** folderu `dziennik-arc/` (czyli `index.html`, `manifest.webmanifest`, `sw.js`, `icon-192.png`, `icon-512.png`).
   ⚠️ **Nie wgrywaj** podfolderu `worker/` — to backend, nie część strony.
4. Kliknij **Deploy**. Dostaniesz adres typu `https://dziennik-arc.pages.dev`.

### B2. (Alternatywa) Wdrożenie z terminala
W folderze `dziennik-arc/`:
```bash
npx wrangler pages deploy . --project-name=dziennik-arc
```

✅ Aplikacja jest online pod adresem `*.pages.dev`.

---

## CZĘŚĆ C — Połączenie aplikacji z chmurą

1. Otwórz aplikację (adres `*.pages.dev`) na komputerze.
2. Kliknij ikonę **☁️** w prawym górnym rogu.
3. W polu **Adres serwera (Worker URL)** wklej adres z kroku **A5** (np. `https://dziennik-arc.twoja-nazwa.workers.dev`).
4. Podaj e-mail i hasło (min. 8 znaków) → **Załóż konto**.
5. Aplikacja automatycznie zsynchronizuje wpisy. Kropka przy ☁️ zrobi się **zielona**.

### Na telefonie
1. Otwórz ten sam adres `*.pages.dev` w przeglądarce telefonu.
2. **Dodaj do ekranu głównego** (instalacja PWA):
   - **iPhone (Safari):** przycisk *Udostępnij* → *Do ekranu początkowego*.
   - **Android (Chrome):** menu ⋮ → *Zainstaluj aplikację* / *Dodaj do ekranu głównego*.
3. Otwórz aplikację, kliknij ☁️, wklej ten sam Worker URL i **zaloguj się tym samym kontem**.
4. Gotowe — wpisy z komputera pojawią się na telefonie i odwrotnie.

---

## Jak działa synchronizacja (w skrócie)

- Każdy wpis ma znacznik czasu `updated`. Przy synchronizacji wygrywa **nowsza wersja** (last-write-wins).
- Usunięcie wpisu to „nagrobek" (tombstone) — usunięcie propaguje się na inne urządzenia.
- Działa **offline-first**: bez internetu zapisujesz normalnie, a po odzyskaniu sieci dane same się doślą.
- Hasła są hashowane (PBKDF2) — serwer nie przechowuje haseł w postaci jawnej.

---

## Koszty i limity

Wszystko mieści się w **darmowych planach** Cloudflare:
- Workers: 100 000 żądań/dzień.
- D1: 5 GB, miliony odczytów/zapisów miesięcznie.
Dla osobistego dziennika to z ogromnym zapasem.

---

## Najczęstsze problemy

- **„Brak adresu serwera"** — nie wkleiłeś Worker URL w oknie ☁️.
- **Błąd CORS / nie łączy** — sprawdź, czy Worker URL jest dokładny (z `https://`, bez ukośnika na końcu nie szkodzi).
- **„Serwer nie skonfigurowany (brak AUTH_SECRET)"** — pomiń­ąłeś krok **A4**. Wykonaj `npx wrangler secret put AUTH_SECRET` i ponów `npx wrangler deploy`.
- **Zmieniłeś kod Workera?** — po każdej zmianie w `worker/` uruchom ponownie `npx wrangler deploy`.
- **PWA nie instaluje się** — instalacja działa tylko po HTTPS (czyli z adresu `*.pages.dev`), nie z pliku otwartego lokalnie.

---

## Aktualizacja aplikacji w przyszłości
- Zmiany w wyglądzie/logice (`index.html` itd.) → ponów **CZĘŚĆ B** (upload / `pages deploy`).
- Zmiany w backendzie (`worker/`) → ponów `npx wrangler deploy`.
- Po aktualizacji `sw.js` podbij numer wersji cache (`arc-cache-v1` → `v2`), aby wymusić odświeżenie u użytkowników.

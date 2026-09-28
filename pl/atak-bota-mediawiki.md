---
title: "Atak bota na self-hostowaną wiki - analiza incydentu i rekonstrukcja"
layout: page
permalink: /pl/atak-bota-mediawiki/
---

> 🇬🇧 **[Read in English](/posts/bot-attack-mediawiki/)**

**Data incydentu:** 28–30 grudnia 2025  
**Rekonstrukcja:** ~pół roku później, z backupu bazy wykonanego przed remediacją  
**Autor:** Krzysztof Pieczka

---

## TLDR

Prowadzę self-hostowaną wiki (MediaWiki, Linux VPS, ~300 użytkowników dziennie) z własnym formularzem uploadu plików napisanym we Flasku. 29 grudnia 2025 serwis został celowo zaatakowany. Atak był **wieloetapowy i adaptacyjny**: gdy blokowałem jeden wektor, napastnik przełączał się na kolejny. Po spamie formularza i wyłączeniu przeze mnie formularza, atakujący zmienił wektor na **masową rejestrację kont** - w ok. 1h20 (16:17–17:35) założył **856 botowych kont** (77% wszystkich kont serwisu; szczyt: **42 rejestracje na minutę**). Atak zatrzymałem, całkowicie zamykając rejestrację; następnego dnia otworzyłem ją ponownie z CAPTCHA, a po kilku dniach prewencyjnie przeniosłem weryfikację do Cloudflare Turnstile. Pół roku później zrekonstruowałem przebieg ataku co do minuty z backupu bazy wykonanego przed czyszczeniem - poniżej pełna analiza, oś czasu, telemetria i wnioski.

---

## 1. Kontekst i infrastruktura

- **Serwis:** self-hostowana wiki (MediaWiki, MySQL/MariaDB), Linux VPS, Apache (MediaWiki/PHP) + mod_wsgi (Flask).
- **Formularz uploadu:** osobna aplikacja Flask (`/formularz`), baza SQLite, przyjmująca linki, obrazy i filmy.
- **Zabezpieczenia rejestracji na wiki (przed atakiem):** własne pytania weryfikacyjne (MediaWiki ConfirmEdit, moduł **QuestyCaptcha**) - pula ok. 8 losowanych, prostych pytań o wiki. Świadomie dobrane pod ówczesny model zagrożeń: masowe spam-boty wklejające linki (kasyna, rosyjskie strony itp.), które nie znają serwisu i nie odpowiedzą na te pytania. Atak kogoś ze strony społeczności był niespodziewany.
- **Ruch:** ~300 użytkowników dziennie.

**Szczery kontekst:** formularz uploadu powstał dzień przed atakiem (28.12), szybko, bez projektowania pod kątem bezpieczeństwa.

---

## 2. Oś czasu ataku

### Faza 0 - 28.12.2025: postawienie formularza
Uruchomienie formularza uploadu, pierwsze zgłoszenia.

### Faza 1 - 29.12 przed południem: spam formularza + rotacja IP
Bot zaczął masowo wysyłać zgłoszenia przez formularz - głównie **upload plików (video/image)**, nie linki.
- **Pierwsza reakcja:** rate limiting + blokada IP (na poziomie aplikacji oraz Apache: `Require not ip ...` → `403`).
- **Adaptacja napastnika:** przejście na **rotujące adresy IP** - blokada per-IP przestała być skuteczna.
- **Druga reakcja:** blokada po zawartości (deduplikacja/cooldown identycznych zgłoszeń) + rozważana CAPTCHA.
- **Containment:** nie mając pewnego sposobu na odróżnienie legalnego ruchu od bota przy rotacji IP, **czasowo wyłączyłem formularz**, by zatrzymać atak i ochronić resztę serwisu.

> Obserwacja z tamtego dnia: pierwsza fala ruszyła ok. 11:45, a rotacja IP zaczęła się ok. 12:00. Dane, które przetrwały, dotyczą zgłoszeń zaakceptowanych - ruch zablokowany nie zapisał się, więc szczyt tej fazy nie jest widoczny w danych. Podaję te godziny jako obserwację naoczną.

### Faza 2 - 29.12, 16:17–17:35: zmiana wektora na masową rejestrację kont
Po wyłączeniu formularza napastnik **zmienił wektor ataku**: zaczął masowo rejestrować konta na wiki, obchodząc pytania weryfikacyjne. Pula liczyła ok. 8 pytań o stałych, prostych odpowiedziach, a treść losowanego pytania trafiała do formularza jako zwykły tekst. Wystarczyło raz zebrać wszystkie pytania i wpisać je wraz z odpowiedziami do bota jako tablicę. Losowanie z 8 pytań niczego nie utrudnia - to nadal **statyczny sekret**: raz rozwiązany, działa w nieskończoność. Ten etap jest w pełni udokumentowany danymi:

- **856 kont** założonych 29.12 w oknie ok. 1h20 (potwierdzone zapytaniem SQL na polu `user_registration`).
- Rozkład godzinowy: **16:00 → 142, 17:00 → 712, 18:00 → 2.** (83% wolumenu w godzinie 17:00–17:59.)
- Szczyt: **42 rejestracje w ciągu jednej minuty (17:18)**.
- Krzywa (minutowo): rozgrzewka (16:17–16:40, kilka kont/min) → eskalacja (16:43–17:10, do ~15/min) → **3 minuty ciszy (ok. 17:11–17:13)** → pełny szturm (17:14–17:35, 23–42/min) → gwałtowny spadek po 17:35.
- **Interwencja:** spadek po 17:35 to moment, w którym **całkowicie zamknąłem możliwość rejestracji nowych kont**. Szturm się urwał, ale nie do zera - do 18:00 powstało jeszcze **19 kont** (szczegóły i hipotezy w sekcji 3).

### Faza 3 - 30.12: ponowne otwarcie rejestracji z CAPTCHA
Następnego dnia otworzyłem rejestrację z powrotem, dodając **CAPTCHA** (rozszerzenie MediaWiki **ConfirmEdit** z reCAPTCHA - wymaga klucza API). To była pierwsza warstwa po zamknięciu rejestracji.

### Faza 4 - 2.01: Cloudflare Turnstile
Po kilku dniach **prewencyjnie** wymieniłem CAPTCHA na Turnstile, bo uznałem go za lepszą warstwę. Różnica względem CAPTCHA: Turnstile nie daje botowi zagadki do rozwiązania - ocenę „człowiek czy bot” wykonuje Cloudflare na podstawie sygnałów z przeglądarki, a aplikacja jedynie weryfikuje wydany token po stronie serwera. Turnstile przy rejestracji działa do dziś (infrastruktura serwisu zmieniła się od tamtej pory, ale ta warstwa została).

> Formularz po pewnym czasie ostatecznie został wyłączony.

---

## 3. Analiza danych - rekonstrukcja forensyczna

### Źródło danych
Główne źródło: **`backup_before_delete_20251229.sql`** - pełny dump bazy MediaWiki (MySQL) wykonany **29.12 przed usunięciem botowych kont**. Wykonanie backupu przed czyszczeniem okazało się kluczowe - bez niego dane przepadłyby. Dane formularza (baza SQLite) niestety wyczyściłem bez kopii, więc telemetria dotyczy głównie fazy rejestracyjnej.

### Skala
- **856 kont** zarejestrowanych 29.12.2025 (potwierdzone: `SELECT COUNT(*) FROM wiki_user WHERE user_registration LIKE '20251229%'`).
- **1114 kont łącznie** w całej bazie → botowe **856 z 1114 to ~77% wszystkich kont, jakie kiedykolwiek powstały na serwisie**.
- *Uwaga metodologiczna:* pierwotny `grep` na surowym dumpie dawał 1713 dopasowań daty - zawyżone, bo rekord `wiki_user` zawiera dwa pola z timestampem (`user_registration` i `user_touched`). Dokładny wynik daje dopiero zapytanie SQL po samym polu rejestracji. Dobra ilustracja, czemu warto weryfikować dane u źródła, a nie ufać pierwszemu grepowi.

### Krzywa ataku

![Krzywa ataku – rejestracje kont na minutę, 29.12.2025](/assets/img/krzywa-ataku-29122025.png)

*Rejestracje kont bota na minutę.*

| Godzina (29.12) | Liczba rejestracji |
|---|---|
| 16:00–16:59 | 142 |
| 17:00–17:59 | 712 |
| 18:00 | 2 |

Kształt wykresu to typowa sygnatura zautomatyzowanego ataku.

### Przerwa przed szturmem
Najciekawszy moment na wykresie to nie szczyt, tylko **trzy minuty ciszy tuż przed nim**. Do ok. 17:10 bot rejestruje w tempie kilku–kilkunastu kont na minutę, potem przez ok. 17:11–17:13 nie powstaje ani jedno konto, a od 17:14 tempo skacze skokowo do 23–37 kont/min i utrzymuje się na tym poziomie przez ~20 minut.

Nie wygląda to na naturalne przyspieszanie narzędzia, tylko na **zatrzymanie i ponowne uruchomienie z inną konfiguracją** - najprawdopodobniej z większą liczbą wątków/równoległych sesji. Skokowa zmiana przepustowości po krótkiej pauzie sugeruje, że napastnik **aktywnie obserwował atak i go stroił.** Pasuje to do reszty obrazu: zmiana wektora po wyłączeniu formularza i ręczne konta testowe przed automatem.


### Ogon po zamknięciu rejestracji
Po zamknięciu rejestracji (~17:36) powstało jeszcze **19 kont**: 7 o 17:37, potem pojedyncze paczki (4, 5, 1) do 17:59 i 2 o 18:00. Szturm się urwał, ale nie do zera. Dwie hipotezy:

1. **Zamknięcie nie było natychmiastowym ruchem** - np. pierwsza zmiana konfiguracji nie domknęła wszystkiego od razu, a część żądań przechodziła jeszcze przez kilkanaście minut.
2. **Inna ścieżka rejestracji.** Jeśli zamknięcie objęło tylko stronę rejestracji (formularz), a nie samo uprawnienie do tworzenia kont, konta dało się dalej zakładać np. przez API MediaWiki (`action=createaccount`).

Dane z backupu tego nie rozstrzygają, a logów Apache z tego dnia już nie ma. Zostawiam to jako otwarte pytanie.

### Sygnatura generatora nazw (mini threat-intel)
Rozkład „rodzin" nazw botowych kont ujawnia mechanikę narzędzia:

| Wzorzec nazwy | Liczba kont | Interpretacja |
|---|---|---|
| `DawidKamilPatryk NNNNNNN` (jedna baza + losowy 7-cyfrowy sufiks) | 844 | trzon ataku, generowany maszynowo |
| `PentestUser NNNNN` | 5 | domyślne nazwy narzędzia |
| `Tester NNNN` | 4 | faza testowa |
| `BotTest` | 1 | test przed automatem |
| pojedyncze nazwy wpisane ręcznie (MatiPiw, Kuszot) | 2 | ręczna rozgrzewka przed automatem |

**Wnioski z sygnatury:**
-  Prefiks to nawiązanie do **rozpoznawalnej polskiej postaci internetowej** - co wskazuje na **celowy, ukierunkowany atak** (napastnik znający środowisko/serwis), a nie przypadkowy skan botneta.
- Obecność nazw typu `PentestUser`/`Tester`/`BotTest` sugeruje użycie gotowego narzędzia do rejestracji/spamu z domyślnym nazewnictwem.

---

## 4. Wnioski
- **Przeciwnik może być inteligentny i adaptacyjny**. Każda pojedyncza obrona - blokada IP, pytanie weryfikacyjne - była obchodzona zmianą podejścia po stronie napastnika. Atak zakończył się w momencie zamknięcia rejestracji, a lukę, którą bot wykorzystał, zamknęła reCAPTCHA, a potem Cloudflare Turnstile. Łatanie pojedynczych wektorów nie skaluje się przeciw adaptacyjnemu napastnikowi.
- **Blokada per-IP nie działa przeciw rotacji IP.** Sensowna pierwsza linia, ale łatwa do obejścia.
- **Containment to dobra decyzja, nie porażka.** Wyłączenie formularza, a potem rejestracji ograniczyło szkody i dało czas na lepszą obronę.
- **Statyczne pytania były dobre - ale pod inny model zagrożeń.** Odsiewały spam-boty, ale to statyczny sekret: kto zna społeczność, zbiera odpowiedzi raz i automatyzuje w nieskończoność.
- **Dowody zabezpiecza się przed sprzątaniem.** Backup bazy wiki przed usunięciem kont uratował całą analizę; bazę formularza wyczyściłem bez kopii i ta faza przepadła. Logi Apache (`rotate 14`) też nie dotrwały do rekonstrukcji.

**Co zrobiłbym inaczej:** Turnstile i rate limiting na brzegu od startu, a nie reaktywnie; polityka backupów i retencji logów ustalona przed wystawieniem serwisu, a nie po incydencie.

---

*Analiza oparta na danych odzyskanych z pełnego backupu bazy MediaWiki wykonanego przed remediacją (29.12.2025).*

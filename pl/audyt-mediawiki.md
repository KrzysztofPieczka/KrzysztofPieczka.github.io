---
title: "Audyt bezpieczeństwa własnego serwera MediaWiki"
layout: post
permalink: /pl/audyt-mediawiki/
categories: [Security Audit]
tags: [mediawiki, apache, cloudflare, web-security]
description: "White-box audyt własnej wiki na MediaWiki przed migracją. Jak jeden włączony directory listing wystawił źródło aplikacji i bazę z danymi userów."
date: 2026-10-06 12:00:00 +0200
image:
  path: /assets/img/audit/audit-thumb.png
  alt: "Security audit - directory listing"
---

> 🇬🇧 **[Read in English](/posts/audit-mediawiki/)**

Od sierpnia 2025 prowadzę self-hostowaną wiki na MediaWiki. Postawiłem ją sam na VPS-ie i dziś ma około 300 użytkowników dziennie. Robiłem ją po omacku z użyciem AI: uczyłem się na bieżąco i dokładałem rzeczy wtedy, kiedy były potrzebne. W grudniu 2025 dorobiłem do niej własny formularz uploadu w Python Flask, szybko i bez myślenia o bezpieczeństwie. Dwa dni później serwis został zaatakowany, co opisałem w [osobnym wpisie](/pl/atak-bota-mediawiki/).

We wrześniu 2026 zdecydowałem się na migrację wiki na inny VPS i wykorzystałem tę okazję, żeby porządnie prześwietlić cały serwis przed migracją. Przez ten rok sporo się nauczyłem w kwestii bezpieczeństwa i dziś rozumiem dużo więcej niż wtedy, kiedy to wszystko stawiałem. Zanim cokolwiek przeniosłem, przejrzałem wszystko i spisałem błędy, które przez ten czas w nim siedziały.

Audyt jest white-box: to moja własna infrastruktura, znam kod i mam dostęp do wszystkiego. To niepowtarzalna okazja, bo na własnym systemie mogę sprawdzić rzeczy, na które przy cudzym nigdy bym sobie nie pozwolił.

> Wszystko poniżej to stan sprzed migracji. Stan na październik 2026 to świeżo postawiona strona na nowym VPS-ie, każdy ujawniony sekret został zrotowany. Nic z tego nie da się dziś odtworzyć!

W tym wpisie opisuję najciekawsze znaleziska. Pełny raport ze wszystkimi findingami, PoC i screenami: **[pobierz PDF](/assets/audit-report.pdf)**.

## 1. Directory listing ON (HIGH)

Miałem włączony directory listing i tak naprawdę każdy mógł pobrać bazę z danymi userów. W konfiguracji Apache było `Options Indexes`, więc każdy katalog bez pliku `index` był po prostu listowany. Wejście na `/website/` zwracało „Index of /website” i pokazywało, co tam leży: `formularz/` i `upload/`.

![Index of /website - directory listing przez Cloudflare](/assets/img/audit/audit1.png)


A w środku leży `app.py`. Formularz uploadu (aplikacja Flask) leżał fizycznie w docroot Apache. Normalnie ruch szedł przez reverse proxy na `/formularz/` i obsługiwał go Flask. Ale skoro pliki fizycznie leżą w docroot, to ta sama aplikacja jest osiągalna drugą drogą: pod `/website/formularz/...` Apache serwuje ją statycznie, z pominięciem Flaska. Apache nie ma handlera do `.py` ani `.db`, więc zamiast wykonać te pliki, oddaje je jako zwykły plik do pobrania.

```
$ curl -s -o /dev/null -w "%{http_code}" http://127.0.0.1/website/formularz/app.py
200

GET https://[REDACTED]/website/formularz/app.py
StatusCode: 200 | Content-Type: text/x-python | Content-Length: 8466
```

![app.py serwowane przez Apache, hasło zamaskowane](/assets/img/audit/audit2.png)

W tym źródle było hasło admina otwartym tekstem i klucz secret Turnstile - ten sam, którego używa produkcyjna wiki. Z jednego pobranego pliku dostajesz klucze do panelu admina formularza i sekret, który miał chronić rejestrację na całej wiki.

Obok, w tym samym katalogu, leżała baza `submissions.db`. Też do pobrania jednym GET-em:

![Pobranie submissions.db](/assets/img/audit/audit3.png)

W bazie były nicki, opisy, realne IP zgłaszających, ścieżki plików i timestampy. Baza była pobieralna od momentu wrzucenia formularza do docroot (katalog datowany 27.12.2025) do migracji, czyli jakieś 9 miesięcy. To **realne dane osobowe publicznie dostępne przez cały ten czas**.

Cały łańcuch: directory listing pokazuje pliki → apka WSGI w docroot → Apache serwuje jej źródło jako plik statyczny → w źródle hardcoded sekrety → publiczne ujawnienie hasła admina i klucza Turnstile → dostęp do panelu admina i całej bazy z danymi userów.

Naprawa: wyłączyć directory listing (`Options -Indexes`) i przenieść WSGI poza docroot (np. `/opt/formularz`). Jeden włączony listing i jeden źle położony katalog zamieniły wewnętrzną apkę w publiczne źródło i bazę z danymi userów.

## 2. Serwer osiągalny z pominięciem Cloudflare (MEDIUM)

Po ataku bota cała ochrona serwisu stała na Cloudflare: challenge, rate-limit, reguły WAF. Ruch miał iść przez CF, a dopiero potem do mojego serwera. Problem w tym, że serwer odpowiadał każdemu, kto zapukał bezpośrednio na jego IP.

Wystarczy wysłać request na IP serwera i podać domenę w nagłówku `Host`:
```
$ curl -skI -H "Host: [REDACTED]" https://[REDACTED_IP]/
HTTP/1.1 200 OK
Server: Apache/2.4.58 (Ubuntu)
```

Serwer oddaje pełną stronę wiki, a Cloudflare w ogóle nie bierze w tym udziału. Nie ma challenge'a ani rate-limitu, a reguły WAF nie mają nawet szansy zadziałać. Wszystko, co stawiałem po ataku bota, **dało się obejść jednym nagłówkiem**.

Prawdziwe IP serwera da się znaleźć bez większego wysiłku: w historii DNS (zanim domena trafiła za Cloudflare, wskazywała prosto na serwer) albo w Shodanie, który skanuje cały internet i indeksuje, co odpowiada na danym adresie.


Naprawa: firewall na portach 80/443 z dostępem wyłącznie dla zakresów IP Cloudflare, a dla całej reszty drop. Wtedy jedyna droga do serwera prowadzi przez CF i ochrona tam postawiona faktycznie ma sens.

## 3. Połowa stacku bez ścieżki aktualizacji (HIGH)

MediaWiki chodziło w wersji 1.43.0. To pierwsze wydanie tej gałęzi, z grudnia 2024, i od postawienia serwisu nie dostało ani jednej z kolejnych łatek. Najnowsze wydanie tej samej gałęzi to dziś 1.43.9, czyli jakieś 9 wydań poprawek, których u mnie nigdy nie było. Co więcej, 1.43.0 było nieaktualne już w momencie instalacji.

![Special:Version - wersje zainstalowanego oprogramowania](/assets/img/audit/audit4.png)

Nie całość serwera była zaniedbana. Apache, PHP i MySQL były aktualne (data builda Apache to lipiec 2026), czyli system łatał je na bieżąco. I tu jest sedno: te komponenty instaluje `apt`, więc `apt upgrade` sam dociąga łatki bezpieczeństwa. MediaWiki i rozszerzenia postawiłem z tarballa, poza `apt`, więc żaden `apt upgrade` ich nie widzi i nie rusza. Połowa stacku aktualizowała się sama, a **druga połowa, ta najbardziej wystawiona na świat, nie miała żadnego mechanizmu aktualizacji**. Nikt jej nie pilnował. Wtedy w ogóle nie siedziałem w bezpieczeństwie i nie miałem świadomości, że stara wersja to realny problem (głównie XSS), a nie kosmetyka. Zakładałem, że skoro system się łata, to łata się wszystko.

W tych 9 wydaniach siedzi głównie rodzina błędów XSS/escaping i kilka info-disclosure (wyciek metadanych, ukrytych nazw userów itp.). Część ryzyka zbijały kontrole, które zostały po ataku bota: wyłączona edycja anonimowa i Turnstile na rejestracji. Ale XSS przez treść strony dotyczy też zalogowanych, zaufanych edytorów, więc samo „obcy nic nie wrzuci” tu nie wystarcza.

Dokładnie ten sam problem dotyczył rozszerzeń. EmbedVideo chodziło w wersji z 2022 roku, jakieś 4 lata bez aktualizacji, z tego samego powodu: third-party poza `apt`, którego nic nie pilnowało. Akurat to rozszerzenie osadza zewnętrzną treść (URL wideo) i przetwarza input, czyli jest typowym polem na XSS, więc nieaktualne boli tu podwójnie.

Naprawa nie polega na „kliknięciu update”. Automatyczna aktualizacja MediaWiki to zły pomysł, bo wydanie potrafi wymagać migracji schematu bazy (`maintenance/update.php`) i umie wywrócić rozszerzenia. Co faktycznie zamyka dziurę:
- świeża instalacja z aktualnego tarballa przy migracji, bez przenoszenia core 1:1,
- zapis na listę `mediawiki-announce`, żeby w ogóle wiedzieć, że wyszło wydanie bezpieczeństwa,
- dla warstwy systemowej (`apt`) `unattended-upgrades` z samymi security updates, czyli domknięcie tej połowy, która i tak działała.

## Wnioski

Wszystkie trzy znaleziska to w gruncie rzeczy ten sam błąd: za każdym razem **bezpieczeństwo opierało się na jednym zabezpieczeniu**. Sekrety „chronione” tym, że nikt nie zna ścieżki do pliku. Cały ruch „chroniony” tym, że przechodzi przez Cloudflare. Aktualność serwera „zapewniona” założeniem, że `apt` ogarnia wszystko. W każdym przypadku wystarczyło podważyć to jedno założenie i cała ochrona znikała, bo pod spodem nie było już nic.

Najgroźniejszy nie był żaden pojedynczy błąd, tylko to, że się łączyły. Directory listing sam w sobie to drobiazg. Formularz w docroot sam w sobie to drobiazg. Dopiero razem dały pobranie bazy z danymi userów przez jeden klik. Każdy finding osobno brzmi niewinnie, łańcuch już nie.

Z perspektywy czasu najwięcej dało mi nie samo łatanie, tylko patrzenie na własny system oczami kogoś, kto chce się włamać. Rok temu bym tych rzeczy nie zobaczył, bo budowałem „żeby działało”, a nie „żeby nie dało się tego obejść”. To są dwa różne sposoby patrzenia na ten sam kod.
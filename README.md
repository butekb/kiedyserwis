# KiedySerwis – Dokumentacja Developerska

> **Dla nowych sesji AI / nowych deweloperów.**  
> Ten dokument opisuje kompletną architekturę wtyczki, jej moduły, kluczowe funkcje i zasady rozbudowy tak, żeby niczego nie zepsuć.

---

## Spis treści

1. [Przegląd i cel wtyczki](#1-przegląd-i-cel-wtyczki)
2. [Struktura plików](#2-struktura-plików)
3. [Instalacja i aktywacja](#3-instalacja-i-aktywacja)
4. [Niestandardowe typy wpisów (CPT)](#4-niestandardowe-typy-wpisów-cpt)
5. [Role użytkowników](#5-role-użytkowników)
6. [Schemat danych – user meta i post meta](#6-schemat-danych--user-meta-i-post-meta)
7. [System planów (Free / Premium / Multi)](#7-system-planów-free--premium--multi)
8. [Bramka SMS – smsgateway24.com](#8-bramka-sms--smsgateway24com)
9. [System Cron i kolejka SMS](#9-system-cron-i-kolejka-sms)
10. [Zewnętrzny endpoint REST (run-sms-queue)](#10-zewnętrzny-endpoint-rest-run-sms-queue)
11. [Webhook płatności (1koszyk.pl)](#11-webhook-płatności-1koszykpl)
12. [Panel Serwisanta (frontend dashboard)](#12-panel-serwisanta-frontend-dashboard)
13. [Moduł Multi-Account](#13-moduł-multi-account)
14. [Moduł SEO (Wizytówka)](#14-moduł-seo-wizytówka)
15. [Moduł Protokół Serwisowy](#15-moduł-protokół-serwisowy)
16. [Moduł Magazyn](#16-moduł-magazyn)
17. [Panel Admina WordPress](#17-panel-admina-wordpress)
18. [Kluczowe opcje wp_options](#18-kluczowe-opcje-wp_options)
19. [Shortcodes](#19-shortcodes)
20. [Rewrite rules i wizytówka publiczna](#20-rewrite-rules-i-wizytówka-publiczna)
21. [Bezpieczeństwo – zasady nienaruszalne](#21-bezpieczeństwo--zasady-nienaruszalne)
22. [Logi i debugowanie](#22-logi-i-debugowanie)
23. [Jak dodać dostawcę SMS: SMSAPI.pl](#23-jak-dodać-dostawcę-sms-smssapipl)
24. [Jak dodać nowe funkcje premium](#24-jak-dodać-nowe-funkcje-premium)
25. [Integracja płatności 1koszyk.pl – gotowy handler](#25-integracja-płatności-1koszykpl--gotowy-handler)
26. [Pułapki i ważne niuanse](#26-pułapki-i-ważne-niuanse)
27. [Testowalność i debugowanie ręczne](#27-testowalność-i-debugowanie-ręczne)

---

## 1. Przegląd i cel wtyczki

**KiedySerwis Core** to wtyczka WordPress dla platformy **kiedyserwis.pl** – SaaS dla serwisantów urządzeń (AGD, klimatyzacje, kotły itp.). Każdy serwisant po rejestracji dostaje:

- Prywatny panel (`/panel-serwisanta/`) z bazą obsługiwanych urządzeń
- Automatyczne przypomnienia SMS do klientów przed terminem serwisu
- Kalendarz zleceń
- Publiczną wizytówkę (`/serwisant/nazwa-firmy/`)
- (Premium) Protokoły serwisowe e-mail, Magazyn części, SEO, Multi-Account

Cały kod logiki biznesowej żyje w **jednym pliku głównym** + 3 klasach + 7 szablonach.

---

## 2. Struktura plików

```
kiedyserwis-core/
├── kiedyserwis-core.php          ← GŁÓWNY PLIK (3 100 linii) – logika, handlery, shortcodes, REST, cron
├── inc/
│   ├── class-ks-admin-users.php  ← Rozszerzenie panelu Users w WP-Admin (kolumny, filtry, profil)
│   ├── class-ks-multi-account.php← Moduł Multi-Account (manager + pracownicy)
│   └── class-ks-seo-handler.php  ← Obsługa meta SEO i Schema.org dla wizytówek
└── templates/
    ├── auth-forms.php             ← Formularze logowania/rejestracji/aktywacji (shortcode [ks_dashboard])
    ├── dashboard-view.php         ← Główny panel serwisanta (zakładki devices/calendar/wizytowka/...)
    ├── author-card.php            ← Publiczna wizytówka (/serwisant/slug/)
    ├── cennik-shortcode.php       ← Shortcode [ks_cennik] – publiczna strona cennika
    ├── premium-magazyn.php        ← Panel Magazyn (Premium)
    ├── premium-protokol.php       ← Panel Protokół Serwisowy (Premium)
    └── premium-seo.php            ← Panel SEO (Premium)
```

**Klasy są inicjalizowane na początku kiedyserwis-core.php:**
```php
require_once plugin_dir_path( __FILE__ ) . 'inc/class-ks-seo-handler.php';
new KS_SEO_Handler();

require_once plugin_dir_path( __FILE__ ) . 'inc/class-ks-admin-users.php';
new KS_Admin_Users();

require_once plugin_dir_path( __FILE__ ) . 'inc/class-ks-multi-account.php';
new KS_Multi_Account();
```

---

## 3. Instalacja i aktywacja

Hook `register_activation_hook` (`ks_activation_handler`) robi przy aktywacji:
1. Generuje `ks_cron_secret` (token zewnętrznego crona)
2. Generuje `ks_webhook_secret` (token webhooka 1koszyk.pl)
3. Planuje WP-Cron: `ks_daily_sms_cron` co minutę (`ks_every_minute`)
4. Tworzy rolę `serwisant`
5. Tworzy role Multi-Account (`service_manager`, `service_worker`) – przez `KS_Multi_Account::register_roles()`
6. Rejestruje CPT i rewrite rules → `flush_rewrite_rules()`

Hook deaktywacji (`ks_deactivation_unschedule`) usuwa zaplanowany cron.

> ⚠️ **Po każdej zmianie rewrite rules (dodaniu nowych) musisz** wywołać `flush_rewrite_rules()` lub przejść do **Ustawienia → Bezpośrednie odnośniki** i kliknąć Zapisz.

---

## 4. Niestandardowe typy wpisów (CPT)

### `ks_urzadzenie` – Urządzenie serwisowe

| Właściwość | Wartość |
|---|---|
| `public` | `false` |
| `publicly_queryable` | `false` |
| `show_in_rest` | `false` |
| `show_ui` | `true` (widoczne w WP-Admin) |
| `supports` | `title`, `author` |

> **Dlaczego `public=false`?** CPT zawiera dane PII klientów (numery telefonów). Wyłączenie publicznego dostępu chroni przed przypadkowym ujawnieniem przez WordPress REST API (`/wp-json/wp/v2/ks_urzadzenie`).

**Post meta urządzenia:**

| Klucz | Opis |
|---|---|
| `ks_klient_tel` | Numer telefonu klienta |
| `ks_adres_urzadzenia` | Adres urządzenia |
| `ks_data_serwisu` | Data ostatniego serwisu (`Y-m-d`) |
| `ks_interwal` | Interwał serwisu w miesiącach |
| `ks_dni_wyprzedzenia` | Ile dni przed serwisem wysłać SMS |
| `ks_notatki` | Notatki serwisowe (z datownikami) |
| `ks_status_sms` | `Oczekuje` / `Wysłano` / `Błąd` |
| `ks_last_sms_date` | Data ostatniego wysłanego SMS (`Y-m-d`) |
| `ks_app_date` | Data wizyty z kalendarza |
| `ks_app_time` | Godzina wizyty z kalendarza |
| `ks_is_custom_app` | `'true'` jeśli to wpis kalendarza (nie urządzenie) |
| `ks_parts_history` | JSON – historia użytych części po serwisach |

### `ks_czesc` – Część magazynowa (Premium)

| Właściwość | Wartość |
|---|---|
| `public` | `false` |
| `show_ui` | `false` |
| `supports` | `title`, `author` |

**Post meta części:**

| Klucz | Opis |
|---|---|
| `ks_stan` | Aktualny stan magazynowy (int) |
| `ks_minimum` | Minimalna ilość (notyfikacja gdy stan ≤ minimum) |

---

## 5. Role użytkowników

| Rola | Opis | Capabilities |
|---|---|---|
| `serwisant` | Zwykły serwisant | `read`, `edit_posts`, `publish_posts`, `upload_files` |
| `service_manager` | Manager Multi-Account | j.w. + może zarządzać pracownikami |
| `service_worker` | Pracownik w systemie Multi-Account | j.w. (dziedziczy plan od managera) |
| `administrator` | Administrator WordPress | pełne uprawnienia |

**Ważne:** Serwisanci są przekierowywani z `/wp-admin/` do `/panel-serwisanta/` (hook `admin_init`). Admin bar jest ukryty dla roli `serwisant`.

---

## 6. Schemat danych – user meta i post meta

### User meta – dane firmy

| Klucz | Opis |
|---|---|
| `ks_company_name` | Nazwa firmy |
| `ks_company_phone` | Telefon firmy (na wizytówce) |
| `ks_company_desc` | Opis firmy |
| `ks_company_city` | Miasto |
| `ks_company_url` | URL strony firmy |
| `ks_company_logo` | ID załącznika (logo) |
| `ks_company_slug` | Slug URL wizytówki |

### User meta – plan / subskrypcja

| Klucz | Opis |
|---|---|
| `ks_plan_status` | `free` / `premium` / `multi` |
| `ks_plan_expiry` | Unix timestamp wygaśnięcia |
| `ks_free_limit` | Limit urządzeń dla free (domyślnie 10) |
| `ks_plan_days` | Ile dni kupiono (informacyjnie) |

### User meta – SEO (Premium)

| Klucz | Opis |
|---|---|
| `ks_seo_title` | Własny `<title>` strony |
| `ks_seo_description` | Meta description |
| `ks_seo_cities` | Obsługiwane miasta (przecinkami) |
| `ks_seo_services` | Rodzaje usług |
| `ks_seo_brands` | Obsługiwane marki |
| `ks_seo_category` | Kategoria biznesu |
| `ks_seo_category_custom` | Kategoria własna |
| `ks_opening_hours` | Godziny otwarcia |

### User meta – Protokół (Premium)

| Klucz | Opis |
|---|---|
| `ks_protocol_enabled` | `'1'` jeśli protokół włączony |
| `ks_protocol_checklist` | Tekst; każda linia = jeden punkt listy |
| `ks_protocol_footer` | Stopka protokołu |

### User meta – Multi-Account

| Klucz | Opis |
|---|---|
| `_parent_manager_id` | (service_worker) ID managera |

---

## 7. System planów (Free / Premium / Multi)

### Kluczowe funkcje

```php
ks_get_plan_data( $user_id )    // → array( 'status', 'expiry', 'limit' )
ks_is_premium_active( $user_id ) // → bool (true gdy premium/multi i nie wygasło)
ks_can_add_device( $user_id )    // → bool (uwzględnia limit free)
```

### Logika działania

- **Free**: max 10 urządzeń (konfigurowalny przez `ks_free_limit` w user meta)
- **Premium** (`ks_plan_status = 'premium'`): nieograniczona liczba urządzeń + wszystkie moduły premium
- **Multi** (`ks_plan_status = 'multi'`): jak premium + system Sub-kont (rola `service_manager`)
- Gdy premium wygaśnie (`ks_plan_expiry < time()`): `ks_get_plan_data()` automatycznie downgrade'uje do `free` i aktualizuje meta
- `service_worker` **dziedziczy plan od swojego managera** – sprawdź w `ks_get_plan_data()`

### Ręczna zmiana planu (admin)

Panel WordPress → Użytkownicy → Edytuj użytkownika → sekcja **Ustawienia KiedySerwis Premium**.  
Pola: Typ planu (dropdown) + Data wygaśnięcia.

---

## 8. Bramka SMS – smsgateway24.com

### Jak działa (Android Gateway)

1. Wtyczka loguje się do `smsgateway24.com` używając e-maila i hasła (dane konta administratora platformy).
2. API zwraca **jednorazowy token sesji**.
3. Wtyczka wysyła SMS przez ten token na podłączony telefon Android (Device ID).
4. SMS fizycznie wychodzi z **karty SIM** telefonu Android.

> **Wymagany plan:** `Startup` (jedno urządzenie). Plan `Profi` (API requests) to inny produkt – nie jest używany.

### Endpointy API

```
POST https://smsgateway24.com/getdata/gettoken
  body: { email, pass }
  response: { error: 0, token: "..." }

POST https://smsgateway24.com/getdata/addsms
  body: { token, sendto, body, device_id, sim }
  response: { error: 0, sms_id: 12345, message: "..." }
```

### Funkcja ks_send_sms()

```php
function ks_send_sms( $phone, $message ) : bool
```

- Normalizuje numer telefonu: usuwa spacje/myślniki, dodaje `+48` jeśli brak kodu kraju
- Czyta `ks_sms_provider` z opcji (aktualnie `android`, w przyszłości `smsapi`)
- Logi sukcesu/błędu przez `ks_log()`
- **NIE loguje numeru telefonu** – tylko ID urządzenia i sms_id

### Opcje konfiguracji SMS (wp_options)

| Opcja | Opis |
|---|---|
| `ks_sms_provider` | `android` (jedyna działająca) lub `smsapi` (zarezerwowane) |
| `ks_sg24_email` | E-mail do konta smsgateway24.com |
| `ks_sg24_password` | Hasło do konta smsgateway24.com |
| `ks_sg24_device_id` | ID urządzenia Android z panelu smsgateway24.com |
| `ks_sg24_sim_slot` | `0` = SIM 1, `1` = SIM 2 |
| `ks_sms_message_template` | Szablon SMS; zmienne: `[Nazwa]`, `[Firma]`, `[Link]` |
| `ks_sms_send_hour` | Godzina wysyłki SMS (0–23, domyślnie 10) |

---

## 9. System Cron i kolejka SMS

### Architektura

```
┌─────────────────────────────────────────────────────────────────┐
│  Co minutę (od skonfigurowanej godziny):                        │
│                                                                 │
│  [WP-Cron: ks_daily_sms_cron] ─────→ ks_process_daily_sms()   │
│                                           ↓                     │
│  [Zewnętrzny cron serwera] ──────→ REST: /ks/v1/run-sms-queue  │
│                                           ↓                     │
│                              ks_handle_external_cron()          │
│                                           ↓                     │
│                              ks_run_sms_batch_for_today()       │
│                                     (wspólna logika)            │
└─────────────────────────────────────────────────────────────────┘
```

### ks_run_sms_batch_for_today()

Główna funkcja – przetwarzanie wsadowe:

1. Jeśli `get_current_user_id() === 0` (kontekst unauthenticated – zewnętrzny cron): tymczasowo przełącza na konto pierwszego administratora, aby `get_posts()` zwracało private CPT
2. Przetwarza urządzenia partiami po **200** (`$batch_size = 200`)
3. Dla każdego urządzenia oblicza: `next_service_date = last_service + interval_months`, `trigger_date = next_service - days_before`
4. Pomija urządzenia z `ks_status_sms = 'Wysłano'`
5. Mechanizm deduplikacji: `ks_last_sms_date === today` → pomija (zapobiega wielokrotnemu wysłaniu przy cronie co minutę)
6. Wysyła przez `do_action('ks_trigger_sms_reminder', $phone, $message, $device_id)`
7. Po sukcesie: ustawia `ks_status_sms = 'Wysłano'` + `ks_last_sms_date = today`
8. Przywraca poprzedniego użytkownika (`wp_set_current_user`)

### Harmonogram

WP-Cron jest ustawiony na `ks_every_minute` (co minutę). Godzina wysyłki jest sprawdzana wewnątrz `ks_process_daily_sms()` – jeśli `current_hour < send_hour`, funkcja kończy bez działania.

**Migracja starego harmonogramu**: `plugins_loaded` hook automatycznie migruje ze starego `daily` do `ks_every_minute` przy aktualizacji.

### Reset statusu SMS

Gdy serwisant loguje serwis lub edytuje urządzenie, status wraca do `Oczekuje` i usuwane jest `ks_last_sms_date`. To pozwala na ponowne wysłanie SMS po kolejnym serwisie.

---

## 10. Zewnętrzny endpoint REST (run-sms-queue)

```
POST /wp-json/ks/v1/run-sms-queue?token=SECRET
```

- Autentykacja: `hash_equals()` tokena z `ks_cron_secret`
- Zwraca JSON: `{ status: 'ok', sent: N, time: '...' }` lub `{ status: 'skipped', reason: 'too_early' }`
- Nagłówki `Cache-Control: no-store` – nigdy nie będzie buforowany
- **Używaj metody POST** w komendzie crona (GET może być buforowany przez LiteSpeed/Nginx)

**Komenda crona serwera (przykład):**
```bash
curl --silent -X POST "https://kiedyserwis.pl/wp-json/ks/v1/run-sms-queue?token=TOKEN" > /dev/null 2>&1
```

**Regeneracja tokena**: Admin → Ustawienia → KiedySerwis → przycisk "Regeneruj token"  
Uwaga: po regeneracji trzeba zaktualizować URL w panelu Seohost.pl.

---

## 11. Webhook płatności (1koszyk.pl)

```
POST /wp-json/ks/v1/payment-webhook
Header: X-KS-Webhook-Secret: SECRET
Body (JSON/form): { status, product_id, attr1: USER_ID, ... }
```

### Jak to działa

1. 1koszyk.pl wywołuje webhook po udanej płatności
2. Wtyczka weryfikuje sekret (`hash_equals`)
3. Odczytuje `attr1` = WordPress user ID (musi być ustawiony w 1koszyk.pl jako parametr produktu)
4. Na podstawie `product_id` oblicza liczbę dni:
   - `product_id` zawiera `'365'` → 365 dni
   - wszystko inne → 30 dni (domyślnie; **rozszerz tę logikę dla planów 6-miesięcznych!**)
5. Aktualizuje `ks_plan_status = 'premium'` i `ks_plan_expiry`
6. Jeśli klient odnawia przed wygaśnięciem: dodaje dni do aktualnego `ks_plan_expiry`

### Konfiguracja w 1koszyk.pl

- URL webhooka: widoczny w Ustawienia → KiedySerwis → sekcja "Webhook płatności"
- Sekret: widoczny obok URL (skopiuj go do 1koszyk.pl)
- Parametr `attr1`: musi zawierać WordPress user ID kupującego

> ⚠️ **TODO dla 6-miesięcznego planu**: Aktualnie jeśli `product_id` nie zawiera `'365'`, zawsze przypisuje 30 dni. Przy dodaniu planu 6-miesięcznego i Multi dodaj warunki dla `'180'` → 180 dni i `'firma'` → 365 dni + rola `multi`.

---

## 12. Panel Serwisanta (frontend dashboard)

### Dostęp

Strona WordPress z shortcodem `[ks_dashboard]` na URL `/panel-serwisanta/`.

- Niezalogowany → formularze logowania/rejestracji (`templates/auth-forms.php`)
- Zalogowany serwisant → panel (`templates/dashboard-view.php`)
- Brak uprawnień `edit_posts` → komunikat błędu

### Zakładki panelu

| `ks_tab` | Zawartość |
|---|---|
| `devices` (domyślna) | Lista urządzeń z filtrowaniem i sortowaniem |
| `calendar` | Kalendarz zleceń |
| `wizytowka` | Edycja wizytówki publicznej |
| `magazyn` | Magazyn części (Premium) |
| `protokol` | Protokoły serwisowe (Premium) |
| `seo` | Ustawienia SEO (Premium) |
| `plan` | Twój plan – cennik i stan subskrypcji |
| `multi` | Zarządzanie Multi-Account |
| `tutorial` | Samouczek |
| `sugestie` | Formularz sugestii |

### Handlery formularzy (admin-post.php)

Wszystkie formularze wysyłają na `action="<?php echo admin_url('admin-post.php'); ?>"`. Każdy handler:
1. Sprawdza `is_user_logged_in()` + `current_user_can('edit_posts')`
2. Weryfikuje nonce (`wp_verify_nonce`)
3. Weryfikuje własność zasobu (`ks_verify_device_ownership()` / `ks_verify_part_ownership()`)
4. Sanitizuje dane
5. Przekierowuje przez `wp_safe_redirect()`

| Action | Handler |
|---|---|
| `ks_add_device` | `ks_handle_device_creation()` |
| `ks_log_service` | `ks_handle_service_log()` |
| `ks_save_appointment` | `ks_handle_save_appointment()` |
| `ks_save_card` | `ks_handle_save_business_card()` |
| `ks_save_custom_app` | `ks_handle_save_custom_app()` |
| `ks_complete_app` | `ks_handle_complete_appointment()` |
| `ks_delete_entry` | `ks_handle_delete_entry()` |
| `ks_delete_device` | `ks_handle_delete_device()` |
| `ks_delete_logo` | `ks_handle_delete_logo()` |
| `ks_send_manual_sms` | `ks_handle_manual_sms_send()` |
| `ks_save_protocol_settings` | `ks_handle_save_protocol_settings()` |
| `ks_send_protocol` | `ks_handle_send_protocol()` |
| `ks_save_czesc` | `ks_handle_save_czesc()` |
| `ks_delete_czesc` | `ks_handle_delete_czesc()` |
| `ks_save_seo` | `KS_SEO_Handler::handle_save_seo_settings()` |
| `ks_send_suggestion` | `ks_handle_suggestion()` |
| `ks_force_run_cron` | `ks_handle_force_run_cron()` (admin only) |
| `ks_regenerate_cron_secret` | `ks_handle_regenerate_cron_secret()` (admin only) |
| `ks_admin_queue_send_sms` | `ks_handle_admin_queue_send_sms()` (admin only) |

### Rejestracja (auth-flow)

1. Formularz → `admin_post_nopriv_ks_register` → `ks_handle_registration()`
2. Dane zapisywane w transient (`ks_reg_KEY`) na 48h (nie w bazie, żeby boty nie zapychały tabel)
3. E-mail aktywacyjny z linkiem do `?ks_action=verify&ks_key=KEY`
4. Kliknięcie linku → formularz ustawienia hasła → `admin_post_nopriv_ks_activate` → `ks_handle_activation()`
5. Tworzony użytkownik z rolą `serwisant`, usuwany transient

---

## 13. Moduł Multi-Account

**Plik:** `inc/class-ks-multi-account.php`

### Architektura

```
service_manager (1 osoba)
    └── service_worker 1
    └── service_worker 2  (max 10 łącznie)
    └── ...
```

- Manager zarządza swoją firmą (urządzenia, protokoły, magazyn)
- Workery działają w imieniu managera: widzą urządzenia firmy, mogą logować serwisy
- Każdy worker ma `_parent_manager_id` wskazujący na ID managera
- `ks_get_plan_data($worker_id)` → rekurencyjnie czyta plan od managera

### Kluczowe metody klasy

| Metoda | Opis |
|---|---|
| `register_roles()` | Tworzy role `service_manager` i `service_worker` (static) |
| `scope_device_queries()` | Hook `pre_get_posts` – worker widzi urządzenia managera |
| `process_add_worker()` | Tworzenie konta pracownika |
| `process_remove_worker()` | Usuwanie pracownika |
| `render_multi_tab()` | Renderowanie zakładki multi w dashboardzie |

### Zakładka Multi w panelu

- Manager widzi listę pracowników + formularz dodania nowego
- Może ustawić hasło dla pracownika i zarządzać jego kontem
- Limit: 10 pracowników na konto Multi

---

## 14. Moduł SEO (Wizytówka)

**Plik:** `inc/class-ks-seo-handler.php` + `templates/premium-seo.php`

### Co robi

- Generuje `<meta name="description">` na stronie wizytówki
- Generuje **Schema.org JSON-LD** (`LocalBusiness`) w `<head>`
- Nadpisuje `<title>` strony przez `pre_get_document_title`
- Dane zapisywane w user meta (patrz [schemat danych](#6-schemat-danych--user-meta-i-post-meta))

### Wizytówka publiczna

URL: `/serwisant/SLUG/` (np. `/serwisant/serwis-agd-marek/`)

- Rewrite rule: `^serwisant/([^/]+)/?$` → `index.php?ks_company_slug=$matches[1]`
- Template: `templates/author-card.php`
- `ks_custom_author_base()` ustawia WordPress author base na `serwisant`

> ⚠️ Po zmianie reguł rewrite: Ustawienia → Bezpośrednie odnośniki → Zapisz.

---

## 15. Moduł Protokół Serwisowy

**Plik:** `templates/premium-protokol.php`

### Funkcje

- Serwisant definiuje szablon protokołu (lista punktów kontrolnych)
- Przy serwisie wybiera wykonane czynności + zbiera podpis klienta (canvas HTML)
- System wysyła e-mail HTML z:
  - Logo firmy
  - Danymi urządzenia i datą serwisu
  - Listą wykonanych czynności (checkboxes)
  - Podpisem klienta (base64 PNG)
  - Stopką z notatkami serwisanta
- Automatyczna integracja z Magazynem: po wybraniu części z protokołu, są one odejmowane ze stanu

### Integracja Magazyn ↔ Protokół

W `dashboard-view.php` dane historii części (`ks_parts_history`) są eksportowane do JS jako `var ksDevicePartsHistory`. Formularz protokołu odczytuje ostatnio użyte części i wstrzykuje je do listy protokołu jako pozycje "Użyte części: …".

---

## 16. Moduł Magazyn

**Plik:** `templates/premium-magazyn.php`

### Funkcje

- CRUD części zamiennych (CPT `ks_czesc`)
- Każda część: nazwa, stan magazynowy, minimum alarmowe
- Odejmowanie stanu przy logowaniu serwisu (funkcja `ks_deduct_posted_parts()`)
- Powiadomienie e-mail gdy stan ≤ minimum (transient `ks_mag_notif_{user_id}` = raz dziennie)
- Badge z liczbą niskich stanów w menu bocznym

### ks_deduct_posted_parts()

```php
ks_deduct_posted_parts( $user_id, $device_id = 0 )
```

Czyta `$_POST['log_parts']` (mapa part_id → checked) i `$_POST['log_parts_qty']` (ilości), weryfikuje własność każdej części (`ks_verify_part_ownership`), odejmuje stan i zapisuje historię w `ks_parts_history` na urządzeniu.

---

## 17. Panel Admina WordPress

**Ustawienia → KiedySerwis** (dostęp: `manage_options`)

### Zakładka: Ustawienia

- Szablon SMS (zmienne: `[Nazwa]`, `[Firma]`, `[Link]`)
- Nazwa PWA
- Godzina wysyłki SMS (0–23)
- Konfiguracja bramki SMS (e-mail, hasło, device_id, sim_slot)
- Cron serwera: URL z tokenem (do wklejenia w cronie hostingu)
- Regeneracja tokena crona
- Webhook 1koszyk.pl: URL i sekret
- Test SMS: wyślij próbny SMS na podany numer

### Zakładka: Cennik

Edycja 4 planów cennikowych (`ks_pricing_plans` w wp_options). Każdy plan:
- Badge, okres, nazwa, cena, podpis ceny
- Lista zalet (jedna pozycja na linię → checkmarki)
- Tekst i URL przycisku (dwa linki: strona publiczna i panel serwisanta)

### Zakładka: Kolejka SMS

- Status WP-Cron (kiedy następne uruchomienie)
- Lista urządzeń oczekujących na SMS (kolorowa tabela: żółte=dziś, czerwone=termin minął)
- Przycisk "Wymuś sprawdzenie kolejki teraz"
- Przycisk "Wyślij SMS" dla wierszy czerwonych/żółtych po godzinie wysyłki

### Rozszerzenie WP-Admin Users (KS_Admin_Users)

- Dodatkowe kolumny: Telefon, Status Planu, Data wygaśnięcia
- Filtr dropdown: Free / Premium / Multi
- Statystyki na górze listy użytkowników
- Edycja planu i daty wygaśnięcia w profilu użytkownika

---

## 18. Kluczowe opcje wp_options

| Opcja | Opis |
|---|---|
| `ks_sms_provider` | Dostawca SMS (`android` lub `smsapi`) |
| `ks_sg24_email` | E-mail konta smsgateway24.com |
| `ks_sg24_password` | Hasło konta smsgateway24.com |
| `ks_sg24_device_id` | Device ID smsgateway24.com |
| `ks_sg24_sim_slot` | Slot SIM (0 lub 1) |
| `ks_sms_message_template` | Szablon wiadomości SMS |
| `ks_sms_send_hour` | Godzina wysyłki SMS (int) |
| `ks_cron_secret` | Token zewnętrznego crona (48-char hex) |
| `ks_webhook_secret` | Token webhooka 1koszyk.pl (48-char hex) |
| `ks_log_suffix` | Losowy sufiks nazwy pliku logów (16-char hex) |
| `ks_pricing_plans` | Dane 4 planów cennikowych (serialized array) |
| `ks_pwa_name` | Nazwa aplikacji PWA |

---

## 19. Shortcodes

| Shortcode | Opis |
|---|---|
| `[ks_dashboard]` | Panel serwisanta (login + dashboard) |
| `[ks_cennik]` | Publiczna strona cennika (4 karty planów) |

---

## 20. Rewrite rules i wizytówka publiczna

```php
// Baza URL autorów zmieniona z /author/ na /serwisant/
ks_custom_author_base() // ustawia $wp_rewrite->author_base = 'serwisant'

// Custom rewrite rule dla sluga firmy
ks_add_custom_rewrite_rules() // ^serwisant/([^/]+)/?$ → index.php?ks_company_slug=$matches[1]

// Query var
ks_register_query_vars() // dodaje 'ks_company_slug'

// Template loader
ks_custom_template_loader() // gdy ks_company_slug ustawiony → ładuje templates/author-card.php
```

> **Ważne**: Funkcja `ks_author_card_template()` służy jako fallback dla `is_author()`, natomiast `ks_custom_template_loader()` obsługuje URL z `ks_company_slug`. Obie mogą załadować `author-card.php`.

---

## 21. Bezpieczeństwo – zasady nienaruszalne

### Własność zasobów

**Zawsze** weryfikuj własność przed odczytem/modyfikacją:

```php
// Urządzenie – MUSI być wywołane przed każdą operacją na device_id
if ( ! ks_verify_device_ownership( $device_id ) ) {
    wp_die( 'Nieprawidłowe urządzenie.' );
}

// Część magazynowa
if ( ! ks_verify_part_ownership( $part_id, $user_id ) ) {
    continue; // lub wp_die()
}
```

Funkcje te sprawdzają `post_type`, `post_status=publish` i `post_author === current_user_id`.

### Nonce

Każdy formularz musi mieć `wp_nonce_field()` i każdy handler musi weryfikować nonce przez `wp_verify_nonce()`. Brak nonce = `wp_die('Błąd bezpieczeństwa.')`.

### Sanitizacja danych

- `sanitize_text_field()` – dla krótkich tekstów
- `sanitize_textarea_field()` – dla notatek/opisów
- `sanitize_email()` – dla e-maili
- `absint()` / `intval()` – dla liczb
- `esc_url_raw()` – dla URL przy zapisie
- `esc_html()` / `esc_attr()` / `esc_url()` – przy wyświetlaniu

### Tokeny bezpieczeństwa

- `ks_get_cron_secret()` – 48-char hex, generowany z `random_bytes(24)`, porównywany przez `hash_equals()`
- `ks_get_webhook_secret()` – j.w. dla webhooka płatności
- `ks_get_log_path()` – losowy 16-char hex sufiks nazwy pliku logów (uniemożliwia odgadnięcie URL na Nginx)

### SQL

Wszystkie zapytania SQL przez `$wpdb->prepare()`. Nie używaj surowych stringów SQL z danymi użytkownika.

### CPT nie jest publiczny

`ks_urzadzenie` ma `public=false`, `publicly_queryable=false`, `show_in_rest=false`. **Nie zmieniaj tego** – dane klientów (telefony) nie mogą być publicznie dostępne.

---

## 22. Logi i debugowanie

### Lokalizacja logów

```
wp-content/ks-logs/ks-debug-SUFFIX.log
```

`SUFFIX` to 16-znakowy hex generowany losowo przy pierwszym użyciu i zapisywany jako `ks_log_suffix` w wp_options. Znajdź aktualną nazwę pliku w:
- Ustawienia → KiedySerwis → przy informacji o godzinie crona
- W bazie: `SELECT option_value FROM wp_options WHERE option_name = 'ks_log_suffix';`

### Ochrona katalogu

- `.htaccess` w `wp-content/ks-logs/` zabrania dostępu (Apache)
- `index.php` zapobiega listingowi katalogów
- Losowa nazwa pliku chroni przed odgadnięciem URL (Nginx)

### Funkcja ks_log()

```php
ks_log( '[KiedySerwis] Moja wiadomość' );
```

Zapisuje linie w formacie `[2026-03-14 18:00:01] [wiadomość]`. Fallback na PHP `error_log()` gdy zapis się nie powiedzie.

### Jak debugować cron

1. Ustawienia → KiedySerwis → **Wymuś sprawdzenie kolejki teraz** → sprawdź log
2. Lub wywołaj ręcznie z terminala:
   ```bash
   curl -X POST "https://kiedyserwis.pl/wp-json/ks/v1/run-sms-queue?token=TOKEN"
   ```
3. Log powinien pokazać `Batch SMS: znaleziono N urządzeń`
4. Jeśli `znaleziono 0 urządzeń`: sprawdź czy status SMS nie jest już `Wysłano` dla tych urządzeń

---

## 23. Jak dodać dostawcę SMS: SMSAPI.pl

Infrastruktura pod SMSAPI.pl jest już **częściowo przygotowana**. Dropdown w ustawieniach ma opcję `smsapi` (wyświetla "wkrótce"). Oto co trzeba zrobić:

### Krok 1: Dodaj opcje konfiguracji

W `ks_register_settings()` dodaj:
```php
register_setting( 'ks_settings_group', 'ks_smsapi_token', array( 'sanitize_callback' => 'sanitize_text_field' ) );
register_setting( 'ks_settings_group', 'ks_smsapi_sender', array( 'sanitize_callback' => 'sanitize_text_field' ) );
```

### Krok 2: Dodaj pola w ks_settings_page_html()

```php
<tr class="ks-smsapi-row">
    <th><label for="ks_smsapi_token">Token API (SMSAPI.pl)</label></th>
    <td>
        <input type="text" name="ks_smsapi_token" value="<?php echo esc_attr( get_option('ks_smsapi_token') ); ?>" class="regular-text">
        <p class="description">Znajdziesz go w panelu SMSAPI.pl → API → Token.</p>
    </td>
</tr>
<tr class="ks-smsapi-row">
    <th><label for="ks_smsapi_sender">Nadawca SMS</label></th>
    <td>
        <input type="text" name="ks_smsapi_sender" value="<?php echo esc_attr( get_option('ks_smsapi_sender', 'KiedySerwis') ); ?>" class="regular-text">
    </td>
</tr>
```

### Krok 3: Dodaj logikę w ks_send_sms()

W funkcji `ks_send_sms()` znajdź blok z komentarzem `// Placeholder for future providers`:

```php
if ( $provider === 'smsapi' ) {
    $token  = get_option( 'ks_smsapi_token', '' );
    $sender = get_option( 'ks_smsapi_sender', 'KiedySerwis' );

    if ( empty( $token ) ) {
        ks_log( '[KiedySerwis] ks_send_sms: Brak tokena SMSAPI.pl.' );
        return false;
    }

    $response = wp_remote_post(
        'https://api.smsapi.pl/sms.do',
        array(
            'timeout' => 15,
            'headers' => array(
                'Authorization' => 'Bearer ' . $token,
            ),
            'body' => array(
                'to'      => $phone,
                'message' => $message,
                'from'    => $sender,
                'format'  => 'json',
            ),
        )
    );

    if ( is_wp_error( $response ) ) {
        ks_log( '[KiedySerwis] SMSAPI.pl WP_Error: ' . $response->get_error_message() );
        return false;
    }

    $code = wp_remote_retrieve_response_code( $response );
    $body = wp_remote_retrieve_body( $response );
    $data = json_decode( $body, true );

    ks_log( '[KiedySerwis] SMSAPI.pl odpowiedź: ' . $body );

    if ( $code !== 200 || isset( $data['error'] ) ) {
        ks_log( '[KiedySerwis] SMSAPI.pl błąd: ' . $body );
        return false;
    }

    ks_log( '[KiedySerwis] SMSAPI.pl sukces. list_id=' . ( $data['list'][0]['id'] ?? '?' ) );
    return true;
}
```

### Krok 4: Usuń "wkrótce" z dropdownu

W `ks_settings_page_html()` zmień opcję:
```php
<option value="smsapi" <?php selected( $provider, 'smsapi' ); ?>>SMSAPI.pl</option>
```

### Krok 5: Zaktualizuj JS toggle

```javascript
// Zmień toggle żeby pokazywać właściwe pola dla każdego dostawcy
var rows_sg24   = document.querySelectorAll('.ks-sg24-row');
var rows_smsapi = document.querySelectorAll('.ks-smsapi-row');
function toggle() {
    var prov = sel.value;
    rows_sg24.forEach(function(r){ r.style.display = prov === 'android' ? '' : 'none'; });
    rows_smsapi.forEach(function(r){ r.style.display = prov === 'smsapi' ? '' : 'none'; });
}
```

---

## 24. Jak dodać nowe funkcje premium

### Sprawdzanie dostępu

```php
// W handlerze lub szablonie:
if ( ! ks_is_premium_active( get_current_user_id() ) ) {
    // Przekieruj do strony cennika lub wyświetl upsell
    wp_safe_redirect( add_query_arg( 'ks_tab', 'plan', home_url('/panel-serwisanta/') ) );
    exit;
}
```

### Dodanie nowej zakładki w dashboardzie

1. Dodaj link w menu nawigacyjnym (`dashboard-view.php`) w `<nav class="ks-nav">`:
   ```php
   <a href="?ks_tab=moja_funkcja" class="ks-nav-item <?php echo $active_tab === 'moja_funkcja' ? 'active' : ''; ?>">
       Moja Funkcja
   </a>
   ```

2. Dodaj obsługę w głównym `include` zakładek (w `dashboard-view.php`):
   ```php
   } elseif ( $active_tab === 'moja_funkcja' && $_is_premium ) {
       include plugin_dir_path( dirname(__FILE__) ) . 'kiedyserwis-core/templates/premium-moja-funkcja.php';
   }
   ```

3. Stwórz szablon `templates/premium-moja-funkcja.php`

4. Dodaj handler w `kiedyserwis-core.php`:
   ```php
   function ks_handle_moja_akcja() {
       if ( ! is_user_logged_in() || ! current_user_can( 'edit_posts' ) ) { wp_die('Brak uprawnień.'); }
       if ( ! wp_verify_nonce( $_POST['ks_moja_nonce'], 'ks_moja_akcja' ) ) { wp_die('Błąd bezpieczeństwa.'); }
       $user_id = get_current_user_id();
       if ( ! ks_is_premium_active( $user_id ) ) { wp_die('Wymagany Premium.'); }
       // ... logika ...
       wp_safe_redirect( add_query_arg( 'ks_tab', 'moja_funkcja', home_url('/panel-serwisanta/') ) );
       exit;
   }
   add_action( 'admin_post_ks_moja_akcja', 'ks_handle_moja_akcja' );
   ```

### Dodanie nowej opcji do cennika

Cennik pobiera dane z `ks_get_pricing_plans()` która scala `wp_options['ks_pricing_plans']` z wartościami domyślnymi z `ks_get_pricing_defaults()`. Zmień domyślne URL w `ks_get_pricing_defaults()` dla poszczególnych planów.

---

## 25. Integracja płatności 1koszyk.pl – gotowy handler

Handler webhooka (`ks_handle_1koszyk_webhook`) jest już gotowy. Przy dodaniu nowych planów cennikowych rozszerz mapowanie `product_id → dni`:

```php
// W ks_handle_1koszyk_webhook(), fragment do rozszerzenia:
$days_to_add = 30; // Default: 1 miesiąc
if ( strpos( $product_id, '365' ) !== false ) {
    $days_to_add = 365;
} elseif ( strpos( $product_id, '180' ) !== false ) {
    $days_to_add = 180; // 6 miesięcy – DODAJ TEN WARUNEK
} elseif ( strpos( $product_id, 'firma' ) !== false ) {
    $days_to_add = 365;
    // Dodatkowo: przypisz rolę service_manager
    $user = get_userdata( $user_id );
    if ( $user ) {
        KS_Multi_Account::register_roles();
        update_user_meta( $user_id, 'ks_plan_status', 'multi' );
        if ( ! in_array( 'service_manager', (array) $user->roles, true ) ) {
            $user->add_role( 'service_manager' );
        }
    }
}
```

### Parametr attr1 w 1koszyk.pl

W panelu 1koszyk.pl przy tworzeniu produktu/linku płatności ustaw `attr1` jako WordPress user ID. Możesz przekazać go automatycznie dodając `?attr1=<?php echo get_current_user_id(); ?>` do URL przycisku zakupu (jest to obsługiwane przez `btn_url_panel` w cennikach).

---

## 26. Pułapki i ważne niuanse

### 1. get_posts() wymaga zalogowanego użytkownika dla private CPT

`ks_urzadzenie` jest private (`public=false`). WordPress zwróci puste wyniki dla `get_posts()` gdy `get_current_user_id() === 0`. Dlatego `ks_run_sms_batch_for_today()` przełącza na admina gdy wywołana bez kontekstu użytkownika. **Nie usuwaj tej logiki.**

### 2. Deduplication guard – ks_last_sms_date

Zewnętrzny cron odpala się co minutę. Mechanizm deduplication (`ks_last_sms_date === today`) zapobiega wysłaniu wielu SMS tego samego dnia do jednego urządzenia. Gdy chcesz pozwolić na ponowne wysłanie (np. po ręcznym resecie), usuń `ks_last_sms_date` przez `delete_post_meta($device_id, 'ks_last_sms_date')`.

### 3. Caching (LiteSpeed/Nginx/WP Rocket)

- Dashboard ma `Cache-Control: no-cache` ustawiony w `ks_dashboard_shortcode()`
- REST endpoint `/run-sms-queue` ma `nocache_headers()` + nagłówki `Cache-Control: no-store`
- Cron powinien używać metody **POST** (nigdy GET) – POST nie jest buforowany przez serwery
- Po każdej zmianie danych użytkownika wywoływane jest `ks_purge_user_cache($user_id)` (obsługuje LiteSpeed, WP Rocket, W3TC, WP Super Cache)

### 4. Strefa czasowa – zawsze Warsaw

Wszystkie obliczenia dat używają `ks_warsaw_now()` (DateTimeImmutable w `Europe/Warsaw`). Nigdy nie używaj `date()` lub `time()` bezpośrednio – serwer może być w UTC.

```php
$today = ks_warsaw_now()->format('Y-m-d');
```

### 5. flush_rewrite_rules()

Po każdej zmianie rewrite rules (nowe reguły, nowe CPT) musisz wywołać `flush_rewrite_rules()` lub przejść do Ustawienia → Bezpośrednie odnośniki i kliknąć Zapisz.

### 6. ks_is_custom_app – wpisy kalendarza vs urządzenia

Wpisy kalendarza (bez urządzenia) są przechowywane jako `ks_urzadzenie` z metą `ks_is_custom_app = 'true'`. Batch SMS wyklucza je przez `meta_query NOT EXISTS ks_is_custom_app`. Przy usuwaniu: wpisy kalendarza są usuwane permanentnie, urządzenia tylko czyszczone z daty wizyty.

### 7. Token w URL crona

Token crona jest widoczny w logach serwera (URL query string). W razie kompromitacji: Ustawienia → KiedySerwis → Regeneruj token → zaktualizuj URL w cronie serwera.

### 8. Hasło smsgateway24.com w bazie

Hasło jest przechowywane w plaintext w `wp_options`. To ograniczenie architektury WordPress (brak natywnego keystore). Upewnij się, że dostęp do bazy MySQL jest odpowiednio chroniony.

---

## 27. Testowalność i debugowanie ręczne

### Test wysyłki SMS

Ustawienia → KiedySerwis → "Wyślij SMS testowy" → podaj numer i kliknij.

### Test crona ręcznie

```bash
curl -v -X POST "https://kiedyserwis.pl/wp-json/ks/v1/run-sms-queue?token=TOKEN"
```

Odpowiedź JSON powinna zawierać `"status":"ok"` i `"sent":N`.

### Sprawdzenie logów

```bash
# Znajdź nazwę pliku
wp option get ks_log_suffix

# Odczytaj ostatnie 50 linii
tail -50 wp-content/ks-logs/ks-debug-SUFFIX.log
```

### Reset statusu SMS urządzenia

Przez WP-Admin → Urządzenia → Edytuj → zmień "Status powiadomienia" na "Oczekuje" → Zapisz.

Lub przez PHP (np. w snippecie):
```php
update_post_meta( $device_id, 'ks_status_sms', 'Oczekuje' );
delete_post_meta( $device_id, 'ks_last_sms_date' );
```

### Test webhooka płatności

```bash
curl -X POST "https://kiedyserwis.pl/wp-json/ks/v1/payment-webhook" \
  -H "X-KS-Webhook-Secret: SECRET" \
  -H "Content-Type: application/json" \
  -d '{"status":"completed","product_id":"product-365","attr1":"USER_ID"}'
```

### WP-CLI (jeśli dostępne)

```bash
wp option get ks_cron_secret
wp user meta get USER_ID ks_plan_status
wp user meta update USER_ID ks_plan_expiry $(date -d "+30 days" +%s)
```

---

*Dokument wygenerowany na podstawie kodu wtyczki w wersji commit `84a26f9` (branch `copilot/security-audit-wordpress-plugin`).*  
*Ostatnia aktualizacja: 2026-03-14*

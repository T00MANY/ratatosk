# ratatosk

Integracja z Allegro REST API dla VIPTechnology. Synchronizacja
stanów i cen ofert, wystawianie nowych ofert, obsługa zamówień,
faktur, sporów i wiadomości. Niezależna od sklepu WooCommerce
viptechnology.pl (tamten ma osobną aplikację `woocommerce_viptechnology`).

## Komponenty

Aplikacja działa rozproszenie na trzech serwerach. Wszystkie używają
tego samego `client_id`, dzielą tokeny OAuth i wysyłają identyczny
nagłówek `User-Agent: ratatosk/1.0 (+https://github.com/T00MANY/ratatosk)`.

### 1. stock-sync (Celery + Redis)

Synchronizacja stanów magazynowych, cen i statusu publikacji ofert.
Wystawianie nowych ofert, naprawa kodów EAN z puli GS1.

Triggery:

- Celery Beat co 5 min — `sync_allegro` (stock/price/publication)
- Manualnie — listing nowych ofert, fix EAN, relisting

Endpointy:

- `GET /sale/offers` — lista ofert (paginacja)
- `PUT /sale/offer-quantity-change-commands/{commandId}` — stock
- `PUT /sale/offer-price-change-commands/{commandId}` — ceny
- `PUT /sale/offer-publication-commands/{commandId}` — aktywacja/end
- `GET /sale/{endpoint}/tasks` — status poleceń batch
- `GET /sale/product-offers` — dane ofert
- `GET /sale/products`, `GET /sale/categories` — katalog produktów
- `GET /sale/offer-attachments` — załączniki ofert
- `GET /sale/responsible-persons` — dane GPSR (producent, RP)

### 2. OMS / Helpdesk

Obsługa zamówień, trackingu, faktur, sporów i wiadomości od kupujących.

Triggery:

- Polling `/order/events` co 1 min (nowe zamówienia)
- Polling `/messaging/threads` (nieprzeczytane wiadomości)
- Polling `/sale/disputes` i `/sale/issues` (spory, roszczenia)
- Po pickingu/packingu w WMS — dodanie trackingu, upload faktury

Endpointy:

- `GET /order/events`, `GET /order/checkout-forms/{id}`
- `POST /order/checkout-forms/{id}/shipments` — tracking number
- `POST /order/checkout-forms/{id}/invoices` — rejestracja faktury
- `PUT /order/checkout-forms/{id}/invoices/{invoiceId}/file` — PDF
- `PUT /order/checkout-forms/{id}/fulfillment` — status realizacji
- `GET /order/carriers` — kurierzy
- `GET /sale-settings/delivery/shipping-rates`,
  `/delivery-methods` — metody dostawy
- `GET /messaging/threads`, `POST /messaging/threads/{id}/messages`
- `GET /sale/disputes`, `GET /sale/issues` — spory i roszczenia

### 3. Messenger (n8n)

Wysyłanie wiadomości do kupujących z opcjonalnymi załącznikami
(np. kody QR eSIM). Multi-konto.

Triggery:

- Webhook z OMS po realizacji zamówienia
- Manualnie z UI helpdesku

Endpointy:

- `POST /messaging/threads/{threadId}/messages`
- `POST /messaging/threads/{threadId}/attachments`

## Autoryzacja

OAuth2 (authorization code + refresh). Tokeny przechowywane w Redis,
odświeżane automatycznie w warstwie auth (`get_token`).

## Kontakt

ok@viptechnology.pl

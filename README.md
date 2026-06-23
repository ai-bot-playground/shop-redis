# Redis — gorąca ścieżka rezerwacji stocku

Standardowy obraz `redis`, ale to repo trzyma jego konfigurację (`redis.conf`).
`docker-compose.yml` (w repo shop-infra) montuje ten plik i uruchamia
`redis-server /usr/local/etc/redis/redis.conf`.
Najważniejszy element dla scenariusza „tysiące kupujących naraz". Używany głównie
przez **shop-inventory** (liczniki i rezerwacje), dodatkowo przez **shop-gateway**
(liczniki rate limitingu) i opcjonalnie cache w shop-catalog.

## Dlaczego Redis, a nie SQL

Przy flash sale tysiące żądań walczy o ten sam licznik. Blokady w bazie
relacyjnej stają się wąskim gardłem. Redis wykonuje operacje na danych
jednowątkowo, więc pojedyncza komenda jest atomowa — idealne do bezpiecznego
dekrementowania licznika bez wyścigów.

## Projekt kluczy

- `stock:{productId}` → liczba dostępnych sztuk (integer).
- `reservation:{orderId}` → rezerwacja z **TTL** (np. 600 s).
- `idemp:{key}` → znaczniki idempotencji.
- `ratelimit:{clientId}` → liczniki dla shop-gateway.

## Atomowa rezerwacja (skrypt Lua)

Rezerwacja w **jednej** atomowej operacji: sprawdź dostępność, zmniejsz licznik,
załóż klucz rezerwacji z TTL. Zwykłe „GET, sprawdź, DECR" z poziomu aplikacji to
wyścig. Logikę i pseudokod opisuje README shop-inventory. Zwolnienie (kompensacja)
to operacja odwrotna (`INCRBY` + `DEL`), również atomowa i idempotentna.

## TTL jako bezpiecznik

Każda rezerwacja ma czas życia — gdy zdarzenie kompensujące zaginie albo serwis
padnie, rezerwacja wygaśnie sama i stock wróci do puli.

## Pamięć i persystencja (`redis.conf`)

- `appendonly yes` (AOF) — odtworzenie stanu po restarcie.
- `maxmemory-policy noeviction` — **nigdy** nie eksmituj liczników stocku
  (utrata licznika = błędna sprzedaż); przy braku pamięci zapis zwraca błąd.
- `maxmemory 256mb` — limit dema; produkcyjnie dobierz do danych / Redis Cluster.
- shop-inventory rozgrzewa liczniki z Postgresa przy starcie i synchronizuje w tle.

## Skalowanie
Redis Cluster z shardingiem po `productId`, repliki. Healthcheck: `redis-cli ping`.

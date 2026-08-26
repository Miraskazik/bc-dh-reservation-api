# bc-dh-reservation-api

API kontrakt služby **bc-dh-reservation** (rezervace pobytů psů) systému
**Dog Hotel**. Repo drží `openapi.yaml` jako zdroj pravdy a přes
[openapi-generator](https://openapi-generator.tech/) (Apache-2.0) z něj generuje:

- **klienta** — Spring **HTTP Interface** (`@HttpExchange` + `RestClient`),
  balíček `cz.doghotel.reservation.api.client` — konzumuje BFF;
- **server stub** — interfaces + DTO, **delegate pattern**, balíček
  `cz.doghotel.reservation.api.server` — implementuje `bc-dh-reservation`.

Artefakt `cz.doghotel:bc-dh-reservation-api` se publikuje do **GitHub Packages**
(SemVer, start `0.1.0`).

Související repa: [dh-infra](https://github.com/Miraskazik/dh-infra) ·
[bc-dh-auth](https://github.com/Miraskazik/bc-dh-auth) · `bc-dh-reservation` (služba, implementace).

## Contract-first

`openapi.yaml` je **jediný zdroj pravdy**. Změna API = PR do **tohoto** repa
(nikdy ne přímo ve službě). CI vygeneruje klienta + server stub a na push do
`main` publikuje nový artefakt.

## Endpointy v1 (`/api/reservations`)

| Metoda | Cesta | Popis |
|---|---|---|
| POST | `/api/reservations` | Vytvoří rezervaci (ověří existenci zákazníka a psa) |
| GET | `/api/reservations/{id}` | Detail rezervace |
| GET | `/api/reservations` | Stránkovaný seznam + filtry `customerId`, `dogId`, `status` |
| POST | `/api/reservations/{id}/finish` | PENDING → FINISHED (publikuje `dh.reservation.finished`) |
| POST | `/api/reservations/{id}/cancel` | PENDING → CANCELLED (při `refundAmount > 0` publikuje `dh.reservation.refunded`) |

Entita `Reservation`: `id`, `customerId`, `dogId`, `dateFrom`, `dateTo`,
`pricePerDay` (snapshot z requestu), `status` (PENDING/FINISHED/CANCELLED),
`refundAmount`, `cancelledAt`, `createdAt`, `updatedAt`.
Chyby: RFC 9457 `application/problem+json`. Zabezpečení: Keycloak JWT (`bearerAuth`).

## Kafka eventy

Dokončení a refundované zrušení publikuje reservation na Kafku — payloady jsou
popsané v [dh-infra/docs/event-catalog.md](https://github.com/Miraskazik/dh-infra/blob/main/docs/event-catalog.md)
(`dh.reservation.finished`, `dh.reservation.refunded`). Kontrakt eventů **není**
součástí tohoto openapi (to popisuje jen synchronní REST API).

## Build

```bash
mvn verify          # vygeneruje klienta + server stub a zkompiluje (validuje spec)
mvn deploy          # publikace do GitHub Packages (dělá CI na push do main)
```

## Použití artefaktu (konzument)

```xml
<dependency>
  <groupId>cz.doghotel</groupId>
  <artifactId>bc-dh-reservation-api</artifactId>
  <version>0.1.0</version>
</dependency>
```

Stažení z GitHub Packages vyžaduje v `~/.m2/settings.xml` server `github`
s tokenem se scope `read:packages` (v CI stačí `GITHUB_TOKEN`).

## CI

- **PR do `main`:** `mvn verify` — build z `openapi.yaml` (nevalidní spec build shodí).
- **Push do `main`:** `mvn deploy` → GitHub Packages. Nová verze = bump `<version>`
  v `pom.xml` (GitHub Packages release verzi nepřepisuje).
- **PR review:** `claude.yml` (`@claude` mention / automaticky při otevření PR) —
  vyžaduje secret `ANTHROPIC_API_KEY`.

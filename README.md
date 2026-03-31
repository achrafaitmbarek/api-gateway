# api-gateway

Point d'entrée unique de la **Bank Platform**. Toutes les requêtes externes passent par ce service avant d'atteindre les microservices internes.

## Rôle dans l'architecture

```
Client (mobile / web)
        │
        ▼
 ┌─────────────────────────────────┐
 │           api-gateway           │
 │  ● JWT Validation               │
 │  ● Rate Limiting (Bucket4j)     │
 │  ● Circuit Breaker (R4j)        │
 │  ● Correlation ID               │
 └───────────────┬─────────────────┘
                 │  HTTP Routing
                 │
                 ├──────────► bank-auth-service       :8081  (IAM, OAuth2, Keycloak)
                 │                   │
                 ├──────────► bank-account-service    :8083  (comptes, soldes, mouvements)
                 │                   │
                 ├──────────► notification-service    :8084  (emails, alertes)
                 │                   │
                 └──────────► bank-reporting-service  :8085  (CQRS, rapports)
                                     │
                           publient des events
                                     │
                                     ▼
                      ┌──────────────────────────┐
                      │     Kafka Event Bus      │
                      └──────────────┬───────────┘
                                     │  consomment les events
                                     │
                                     ├──────────► bank-audit-service
                                     │            (traçabilité, conformité)
                                     │
                                     └──────────► bank-fraud-detection-service
                                                  (fraude temps réel)

┌──────────────────────────────────────────────────────────────────┐
│   bank-config-service — configuration centralisée (12-Factor)    │
│   Utilisé par tous les services au démarrage.                    │
│   Non exposé publiquement via la gateway.                        │
└──────────────────────────────────────────────────────────────────┘
```

## Fonctionnalités prévues

### Routing
Redirection des requêtes vers les services cibles selon le path :

| Path entrant | Service cible |
| :--- | :--- |
| `/api/auth/**` | bank-auth-service |
| `/api/accounts/**` | bank-account-service |
| `/api/notifications/**` | notification-service |
| `/api/reports/**` | bank-reporting-service |

### Sécurité — Validation JWT globale
Filtre global qui valide le token JWT sur chaque requête avant de la transmettre. Les endpoints publics (`/api/auth/login`, `/api/auth/register`) sont explicitement exclus.

### Rate Limiting — Bucket4j
Limitation du nombre de requêtes par IP pour protéger les endpoints sensibles du brute force. Configuré par route (plus strict sur `/api/auth/**`).

### Résilience — Circuit Breaker (Resilience4j)
Si un service interne est indisponible, le circuit s'ouvre et renvoie une réponse de fallback immédiate plutôt que de laisser les requêtes s'accumuler en timeout.

### Observabilité — Correlation ID
Chaque requête entrante reçoit un `X-Correlation-ID` unique injecté par le gateway et propagé vers tous les microservices. Permet de tracer une transaction de bout en bout dans les logs de chaque service.

## Stack technique

- **Java 21** + **Spring Boot 4**
- **Spring Cloud Gateway**
- **Resilience4j** — Circuit Breaker, Retry
- **Bucket4j** — Rate Limiting
- **OpenTelemetry** — Distributed Tracing (prévu)
- **Docker** — image multi-stage
- **GitLab CI/CD** — build, test, push ECR, déploiement AWS

## Variables d'environnement

| Variable | Description | Défaut local |
| :--- | :--- | :--- |
| `KEYCLOAK_ISSUER_URI` | URI du realm Keycloak pour validation JWT | `http://localhost:8180/realms/bank-app` |
| `AUTH_SERVICE_URL` | URL du bank-auth-service | `http://localhost:8081` |
| `ACCOUNT_SERVICE_URL` | URL du bank-account-service | `http://localhost:8083` |
| `NOTIFICATION_SERVICE_URL` | URL du notification-service | `http://localhost:8084` |
| `REPORTING_SERVICE_URL` | URL du bank-reporting-service | `http://localhost:8085` |

## Lancer en local

```bash
# Démarrer l'infrastructure (Keycloak, Postgres, Kafka)
docker compose -f infrastructure/docker-compose.yml up -d

# Lancer le gateway
./mvnw spring-boot:run
```

## Auteur

Achraf Ait M'Barek — the-crazy-achraf@hotmail.fr
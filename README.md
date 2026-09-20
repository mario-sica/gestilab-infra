# gestilab-infra

Infrastruttura di deploy per [GestiLab](https://github.com/mario-sica/gestilab-app), piattaforma SaaS multi-tenant per gli assistenti tecnici delle scuole superiori italiane. Questo repository contiene **solo** ciò che serve a mettere in produzione il sistema — configurazione di deploy, reverse proxy, documentazione operativa — mai codice applicativo.

## Perché tre repository

GestiLab è diviso in tre repository con confini di responsabilità distinti:

| Repository | Contiene | Visibilità |
|---|---|---|
| [`gestilab-app`](https://github.com/mario-sica/gestilab-app) | Codice applicativo (`apps/web`, `apps/api`), pacchetti condivisi, Dockerfile, CI/CD, ambiente di sviluppo locale (`compose.yaml`, `compose.dev.yaml`, `compose.local-prod.yaml`) | Pubblico |
| `gestilab-auth-service` | Autenticazione: login, password, PIN, TOTP, emissione e revoca sessioni | **Privato** |
| `gestilab-infra` (questo repo) | Deploy di produzione: `compose.prod.yaml`, Traefik, documentazione operativa | Pubblico |

Non è una scelta di moda: `gestilab-auth-service` resta privato perché il codice che gestisce le credenziali è la superficie più sensibile del sistema — separarlo in un repository proprio, distinto da quello applicativo pubblico, riduce chi/cosa può vederlo per motivi di sicurezza reali, non solo organizzativi. Isolarlo *anche* dal punto di vista del deploy (`gestilab-infra` separato da `gestilab-app`) tiene i segreti di produzione (`.env` di questo repo) fuori dal repository dove vive il codice, e la cronologia di ogni repository pulita e specifica al proprio ambito.

L'autenticazione condivide comunque lo stesso database Postgres di `gestilab-app` (stesso schema, tabelle `utenti`/`persone`, un ruolo Postgres dedicato `gestilab_auth`): la separazione è di codice e di processo di deploy, non di dati — duplicare l'identità degli utenti tra due database sarebbe un problema di coerenza distribuita che un pilota su un solo istituto non ha bisogno di affrontare.

## Cosa NON è qui

`compose.local-prod.yaml` e il Traefik che usa (`docker/traefik/traefik.yml`, `dynamic/tls.yml`, i certificati mkcert) **restano in `gestilab-app`**: sono uno strumento di sviluppo/verifica locale (dimostrano la prontezza alla produzione senza costare nulla), non un artefatto di deploy reale — un contributor che clona solo `gestilab-app` deve poter verificare tutto in locale senza toccare questo repository.

## Deploy

**Prerequisiti** (vedi "Prerequisiti a spesa" nel backlog di `gestilab-app`): un dominio registrato e un VPS in UE. Finché non ci sono, `compose.prod.yaml` resta scritto e versionato ma **mai eseguito**.

`compose.prod.yaml` è un *overlay*: la definizione base dei servizi (`compose.yaml`) vive in `gestilab-app`, un repository diverso. Procedura:

```bash
# 1. Clona la release da deployare accanto a questo repository
git clone --branch <tag-di-release> https://github.com/mario-sica/gestilab-app.git ../gestilab-app-deploy

# 2. Copia .env.example -> .env in questa cartella e valorizzalo
cp .env.example .env

# 3. Avvia dalla cartella di gestilab-infra (Docker Compose legge .env da qui)
docker compose -f ../gestilab-app-deploy/compose.yaml -f compose.prod.yaml up -d
```

Nessuna modifica al codice applicativo per passare da `local-prod` a `prod`: solo `BASE_DOMAIN`, il token del provider DNS e i DSN dei servizi esterni in `.env` — la stessa filosofia di `gestilab-app` ("nessuna differenza di codice tra ambienti") estesa al deploy.

## Stato

Repository appena creato. `compose.prod.yaml` non è mai stato eseguito (manca ancora il dominio e il VPS). `gestilab-auth-service` non esiste ancora come codice: il backlog di autenticazione (Fase 1 di `gestilab-app`) è in fase di progettazione.

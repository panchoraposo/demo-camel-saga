## Keycloak HA multi-cluster (Site-A / Site-B) + theme TicketBlaster

Estos manifiestos son un **blueprint** para probar Red Hat build of Keycloak 26.4 en una topología **multi-cluster** (dos clusters OpenShift, “Site A” y “Site B”), con:

- **sesiones replicadas** vía **external Data Grid (Infinispan)** con XSite
- **DB síncrona** (ej. Aurora multi-AZ)
- **failover de cliente** usando un **entrypoint único** (LB/Global Accelerator/dominio) para evitar problemas de caché DNS
- **theme** de login “ticketblaster” aplicado al realm `ticketblaster`

### Qué hay aquí

- `site-a/`: Keycloak CR + realm import para Site A
- `site-b/`: Keycloak CR + realm import para Site B

### Qué debes ajustar

En `Keycloak.spec.hostname.hostname`:

- reemplaza `<KEYCLOAK_URL_HERE>` por el hostname “global” que usen los clientes (LB/GA/dominio)

En `Keycloak.spec.db.url`:

- reemplaza `<AWS_AURORA_URL_HERE>` por tu writer endpoint (o equivalente) de DB síncrona

En `Keycloak.spec.http.tlsSecret`:

- crea/provee `keycloak-tls-secret` con el certificado que corresponda al hostname global

### Dependencias

Antes de aplicar Keycloak HA, instala:

- **RHBK Operator** (ya existe en este repo)
- **Data Grid Operator** (ver `../../operators/datagrid/`)
- **Data Grid XSite** en ambos sites (ver `../datagrid-ha/`)

### Notas sobre cachés / sesiones

El blueprint usa los `additionalOptions` recomendados por el HA guide:

- `cache-remote-*` para apuntar al Data Grid
- `features.enabled: [multi-site]` para habilitar `/lb-check` y soporte multi-cluster

### Aplicación por site

En Site A:

- `oc apply -k apps/keycloak-ha/site-a/`

En Site B:

- `oc apply -k apps/keycloak-ha/site-b/`


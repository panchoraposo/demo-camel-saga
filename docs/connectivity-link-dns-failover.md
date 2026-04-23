## Experimento: failover DNS con Red Hat Connectivity Link (vs. Global Accelerator)

El HA guide de Red Hat build of Keycloak para multi-cluster recomienda un **entrypoint “estable”** (ej. AWS Global Accelerator) para minimizar problemas por **DNS caching** del lado del cliente. Aun así, vale la pena probar un enfoque alternativo: **failover a nivel DNS** usando **Red Hat Connectivity Link** (basado en Gateway API + políticas).

Connectivity Link ofrece **“Dynamic DNS and multi-cluster routing with failover”** y estrategias como **weighted** o **GEO routing** mediante proveedores como Route53/Azure/GCP DNS (ver descripción del producto en `https://developers.redhat.com/products/red-hat-connectivity-link`).

### Objetivo del experimento

- Publicar un hostname único para Keycloak, por ejemplo `sso.ticketblaster.example.com`
- Tener **dos gateways** (uno por cluster: Site A / Site B)
- Usar Connectivity Link para:
  - crear/actualizar automáticamente el DNS del hostname
  - conmutar tráfico hacia el site saludable cuando el otro falle

### Health check recomendado

Para Keycloak multi-site, el HA guide introduce el endpoint:

- `GET /lb-check`

La idea es que el health check del DNS (o del gateway) evalúe `https://sso.ticketblaster.example.com/lb-check` por site y, cuando un site falle, deje de anunciarlo.

### Advertencias (importantes para tu prueba)

- **DNS caching**: en navegadores y resolvers, incluso con TTL bajo, el failover puede tardar más que un LB con IP estática.
- **Split-brain / fencing**: si el problema no es que “se cayó el site” sino que se rompe la conectividad entre sites (Data Grid XSite), el HA guide sugiere “fencing” para dejar solo un site atendiendo y luego re-sincronizar. Connectivity Link puede ayudarte a redirigir tráfico, pero **no reemplaza** los procedimientos de re-sync del Data Grid.

### Propuesta de pasos (alto nivel)

1. **Crear 2 Gateways** (Site A y Site B) que publiquen el mismo hostname del listener (ej. `sso.ticketblaster.example.com`).
2. **Adjuntar política DNS** de Connectivity Link al/los gateway(s) para:
   - crear el record (A/AAAA o CNAME según proveedor)
   - habilitar estrategia **failover** (o weighted con health checks)
3. Configurar health checks con path `/lb-check`.
4. **Prueba**: iniciar login, navegar en TicketBlaster, “tumbar” el Keycloak de Site A y medir:
   - tiempo hasta que refrescos de token vuelvan a funcionar
   - si el usuario evita re-login (objetivo)

### Resultado esperado vs. Global Accelerator

- **Connectivity Link (DNS)**: failover funcional pero potencialmente más lento/variable.
- **Global Accelerator / IP estable**: failover más predecible y resistente a caching.


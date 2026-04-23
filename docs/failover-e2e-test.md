## Prueba end-to-end: failover “en vivo” con usuario navegando

Objetivo: validar el escenario **“estoy logueado, seleccionando un asiento, cae Keycloak en Site A y sigo sin re-login”**.

### Precondiciones

- Keycloak HA multi-cluster desplegado en **Site A** y **Site B** (ver `apps/keycloak-ha/`)
- Data Grid XSite desplegado y saludable en ambos sites (ver `apps/datagrid-ha/`)
- Un entrypoint “global” para Keycloak (LB/Global Accelerator o DNS failover):
  - `https://<KEYCLOAK_URL_HERE>/realms/ticketblaster`
- Frontend configurado para usar ese entrypoint:
  - `REACT_APP_KEYCLOAK_URL=https://<KEYCLOAK_URL_HERE>`
  - `REACT_APP_KEYCLOAK_REALM=ticketblaster`
  - `REACT_APP_KEYCLOAK_CLIENT_ID=frontend`
- Backend `order` apuntando a la misma URL global (recomendado para multi-cluster):
  - set `KEYCLOAK_AUTH_SERVER_URL=https://<KEYCLOAK_URL_HERE>/realms/ticketblaster`

### Paso a paso (happy path)

1. Abre la app TicketBlaster y autentícate (usuario demo, ej. `johndoe/demo`).
2. En “Reservar asiento” selecciona un asiento (no confirmes todavía).
3. **Provoca la caída de Keycloak en Site A**:
   - opción A (simple): escala el `Keycloak`/pods del site A a 0 temporalmente
   - opción B (más realista): “saca” el site A del load balancer/entrypoint global
4. Sin recargar la página:
   - intenta refrescar asientos (botón “Asientos”)
   - intenta navegar a “Órdenes / Timeline”
   - confirma la compra (si el token sigue válido y el backend puede validar JWT, debería funcionar)
5. Restaura Site A (o déjalo abajo) y verifica que:
   - el usuario **no fue deslogueado** por un fallo transitorio de refresh
   - luego de la conmutación, los refresh tokens vuelven a funcionar (Keycloak atiende desde Site B)

### Qué observar / medir

- Tiempo hasta que el refresh token vuelve a funcionar tras el failover
- Errores HTTP en:
  - `.../protocol/openid-connect/token` (refresh)
  - `.../lb-check` (health)
- Impacto en la UX:
  - la UI debería mantenerse autenticada mientras el access token sea válido

### Variante: “partition” entre sites (split-brain)

Si la falla es pérdida de conectividad entre sites (Data Grid XSite), el HA guide menciona que el failure policy `FAIL` prioriza consistencia y puede hacer que el servicio responda error hasta que se ejecute “fencing” (dejar 1 site activo) y luego re-sync.

En esta variante, además de redirigir tráfico, hay que validar:

- qué site queda “online” (y cuál se deshabilita)
- el procedimiento de re-sincronización antes de re-activar el segundo site


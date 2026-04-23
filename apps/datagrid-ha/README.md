## Data Grid (Infinispan) HA multi-cluster (Site-A / Site-B)

Estos manifiestos son un **blueprint** para desplegar Red Hat Data Grid (Infinispan) en **dos clusters OpenShift distintos** y habilitar **cross-site replication** (XSite), siguiendo el enfoque del *High Availability Guide* de Red Hat build of Keycloak 26.4 (capítulo 3.14).

### Qué hay aquí

- `site-a/`: recursos para el cluster **Site A**
- `site-b/`: recursos para el cluster **Site B**

Incluye:

- `Secret` `connect-secret`: credenciales admin del Data Grid (formato `identities.yaml`)
- `Secret` `remote-store-secret`: credenciales que usará Keycloak para conectarse al Data Grid (usuario/clave)
- `Infinispan` CR con `service.sites.*` habilitado (XSite)
- `Cache` CRs requeridos por Keycloak (`actionTokens`, `authenticationSessions`, `loginFailures`, `work`) con backups síncronos

### Prerrequisitos importantes (no incluidos)

Para que XSite funcione en OpenShift, el operador requiere varios artefactos que **no** están versionados aquí porque dependen de tu entorno:

- **TLS keystore/truststore** para la comunicación cross-site (`xsite-keystore-secret`, `xsite-truststore-secret`)
- **Token secret** para que el Data Grid Operator configure la conexión a la otra site (`xsite-token-secret`)
- Conectividad de red entre sites con latencia baja (objetivo < 5 ms, máximo < 10 ms RTT)

### Cómo aplicar

En Site A:

- Apunta ArgoCD (o aplica con `oc apply -k`) a `apps/datagrid-ha/site-a/`

En Site B:

- Apunta ArgoCD (o aplica con `oc apply -k`) a `apps/datagrid-ha/site-b/`

Luego, despliega Keycloak HA apuntando a este Data Grid (ver `../keycloak-ha/`).


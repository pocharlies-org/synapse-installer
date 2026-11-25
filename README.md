# synapse-installer

El esqueleto de [Synapse](#qué-instala) como árbol de kustomize que Flux puede leer. **Este repositorio es generado**:
sale de la fuente de la base, que es privada, y nadie lo edita a mano; un cambio aquí se pierde en la siguiente
publicación. Cada versión es un tag, y un tag no se mueve: quien instala apunta a uno y sube al siguiente cuando quiere.

## Qué instala

Gateway, motor, dispatcher, el migrador del esquema y el plugin de la compañía (`synapse-adapter-supervisor`): lo que
necesita el supervisor de la compañía para funcionar, y ningún otro adapter. Tres raíces, que se aplican en este orden:

| raíz | qué es | cuándo |
|---|---|---|
| `sustrato/` | un PostgreSQL y un RabbitMQ propios (dos StatefulSet, un volumen cada uno) | solo si no hay ya unos que se puedan usar |
| `provision/` | el Job que crea roles, base, cadena de migraciones, vhost y usuarios | una vez, con el sustrato arriba |
| `esqueleto/` | los cuatro servicios y el Job del migrador | después de que `synapse-provision` termine |

Todo lo que renderizan son objetos del namespace (ConfigMap, Service, ServiceAccount, Deployment, StatefulSet, Job):
ningún Namespace, ningún Secret, nada de ámbito de clúster. El namespace lo pone quien instala.

La base de datos nace vacía. La cadena de migraciones crea el esquema y dos tenants de estructura sin perfil: `system`,
para los eventos que no son de ningún tenant, y `company`, el del plugin de la compañía. Ningún otro tenant, ninguna
persona, ninguna identidad y ninguna configuración: eso es de cada instalación y lo da de alta quien instala.

## Lo que pone quien instala

**El namespace.** Los objetos no llevan ninguno: en Flux, `spec.targetNamespace`.

**Las imágenes.** La de Synapse y la del broker no son públicas. La de Synapse es `synapse-core`: solo el núcleo
(gateway, motor, dispatcher, migrador y el plugin de la compañía), sin el código ni los flujos de ninguna otra
instalación. El árbol las nombra con un registro de relleno, `registro.invalid/…`, que no resuelve; `imagenes.txt`
lista cada una con su digest. Quien instala las copia a su
registro y las apunta ahí conservando el digest (en Flux, `spec.images` con `name` y `newName`). El broker tiene que
ser el de `imagenes.txt` o uno equivalente con el plugin de mensajes retardados: el RabbitMQ oficial no lo trae.

**Los Secrets**, con estos nombres y ficheros:

| Secret | ficheros | quién lo monta |
|---|---|---|
| `synapse-secrets` | `synapse_gateway.password`, `synapse_engine.password`, `synapse_dispatcher.password`, `synapse_adapter.password`, `synapse_admin.password`, `synapse_migration.password` (PostgreSQL) y `synapse_gateway.rmq.password`, `synapse_engine.rmq.password`, `synapse_dispatcher.rmq.password`, `synapse_adapter_supervisor.rmq.password`, `synapse_adapter_company.rmq.password` (RabbitMQ) | los servicios y la provisión |
| `synapse-adapter-supervisor` | `company.supervisor.token` | el plugin de la compañía |
| `synapse-provision` | `postgres_superuser.password`, `rabbitmq_admin.password` | solo la provisión y el sustrato |
| `harbor-pull` (de pull) | — | todos; solo si el registro pide credenciales |

El valor de cada contraseña lo elige quien instala: la provisión crea los roles y los usuarios con esos valores.

**Las variables**, que Flux sustituye con `postBuild.substitute`:

| variable | por defecto | qué es |
|---|---|---|
| `synapse_supervisor_url` | — (obligatoria) | la URL del supervisor de la compañía, por ejemplo `http://company:19911` |
| `synapse_pg_host` | `synapse-pg` | el host de PostgreSQL |
| `synapse_rmq_host` | `synapse-rmq` | el host de RabbitMQ |

Flux no sustituye nada si no hay al menos una variable definida, y entonces los valores por defecto tampoco se
aplican: por eso `synapse_supervisor_url` no tiene uno. El ConfigMap de los guiones de provisión lleva la anotación
`kustomize.toolkit.fluxcd.io/substitute: disabled`: es shell, y la sustitución de Flux lo rompería.

## Con Flux

```yaml
apiVersion: source.toolkit.fluxcd.io/v1
kind: GitRepository
metadata: {name: synapse-installer, namespace: flux-system}
spec:
  interval: 30m
  url: https://github.com/pocharlies-org/synapse-installer
  ref: {tag: <tag>}          # un tag, no una rama: lo que corre no cambia hasta que alguien lo sube
---
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata: {name: synapse, namespace: flux-system}
spec:
  interval: 10m
  sourceRef: {kind: GitRepository, name: synapse-installer}
  path: ./esqueleto
  targetNamespace: synapse
  prune: true
  images:
    - {name: registro.invalid/synapse-core, newName: <tu registro>/synapse-core}
  postBuild:
    substitute:
      synapse_supervisor_url: http://company:19911
```

Una `Kustomization` por raíz, cada una con `dependsOn` de la anterior. Un Job no se puede reaplicar cambiado: para
repetir la provisión o el migrador con otra imagen hay que borrar antes el Job.

## Con kustomize o ArgoCD

Sin el controlador de Flux nadie sustituye las variables: quien instala pone sus valores encima, con su propio
`kustomization.yaml`.

```yaml
namespace: synapse
resources:
  - https://github.com/pocharlies-org/synapse-installer//esqueleto?ref=<tag>
images:
  - {name: registro.invalid/synapse-core, newName: <tu registro>/synapse-core}
configMapGenerator:
  - name: synapse-config
    behavior: merge          # tus ficheros sustituyen enteros a los del árbol
    options: {disableNameSuffixHash: true}
    files: [gateway.yaml, engine.yaml, dispatcher.yaml, adapter-supervisor.yaml]
```

Las variables están en dos sitios, y los dos se pisan: los cuatro ficheros de `synapse-config` (copia los del árbol y
escribe en ellos tus hosts) y las variables de entorno `SYNAPSE_POSTGRES_HOST` y `SYNAPSE_POSTGRES_PORT` del Job
`synapse-schema-migrate`. El ConfigMap `adapter-supervisor-config-checksum-…` llega con el hash ya puesto: no cambia
de nombre cuando cambias tu config, así que el plugin hay que reiniciarlo a mano o sustituir ese ConfigMap por uno
generado por ti. Comprueba que en tu render no queda ningún `${synapse_` ni ningún `registro.invalid`.

## Sin Flux

```sh
flux build kustomization synapse --path ./esqueleto --kustomization-file ./kustomization-flux.yaml --dry-run \
  | kubectl apply -f -
```

`flux build` hace en local lo mismo que el controlador: namespace, imágenes y variables. `kubectl kustomize` a secas
construye el árbol pero deja las variables sin sustituir.

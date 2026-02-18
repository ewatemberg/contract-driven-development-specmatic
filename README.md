# Contract Driven Development con Specmatic

Capacitacion practica sobre como usar **Specmatic** para implementar **Contract Testing** y **Service Virtualization (Stubs)** en equipos que trabajan con **API First Development**.

---

## Tabla de contenidos

1. [El problema que resolvemos](#1-el-problema-que-resolvemos)
2. [Que es Specmatic](#2-que-es-specmatic)
3. [Flujo de trabajo API First](#3-flujo-de-trabajo-api-first)
4. [Estructura del proyecto](#4-estructura-del-proyecto)
5. [Convencion de expectativas](#5-convencion-de-expectativas)
6. [Prerequisitos e instalacion](#6-prerequisitos-e-instalacion)
7. [Demo 1 - Faker: stub sin expectativas](#7-demo-1---faker-stub-sin-expectativas)
8. [Demo 2 - Levantar el Stub con expectativas (FE / QAA)](#8-demo-2---levantar-el-stub-con-expectativas-fe--qaa)
9. [Demo 3 - Expectativas externas (QAA)](#9-demo-3---expectativas-externas-qaa)
10. [Demo 4 - Contract Testing (BE)](#10-demo-4---contract-testing-be)
11. [Demo 5 - Repositorio central de contratos](#11-demo-5---repositorio-central-de-contratos)
12. [Demo 6 - Que pasa cuando alguien rompe el contrato](#12-demo-6---que-pasa-cuando-alguien-rompe-el-contrato)
13. [Integracion con Maven (BE)](#13-integracion-con-maven-be)
14. [Comparacion con otras herramientas](#14-comparacion-con-otras-herramientas)
15. [Resumen de beneficios](#15-resumen-de-beneficios)

---

## 1. El problema que resolvemos

Cuando varios equipos (BE, FE, QAA) trabajan en paralelo sobre las mismas APIs:

| Sin Contract Testing | Con Contract Testing |
|---|---|
| FE espera que BE termine para integrar | FE trabaja con stubs desde el dia 1 |
| QAA escribe tests contra mocks manuales que se desactualizan | QAA usa stubs validados contra el contrato |
| BE cambia la API y nadie se entera hasta integracion | BE ejecuta contract tests en cada PR |
| Integracion dolorosa al final del sprint | Integracion continua y sin sorpresas |
| Mocks escritos a mano se dessincronizan | Stubs auto-generados desde la spec |

---

## 2. Que es Specmatic

Specmatic convierte especificaciones de API (OpenAPI, AsyncAPI, gRPC, GraphQL) en:

- **Tests ejecutables** - valida que tu backend cumple el contrato
- **Stubs inteligentes** - genera un servidor mock que respeta la especificacion
- **Validador de backward compatibility** - detecta breaking changes

Todo esto **sin escribir codigo de test manual**.

---

## 3. Flujo de trabajo API First

```
  Paso 1: Disenar API              Paso 2: Trabajo en paralelo
  ┌──────────────────┐         ┌─────────────────────────────────┐
  │  BE + FE + QAA   │         │  FE  --> Stub (mock server)     │
  │  disenan juntos  │  ───>   │  QAA --> Stub + expectativas    │
  │  la OpenAPI spec │         │  BE  --> Contract tests         │
  └──────────────────┘         └─────────────────────────────────┘
                                          │
                                          v
                               Paso 3: Integracion sin sorpresas
                               ┌─────────────────────────────────┐
                               │  CI/CD valida contratos en cada │
                               │  PR automaticamente             │
                               └─────────────────────────────────┘
```

---

## 4. Estructura del proyecto

```
specmatic_tl/
├── README.md                                    # Esta guia
├── .gitignore
├── specmatic.jar                                # Ejecutable de Specmatic
├── specmatic.yaml                               # Config principal (ambas APIs)
├── specmatic-stub-products.yaml                 # Config stub solo Products
├── specmatic-stub-orders.yaml                   # Config stub solo Orders
├── specmatic-test.yaml                          # Config contract testing (BE)
├── specmatic-git.yaml                           # Ejemplo: config apuntando a GitHub
│
└── contracts/                                   # Contratos OpenAPI (fuente de verdad)
    ├── users_api.yaml                           # Contrato SIN examples (demo faker)
    ├── products_api.yaml                        # Contrato de Products API
    ├── products_api_examples/                   # Expectativas externas de Products
    │   ├── get-products/                        #   GET /products
    │   │   ├── 200-ok.json
    │   │   ├── 200-empty.json
    │   │   └── 200-filtrado-por-categoria.json
    │   ├── post-products/                       #   POST /products
    │   │   ├── 201-ok.json
    │   │   ├── 201-minimos.json
    │   │   └── 400-body-invalido.json
    │   ├── get-products-id/                     #   GET /products/{id}
    │   │   ├── 200-ok.json
    │   │   ├── 404-no-existe.json
    │   │   └── 500-error-interno.json
    │   ├── put-products-id/                     #   PUT /products/{id}
    │   │   ├── 200-ok.json
    │   │   └── 404-no-existe.json
    │   └── delete-products-id/                  #   DELETE /products/{id}
    │       ├── 204-ok.json
    │       └── 404-no-existe.json
    │
    ├── orders_api.yaml                          # Contrato de Orders API
    └── orders_api_examples/                     # Expectativas externas de Orders
        ├── get-orders/                          #   GET /orders
        │   ├── 200-ok.json
        │   ├── 200-empty.json
        │   └── 200-filtrado-por-status.json
        ├── post-orders/                         #   POST /orders
        │   ├── 201-ok.json
        │   └── 400-body-invalido.json
        ├── get-orders-id/                       #   GET /orders/{id}
        │   ├── 200-ok.json
        │   ├── 404-no-existe.json
        │   └── 500-error-interno.json
        └── patch-orders-id-status/              #   PATCH /orders/{id}/status
            ├── 200-ok.json
            └── 404-no-existe.json
```

**Convencion de Specmatic:** La carpeta de expectativas debe llamarse `{nombre_del_spec}_examples/` para que Specmatic las detecte automaticamente. Dentro de ella, aplicamos nuestra convencion de equipo (ver seccion siguiente).

---

## 5. Convencion de expectativas

Nuestro equipo definió un estandar para organizar las expectativas de forma que cualquier persona pueda entender que cubre cada archivo **sin abrirlo**.

### 5.1 Regla de nombrado

```
<status>-<descripcion>.json
```

Un **folder por endpoint**, un **archivo por escenario**:

```
{spec}_examples/
  {method}-{path}/
    <status>-<descripcion>.json
```

### 5.2 Nombres de archivo por status code

| Caso | Nombre sugerido |
|---|---|
| Respuesta OK tipica | `200-ok.json` |
| OK con lista vacia | `200-empty.json` |
| OK solo campos obligatorios | `200-minimos.json` / `201-minimos.json` |
| OK todos los campos presentes | `200-maximos.json` |
| Parametro invalido | `400-parametro-invalido.json` |
| Body mal formado | `400-body-invalido.json` |
| Falta campo obligatorio | `400-falta-campo-obligatorio.json` |
| No autorizado | `401-no-auth.json` |
| Sin permisos | `403-sin-permisos.json` |
| No encontrado | `404-no-existe.json` |
| Conflicto | `409-conflicto.json` |
| Regla de negocio | `422-regla-negocio.json` |
| Error interno | `500-error-interno.json` |
| Servicio no disponible | `503-servicio-no-disponible.json` |

### 5.3 Cobertura minima por endpoint

Cada endpoint deberia tener **al menos** estas categorias de expectativas:

**2xx - Casos felices y bordes validos:**
- `200-ok.json` / `201-ok.json` - caso estandar
- `200-empty.json` - lista vacia / sin resultados (si aplica)
- `201-minimos.json` - solo campos obligatorios (para POST/PUT)

**4xx - Errores del cliente (clave para FE):**
- `400-body-invalido.json` - validaciones de request
- `404-no-existe.json` - recurso no encontrado
- `401-no-auth.json` / `403-sin-permisos.json` - si la API tiene auth

> Los errores 4xx son **oro para frontend** porque definen que error mostrar y cuando.

**5xx - Fallos del servidor:**
- `500-error-interno.json` - permite a FE probar pantallas de error realistas
- `503-servicio-no-disponible.json` - si el servicio integra con otros sistemas

### 5.4 Ejemplo concreto aplicado

```
products_api_examples/
  get-products/                        # GET /products
    200-ok.json                        #   Lista con datos
    200-empty.json                     #   Lista vacia (filtro sin resultados)
    200-filtrado-por-categoria.json    #   Filtro por query param
  post-products/                       # POST /products
    201-ok.json                        #   Creacion exitosa
    201-minimos.json                   #   Solo campos obligatorios
    400-body-invalido.json             #   Body con datos invalidos
  get-products-id/                     # GET /products/{id}
    200-ok.json                        #   Producto encontrado
    404-no-existe.json                 #   ID inexistente
    500-error-interno.json             #   Error del servidor
```

### 5.5 Por que esta convencion

- **QAA** entiende que cubre cada archivo sin abrirlo
- **FE** sabe exactamente que respuestas de error existen para manejar en la UI
- **BE** identifica rapido que casos de prueba estan cubiertos
- **Code review** de expectativas es mucho mas rapido
- **Onboarding** de nuevos integrantes al equipo es inmediato

---

## 6. Prerequisitos e instalacion

### Opcion A: JAR directo (recomendado para esta capacitacion)

El JAR de Specmatic esta incluido en este repositorio (`specmatic.jar`). Solo necesitas Java 17+.

```bash
# Verificar que tengas Java
java -version

# Verificar que el JAR funciona
java -jar specmatic.jar --help
```

**Tip:** Para no escribir `java -jar specmatic.jar` cada vez, podes crear un alias:

```bash
# Linux / macOS - agregar a ~/.bashrc o ~/.zshrc
alias specmatic='java -jar /ruta/completa/specmatic.jar'

# Windows - crear specmatic.bat en una carpeta del PATH
# Contenido: java -jar C:\ruta\completa\specmatic.jar %*
```

Descarga de versiones nuevas: [GitHub Releases](https://github.com/specmatic/specmatic/releases) o [Maven Central](https://central.sonatype.com/artifact/io.specmatic/specmatic-executable)

### Opcion B: Docker

```bash
docker pull specmatic/specmatic
```

### Opcion C: npm

```bash
npm install specmatic -g
# o usar con npx sin instalar globalmente
npx specmatic --help
```

---

## 7. Demo 1 - Faker: stub sin expectativas

> **Escenario:** Queres ver como responde Specmatic cuando el contrato **no tiene examples ni expectativas**. Specmatic genera respuestas random pero **siempre validas** segun el schema. Esta es la forma mas rapida de arrancar: solo necesitas el contrato.

### 7.1 Contrato sin examples

En este repo tenemos `contracts/users_api.yaml`, un contrato minimo que **no tiene examples definidos** (ni inline ni externos). Solo define schemas.

Fragmento clave del contrato:

```yaml
components:
  schemas:
    User:
      type: object
      required:
        - id
        - name
        - email
        - role
        - active
      properties:
        id:
          type: integer
        name:
          type: string
        email:
          type: string
          format: email
        role:
          type: string
          enum: [admin, editor, viewer]
        active:
          type: boolean
```

### 7.2 Levantar el stub

```bash
java -jar specmatic.jar stub contracts/users_api.yaml
```

No hay carpeta `users_api_examples/`, no hay inline examples. Solo el contrato.

### 7.3 Probar el faker

```bash
# GET lista de usuarios - Specmatic genera un array con datos random
curl http://localhost:9000/users
```

Respuesta generada automaticamente (ejemplo, los valores varian cada vez):

```json
[
  {
    "id": 382,
    "name": "KBMLY",
    "email": "XRQFP@j.com",
    "role": "viewer",
    "active": true
  }
]
```

```bash
# GET usuario por ID - Specmatic genera un objeto random
curl http://localhost:9000/users/42
```

```json
{
  "id": 819,
  "name": "QZTMR",
  "email": "YPWNL@m.com",
  "role": "admin",
  "active": false
}
```

```bash
# POST crear usuario - Specmatic valida el body y genera la respuesta
curl -X POST http://localhost:9000/users \
  -H "Content-Type: application/json" \
  -d "{\"name\": \"Juan Perez\", \"email\": \"juan@example.com\", \"role\": \"editor\"}"
```

```json
{
  "id": 547,
  "name": "HMVKW",
  "email": "DCFLS@b.com",
  "role": "editor",
  "active": true
}
```

### 7.4 Que garantiza el faker

Aunque los valores son random, **siempre cumplen el contrato**:

| Propiedad | Schema dice | Faker respeta |
|---|---|---|
| `id` | `type: integer` | Siempre devuelve un entero |
| `name` | `type: string` | Siempre devuelve un string |
| `email` | `format: email` | Siempre devuelve formato email valido |
| `role` | `enum: [admin, editor, viewer]` | Siempre devuelve uno de los 3 valores |
| `active` | `type: boolean` | Siempre devuelve true o false |
| campos required | `required: [id, name, email, role, active]` | Nunca falta un campo requerido |

### 7.5 Que pasa si el request es invalido

El faker tambien **valida el request** contra el contrato:

```bash
# Body sin campo obligatorio "email" → Specmatic rechaza
curl -X POST http://localhost:9000/users \
  -H "Content-Type: application/json" \
  -d "{\"name\": \"Juan\", \"role\": \"admin\"}"
```

Specmatic devuelve **HTTP 400** con el detalle de por que el request no es valido.

```bash
# Enum invalido → Specmatic rechaza
curl -X POST http://localhost:9000/users \
  -H "Content-Type: application/json" \
  -d "{\"name\": \"Juan\", \"email\": \"j@x.com\", \"role\": \"superadmin\"}"
```

Specmatic devuelve **HTTP 400** porque `"superadmin"` no esta en el enum.

### 7.6 Faker vs Expectativas: cuando usar cada uno

| | Faker (sin examples) | Expectativas (con examples) |
|---|---|---|
| **Respuestas** | Random pero schema-valid | Exactas, predecibles |
| **Ideal para** | Prototipar rapido, explorar la API | Tests automatizados, desarrollo FE |
| **FE lo usa para** | Ver la estructura de la respuesta | Renderizar datos concretos en la UI |
| **QAA lo usa para** | Validar que el contrato tiene sentido | Cubrir escenarios especificos |

> **Flujo tipico:** Primero el faker te deja arrancar rapido con el contrato solo. Despues, a medida que FE y QAA necesitan respuestas predecibles, agregan expectativas archivo por archivo. Eso es exactamente lo que vemos en la siguiente demo.

---

## 8. Demo 2 - Levantar el Stub con expectativas (FE / QAA)

> **Escenario:** El FE necesita un servidor que responda con **datos predecibles** para desarrollar la UI. A diferencia del faker, aca usamos expectativas (examples) que definen respuestas exactas.

### 8.1 Levantar stub de Products API (puerto 9000)

**Con JAR (recomendado):**

```bash
java -jar specmatic.jar stub --config specmatic-stub-products.yaml --port 9000
```

**Con Docker (Windows):**

```bash
docker run -p 9000:9000 -v "%cd%:/usr/src/app" specmatic/specmatic stub --config specmatic-stub-products.yaml
```

**Con Docker (Linux/macOS):**

```bash
docker run -p 9000:9000 -v "$(pwd):/usr/src/app" specmatic/specmatic stub --config specmatic-stub-products.yaml
```

**Con npx:**

```bash
npx specmatic stub --config specmatic-stub-products.yaml --port 9000
```

> Por defecto el stub levanta en `http://localhost:9000`. A diferencia del faker (Demo 1), aca Specmatic carga las expectativas y devuelve respuestas exactas cuando matchean.

### 8.2 Probar que funciona

Abrir otra terminal y ejecutar:

```bash
# Listar productos (responde con datos del inline example)
curl http://localhost:9000/products

# Crear un producto (matchea con el inline example CREATE_PRODUCT_SUCCESS)
curl -X POST http://localhost:9000/products \
  -H "Content-Type: application/json" \
  -d "{\"name\": \"Bluetooth Speaker\", \"price\": 49.99, \"category\": \"electronics\", \"stock\": 75}"

# Obtener producto por ID (matchea con get-products-id/200-ok.json)
curl http://localhost:9000/products/1

# Producto no encontrado (matchea con get-products-id/404-no-existe.json)
curl http://localhost:9000/products/9999

# Listar por categoria (matchea con get-products/200-filtrado-por-categoria.json)
curl "http://localhost:9000/products?category=clothing"
```

> **Notar la diferencia con Demo 1:** Aca las respuestas son siempre las mismas (predecibles), no random. Eso es clave para que FE pueda renderizar datos concretos y QAA pueda escribir assertions estables.

### 8.3 Que esta pasando internamente

1. Specmatic lee el contrato `products_api.yaml`
2. Carga los **inline examples** definidos en el YAML
3. Carga los **external examples** de la carpeta `products_api_examples/`
4. Levanta un servidor HTTP en el puerto 9000
5. Cada request entrante se valida contra el contrato
6. Si matchea un ejemplo, devuelve esa respuesta exacta
7. Si no matchea un ejemplo pero es valido segun el schema, genera una respuesta automatica (faker)

### 8.4 Levantar ambos stubs en paralelo

```bash
# Terminal 1: Products API en puerto 9000
java -jar specmatic.jar stub --config specmatic-stub-products.yaml --port 9000

# Terminal 2: Orders API en puerto 9001
java -jar specmatic.jar stub --config specmatic-stub-orders.yaml --port 9001
```

### 8.5 Stub directo de un contrato (sin config)

Si solo queres levantar un stub rapido de un contrato sin archivo de configuracion:

```bash
# Stub de un solo archivo OpenAPI
java -jar specmatic.jar stub contracts/products_api.yaml

# Stub con puerto custom
java -jar specmatic.jar stub contracts/products_api.yaml --port 8090

# Stub con debug (muestra que example files carga)
java -jar specmatic.jar stub contracts/products_api.yaml --debug
```

### 8.6 Health check

```bash
curl http://localhost:9000/actuator/health
# Respuesta: {"status":"UP"}
```

---

## 9. Demo 3 - Expectativas externas (QAA)

> **Escenario:** QAA necesita agregar escenarios de prueba adicionales sin modificar el contrato.

### 9.1 Anatomia de una expectativa

Archivo: `contracts/products_api_examples/get-products-id/404-no-existe.json`

```json
{
  "http-request": {
    "method": "GET",
    "path": "/products/9999"
  },
  "http-response": {
    "status": 404,
    "headers": {
      "Content-Type": "application/json"
    },
    "body": {
      "code": 404,
      "message": "Product not found"
    }
  }
}
```

Notar como el nombre del archivo (`404-no-existe.json`) dentro de la carpeta del endpoint (`get-products-id/`) deja claro que escenario cubre sin necesidad de abrirlo.

**Reglas:**
- El request y response deben ser validos segun el contrato OpenAPI
- Si la expectativa viola el contrato, Specmatic la rechaza al arrancar (fail fast)
- Las expectativas no pueden inventar campos que no existen en el schema

### 9.2 Agregar una nueva expectativa en vivo

Con el stub corriendo, crear un archivo `contracts/products_api_examples/post-products/201-categoria-food.json`:

```json
{
  "http-request": {
    "method": "POST",
    "path": "/products",
    "headers": {
      "Content-Type": "application/json"
    },
    "body": {
      "name": "Organic Honey",
      "price": 12.99,
      "category": "food",
      "stock": 100
    }
  },
  "http-response": {
    "status": 201,
    "headers": {
      "Content-Type": "application/json"
    },
    "body": {
      "id": 30,
      "name": "Organic Honey",
      "price": 12.99,
      "category": "food",
      "stock": 100
    }
  }
}
```

Specmatic detecta el nuevo archivo automaticamente (hot-reload) y lo carga.

Probar:

```bash
curl -X POST http://localhost:9000/products \
  -H "Content-Type: application/json" \
  -d "{\"name\": \"Organic Honey\", \"price\": 12.99, \"category\": \"food\", \"stock\": 100}"
```

### 9.3 Expectativas dinamicas via API

QAA tambien puede setear expectativas programaticamente:

```bash
curl -X POST http://localhost:9000/_specmatic/expectations \
  -H "Content-Type: application/json" \
  -d "{\"http-request\": {\"method\": \"GET\", \"path\": \"/products/42\"}, \"http-response\": {\"status\": 200, \"body\": {\"id\": 42, \"name\": \"Test Product\", \"price\": 9.99, \"category\": \"electronics\", \"stock\": 1}}}"
```

### 9.4 Modo estricto

Para que el stub SOLO responda con expectativas definidas (sin generar respuestas automaticas):

```bash
java -jar specmatic.jar stub --config specmatic-stub-products.yaml --strict
```

En modo estricto, si llega un request que no matchea ninguna expectativa, Specmatic devuelve **HTTP 400** en lugar de generar una respuesta.

---

## 10. Demo 4 - Contract Testing (BE)

> **Escenario:** BE esta desarrollando la API y quiere validar que cumple con el contrato.

### 10.1 Ejecutar contract tests contra una API real

Suponiendo que el BE tiene su API corriendo en `http://localhost:8080`:

**Con JAR (recomendado):**

```bash
java -jar specmatic.jar test --config specmatic-test.yaml --testBaseURL http://localhost:8080
```

**Con JAR contra un contrato directo (sin config):**

```bash
java -jar specmatic.jar test contracts/products_api.yaml --testBaseURL http://localhost:8080
```

**Con Docker (Windows):**

```bash
docker run -v "%cd%:/usr/src/app" --network host specmatic/specmatic test --config specmatic-test.yaml --testBaseURL http://localhost:8080
```

**Con npx:**

```bash
npx specmatic test --config specmatic-test.yaml --testBaseURL http://localhost:8080
```

### 10.2 Que testea Specmatic automaticamente

A partir del contrato, Specmatic genera tests para:

| Que valida | Ejemplo |
|---|---|
| Status codes correctos | POST /products devuelve 201 |
| Schema del response | El body tiene `id`, `name`, `price`, `category`, `stock` |
| Tipos de datos | `price` es number, `stock` es integer |
| Campos requeridos | No falta ningun campo required |
| Valores de enum | `category` es uno de: electronics, clothing, food |
| Content-Type headers | Response es application/json |

### 10.3 Generar reporte JUnit

```bash
java -jar specmatic.jar test --config specmatic-test.yaml --testBaseURL http://localhost:8080 --junitReportDir ./build/reports
```

El reporte se genera en `./build/reports/` y puede integrarse en CI/CD (Jenkins, GitHub Actions, etc.).

El reporte HTML se genera en `./build/reports/specmatic/html/index.html`.

### 10.4 Tests generativos (boundary conditions)

Specmatic puede generar automaticamente tests de boundary conditions (valores limite, nulos, strings vacios, etc.):

```bash
# Activar tests generativos
set SPECMATIC_GENERATIVE_TESTS=true
java -jar specmatic.jar test --config specmatic-test.yaml --testBaseURL http://localhost:8080

# Solo tests positivos (sin negativos)
set ONLY_POSITIVE=true
java -jar specmatic.jar test --config specmatic-test.yaml --testBaseURL http://localhost:8080
```

### 10.5 Filtrar tests especificos

```bash
# Solo tests de POST
java -jar specmatic.jar test --config specmatic-test.yaml --testBaseURL http://localhost:8080 --filter="METHOD='POST'"

# Solo tests de un endpoint especifico
java -jar specmatic.jar test --config specmatic-test.yaml --testBaseURL http://localhost:8080 --filter="PATH='/products/{id}'"

# Solo tests que devuelven 200
java -jar specmatic.jar test --config specmatic-test.yaml --testBaseURL http://localhost:8080 --filter="STATUS='200'"

# Por operationId del contrato OpenAPI
java -jar specmatic.jar test --config specmatic-test.yaml --testBaseURL http://localhost:8080 --filter="OPERATION-ID='getProductById'"

# Por tag
java -jar specmatic.jar test --config specmatic-test.yaml --testBaseURL http://localhost:8080 --filter="TAGS='Products'"

# Combinar filtros
java -jar specmatic.jar test --config specmatic-test.yaml --testBaseURL http://localhost:8080 --filter="METHOD='POST' && PATH='/products'"
```

---

## 11. Demo 5 - Repositorio central de contratos

> **Escenario:** Todos los equipos referencian los contratos desde un unico repositorio en GitHub.

### 11.1 Estructura del repo central

```
github.com/tu-org/specmatic-contracts/
├── contracts/
│   ├── products_api.yaml
│   ├── products_api_examples/
│   │   ├── get-products/
│   │   │   └── 200-ok.json, 200-empty.json, ...
│   │   ├── post-products/
│   │   │   └── 201-ok.json, 400-body-invalido.json, ...
│   │   └── ...
│   ├── orders_api.yaml
│   └── orders_api_examples/
│       ├── get-orders/
│       │   └── 200-ok.json, 200-empty.json, ...
│       └── ...
└── README.md
```

### 11.2 Configuracion apuntando a GitHub

Archivo `specmatic-git.yaml` (incluido en este repo como referencia):

```yaml
version: 3
dependencies:
  services:
    - service:
        description: Products API
        definitions:
          - definition:
              source:
                git:
                  url: https://github.com/TU-ORG/specmatic-contracts.git
                  branch: main
              specs:
                - spec:
                    id: productsApi
                    path: contracts/products_api.yaml
```

### 11.3 Levantar stub desde el repo central

```bash
# Con JAR
java -jar specmatic.jar stub --config specmatic-git.yaml

# Con Docker
docker run -p 9000:9000 -v "%cd%:/usr/src/app" specmatic/specmatic stub --config specmatic-git.yaml
```

Specmatic clona el repo, descarga los contratos y las expectativas, y levanta el stub. **Todos los equipos usan la misma fuente de verdad.**

### 11.4 Flujo con branches

Con la opcion `--match-branch`, Specmatic busca una branch con el mismo nombre que la branch actual del repo donde se ejecuta. Esto permite que el equipo trabaje en features branches sin afectar main:

```bash
# Si estas en branch "feature/new-payment", Specmatic busca esa branch en el repo de contratos
java -jar specmatic.jar stub --config specmatic-git.yaml --match-branch
```

---

## 12. Demo 6 - Que pasa cuando alguien rompe el contrato

> **Escenario:** Alguien modifica el contrato y queremos detectarlo temprano.

### 12.1 Simulacion: BE devuelve un campo con tipo incorrecto

Si el BE devuelve `price` como string en lugar de number:

```json
{
  "id": 1,
  "name": "Laptop",
  "price": "mil doscientos",
  "category": "electronics",
  "stock": 25
}
```

**Specmatic contract test falla:**

```
Test Failed: GET /products/1
  Expected: price (number)
  Actual: price (string) "mil doscientos"
```

### 12.2 Simulacion: QAA agrega expectativa invalida

Si QAA crea una expectativa con un campo que no existe en el schema:

```json
{
  "http-response": {
    "body": {
      "id": 1,
      "name": "Laptop",
      "price": 1299.99,
      "category": "electronics",
      "stock": 25,
      "discount": 10
    }
  }
}
```

**Specmatic rechaza la expectativa al arrancar el stub:**

```
Error: Key "discount" in the response body is not defined in the schema
```

### 12.3 En el CI/CD

```yaml
# Ejemplo GitHub Actions
name: Contract Tests
on: [pull_request]

jobs:
  contract-test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Start API
        run: ./start-api.sh &

      - name: Run contract tests (JAR)
        run: |
          java -jar specmatic.jar test \
            --config specmatic-test.yaml \
            --testBaseURL http://localhost:8080 \
            --junitReportDir ./build/reports

      - name: Upload test report
        uses: actions/upload-artifact@v4
        if: always()
        with:
          name: contract-test-report
          path: ./build/reports/
```

---

## 13. Integracion con Maven (BE)

> **Escenario:** El equipo de BE quiere que los contract tests se ejecuten automaticamente con `mvn test`, igual que cualquier otro test unitario o de integracion.

### 13.1 Dependencias Maven

Agregar al `pom.xml` del proyecto BE:

```xml
<dependencies>
    <!-- Specmatic JUnit5 Support - ejecuta contract tests como tests de JUnit -->
    <dependency>
        <groupId>io.specmatic</groupId>
        <artifactId>junit5-support</artifactId>
        <version>2.39.5</version>
        <scope>test</scope>
    </dependency>

    <!-- JUnit 5 -->
    <dependency>
        <groupId>org.junit.jupiter</groupId>
        <artifactId>junit-jupiter</artifactId>
        <version>5.10.2</version>
        <scope>test</scope>
    </dependency>
</dependencies>
```

> **Nota:** Verificar la ultima version disponible en [Maven Central](https://central.sonatype.com/artifact/io.specmatic/junit5-support). El groupId anterior `in.specmatic` esta deprecado, usar siempre `io.specmatic`.

### 13.2 Clase de Contract Test (zero codigo)

Crear la clase en `src/test/java/`:

**Java:**

```java
package com.tu.empresa.tests;

import io.specmatic.test.SpecmaticContractTest;

public class ContractTests implements SpecmaticContractTest {
    // No necesita codigo.
    // Specmatic genera los tests automaticamente desde la spec.
}
```

**Kotlin:**

```kotlin
package com.tu.empresa.tests

import io.specmatic.test.SpecmaticContractTest

class ContractTests : SpecmaticContractTest
```

Eso es todo. **Zero glue code.** Specmatic se encarga de generar y ejecutar los tests basandose en la configuracion.

### 13.3 Configuracion specmatic.yaml (en la raiz del proyecto BE)

El archivo `specmatic.yaml` debe estar en el mismo directorio que el `pom.xml`:

**Opcion A: Contratos en filesystem local (para desarrollo)**

```yaml
version: 3
systemUnderTest:
  service:
    $ref: "#/components/services/productsService"
components:
  services:
    productsService:
      description: Products API
      definitions:
        - definition:
            source:
              filesystem:
                directory: .
            specs:
              - spec:
                  path: ./contracts/products_api.yaml
```

**Opcion B: Contratos desde un repositorio Git central (para CI/CD)**

```yaml
version: 3
systemUnderTest:
  service:
    $ref: "#/components/services/productsService"
components:
  sources:
    contractRepo:
      git:
        url: https://github.com/TU-ORG/specmatic-contracts.git
        branch: main
  services:
    productsService:
      description: Products API
      definitions:
        - definition:
            source:
              $ref: "#/components/sources/contractRepo"
            specs:
              - spec:
                  path: contracts/products_api.yaml
```

**Opcion C: Con autenticacion (repos privados en CI/CD)**

```yaml
components:
  sources:
    contractRepo:
      git:
        url: https://${CONTRACT_REPO_TOKEN}@github.com/TU-ORG/specmatic-contracts.git
        branch: main
```

### 13.4 Ejecutar con Maven

```bash
# Ejecutar contract tests (la app BE debe estar corriendo)
mvn test

# Solo la clase de contract tests
mvn test -Dtest=ContractTests

# Con output verbose
mvn test -Dtest=ContractTests -Dsurefire.useFile=false
```

El resultado se ve como cualquier test de JUnit:

```
[INFO] Running com.tu.empresa.tests.ContractTests
Tests run: 12, Failures: 0, Errors: 0, Skipped: 0
[INFO] BUILD SUCCESS
```

Los reportes se generan en:
- JUnit XML: `target/surefire-reports/`
- HTML (Specmatic): `build/reports/specmatic/html/index.html`

### 13.5 Configurar Maven Surefire (opcional)

Si necesitas configurar el plugin de Surefire para los contract tests:

```xml
<build>
    <plugins>
        <plugin>
            <groupId>org.apache.maven.plugins</groupId>
            <artifactId>maven-surefire-plugin</artifactId>
            <version>3.2.5</version>
            <configuration>
                <!-- Incluir los contract tests -->
                <includes>
                    <include>**/*ContractTests.java</include>
                    <include>**/*Test.java</include>
                </includes>
                <!-- Variables de entorno para Specmatic -->
                <environmentVariables>
                    <!-- Activar tests generativos (boundary conditions) -->
                    <!-- <SPECMATIC_GENERATIVE_TESTS>true</SPECMATIC_GENERATIVE_TESTS> -->
                    <!-- Solo tests positivos -->
                    <!-- <ONLY_POSITIVE>true</ONLY_POSITIVE> -->
                </environmentVariables>
            </configuration>
        </plugin>
    </plugins>
</build>
```

### 13.6 Perfil Maven para contract tests separados

Si queres separar los contract tests de los tests unitarios con un perfil:

```xml
<profiles>
    <profile>
        <id>contract-tests</id>
        <build>
            <plugins>
                <plugin>
                    <groupId>org.apache.maven.plugins</groupId>
                    <artifactId>maven-surefire-plugin</artifactId>
                    <version>3.2.5</version>
                    <configuration>
                        <includes>
                            <include>**/*ContractTests.java</include>
                        </includes>
                    </configuration>
                </plugin>
            </plugins>
        </build>
    </profile>
</profiles>
```

```bash
# Solo contract tests
mvn test -Pcontract-tests

# Solo tests unitarios (excluir contract tests)
mvn test -Dtest='!ContractTests'
```

### 13.7 Ejemplo completo de pom.xml minimo

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0
         http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>com.tu.empresa</groupId>
    <artifactId>products-api</artifactId>
    <version>1.0.0</version>

    <properties>
        <maven.compiler.source>17</maven.compiler.source>
        <maven.compiler.target>17</maven.compiler.target>
        <specmatic.version>2.39.5</specmatic.version>
    </properties>

    <dependencies>
        <!-- Tu framework (Spring Boot, Quarkus, etc.) -->
        <!-- ... -->

        <!-- Specmatic Contract Testing -->
        <dependency>
            <groupId>io.specmatic</groupId>
            <artifactId>junit5-support</artifactId>
            <version>${specmatic.version}</version>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>org.junit.jupiter</groupId>
            <artifactId>junit-jupiter</artifactId>
            <version>5.10.2</version>
            <scope>test</scope>
        </dependency>
    </dependencies>

    <build>
        <plugins>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-surefire-plugin</artifactId>
                <version>3.2.5</version>
            </plugin>
        </plugins>
    </build>
</project>
```

### 13.8 Gradle equivalente

Para equipos que usan Gradle en lugar de Maven:

```kotlin
// build.gradle.kts
dependencies {
    testImplementation("io.specmatic:junit5-support:2.39.5")
    testImplementation("org.junit.jupiter:junit-jupiter:5.10.2")
}

tasks.test {
    useJUnitPlatform()
}
```

```bash
# Ejecutar contract tests con Gradle
./gradlew test

# Solo contract tests
./gradlew test --tests '*ContractTests'
```

### 13.9 Workflow completo por rol con Maven

```
BE Developer:
  1. Clona el repo del servicio
  2. specmatic.yaml ya apunta a los contratos (local o Git)
  3. Desarrolla la API
  4. mvn test               ← Contract tests se ejecutan automaticamente
  5. Si falla → el contrato no se cumple, corregir antes de pushear
  6. git push → CI ejecuta mvn test → contract tests validan en el pipeline
```

---

## 14. Comparacion con otras herramientas

> **Contexto para la capacitacion:** Es comun que otros lideres pregunten "por que no usar WireMock?" o "Pact no hace lo mismo?". Esta seccion responde esas preguntas con datos concretos.

### 14.1 Las 4 herramientas mas comunes

| Herramienta | Enfoque | En pocas palabras |
|---|---|---|
| **Specmatic** | Contract-Driven Development (CDD) | La spec OpenAPI ES el contrato. Todo se genera de ahi. |
| **WireMock** | Mock-First / Record-Replay | Servidor de mocks manual. No valida contratos. |
| **Pact** | Consumer-Driven Contract Testing (CDCT) | El consumer define el contrato en codigo. |
| **Spring Cloud Contract** | Provider-Driven + CDC | Contratos en Groovy DSL, genera stubs y tests. |

### 14.2 Tabla comparativa detallada

| Dimension | Specmatic | WireMock | Pact | Spring Cloud Contract |
|---|---|---|---|---|
| **Fuente del contrato** | OpenAPI spec (ya existente) | JSON mappings manuales | Generado desde codigo del consumer | Groovy DSL / YAML separado |
| **Single source of truth** | Si (spec = contrato = stubs = tests) | No (mocks separados de la API real) | No (pact files + spec son cosas distintas) | No (DSL separado de la spec) |
| **Genera stubs automaticos** | Si, desde la spec | No, se escriben a mano | Solo desde el DSL del consumer | Si, desde el DSL del contrato |
| **Genera contract tests** | Si, zero codigo | No tiene | Si, pero requiere codigo en ambos lados | Si, genera JUnit/Spock |
| **Codigo glue necesario** | Ninguno | Mappings JSON manuales | DSL del consumer + provider states | Groovy DSL / YAML |
| **Detecta breaking changes** | Si, spec vs spec sin codigo | No | Solo con provider corriendo | Solo con provider corriendo |
| **Soporte OpenAPI nativo** | Si, first-class | Import limitado (no valida) | No (Pactflow: parcial, de pago) | No built-in |
| **Trabajo en paralelo** | Si, desde el dia 1 | N/A (solo mocks) | No, consumer primero | Parcial (consumer espera al provider) |
| **Broker/Servidor extra** | No necesita | No necesita | Si, Pact Broker o Pactflow | Maven repo o SCM |
| **Protocolos** | HTTP, gRPC, Kafka, GraphQL, WebSocket | HTTP | HTTP | HTTP |
| **Ecosistema** | Cualquier lenguaje (CLI/Docker) | Java + Docker | 10+ lenguajes nativos | JVM-centric |
| **Curva de aprendizaje** | Baja (solo conocer tu spec) | Baja para mocking | Alta (DSL + Broker + states) | Media (requiere Spring) |
| **Licencia** | MIT | Apache 2.0 | MIT | Apache 2.0 |

### 14.3 Specmatic vs WireMock

WireMock es la herramienta mas conocida para mocking de APIs. La diferencia fundamental:

| | WireMock | Specmatic |
|---|---|---|
| **Proposito** | Servidor de mocks | Contract testing + mocks inteligentes |
| **Quien define los mocks** | Un dev escribe JSONs a mano | Se generan de la spec OpenAPI |
| **Validacion** | Ninguna. Si la API cambia, el mock sigue devolviendo datos viejos | Cada request/response se valida contra la spec |
| **Drift** | Se dessincronizan silenciosamente | Imposible: si viola la spec, Specmatic lo rechaza |
| **Mantenimiento** | Actualizar mappings cada vez que la API cambia | Actualizar solo la spec, stubs se regeneran |

**El problema real con WireMock:**

```
Dia 1:  WireMock devuelve { "price": 19.99 }        ← OK
Dia 30: BE cambia price a string "19.99"             ← WireMock sigue devolviendo number
Dia 45: FE integra con BE real                       ← BUG en produccion
Dia 45: "Pero... en mi mock funcionaba!" 😤
```

Con Specmatic ese escenario es imposible porque el contrato es la unica fuente de verdad.

> **Cuando SI usar WireMock:** Si ya tenes mocks de WireMock y solo necesitas mocking sin contract testing, WireMock cumple. Pero si necesitas **garantizar que los mocks representan la API real**, necesitas Specmatic.

### 14.4 Specmatic vs Pact

Pact es la herramienta mas conocida de contract testing. La diferencia clave es **quien define el contrato**:

| | Pact | Specmatic |
|---|---|---|
| **Quien escribe el contrato** | El consumer, en codigo | Todos juntos, en la spec OpenAPI |
| **Workflow** | Consumer primero → genera pact → provider verifica | Spec primero → todos trabajan en paralelo |
| **Infraestructura extra** | Pact Broker (servidor, DB, mantenimiento) | Nada, lee specs desde filesystem o Git |
| **Provider states** | Si, codigo custom en el provider | No necesita |
| **Curva de aprendizaje** | DSL especifico + Broker + states por lenguaje | Solo tu spec OpenAPI |
| **Stakeholders** | Solo devs/SDETs pueden participar | Cualquiera que lea una spec OpenAPI |

**El problema real con Pact:**

```
1. Consumer escribe tests con Pact DSL           ← Solo devs pueden hacerlo
2. Genera pact file (JSON)                        ← Artefacto intermedio a mantener
3. Publica al Pact Broker                         ← Servidor que hay que operar
4. Provider descarga el pact                      ← Dependencia secuencial
5. Provider implementa "provider states"          ← Mas codigo glue
6. Provider verifica contra el pact               ← Recien aca sabemos si funciona
```

Con Specmatic:

```
1. Equipo escribe la spec OpenAPI                 ← Todos participan
2. specmatic mock / specmatic test                ← Listo, a trabajar en paralelo
```

> **Cuando SI usar Pact:** Si tu equipo no hace API First Development, no tiene specs OpenAPI, y el consumer necesita conducir el contrato. Pero si ya tenes specs OpenAPI (o queres adoptarlas), Specmatic es mas directo.

### 14.5 Specmatic vs Spring Cloud Contract

| | Spring Cloud Contract | Specmatic |
|---|---|---|
| **Ecosistema** | Solo JVM (Java/Kotlin/Groovy) | Cualquier lenguaje |
| **Contrato** | Groovy DSL o YAML separado de la spec | La spec OpenAPI es el contrato |
| **Stubs** | Se publican en Maven repo | Se generan on-the-fly de la spec |
| **Consumer** | Espera que provider publique stubs | Trabaja desde el dia 1 con la spec |
| **Mantenimiento** | DSL + spec + stubs publicados | Solo la spec |

> **Cuando SI usar Spring Cloud Contract:** Si todo tu stack es Spring Boot y ya lo usas. Pero si tenes equipos multi-lenguaje o queres evitar el acoplamiento al ecosistema Spring, Specmatic es mejor opcion.

### 14.6 Resumen visual

```
                    ¿Necesitas contract testing?
                            │
                     ┌──────┴──────┐
                     │ NO          │ SI
                     │             │
                ¿Solo mocking?    ¿Tenes spec OpenAPI?
                     │                   │
                ┌────┴────┐        ┌─────┴─────┐
                │ SI      │        │ SI        │ NO
                │         │        │           │
            WireMock   Nada    Specmatic    ¿Quien define
            (pero         ←────           el contrato?
            considerar                        │
            Specmatic                   ┌─────┴─────┐
            que tambien                 │ Consumer   │ Provider
            mockea)                     │            │
                                      Pact     Spring Cloud
                                               Contract
```

---

## 15. Resumen de beneficios

### Para el equipo

| Rol | Como usa Specmatic | Beneficio |
|---|---|---|
| **FE** | Levanta stub con `specmatic stub` | Desarrolla sin depender de BE |
| **QAA** | Agrega expectativas + usa stub para E2E | Tests confiables y sincronizados |
| **BE** | Ejecuta `specmatic test` o `mvn test` en cada PR | Valida el contrato automaticamente |
| **TL** | Revisa que el contrato se cumpla en CI/CD | Confianza en la integracion |

### Para la organizacion

- **Reduce tiempo de integracion** - Los problemas se detectan en el PR, no en staging
- **Single source of truth** - El contrato OpenAPI es la unica fuente de verdad
- **Sin mocks manuales** - Los stubs se generan del contrato, no se escriben a mano
- **Backward compatibility** - Specmatic detecta breaking changes antes de mergear
- **Zero glue code** - No hay que escribir codigo de test, se genera del contrato

---

## Comandos rapidos de referencia

```bash
# ================================================================
# JAR (recomendado)
# ================================================================

# === STUB (FE / QAA) ===

# Stub con config
java -jar specmatic.jar stub --config specmatic.yaml

# Stub de un solo contrato
java -jar specmatic.jar stub contracts/products_api.yaml

# Stub con puerto custom
java -jar specmatic.jar stub contracts/products_api.yaml --port 8090

# Stub en modo estricto
java -jar specmatic.jar stub --config specmatic.yaml --strict

# Stub con debug (ver que examples carga)
java -jar specmatic.jar stub contracts/products_api.yaml --debug

# Health check
curl http://localhost:9000/actuator/health


# === CONTRACT TESTING (BE) ===

# Contract tests con config
java -jar specmatic.jar test --config specmatic-test.yaml --testBaseURL http://localhost:8080

# Contract tests directo contra un contrato
java -jar specmatic.jar test contracts/products_api.yaml --testBaseURL http://localhost:8080

# Con reporte JUnit
java -jar specmatic.jar test --config specmatic-test.yaml --testBaseURL http://localhost:8080 --junitReportDir ./build/reports

# Con filtros
java -jar specmatic.jar test --config specmatic-test.yaml --testBaseURL http://localhost:8080 --filter="METHOD='POST'"

# Tests generativos (boundary conditions)
set SPECMATIC_GENERATIVE_TESTS=true
java -jar specmatic.jar test --config specmatic-test.yaml --testBaseURL http://localhost:8080


# === DESDE GITHUB ===

# Stub desde repo central
java -jar specmatic.jar stub --config specmatic-git.yaml

# Stub con match-branch
java -jar specmatic.jar stub --config specmatic-git.yaml --match-branch


# === MAVEN (BE) ===

# Ejecutar contract tests via Maven
mvn test

# Solo contract tests
mvn test -Dtest=ContractTests

# Contract tests con perfil
mvn test -Pcontract-tests


# ================================================================
# DOCKER (alternativa)
# ================================================================

# Stub (Windows)
docker run -p 9000:9000 -v "%cd%:/usr/src/app" specmatic/specmatic stub --config specmatic.yaml

# Stub (Linux/macOS)
docker run -p 9000:9000 -v "$(pwd):/usr/src/app" specmatic/specmatic stub --config specmatic.yaml

# Contract tests
docker run -v "%cd%:/usr/src/app" --network host specmatic/specmatic test --config specmatic-test.yaml --testBaseURL http://localhost:8080
```

---

## Links utiles

- [Documentacion oficial](https://docs.specmatic.io/contract_driven_development)
- [CLI Quick Start](https://docs.specmatic.io/getting_started/cli_quick_start.html)
- [Contract Testing](https://docs.specmatic.io/contract_driven_development/contract_testing.html)
- [Service Virtualization](https://docs.specmatic.io/contract_driven_development/service_virtualization)
- [GitHub de Specmatic](https://github.com/specmatic/specmatic)
- [Maven Central - junit5-support](https://central.sonatype.com/artifact/io.specmatic/junit5-support)
- [Specmatic vs WireMock (oficial)](https://specmatic.io/comparisons/comparison-specmatic-vs-wiremock/)
- [Specmatic vs Pact (oficial)](https://specmatic.io/comparisons/specmatic-vs-pact-io-and-pactflow-io/)
- [Specmatic vs Spring Cloud Contract (oficial)](https://specmatic.io/comparisons/comparison-specmatic-vs-spring-cloud-contract/)
- [Tipos de Contract Testing](https://specmatic.io/updates/types-of-contract-testing/)

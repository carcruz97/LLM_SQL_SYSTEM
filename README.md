# Arquitectura LLM SQL Governada

## Propuesta Técnica

## 1. Objetivo

El sistema permite que usuarios de negocio consulten información analítica en lenguaje natural, sin necesidad de saber SQL ni la estructura interna de la base.

El usuario escribe una consulta como:

> “¿Cuántas ventas tuvimos en marzo por región?”

y recibe una respuesta narrativa, gobernada y basada en datos reales. 📊

---

## 2. Principios de diseño

La arquitectura está organizada en **4 layers funcionales**:

| Layer                                   | Responsabilidad                      |
| --------------------------------------- | ------------------------------------ |
| **L1 · Interaction Layer**              | Entrada y salida del usuario         |
| **L2 · Optional Ingestion Layer**       | Carga y transformación de datos      |
| **L3 · Analytical Engine Layer**        | Generación analítica con LLM         |
| **L4 · Governance & Persistence Layer** | Gobernanza, seguridad y persistencia |

La idea central es simple:
el usuario interactúa solo con la capa de interfaz; el engine razona sobre contexto y metadata; y todo acceso a datos reales pasa por gobernanza. 🔒

---

## 3. Separación entre dato y almacenamiento

Acá conviene distinguir dos cosas:

* **dato lógico**: el contenido que circula por el sistema,
* **storage / persistencia**: el lugar físico donde ese dato se guarda.

### Datos lógicos

| Dato                         | Qué representa                      |
| ---------------------------- | ----------------------------------- |
| `USER_MESSAGE`               | Pregunta del usuario                |
| `ENRICHED_QUESTION_BUSINESS` | Pregunta analítica enriquecida      |
| `RAW_QUERIES`                | SQL generado por el engine          |
| `RESULT_QUERIES`             | Resultado real devuelto por la base |
| `DESCRIPTION_QUERIES`        | Resumen censurado del resultado     |
| `PRELIMINAR_RESPONSE`        | Respuesta narrativa preliminar      |
| `SYSTEM_MESSAGE`             | Respuesta final para mostrar        |

### Persistencia por etapa

| Etapa         | Persistencia lógica                                 | Soporte físico                                                                                                 |
| ------------- | --------------------------------------------------- | -------------------------------------------------------------------------------------------------------------- |
| **PoC / MVP** | `data_business`, `context_business`, `user_history` | **3 schemas en PostgreSQL**                                                                                    |
| **Scale**     | `data_business`, `context_business`, `user_history` | `context_business` y `user_history` pasan a **2 buckets** separados; `data_business` queda como base operativa |

### Qué guarda cada persistencia

| Recurso            | Guarda                                             |
| ------------------ | -------------------------------------------------- |
| `data_business`    | Datos reales del negocio                           |
| `context_business` | Metadata, configuración y contexto semántico       |
| `user_history`     | `USER_MESSAGE` y contexto conversacional censurado |

---

## 4. Arquitectura por layers

---

## L1 · Interaction Layer

Esta capa es el único punto de contacto con el usuario.

| Nodo                          | Tipo        | Descripción                            |
| ----------------------------- | ----------- | -------------------------------------- |
| `USER`                        | Actor       | Usuario de negocio                     |
| `UI_DISPLAY`                  | Frontend    | Interfaz visual de chat                |
| `USER_MESSAGE`                | Data Object | Mensaje de entrada en lenguaje natural |
| `CONTEXT_BUSINESS_CONFIG_LLM` | Data Object | Configuración contextual censurada     |
| `SYSTEM_MESSAGE`              | Data Object | Respuesta final mostrada al usuario    |

**Responsabilidad:** capturar la consulta, transmitir el contexto necesario y mostrar la respuesta.
No interpreta SQL ni accede a datos reales. 🧩

---

## L2 · Optional Ingestion Layer

Esta capa gestiona la carga batch de datos hacia la base de negocio. No participa del flujo online del usuario.

| Nodo                 | Tipo        | Descripción                     |
| -------------------- | ----------- | ------------------------------- |
| `SCHEDULER`          | Workflow    | Dispara procesos programados    |
| `CSV_BUSINESS_RAW`   | Data Object | Archivo crudo de entrada        |
| `ANY_ETL_PROCESS`    | Workflow    | Limpieza y transformación       |
| `CSV_BUSINESS_CLEAN` | Data Object | Archivo limpio listo para carga |

**Responsabilidad:** preparar datos externos para su incorporación al sistema.
Es una capa opcional y desacoplada del flujo conversacional. ⚙️

---

## L3 · Analytical Engine Layer · CORE

Es el núcleo analítico del sistema. Recibe una estructura enriquecida, genera SQL y produce una respuesta preliminar.

| Nodo                  | Tipo          | Descripción                    |
| --------------------- | ------------- | ------------------------------ |
| `GEN_QUERIES`         | API / Service | Engine principal de generación |
| `RAW_QUERIES`         | Data Object   | SQL sin validar                |
| `PRELIMINAR_RESPONSE` | Data Object   | Respuesta narrativa censurada  |

**Responsabilidad:**

* interpretar la intención analítica,
* generar consultas,
* construir una narrativa preliminar,
* trabajar sobre metadata y contexto, no sobre datos crudos.

---

## L4 · Governance & Persistence Layer

Es la capa crítica. Controla el acceso a datos reales, la seguridad, la validación y la composición final de la respuesta.

---

### 4.1 Semantic Governance

Convierte lenguaje natural en una estructura analítica consistente.

| Nodo               | Tipo        | Descripción                                |
| ------------------ | ----------- | ------------------------------------------ |
| `METADATES_REVIEW` | Workflow    | Extracción batch de metadata desde la base |
| `LAYER_METADATES`  | Data Object | Metadata descriptiva censurada             |
| `DET_QUERIES`      | Service     | Convierte NL en estructura analítica       |
| `ENRICHED_Q`       | Data Object | Pregunta enriquecida para el engine        |

**Inputs principales:**

* `USER_MESSAGE`
* `CONTEXT_BUSINESS`
* `USER_HISTORY` como **storage de contexto**, no como input lógico

**Output principal:**

* `ENRICHED_QUESTION_BUSINESS`

---

### 4.2 Query Governance

Valida y controla toda query generada por el engine antes de tocar la base real.

| Nodo                  | Tipo               | Descripción                         |
| --------------------- | ------------------ | ----------------------------------- |
| `FILTER_QUERIES`      | Service            | Valida y sanitiza SQL               |
| `CLEAN_QUERIES`       | Data Object        | Query validada                      |
| `RESULT_QUERIES`      | Data Object        | Resultado real de la DB             |
| `DESCRIPTION_QUERIES` | Data Object        | Descripción censurada del resultado |
| `FILTER_SECURITY`     | Security Component | Filtrado transversal de seguridad   |

**Validaciones principales:**

* solo `SELECT`,
* allowlist de tablas,
* límite de filas,
* sanitización SQL,
* control de joins y comportamiento explosivo. 🔐

---

### 4.3 Response Governance

Compone la respuesta final a partir de la narrativa preliminar y los resultados filtrados.

| Nodo        | Tipo        | Descripción                       |
| ----------- | ----------- | --------------------------------- |
| `JOIN_DATA` | Service     | Composición final de la respuesta |
| `SYS_MSG`   | Data Object | Respuesta final para la UI        |

---

### 4.4 Persistence Resources

Los recursos de persistencia sostienen todo el sistema.

| Recurso               | Tipo    | Contenido                            |
| --------------------- | ------- | ------------------------------------ |
| `Data Business DB`    | Storage | Datos reales del negocio             |
| `Context Business DB` | Storage | Metadata + configuración de contexto |
| `User History DB`     | Storage | `USER_MESSAGE` + historial censurado |

---

## 5. Flujo funcional

| Paso | Qué ocurre                                                                   |
| ---- | ---------------------------------------------------------------------------- |
| 1    | El usuario escribe una pregunta en la UI                                     |
| 2    | `USER_MESSAGE` entra en la capa de interacción                               |
| 3    | `USER_MESSAGE` se almacena en `USER_HISTORY` junto con el contexto censurado |
| 4    | Semantic Governance arma `ENRICHED_QUESTION_BUSINESS`                        |
| 5    | El engine genera `RAW_QUERIES` y `PRELIMINAR_RESPONSE`                       |
| 6    | Query Governance valida la query                                             |
| 7    | La base ejecuta la consulta y devuelve `RESULT_QUERIES`                      |
| 8    | Security filtra lo sensible                                                  |
| 9    | Response Governance compone `SYSTEM_MESSAGE`                                 |
| 10   | La UI lo muestra al usuario                                                  |

---

## 6. Evolución por etapa

### PoC

Objetivo: validar el flujo completo de lenguaje natural → SQL → respuesta narrativa con el menor nivel de infraestructura posible.

| Componente    | Tecnología                        |
| ------------- | --------------------------------- |
| Orquestación  | n8n                               |
| Engine LLM    | Gemini API                        |
| Base de datos | PostgreSQL o SQLite               |
| Metadata      | Extracción manual o script simple |
| Ingesta       | CSV cargado manualmente           |

### MVP

Objetivo: pasar de validación a una solución usable, manteniendo una arquitectura simple pero más gobernada.

| Componente            | Tecnología                           |
| --------------------- | ------------------------------------ |
| Orquestación auxiliar | n8n                                  |
| Engine LLM            | FastAPI + Gemma con function calling |
| Base de datos         | PostgreSQL                           |
| Governance            | Python scripts o nodos controlados   |
| Metadata              | Script batch periódico               |
| Historial             | Tabla simple en PostgreSQL           |

---

## 7. Diagramas Mermaid

### Scale Up

```mermaid
flowchart TB

    %% ─────────────────────────────────────────
    %% ESTILOS NODOS
    %% ─────────────────────────────────────────
    classDef interaction fill:#E3F2FD,stroke:#1E88E5,color:#0D47A1;
    classDef ingestion fill:#E8F5E9,stroke:#43A047,color:#1B5E20;
    classDef engine fill:#FFF3E0,stroke:#FB8C00,color:#E65100;
    classDef governance fill:#F3E5F5,stroke:#8E24AA,color:#4A148C;
    classDef persistence fill:#ECEFF1,stroke:#546E7A,color:#263238;

    %% ─────────────────────────────────────────
    %% L1 · INTERACTION LAYER
    %% ─────────────────────────────────────────
    subgraph L1["L1 · Interaction Layer"]
        direction TB

        USER(("USER"))
        UI_DISPLAY["UI Display\n[display puro]"]
        USER_MSG[/"USER_MESSAGE\n(CHAT ONLINE)"/]
        CONTEXT_CONFIG[/"CONTEXT_BUSINESS_CONFIG_LLM\n[censurado · descriptivo]"/]
    end

    %% ─────────────────────────────────────────
    %% L2 · INGESTION LAYER
    %% ─────────────────────────────────────────
    subgraph L2["L2 · Optional Ingestion Layer"]
        direction TB

        SCHEDULER(("SCHEDULER"))
        CSV_RAW[/"CSV_BUSINESS_RAW"/]
        ETL(["ANY_ETL_PROCESS"])
        CSV_CLEAN[/"CSV_BUSINESS_CLEAN"/]
    end

    %% ─────────────────────────────────────────
    %% L3 · ENGINE LAYER
    %% ─────────────────────────────────────────
    subgraph L3["L3 · Analytical Engine Layer · CORE"]
        direction TB

        GEN_QUERIES(["Generator Queries & Analytical Response\n[API REST]"])
        RAW_QUERIES[/"RAW_QUERIES"/]
        PRELIMINAR_RESPONSE[/"PRELIMINAR_RESPONSE\n[censurada · sin datos reales]"/]
    end

    %% ─────────────────────────────────────────
    %% L4 · GOVERNANCE & PERSISTENCE
    %% ─────────────────────────────────────────
    subgraph L4["L4 · Governance & Persistence Layer"]
        direction TB

        subgraph SG["Semantic Governance"]
            direction TB

            METADATES_REVIEW(["Metadates_Review\n[SQL o Python · batch]"])
            LAYER_METADATES["Layer Metadates\n[censurado · descriptivo]"]
            DET_QUERIES(["Deterministic Queries\n[NL → estructura analítica]"])
            ENRICHED_Q[/"ENRICHED_QUESTION_BUSINESS"/]
        end

        subgraph QG["Query Governance"]
            direction TB

            FILTER_QUERIES(["Filter Queries\n[SQL mode · Descriptive mode]"])
            CLEAN_QUERIES[/"CLEAN_QUERIES"/]
            RESULT_QUERIES[/"RESULT_QUERIES"/]
            DESCRIPTION_QUERIES[/"DESCRIPTION_QUERIES\n[censurado · descriptivo]"/]
            FILTER_SECURITY(["Filter Security\n[transversal]"])
        end

        subgraph RG["Response Governance"]
            direction TB

            JOIN_DATA(["Join Data\n[response composition]"])
            SYS_MSG[/"SYSTEM_MESSAGE\n(CHAT ONLINE)"/]
        end

        subgraph PR["Persistence Resources"]
            direction TB

            DATA_BUSINESS_DB[("Data Business DB")]
            CONTEXT_BUSINESS_DB[("Context Business DB\n[metadata + user config]")]
            USER_HISTORY_DB[("User History DB\n[solo contexto censurado]")]
        end
    end

    %% ─────────────────────────────────────────
    %% FLUJO
    %% ─────────────────────────────────────────
    USER <--> UI_DISPLAY
    UI_DISPLAY --> USER_MSG
    UI_DISPLAY --> CONTEXT_CONFIG

    USER_MSG --> USER_HISTORY_DB
    CONTEXT_CONFIG --> CONTEXT_BUSINESS_DB

    CONTEXT_BUSINESS_DB --> DET_QUERIES
    USER_HISTORY_DB -- ventana contexto --> DET_QUERIES

    DET_QUERIES --> ENRICHED_Q
    ENRICHED_Q --> GEN_QUERIES

    GEN_QUERIES --> RAW_QUERIES
    GEN_QUERIES --> PRELIMINAR_RESPONSE

    PRELIMINAR_RESPONSE --> USER_HISTORY_DB
    PRELIMINAR_RESPONSE --> JOIN_DATA

    RAW_QUERIES --> FILTER_QUERIES
    FILTER_QUERIES --> CLEAN_QUERIES
    CLEAN_QUERIES --> DATA_BUSINESS_DB

    DATA_BUSINESS_DB --> RESULT_QUERIES
    RESULT_QUERIES --> FILTER_QUERIES
    FILTER_QUERIES --> DESCRIPTION_QUERIES
    DESCRIPTION_QUERIES --> GEN_QUERIES

    RESULT_QUERIES --> FILTER_SECURITY
    FILTER_SECURITY --> JOIN_DATA

    DATA_BUSINESS_DB --> METADATES_REVIEW
    METADATES_REVIEW --> LAYER_METADATES
    LAYER_METADATES --> CONTEXT_BUSINESS_DB

    JOIN_DATA --> SYS_MSG
    SYS_MSG --> UI_DISPLAY

    SCHEDULER -. opcional .-> CSV_RAW
    CSV_RAW --> ETL
    ETL --> CSV_CLEAN
    CSV_CLEAN --> DATA_BUSINESS_DB

    %% ─────────────────────────────────────────
    %% COLORES POR NODO
    %% ─────────────────────────────────────────
    class USER,UI_DISPLAY,USER_MSG,CONTEXT_CONFIG interaction;
    class SCHEDULER,CSV_RAW,ETL,CSV_CLEAN ingestion;
    class GEN_QUERIES,RAW_QUERIES,PRELIMINAR_RESPONSE engine;
    class METADATES_REVIEW,LAYER_METADATES,DET_QUERIES,ENRICHED_Q,FILTER_QUERIES,CLEAN_QUERIES,RESULT_QUERIES,DESCRIPTION_QUERIES,FILTER_SECURITY,JOIN_DATA,SYS_MSG governance;
    class DATA_BUSINESS_DB,CONTEXT_BUSINESS_DB,USER_HISTORY_DB persistence;

    %% ─────────────────────────────────────────
    %% COLORES DE LAYERS (SUBGRAPHS)
    %% ─────────────────────────────────────────
    style L1 fill:#E3F2FD,stroke:#1E88E5,stroke-width:2px
    style L2 fill:#E8F5E9,stroke:#43A047,stroke-width:2px
    style L3 fill:#FFF3E0,stroke:#FB8C00,stroke-width:2px
    style L4 fill:#F3E5F5,stroke:#8E24AA,stroke-width:2px
    style SG fill:#F8EAF6,stroke:#8E24AA,stroke-width:1px
    style QG fill:#F3E5F5,stroke:#6A1B9A,stroke-width:1px
    style RG fill:#EDE7F6,stroke:#5E35B1,stroke-width:1px
    style PR fill:#ECEFF1,stroke:#546E7A,stroke-width:1px
```

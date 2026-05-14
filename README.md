# Guía de Configuración: Document Understanding Lab

**Proyecto:** Document Understanding Lab — Análisis de Documentos con IA Generativa  
**Plataforma:** Oracle APEX 24.2 + OCI Generative AI  
**Aplicación APEX:** ID `102` · Alias `DOCUMENT-UNDERSTANDING-LAB`  
**Usuario DB / Schema:** `DEMODU`  
**Workspace APEX:** `WK_DEMODU`  
**Región OCI (Embedding):** `us-ashburn-1` · Modelo: `cohere.embed-v4.0` · Dimensión: `1536 FLOAT32`  
**Región OCI (Chat/LLM):** `us-chicago-1` · Modelo: `openai.gpt-oss-120b`  
**Credencial OCI:** `OCI_GENAI_CRED`  

---

## Tabla de Contenidos

1. [Prerrequisitos](#1-prerrequisitos)
2. [Paso 1 – Creación del Usuario DEMODU](#2-paso-1--creación-del-usuario-demodu)
3. [Paso 2 – Grants y Scripts de Configuración](#3-paso-2--grants-y-scripts-de-configuración)
4. [Paso 3 – Creación del Workspace en APEX](#4-paso-3--creación-del-workspace-en-apex)
5. [Paso 4 – Configuración de Credenciales OCI](#5-paso-4--configuración-de-credenciales-oci)
6. [Paso 5 – Objetos de Base de Datos](#6-paso-5--objetos-de-base-de-datos)
7. [Paso 6 – Paquete RAG_ENGINE](#7-paso-6--paquete-rag_engine)
8. [Paso 7 – Importar la Aplicación APEX](#8-paso-7--importar-la-aplicación-apex)
9. [Paso 8 – Configuración Post-Importación en APEX](#9-paso-8--configuración-post-importación-en-apex)
10. [Referencia de Páginas de la Aplicación](#10-referencia-de-páginas-de-la-aplicación)
11. [Verificación de la Configuración](#11-verificación-de-la-configuración)
12. [Troubleshooting](#12-troubleshooting)

---

## 1. Prerrequisitos

| Requisito | Detalle |
|-----------|---------|
| Oracle ADB / Base de datos | Con soporte para `DBMS_CLOUD`, `DBMS_VECTOR` y `DBMS_VECTOR_CHAIN` |
| Oracle APEX | Versión **24.2** (la app fue exportada en 24.2.15) |
| OCI Generative AI — Embedding | `cohere.embed-v4.0` activo en `us-ashburn-1` |
| OCI Generative AI — Chat | `openai.gpt-oss-120b` activo en `us-chicago-1` |
| Acceso de DBA | Usuario `ADMIN` para ejecutar grants |
| Clave API OCI | Par RSA generado con fingerprint registrado en la consola OCI |
| OCIDs | `user_ocid`, `tenancy_ocid` y `compartment_ocid` disponibles |

---

## 2. Paso 1 – Creación del Usuario DEMODU

Conectarse como `ADMIN` y ejecutar:

```sql
CREATE USER DEMODU IDENTIFIED BY "<password_seguro>";

ALTER USER DEMODU QUOTA UNLIMITED ON DATA;

GRANT CONNECT, RESOURCE TO DEMODU;
GRANT CREATE SESSION     TO DEMODU;
GRANT CREATE TABLE       TO DEMODU;
GRANT CREATE VIEW        TO DEMODU;
GRANT CREATE PROCEDURE   TO DEMODU;
GRANT CREATE PACKAGE     TO DEMODU;
GRANT CREATE SEQUENCE    TO DEMODU;
GRANT CREATE TYPE        TO DEMODU;
```

> Sustituir `<password_seguro>` por una contraseña que cumpla la política de seguridad de tu organización.

---

## 3. Paso 2 – Grants y Scripts de Configuración

Se ejecutan en dos fases: primero como **ADMIN**, luego como **DEMODU**.

### 3.1 Ejecutar como Usuario ADMIN

#### Grants sobre paquetes DBMS_CLOUD, DBMS_VECTOR y GenAI

```sql
GRANT EXECUTE ON DBMS_CLOUD          TO DEMODU;
GRANT EXECUTE ON DBMS_CLOUD_AI       TO DEMODU;
GRANT EXECUTE ON DBMS_VECTOR         TO DEMODU;
GRANT EXECUTE ON DBMS_VECTOR_CHAIN   TO DEMODU;
```

#### Grants sobre tipos OCI Generative AI Inference

Requeridos por `RAG_ENGINE` para construir las solicitudes de embedding:

```sql
GRANT EXECUTE ON dbms_cloud_oci_generative_ai_inference_embed_text_details_t         TO DEMODU;
GRANT EXECUTE ON dbms_cloud_oci_generative_ai_inference_varchar2_tbl                 TO DEMODU;
GRANT EXECUTE ON dbms_cloud_oci_generative_ai_inference_on_demand_serving_mode_t     TO DEMODU;
GRANT EXECUTE ON dbms_cloud_oci_gai_generative_ai_inference_embed_text_response_t    TO DEMODU;
GRANT EXECUTE ON dbms_cloud_oci_gai_generative_ai_inference                          TO DEMODU;
GRANT EXECUTE ON DBMS_CLOUD_OCI_GENERATIVE_AI_INFERENCE_GENERATE_TEXT_DETAILS_T      TO DEMODU;
GRANT EXECUTE ON DBMS_CLOUD_OCI_GENERATIVE_AI_INFERENCE_SERVING_MODE_T               TO DEMODU;
GRANT EXECUTE ON DBMS_CLOUD_OCI_GAI_GENERATIVE_AI_INFERENCE_GENERATE_TEXT_RESPONSE_T TO DEMODU;
```

#### Habilitar autenticación principal OCI

```sql
BEGIN
  DBMS_CLOUD_ADMIN.ENABLE_PRINCIPAL_AUTH(provider => 'OCI');
END;
/
```

#### Configurar ACL de red para OCI

```sql
BEGIN
  SYS.DBMS_NETWORK_ACL_ADMIN.APPEND_HOST_ACE (
    host => '*.oci.oraclecloud.com',
    ace  => xs$ace_type(
              privilege_list => xs$name_list('connect', 'resolve'),
              principal_name => 'DEMODU',
              principal_type => xs_acl.ptype_db
            )
  );
END;
/
```

---

### 3.2 Ejecutar como Usuario DEMODU

#### Crear la credencial OCI para Embedding y Chat GenAI

La credencial `OCI_GENAI_CRED` es utilizada tanto por el paquete `RAG_ENGINE` (embeddings vía PL/SQL) como por la página 3 de la aplicación (chat con `APEX_WEB_SERVICE`). Es la **única credencial necesaria**.

```sql
BEGIN
  DBMS_CLOUD.CREATE_CREDENTIAL(
    credential_name => 'OCI_GENAI_CRED',
    user_ocid       => 'ocid1.user.oc1..<valor_ofuscado>',
    tenancy_ocid    => 'ocid1.tenancy.oc1..<valor_ofuscado>',
    private_key     => '<contenido_de_tu_clave_privada_pem>',
    fingerprint     => 'xx:xx:xx:xx:xx:xx:xx:xx:xx:xx:xx:xx:xx:xx:xx:xx'
  );
END;
/
```

> El valor de `private_key` debe ser el contenido del `.pem` sin las líneas `-----BEGIN/END RSA PRIVATE KEY-----`, como una cadena continua sin saltos de línea.

#### Verificar la credencial

```sql
SELECT credential_name, username, enabled
FROM   user_credentials
WHERE  credential_name = 'OCI_GENAI_CRED';
```

---

## 4. Paso 3 – Creación del Workspace en APEX

### 4.1 Crear el Workspace desde APEX Internal

1. Ingresar a APEX como `INTERNAL` / `ADMIN`.
2. Ir a **Manage Workspaces → Create Workspace**.
3. Completar:

| Campo | Valor |
|-------|-------|
| Workspace Name | `WK_DEMODU` |
| Schema | `DEMODU` |
| Space Quota (MB) | 500 o según política |

4. Clic en **Create Workspace**.

### 4.2 Crear usuario administrador del Workspace

1. **Manage Workspaces → Existing Workspaces** → seleccionar `WK_DEMODU`.
2. **Manage Developers and Users** → crear usuario:

| Campo | Valor |
|-------|-------|
| Username | `DEMODU_ADMIN` (o el que prefieras) |
| Email | correo del administrador |
| Role | Administrator |

---

## 5. Paso 4 – Configuración de Credenciales OCI

### 5.1 Credencial PL/SQL (requerida por RAG_ENGINE)

La credencial `OCI_GENAI_CRED` creada en el Paso 2 es suficiente para el funcionamiento del motor RAG. No se requiere nada adicional en la capa PL/SQL.

### 5.2 Web Credential en APEX (requerida por la Página 3)

La Página 3 (_RAG Search - GenAI Response_) llama al API de OCI GenAI directamente usando `APEX_WEB_SERVICE.MAKE_REST_REQUEST` con `p_credential_static_id => 'OCI_GENAI_CRED'`. Esta credencial **debe existir también como Web Credential** en el workspace:

1. Ingresar al Workspace `WK_DEMODU`.
2. Ir a **App Builder → Workspace Utilities → Web Credentials**.
3. Crear la credencial con estos datos exactos:

| Campo | Valor |
|-------|-------|
| Name | `OCI_GENAI_CRED` |
| Static ID | `OCI_GENAI_CRED` |
| Authentication Type | Oracle Cloud Infrastructure (OCI) |
| OCI User ID | `ocid1.user.oc1..<valor_ofuscado>` |
| OCI Private Key | Contenido del archivo `.pem` |
| OCI Tenancy ID | `ocid1.tenancy.oc1..<valor_ofuscado>` |
| OCI Public Key Fingerprint | `xx:xx:xx:xx:xx:xx:xx:xx:xx:xx:xx:xx:xx:xx:xx:xx` |

> El nombre y Static ID deben ser exactamente `OCI_GENAI_CRED` ya que están hardcodeados en el proceso PL/SQL de la Página 3.

### 5.3 Remote Server para OCI GenAI

La aplicación usa un Remote Server para las llamadas REST a OCI. Se importa automáticamente con la aplicación, pero conviene verificarlo después de la importación:

1. **App Builder → Workspace Utilities → Remote Servers**.
2. Verificar que existe:

| Campo | Valor |
|-------|-------|
| Name | `inference-generativeai-us-chicago-1-oci-oraclecloud-com-20231130-actions` |
| Base URL | `https://inference.generativeai.us-chicago-1.oci.oraclecloud.com/20231130/actions/` |

---

## 6. Paso 5 – Objetos de Base de Datos

Ejecutar todos los DDLs como usuario **DEMODU**.

### 6.1 Tabla RAG_DOCUMENTS

Almacena los documentos originales subidos (PDF). El campo `STATUS` controla el ciclo de vida del documento.

```sql
CREATE TABLE DEMODU.RAG_DOCUMENTS (
  DOC_ID      NUMBER GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  DOC_NAME    VARCHAR2(500 BYTE)  NOT NULL,
  DOC_MIME    VARCHAR2(100 BYTE)  DEFAULT 'application/pdf',
  DOC_BLOB    BLOB,
  UPLOADED_BY VARCHAR2(100 BYTE)  DEFAULT USER,
  UPLOADED_AT TIMESTAMP(6)        DEFAULT SYSTIMESTAMP,
  STATUS      VARCHAR2(50 BYTE)   DEFAULT 'PENDING'  -- PENDING | CHUNKED | ERROR
);
```

### 6.2 Tabla RAG_CHUNKS

Almacena los fragmentos de texto y sus vectores de embedding (1536 dimensiones).

```sql
CREATE TABLE DEMODU.RAG_CHUNKS (
  CHUNK_ID   NUMBER GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  DOC_ID     NUMBER        NOT NULL,
  CHUNK_SEQ  NUMBER        NOT NULL,
  CHUNK_TEXT CLOB          NOT NULL,
  CREATED_AT TIMESTAMP(6)  DEFAULT SYSTIMESTAMP,
  EMBEDDING  VECTOR(1536, FLOAT32),
  CONSTRAINT fk_chunks_doc FOREIGN KEY (DOC_ID)
    REFERENCES DEMODU.RAG_DOCUMENTS (DOC_ID) ON DELETE CASCADE
);

-- Índice vectorial ANN para búsqueda semántica por COSINE
CREATE VECTOR INDEX RAG_CHUNKS_VIDX
  ON DEMODU.RAG_CHUNKS (EMBEDDING);
```

### 6.3 Tabla RAG_QUERY_LOG

Registro de auditoría de consultas realizadas.

```sql
CREATE TABLE DEMODU.RAG_QUERY_LOG (
  QUERY_ID     NUMBER GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  QUERY_TEXT   VARCHAR2(4000 BYTE),
  QUERY_VECTOR VECTOR(1024, FLOAT32),
  TOP_K        NUMBER        DEFAULT 5,
  QUERIED_AT   TIMESTAMP(6)  DEFAULT SYSTIMESTAMP,
  QUERIED_BY   VARCHAR2(100 BYTE) DEFAULT USER
);
```

### 6.4 Tabla RAG_RESULTS

Almacena resultados de búsqueda por sesión de APEX. La columna `SIMILARITY` es virtual (`1 - DISTANCE`). La Página 5 y la Página 3 escriben aquí y leen desde aquí filtrado por `SESSION_ID`.

```sql
CREATE TABLE DEMODU.RAG_RESULTS (
  RESULT_ID   NUMBER GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  SESSION_ID  VARCHAR2(100 BYTE) DEFAULT SYS_CONTEXT('APEX$SESSION','APP_SESSION'),
  QUERY_TEXT  VARCHAR2(4000 BYTE),
  DOC_NAME    VARCHAR2(500 BYTE),
  CHUNK_SEQ   NUMBER,
  CHUNK_TEXT  CLOB,
  DISTANCE    NUMBER(10,6),
  SIMILARITY  NUMBER(10,6) GENERATED ALWAYS AS (1 - DISTANCE) VIRTUAL,
  SEARCHED_AT TIMESTAMP(6)  DEFAULT SYSTIMESTAMP
);
```

---

## 7. Paso 6 – Paquete RAG_ENGINE

El paquete `DEMODU.RAG_ENGINE` es el motor RAG. Expone:

| Elemento | Tipo | Descripción |
|----------|------|-------------|
| `PROCESS_DOCUMENT(p_doc_id)` | Procedimiento | BLOB → texto → chunks → embeddings → `RAG_CHUNKS` |
| `SEARCH(p_query, p_top_k, p_doc_ids)` | Función | Vectoriza consulta → búsqueda ANN COSINE → `SYS_REFCURSOR` |

### 7.1 Constantes internas

| Constante | Valor |
|-----------|-------|
| `c_embed_model` | `cohere.embed-v4.0` |
| `c_region` | `us-ashburn-1` |
| `c_credential` | `OCI_GENAI_CRED` |
| `c_compartment_id` | `ocid1.compartment.oc1..<valor_ofuscado>` |

### 7.2 Especificación

```sql
CREATE OR REPLACE PACKAGE DEMODU.RAG_ENGINE AS

  PROCEDURE PROCESS_DOCUMENT (p_doc_id IN NUMBER);

  FUNCTION SEARCH (
    p_query   IN VARCHAR2,
    p_top_k   IN NUMBER   DEFAULT 5,
    p_doc_ids IN VARCHAR2 DEFAULT NULL
  ) RETURN SYS_REFCURSOR;

END RAG_ENGINE;
/
```

### 7.3 Cuerpo

```sql
CREATE OR REPLACE PACKAGE BODY DEMODU.RAG_ENGINE AS

  c_compartment_id  CONSTANT VARCHAR2(200) := 'ocid1.compartment.oc1..<valor_ofuscado>';
  c_embed_model     CONSTANT VARCHAR2(200) := 'cohere.embed-v4.0';
  c_region          CONSTANT VARCHAR2(50)  := 'us-ashburn-1';
  c_credential      CONSTANT VARCHAR2(100) := 'OCI_GENAI_CRED';

  -- Genera embedding VECTOR(1536, FLOAT32) usando cohere.embed-v4.0
  FUNCTION GET_EMBEDDING (p_text IN VARCHAR2) RETURN VECTOR IS
    l_details   dbms_cloud_oci_generative_ai_inference_embed_text_details_t;
    l_resp      dbms_cloud_oci_gai_generative_ai_inference_embed_text_response_t;
    l_embedding VECTOR(1536, FLOAT32);
    l_json      CLOB;
  BEGIN
    l_details := dbms_cloud_oci_generative_ai_inference_embed_text_details_t(
      inputs         => dbms_cloud_oci_generative_ai_inference_varchar2_tbl(p_text),
      serving_mode   => dbms_cloud_oci_generative_ai_inference_on_demand_serving_mode_t(
                          'ON_DEMAND', c_embed_model),
      compartment_id => c_compartment_id,
      is_echo        => 0,
      truncate       => 'NONE',
      input_type     => 'SEARCH_DOCUMENT'
    );

    l_resp := dbms_cloud_oci_gai_generative_ai_inference.embed_text(
      embed_text_details => l_details,
      region             => c_region,
      credential_name    => c_credential
    );

    l_json := l_resp.response_body.embeddings.to_string;
    l_json := SUBSTR(l_json, 2, LENGTH(l_json) - 2);

    SELECT TO_VECTOR(l_json, 1536, FLOAT32) INTO l_embedding FROM DUAL;
    RETURN l_embedding;
  EXCEPTION
    WHEN OTHERS THEN
      RAISE_APPLICATION_ERROR(-20001, 'Error obteniendo embedding: ' || SQLERRM);
  END GET_EMBEDDING;

  -- Extrae texto del BLOB (PDF) usando DBMS_VECTOR_CHAIN
  FUNCTION EXTRACT_TEXT (p_blob IN BLOB) RETURN CLOB IS
    l_text CLOB;
  BEGIN
    l_text := DBMS_VECTOR_CHAIN.UTL_TO_TEXT(p_blob);
    RETURN l_text;
  EXCEPTION
    WHEN OTHERS THEN
      RAISE_APPLICATION_ERROR(-20002, 'Error extrayendo texto: ' || SQLERRM);
  END EXTRACT_TEXT;

  -- Flujo completo: BLOB → texto → chunks (200 palabras/overlap 20) → embeddings → RAG_CHUNKS
  PROCEDURE PROCESS_DOCUMENT (p_doc_id IN NUMBER) IS
    l_blob      BLOB;
    l_full_text CLOB;
    l_embedding VECTOR(1536, FLOAT32);
    l_seq       NUMBER := 0;
  BEGIN
    SELECT doc_blob INTO l_blob FROM rag_documents WHERE doc_id = p_doc_id;
    l_full_text := EXTRACT_TEXT(l_blob);
    DELETE FROM rag_chunks WHERE doc_id = p_doc_id;

    FOR r IN (
      SELECT JSON_VALUE(column_value, '$.chunk_data') AS chunk_text
        FROM TABLE(
               DBMS_VECTOR_CHAIN.UTL_TO_CHUNKS(
                 l_full_text,
                 JSON('{"by":"words","max":200,"overlap":20,"split":"sentence"}')
               )
             )
    ) LOOP
      CONTINUE WHEN r.chunk_text IS NULL OR LENGTH(TRIM(r.chunk_text)) = 0;
      l_seq       := l_seq + 1;
      l_embedding := GET_EMBEDDING(r.chunk_text);
      INSERT INTO rag_chunks (doc_id, chunk_seq, chunk_text, embedding)
      VALUES (p_doc_id, l_seq, r.chunk_text, l_embedding);
      COMMIT;
    END LOOP;

    UPDATE rag_documents SET status = 'CHUNKED' WHERE doc_id = p_doc_id;
    COMMIT;
  EXCEPTION
    WHEN OTHERS THEN
      UPDATE rag_documents SET status = 'ERROR' WHERE doc_id = p_doc_id;
      COMMIT;
      RAISE;
  END PROCESS_DOCUMENT;

  -- Vectoriza la consulta y retorna top-K chunks por similitud COSINE
  -- p_doc_ids: IDs separados por ':' (ej. '1:3:7'). NULL = todos los documentos.
  FUNCTION SEARCH (
    p_query   IN VARCHAR2,
    p_top_k   IN NUMBER   DEFAULT 5,
    p_doc_ids IN VARCHAR2 DEFAULT NULL
  ) RETURN SYS_REFCURSOR IS
    l_query_vec VECTOR(1536, FLOAT32);
    l_cursor    SYS_REFCURSOR;
  BEGIN
    l_query_vec := GET_EMBEDDING(p_query);

    IF p_doc_ids IS NULL THEN
      OPEN l_cursor FOR
        SELECT c.chunk_id, d.doc_name, c.chunk_seq, c.chunk_text,
               VECTOR_DISTANCE(c.embedding, l_query_vec, COSINE) AS distance
          FROM rag_chunks c JOIN rag_documents d ON d.doc_id = c.doc_id
         ORDER BY distance ASC
         FETCH FIRST p_top_k ROWS ONLY;
    ELSE
      OPEN l_cursor FOR
        SELECT chunk_id, doc_name, chunk_seq, chunk_text, distance
          FROM (
            SELECT c.chunk_id, d.doc_name, c.chunk_seq, c.chunk_text,
                   VECTOR_DISTANCE(c.embedding, l_query_vec, COSINE) AS distance,
                   ROW_NUMBER() OVER (
                     PARTITION BY d.doc_id
                     ORDER BY VECTOR_DISTANCE(c.embedding, l_query_vec, COSINE) ASC
                   ) AS rn
              FROM rag_chunks c JOIN rag_documents d ON d.doc_id = c.doc_id
             WHERE d.doc_id IN (
                     SELECT TO_NUMBER(TRIM(column_value))
                       FROM TABLE(APEX_STRING.SPLIT(p_doc_ids, ':'))
                      WHERE TRIM(column_value) IS NOT NULL
                        AND REGEXP_LIKE(TRIM(column_value), '^\d+$')
                   )
          )
         WHERE rn <= p_top_k
         ORDER BY distance ASC;
    END IF;

    RETURN l_cursor;
  END SEARCH;

END RAG_ENGINE;
/
```

### 7.4 Flujo de datos

```
[Usuario sube PDF en APEX Página 4]
         │
         ▼
  RAG_DOCUMENTS (BLOB + metadata, STATUS = PENDING)
         │
         ▼
  RAG_ENGINE.PROCESS_DOCUMENT(doc_id)
    ├─ EXTRACT_TEXT()   → DBMS_VECTOR_CHAIN.UTL_TO_TEXT
    ├─ UTL_TO_CHUNKS()  → 200 palabras, overlap 20, split por oración
    └─ GET_EMBEDDING()  → OCI GenAI · cohere.embed-v4.0 · us-ashburn-1
                          Credencial: OCI_GENAI_CRED
         │
         ▼
  RAG_CHUNKS (chunk_text + VECTOR(1536,FLOAT32), índice RAG_CHUNKS_VIDX)
  STATUS del documento → CHUNKED
         │
  [Usuario busca en APEX Página 5 o Página 3]
         │
         ▼
  RAG_ENGINE.SEARCH(query, top_k, doc_ids)
    └─ GET_EMBEDDING(query) → VECTOR_DISTANCE(COSINE) → top-K chunks
         │
         ▼
  RAG_RESULTS (por sesión APEX)
         │
  [Solo Página 3]
         │
         ▼
  APEX_WEB_SERVICE → OCI GenAI Chat · openai.gpt-oss-120b · us-chicago-1
  Respuesta renderizada con marked.js en el div #ai-answer-rendered
```

---

## 8. Paso 7 – Importar la Aplicación APEX

### 8.1 Importar el archivo de exportación

1. Ingresar al Workspace `WK_DEMODU` en APEX.
2. Ir a **App Builder → Import**.
3. Seleccionar el archivo `APEXAPP.sql` (exportado de APEX 24.2.15).
4. Clic en **Next**.
5. En la pantalla de instalación:

| Campo | Valor |
|-------|-------|
| Parsing Schema | `DEMODU` |
| Build Status | Run and Build Application |
| Install Supporting Objects | Yes |

6. Clic en **Install Application**.

> La aplicación se importará con el ID `102` por defecto. Si ya existe una aplicación con ese ID en el workspace, APEX pedirá confirmación para sobreescribirla o asignará un nuevo ID.

### 8.2 Verificar que la importación fue exitosa

1. Ir a **App Builder** → debe aparecer **Document Understanding Lab** (ID 102).
2. Hacer clic en **Run Application** para verificar que abre correctamente.
3. El login usa el sistema de autenticación de APEX (usuarios del workspace).

---

## 9. Paso 8 – Configuración Post-Importación en APEX

### 9.1 Verificar y actualizar el Remote Server

El Remote Server para OCI GenAI se importa con la aplicación pero conviene validarlo:

1. **App Builder → Workspace Utilities → Remote Servers**.
2. Abrir `inference-generativeai-us-chicago-1-oci-oraclecloud-com-20231130-actions`.
3. Verificar que **Base URL** es:
   ```
   https://inference.generativeai.us-chicago-1.oci.oraclecloud.com/20231130/actions/
   ```
4. Si el campo está vacío (puede ocurrir en algunas importaciones), ingresarlo manualmente.

### 9.2 Verificar la Web Credential OCI_GENAI_CRED

1. **App Builder → Workspace Utilities → Web Credentials**.
2. Confirmar que existe `OCI_GENAI_CRED` con Authentication Type `OCI`.
3. Si no existe, crearla siguiendo el Paso 4 de esta guía.
4. Si existe pero fue creada durante la importación sin los datos reales, editarla y completar los valores de OCI User ID, Private Key, Tenancy ID y Fingerprint.

### 9.3 Actualizar el Compartment ID en el paquete RAG_ENGINE

El `compartment_id` está hardcodeado en la constante `c_compartment_id` del paquete. Si el compartment cambió o se instaló en un ambiente diferente, recompilar el paquete con el valor correcto:

```sql
-- Editar la constante en el cuerpo del paquete antes de recompilar:
-- c_compartment_id CONSTANT VARCHAR2(200) := 'ocid1.compartment.oc1..<tu_compartment_id>';
ALTER PACKAGE DEMODU.RAG_ENGINE COMPILE BODY;
```

### 9.4 Actualizar el Compartment ID en la Página 3

El `compartmentId` también está en el payload JSON de la Dynamic Action `GenAIResponse` de la Página 3. Para actualizarlo:

1. **App Builder → Application 102 → Page 3 (RAG Search - GenAI Response)**.
2. En el panel izquierdo, expandir **Dynamic Actions → ExecuteSearch**.
3. Seleccionar la acción **GenAIResponse** (secuencia 40, tipo PL/SQL).
4. En el bloque PL/SQL, localizar la línea:
   ```plsql
   '{\"compartmentId\":\"ocid1.compartment.oc1..<valor_ofuscado>\", ...'
   ```
5. Reemplazar con tu `compartment_ocid` real.
6. Guardar la página.

---

## 10. Referencia de Páginas de la Aplicación

La aplicación **Document Understanding Lab** (ID 102) tiene 6 páginas:

### Resumen de Páginas

| Página | Nombre | Alias | Descripción |
|--------|--------|-------|-------------|
| 0 | Global Page | — | Componentes globales (header, footer, CSS compartido) |
| 1 | Home | `HOME` | Pantalla de bienvenida con logo de la aplicación |
| 3 | RAG Search - GenAI Response | `RAG-SEARCH-GENAI-RESPONSE` | Búsqueda semántica + respuesta generativa con `openai.gpt-oss-120b` |
| 4 | Upload de PDF | `UPLOAD-DE-PDF` | Carga de documentos PDF y tabla de documentos |
| 5 | RAG Search | `RAG-SEARCH` | Búsqueda semántica sin respuesta generativa |
| 9999 | Login Page | — | Pantalla de autenticación estándar APEX |

---

### Página 1 – Home

Pantalla de bienvenida de la aplicación. Contiene:

- **Región Hero** con el ícono de la aplicación (`#APP_FILES#icons/app-icon-512.png`) y el título `Document Understanding Lab`.
- No tiene items, procesos ni Dynamic Actions propios.
- Sirve como punto de entrada; la navegación lateral lleva a las páginas funcionales.

---

### Página 4 – Upload de PDF

**Propósito:** Permite al usuario cargar archivos PDF y visualizar el estado de los documentos ya cargados.

#### Estructura de la página

```
Page 4 - Upload de PDF
├── Breadcrumb (REGION_POSITION_01)
├── UploadArea [Región — columna izquierda 6/12]
│   ├── Item: P1_PDF_FILE (File Upload — solo PDF, almacenado en APEX_APPLICATION_TEMP_FILES)
│   ├── Item: P1_DOC_NAME (Hidden)
│   └── Button: PB_CARGAR [Submit — icono fa-upload, Hot]
└── RAG_Documents [Classic Report — columna derecha 6/12]
    └── Query: SELECT * FROM RAG_DOCUMENTS
        Columnas: DOC_ID, DOC_NAME, DOC_MIME, UPLOADED_BY, UPLOADED_AT, STATUS
```

#### Items de la página

| Item | Tipo | Descripción |
|------|------|-------------|
| `P1_PDF_FILE` | File Upload (Native) | Acepta solo `application/pdf`. Almacenamiento: `APEX_APPLICATION_TEMP_FILES`. Un archivo a la vez. |
| `P1_DOC_NAME` | Hidden | Auxiliar para el nombre del documento (llenado por proceso) |

#### Botón

| Botón | Acción | Comportamiento |
|-------|--------|----------------|
| `PB_CARGAR` | Submit | Envía el formulario y dispara el proceso de servidor |

#### Proceso de servidor: _Insertar y Procesar PDF_

- **Punto de ejecución:** `ON_SUBMIT_BEFORE_COMPUTATION`
- **Tipo:** PL/SQL nativo

```sql
DECLARE
  l_blob    BLOB;
  l_name    VARCHAR2(500);
  l_doc_id  NUMBER;
BEGIN
  -- Recuperar el archivo desde la colección temporal de APEX
  SELECT blob_content, filename
    INTO l_blob, l_name
    FROM apex_application_temp_files
   WHERE name = :P1_PDF_FILE;

  -- Insertar en RAG_DOCUMENTS
  INSERT INTO rag_documents (doc_name, doc_blob, status)
  VALUES (l_name, l_blob, 'PENDING')
  RETURNING doc_id INTO l_doc_id;

  COMMIT;

  -- Procesar el documento de forma síncrona (extracción + chunking + embedding)
  rag_engine.process_document(l_doc_id);

  apex_application.g_print_success_message :=
    'Documento "' || l_name || '" vectorizado correctamente (' || l_doc_id || ')';
EXCEPTION
  WHEN NO_DATA_FOUND THEN
    raise_application_error(-20010, 'No se encontró el archivo subido.');
END;
```

> **Nota:** El proceso es **síncrono**. Para documentos grandes el tiempo de espera puede ser considerable ya que genera todos los embeddings antes de responder. Considera mover a un APEX Automation o Background Process para producción.

---

### Página 5 – RAG Search

**Propósito:** Búsqueda semántica pura sobre los documentos vectorizados. Muestra los chunks más relevantes sin llamar al LLM.

#### Estructura de la página

```
Page 5 - RAG Search
├── Breadcrumb (REGION_POSITION_01)
└── RAGSearch [Región principal]
    ├── Consulta [Sub-región]
    │   └── Item: P2_QUERY (Textarea — "Escribe aquí tu consulta")
    ├── Parametros [Sub-región Tabs]
    │   ├── Item: P2_TOP_K (Select — Nivel de correlacionamiento, valor 1-5)
    │   └── Item: P2_DOCLIST (Select Multiple — documentos con STATUS='CHUNKED', separador ':')
    ├── Button: Search [DEFINED_BY_DA — icono fa-database-search, Hot]
    ├── Item: P2_SESSION (Hidden — almacena APP_SESSION)
    └── ReporteBusqueda [Classic Report]
        └── SELECT * FROM RAG_RESULTS WHERE session_id = :P2_SESSION
```

#### Items de la página

| Item | Tipo | Descripción |
|------|------|-------------|
| `P2_QUERY` | Textarea | Consulta en lenguaje natural del usuario |
| `P2_TOP_K` | Select One | Nivel de correlacionamiento: `Determinístico=1`, `Muy Alto=2`, `Alto=3`, `Medio=4`, `Bajo=5` |
| `P2_DOCLIST` | Select Many | Lista de documentos disponibles (STATUS='CHUNKED'). Los valores se concatenan con `:` y se pasan a `p_doc_ids` de `RAG_ENGINE.SEARCH` |
| `P2_SESSION` | Hidden | Almacena el valor de `APP_SESSION` para filtrar `RAG_RESULTS` |

#### Dynamic Actions

**DA: `LimpiezadeResultados`** — se ejecuta en `Page Load (ready)`:
```sql
DECLARE
  l_session VARCHAR2(100) := V('APP_SESSION');
BEGIN
  DELETE FROM rag_results WHERE session_id = l_session;
  COMMIT;
  :P2_SESSION := V('APP_SESSION');
END;
```

**DA: `ExecuteSearch`** — se dispara al hacer clic en el botón `Search`:

Acción 1 — PL/SQL (secuencia 20): ejecuta la búsqueda y guarda en `RAG_RESULTS`:
```sql
DECLARE
  l_cursor     SYS_REFCURSOR;
  l_chunk_id   NUMBER;
  l_doc_name   VARCHAR2(500);
  l_chunk_seq  NUMBER;
  l_chunk_text CLOB;
  l_distance   NUMBER;
  l_session    VARCHAR2(100) := V('APP_SESSION');
BEGIN
  DELETE FROM rag_results WHERE session_id = l_session;

  l_cursor := rag_engine.search(
    p_query   => :P2_QUERY,
    p_top_k   => NVL(:P2_TOP_K, 5),
    p_doc_ids => :P2_DOCLIST
  );

  LOOP
    FETCH l_cursor INTO l_chunk_id, l_doc_name, l_chunk_seq, l_chunk_text, l_distance;
    EXIT WHEN l_cursor%NOTFOUND;
    INSERT INTO rag_results (session_id, query_text, doc_name, chunk_seq, chunk_text, distance)
    VALUES (l_session, :P2_QUERY, l_doc_name, l_chunk_seq, l_chunk_text, l_distance);
  END LOOP;

  CLOSE l_cursor;
  COMMIT;
END;
```
_Items enviados al servidor: `P2_TOP_K, P2_QUERY, P2_DOCLIST`_

Acción 2 — Refresh (secuencia 30): refresca la región `ReporteBusqueda`.

---

### Página 3 – RAG Search - GenAI Response

**Propósito:** Búsqueda semántica + generación de respuesta en lenguaje natural usando `openai.gpt-oss-120b` en `us-chicago-1`. La respuesta se renderiza en Markdown usando la librería `marked.js`.

#### Estructura de la página

```
Page 3 - RAG Search - GenAI Response
├── Breadcrumb (REGION_POSITION_01)
├── JS externo: https://cdnjs.cloudflare.com/ajax/libs/marked/9.1.6/marked.min.js
├── JS onLoad: función renderAnswer() — parsea P2_ANSWER con marked.js
└── RAGSearch [Región principal]
    ├── Consulta [Sub-región]
    │   ├── Item: P2_QUERY2 (Textarea — "Escribe aquí tu consulta")
    │   └── Button: Search [DEFINED_BY_DA — Hot]
    ├── Parametros [Sub-región Tabs]
    │   ├── Item: P2_TOP_K2 (Select — igual que Página 5)
    │   └── Item: P2_DOCLIST2 (Select Many — igual que Página 5)
    ├── Item: P2_SESSION2 (Hidden — APP_SESSION)
    ├── Respuesta [Sub-región]
    │   └── GenAIResponse [Sub-región]
    │       └── Imagen [HTML estático]
    │           └── Item: P2_ANSWER (Hidden — contiene el texto Markdown de la respuesta)
    │               <div id="ai-answer-rendered"> ← aquí se renderiza el Markdown
    └── REPORTEBUSQUEDA [Classic Report]
        └── SELECT * FROM RAG_RESULTS WHERE session_id = :P2_SESSION2
```

#### Items de la página

| Item | Tipo | Descripción |
|------|------|-------------|
| `P2_QUERY2` | Textarea | Consulta del usuario |
| `P2_TOP_K2` | Select One | Nivel de correlacionamiento (igual que Página 5) |
| `P2_DOCLIST2` | Select Many | Documentos a buscar (igual que Página 5) |
| `P2_SESSION2` | Hidden | APP_SESSION para filtrar resultados |
| `P2_ANSWER` | Hidden | Almacena la respuesta generada por el LLM en texto Markdown |

#### Dynamic Actions — Secuencia de ejecución al hacer clic en Search

**Paso 1 — `CleanAnswer`** (PL/SQL, secuencia 10):
```sql
BEGIN
  :P2_ANSWER := 'Estoy analizando tu consulta...';
END;
```
_Devuelve `P2_ANSWER` al cliente para mostrar mensaje de carga._

**Paso 2 — `RefreshAnswer`** (JavaScript, secuencia 20):
```javascript
var answer = "Analizando tu consulta...";
window.miSpinner = apex.util.showSpinner();  // Muestra spinner de carga

if (answer && typeof marked !== 'undefined') {
  $('#ai-answer-rendered').html(marked.parse(answer));
}

apex.region('REPORTEBUSQUEDA').refresh();
```

**Paso 3 — Búsqueda RAG** (PL/SQL, secuencia 30):

Idéntico al proceso de la Página 5, pero usando los items `P2_QUERY2`, `P2_TOP_K2` y `P2_DOCLIST2`.

**Paso 4 — `GenAIResponse`** (PL/SQL, secuencia 40):

Toma los chunks de `RAG_RESULTS` de la sesión actual, construye un prompt estructurado y llama al API de OCI GenAI Chat:

```sql
DECLARE
  l_payload   CLOB;
  l_response  CLOB;
  l_prompt    VARCHAR2(32767);
BEGIN
  -- Construir contexto desde RAG_RESULTS de la sesión actual
  FOR r IN (
    SELECT doc_name, chunk_seq, chunk_text, similarity
      FROM rag_results
     WHERE session_id = V('APP_SESSION')
     ORDER BY distance ASC
  ) LOOP
    l_prompt := l_prompt
      || 'Documento: ' || r.doc_name || CHR(10)
      || 'Chunk: '     || r.chunk_seq || CHR(10)
      || 'Texto: '     || SUBSTR(r.chunk_text, 1, 500) || CHR(10)
      || '---' || CHR(10);
  END LOOP;

  -- Prompt del sistema con reglas de formato y consulta del usuario
  l_prompt :=
    'Eres un asistente experto en análisis de documentos. '
    || 'Tu misión es responder de forma clara, profesional y ejecutiva. '
    || 'Utiliza ÚNICAMENTE la información contenida en la evidencia proporcionada. '
    || CHR(10)
    || 'REGLAS DE FORMATO:' || CHR(10)
    || '- Responde siempre en español.' || CHR(10)
    || '- Usa un tono profesional pero cercano.' || CHR(10)
    || '- Estructura tu respuesta con secciones claras cuando sea necesario.' || CHR(10)
    || '- Usa tablas Markdown para datos comparativos o numéricos.' || CHR(10)
    || '- Usa negritas para destacar datos clave.' || CHR(10)
    || '- Al final incluye una sección **Fuentes** con los nombres de documentos consultados.' || CHR(10)
    || '- Sé conciso pero completo.' || CHR(10)
    || CHR(10)
    || 'Consulta del usuario: ' || :P2_QUERY2
    || CHR(10) || CHR(10)
    || 'Evidencia disponible:' || CHR(10)
    || l_prompt;

  -- Escapar caracteres especiales para JSON
  l_prompt := REPLACE(l_prompt, '\',   '\\');
  l_prompt := REPLACE(l_prompt, '"',   '\"');
  l_prompt := REPLACE(l_prompt, CHR(10), '\n');
  l_prompt := REPLACE(l_prompt, CHR(13), '\r');
  l_prompt := REPLACE(l_prompt, CHR(9),  '\t');

  -- Construir payload JSON para OCI GenAI Chat
  l_payload :=
    '{"compartmentId":"ocid1.compartment.oc1..<valor_ofuscado>",'
    || '"servingMode":{"modelId":"openai.gpt-oss-120b","servingType":"ON_DEMAND"},'
    || '"chatRequest":{"apiFormat":"GENERIC",'
    || '"messages":[{"role":"USER","content":[{"type":"TEXT","text":"' || l_prompt || '"}]}],'
    || '"maxTokens":1200,"temperature":0.2,"topP":1,'
    || '"frequencyPenalty":0,"presencePenalty":0,"isStream":false}}';

  -- Llamada al API REST de OCI GenAI Chat
  APEX_WEB_SERVICE.SET_REQUEST_HEADERS(
    p_name_01  => 'Content-Type',
    p_value_01 => 'application/json'
  );

  l_response := APEX_WEB_SERVICE.MAKE_REST_REQUEST(
    p_url                  => 'https://inference.generativeai.us-chicago-1.oci.oraclecloud.com/20231130/actions/chat',
    p_http_method          => 'POST',
    p_body                 => l_payload,
    p_credential_static_id => 'OCI_GENAI_CRED'
  );

  -- Extraer texto de la respuesta JSON
  :P2_ANSWER := JSON_VALUE(
    l_response,
    '$.chatResponse.choices[0].message.content[0].text'
  );
END;
```
_Items enviados: `P2_QUERY2`. Item retornado: `P2_ANSWER`._

**Paso 5 — `FormatedResponse`** (JavaScript, secuencia 50):
```javascript
var answer = apex.item('P2_ANSWER').getValue();
if (answer && typeof marked !== 'undefined') {
  $('#ai-answer-rendered').html(marked.parse(answer));
}

apex.region('REPORTEBUSQUEDA').refresh();

// Ocultar spinner iniciado en el Paso 2
if (window.miSpinner) {
  window.miSpinner.remove();
  window.miSpinner = null;
}
```

**Paso 6 — `RefreshRegionBusqueda`** (Refresh, secuencia 60): refresca la región `Respuesta` completa.

---

### Menú de Navegación

La aplicación usa un menú lateral (Tree Navigation) con las siguientes entradas:

| Entrada | Página destino |
|---------|---------------|
| Home | Página 1 |
| Upload de PDF | Página 4 |
| RAG Search | Página 5 |
| RAG Search - GenAI Response | Página 3 |

El header incluye el nombre del usuario actual (`&APP_USER.`) y la opción **Sign Out**.

---

## 11. Verificación de la Configuración

Ejecutar como usuario `DEMODU`:

### 11.1 Verificar credencial OCI

```sql
SELECT credential_name, username, enabled
FROM   user_credentials
WHERE  credential_name = 'OCI_GENAI_CRED';
```

### 11.2 Verificar tablas creadas

```sql
SELECT table_name
FROM   user_tables
WHERE  table_name IN ('RAG_DOCUMENTS','RAG_CHUNKS','RAG_QUERY_LOG','RAG_RESULTS')
ORDER BY table_name;
```

### 11.3 Verificar índice vectorial

```sql
SELECT index_name, index_type, status
FROM   user_indexes
WHERE  table_name = 'RAG_CHUNKS';
```

### 11.4 Verificar paquete compilado sin errores

```sql
SELECT object_name, object_type, status
FROM   user_objects
WHERE  object_name = 'RAG_ENGINE'
ORDER BY object_type;
```

Si el status es `INVALID`:

```sql
SHOW ERRORS PACKAGE BODY DEMODU.RAG_ENGINE;
```

### 11.5 Verificar ACL de red

```sql
SELECT host, privilege, principal
FROM   dba_network_acl_privileges
WHERE  principal = 'DEMODU';
```

### 11.6 Prueba end-to-end desde la aplicación

1. Ingresar a la aplicación APEX.
2. Ir a **Upload de PDF** → subir un PDF de prueba → esperar mensaje de éxito.
3. Verificar en `RAG_DOCUMENTS` que el status sea `CHUNKED`:
   ```sql
   SELECT doc_id, doc_name, status FROM rag_documents ORDER BY uploaded_at DESC;
   ```
4. Ir a **RAG Search** → seleccionar el documento → escribir una consulta → clic en **Search**.
5. Verificar que aparecen resultados en la tabla `Resultados de la Búsqueda`.
6. Ir a **RAG Search - GenAI Response** → repetir la búsqueda → verificar que aparece la respuesta del LLM renderizada.

---

## 12. Troubleshooting

### Error: `ORA-20001: Error obteniendo embedding`
- **Causa:** Falla en la llamada a OCI GenAI para embeddings. Puede ser credencial incorrecta, compartment erróneo o modelo no activo.
- **Solución:** Verificar que `OCI_GENAI_CRED` es válida y que `cohere.embed-v4.0` está activo en `us-ashburn-1`.

### Error: `ORA-20002: Error extrayendo texto`
- **Causa:** El BLOB no es un PDF válido o el archivo está corrupto.
- **Solución:** Verificar el archivo subido. `DBMS_VECTOR_CHAIN.UTL_TO_TEXT` soporta PDF; otros formatos pueden requerir configuración adicional.

### Error: `ORA-20010: No se encontró el archivo subido`
- **Causa:** El item `P1_PDF_FILE` está vacío al hacer submit en la Página 4.
- **Solución:** Asegurarse de seleccionar un archivo antes de hacer clic en **Upload Files**.

### Error: `ORA-29273: HTTP request failed`
- **Causa:** ACL de red no configurada.
- **Solución:** Verificar con `dba_network_acl_privileges` y volver a ejecutar `APPEND_HOST_ACE`.

### Error: `ORA-20000: OCI credentials error`
- **Causa:** Private key, fingerprint o OCID incorrecto en la credencial.
- **Solución:**
  ```sql
  BEGIN DBMS_CLOUD.DELETE_CREDENTIAL(credential_name => 'OCI_GENAI_CRED'); END;
  /
  -- Recrear con datos correctos
  ```

### La respuesta de GenAI en Página 3 es NULL o vacía
- **Causas posibles:**
  - El `compartmentId` en el payload de la DA `GenAIResponse` es incorrecto.
  - El modelo `openai.gpt-oss-120b` no está disponible en `us-chicago-1` para el compartment.
  - La Web Credential `OCI_GENAI_CRED` no está correctamente configurada en el workspace.
- **Diagnóstico:** Ejecutar el bloque PL/SQL de la DA `GenAIResponse` directamente en SQL\*Plus o SQL Developer con `DBMS_OUTPUT` habilitado para ver el JSON completo de `l_response`.

### El spinner no desaparece después de la búsqueda
- **Causa:** El proceso PL/SQL de la DA lanzó una excepción y el JavaScript del Paso 5 no se ejecutó.
- **Solución:** Revisar los errores en la consola del navegador (F12) y en los logs de APEX (`APEX_DEBUG`).

### El documento queda en STATUS = 'ERROR'
- **Causa:** Falla en `PROCESS_DOCUMENT` durante extracción, chunking o embedding.
- **Solución:**
  ```sql
  -- Re-ejecutar manualmente para ver el error:
  BEGIN DEMODU.RAG_ENGINE.PROCESS_DOCUMENT(p_doc_id => <id>); END;
  /
  ```

### El paquete RAG_ENGINE tiene errores de compilación
- **Causa:** Grants faltantes sobre los tipos OCI Inference o `DBMS_VECTOR_CHAIN`.
- **Solución:**
  ```sql
  SHOW ERRORS PACKAGE BODY DEMODU.RAG_ENGINE;
  ALTER PACKAGE DEMODU.RAG_ENGINE COMPILE BODY;
  ```

### La aplicación APEX no encuentra el schema DEMODU
- **Causa:** El workspace no tiene `DEMODU` asignado como schema de parsing.
- **Solución:** APEX Internal → Manage Workspaces → Manage Schemas → asociar `DEMODU` a `WK_DEMODU`.

---

## Referencia Rápida de Scripts

| Script | Usuario | Descripción |
|--------|---------|-------------|
| `CREATE USER DEMODU` | ADMIN | Crear usuario de la aplicación |
| Grants `DBMS_CLOUD/VECTOR/VECTOR_CHAIN` | ADMIN | Permisos sobre paquetes Oracle Cloud |
| Grants tipos OCI GAI Inference (8 grants) | ADMIN | Permisos para embedding vía PL/SQL |
| `ENABLE_PRINCIPAL_AUTH` | ADMIN | Habilitar autenticación OCI |
| `APPEND_HOST_ACE` (*.oci.oraclecloud.com) | ADMIN | ACL de red para endpoints OCI |
| `CREATE_CREDENTIAL('OCI_GENAI_CRED')` | DEMODU | Credencial PL/SQL para embeddings |
| Web Credential `OCI_GENAI_CRED` en APEX | DEMODU_ADMIN | Credencial REST para llamadas chat de Página 3 |
| DDL `RAG_DOCUMENTS` | DEMODU | Almacén de documentos fuente |
| DDL `RAG_CHUNKS` + índice `RAG_CHUNKS_VIDX` | DEMODU | Chunks + embeddings `VECTOR(1536, FLOAT32)` |
| DDL `RAG_QUERY_LOG` | DEMODU | Auditoría de consultas |
| DDL `RAG_RESULTS` | DEMODU | Resultados por sesión APEX |
| `CREATE PACKAGE RAG_ENGINE` (spec + body) | DEMODU | Motor RAG completo |
| Importar `APEXAPP.sql` en workspace | DEMODU_ADMIN | Aplicación APEX ID 102 con 6 páginas |
| Actualizar Remote Server post-importación | DEMODU_ADMIN | Verificar URL base de OCI GenAI |
| Actualizar `compartmentId` en Página 3 | DEMODU_ADMIN | Ajustar al compartment real del ambiente |

---

*Documento generado para el proyecto **Document Understanding Lab** — Oracle APEX 24.2 + OCI Generative AI.*  
*Embedding: `cohere.embed-v4.0` · `us-ashburn-1` · `1536 FLOAT32` | Chat LLM: `openai.gpt-oss-120b` · `us-chicago-1` | Credencial: `OCI_GENAI_CRED`*

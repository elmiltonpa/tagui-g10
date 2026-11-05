# 📐 Plan de Implementación de Arquitectura: Módulo de Ítems y Cruce con Catálogo

## 1. Resumen Ejecutivo y Objetivos

El objetivo de esta iniciativa es evolucionar el bot de automatización de licitaciones públicas ([comprar.gob.ar](https://comprar.gob.ar/)) para pasar de un filtrado superficial por metadatos (rubro y fechas) a un **análisis de valor semántico a nivel renglón de compra**.

### Objetivos Clave:
1. **Extracción Granular:** Recolectar renglones, códigos oficiales de catálogo, descripciones técnicas, cantidades y unidades de medida de cada contratación.
2. **Motor de Cruce Híbrido (Matching Engine):** Comparar los ítems demandados por el Estado contra el catálogo de bienes y servicios propios de la empresa, ponderando código oficial, palabras clave y similitud difusa.
3. **Reportes Bivariados:** Generar libros Excel (`.xlsx`) estructurados en 2 hojas (`Procesos` consolidados e `Items_Licitados` desglosados renglón por renglón).
4. **Filtrado Inteligente de Alertas:** Priorizar notificaciones periódicas en Telegram, silenciando licitaciones que no posean ningún ítem comercializable para la empresa.
5. **Gestión Interactiva:** Proveer un comando `/catalogo` en Telegram para CRUD de catálogo, sincronización de archivos Excel y búsqueda en vivo en `BuscarItemCiudadano`.

---

## 2. Arquitectura del Sistema

El diseño sigue los principios de **Arquitectura Limpia / Separación de Responsabilidades**, desacoplando la recolección, el motor de reglas de negocio, la persistencia y la interfaz de usuario.

```mermaid
flowchart TD
    subgraph UI_Telegram [Capa de Presentación / Telegram]
        B[Handlers de Bot: /buscar, /alertas, /catalogo]
        T[Teclados Inline y Callbacks]
    end

    subgraph Business_Logic [Capa de Negocio y Cruce]
        M[Motor de Matching: Código | Keywords | Fuzzy]
        R[Generador de Reportes Excel: 2 Hojas]
    end

    subgraph Scraping_Engine [Capa de Extracción RPA / HTTP]
        S[scraper.py]
        TL[TAGUI_LOCK Mutex]
        H[HTTP Client: _GrillaItems y Pliegos]
        TUI[TagUI + Headless Chrome]
    end

    subgraph Storage [Capa de Persistencia Atómica]
        CAT[(catalogo_empresa.xlsx)]
        JSON[(JSONs: Alertas y Presets)]
    end

    B --> S
    S --> TL
    TL --> H
    TL --> TUI
    S --> M
    M --> CAT
    M --> R
    B --> JSON
```

---

## 3. Desglose de Fases de Implementación

### Fase 1: Modelo de Datos del Catálogo y Persistencia Concurrente
* **Estructura del Catálogo (`catalogo_empresa.xlsx`):**
  * `Codigo`: Código oficial del nomenclador de COMPR.AR (ej. `2.3.2-1571.23`).
  * `Descripcion`: Denominación del producto o servicio.
  * `Palabras_Clave`: Tokens o frases separadas por coma para match flexible.
  * `Rubro`: Rubro asociado en COMPR.AR.
* **Persistencia Atómica y Thread-Safe:**
  * Uso de buffers temporales (`tempfile.NamedTemporaryFile`) y reemplazo atómico (`os.replace`).
  * Locks de exclusión mutua (`threading.Lock`) para evitar corrupción ante accesos concurrentes entre el worker en segundo plano y comandos del usuario.
  * Backup sincronizado en la carpeta montada en Docker (`resultados/`).

### Fase 2: Motor de Matching Inteligente en Tres Niveles
Para superar la variabilidad de redacción entre organismos públicos, el algoritmo evalúa:

$$\text{Score}_{\text{final}} = \max\left(\text{Score}_{\text{código}},\, \text{Score}_{\text{keywords}},\, \text{Score}_{\text{similitud}}\right)$$

1. **Nivel 1: Código Oficial de COMPR.AR (Precisión Alta)**
   * Coincidencia exacta: `100%`.
   * Coincidencia de clase/prefijo (mismo segmento de producto): `88%`.
2. **Nivel 2: Palabras Clave y Tokenización (Cobertura Media-Alta)**
   * Normalización de texto: remoción de diacríticos/tildes, caracteres especiales y minúsculas.
   * Filtrado de *stopwords* en español.
   * Rango de puntaje: `65%` a `95%` proporcional a la densidad de keywords presentes en la descripción oficial.
3. **Nivel 3: Similitud Difusa y Solapamiento Léxico (Red de Seguridad)**
   * Comparación de secuencias (`difflib.SequenceMatcher` o `RapidFuzz`).
   * Umbral mínimo de activación: $\ge 60\%$.

### Fase 3: Extracción Eficiente de Renglones (Scraping Híbrido)
* **El Cuello de Botella:** La automatización tradicional con TagUI abriendo proceso por proceso añade una latencia inaceptable (~2 a 4 segundos por página).
* **Solución Técnica:**
  * **Cabecera general:** Descarga directa de `ReporteProcesos.xls` mediante TagUI.
  * **Inspección de renglones:** Extracción acelerada mediante cliente HTTP asíncrono/síncrono (`httpx`) contra `VistaPreviaPliegoCiudadano.aspx` replicando los parámetros de postback ASP.NET (`__VIEWSTATE`, `__EVENTVALIDATION`, headers de sesión).
  * **Fallback resiliente:** Si el endpoint HTTP falla o rechaza la sesión, TagUI ejecuta la navegación visual por DOM (`gvLineaPliego`).
  * **Control de profundidad:** Parámetro `MAX_PROCESOS_INSPECCION` configurable para acotar la búsqueda a los N procesos más relevantes.

### Fase 4: Enriquecimiento de Reportes Excel
Transformación del archivo transitorio `.xls` en un libro `.xlsx` con dos hojas:
* **Hoja 1 (`Procesos`):** Mantiene las columnas originales del portal del Estado y agrega métricas de negocio:
  * `Coincide con Catálogo` (`SÍ` / `NO`).
  * `Ítems Coincidentes` (ej. `3 de 7`).
  * `Grado de Coincidencia` (% de renglones que la empresa puede abastecer).
  * `Detalle de Coincidencias` (resumen de renglones y criterios de match).
* **Hoja 2 (`Items_Licitados`):** Base de datos plana con cada renglón extraído:
  * Número de proceso, número de renglón, código de ítem, descripción técnica, cantidad solicitada, unidad de medida, ítem coincidente de la empresa y porcentaje de score obtenido.

### Fase 5: Interfaz de Usuario y Filtrado en Telegram
* **Comando `/catalogo`:**
  * Visualización del catálogo en chat con paginación limpia.
  * Conexión en vivo con `BuscarItemCiudadano` (`_GrillaItems` HTTP-first) para agregar ítems oficiales tocando un botón.
  * Carga manual por formato estructurado (`CODIGO | DESCRIPCION | KEYWORDS`).
  * Descarga y carga masiva mediante archivos `.xlsx`.
* **Priorización de Alertas (`/alertas`):**
  * Opción explícita: *“Solo si incluye ítems de mi catálogo”*.
  * Si la licitación no contiene renglones coincidentes, el worker omite el aviso y registra el proceso como visto para no saturar al usuario.

---

## 4. Matriz de Riesgos Técnicos y Mitigaciones

| Riesgo Técnico | Impacto | Mitigación Arquitectónica |
| :--- | :---: | :--- |
| **Colisión de procesos en TagUI** (Chrome headless colapsa si dos hilos llaman a `r.init()`). | **Crítico** | `TAGUI_LOCK` global (`threading.Lock`) que serializa ejecuciones y garantiza el cierre en bloques `finally`. |
| **Pérdida de sesión / CSRF en ASP.NET** (fallo en requests directos de pliegos). | **Medio** | Headers completos de navegador (`User-Agent`, `Referer`, `Origin`), cookies automáticas y fallback a RPA visual. |
| **Corrupción de archivos JSON / Excel** por escrituras concurrentes. | **Alto** | Persistencia atómica mediante buffers temporales y `os.replace`. |
| **Falsos positivos por descripciones genéricas** (ej. "servicio"). | **Medio** | Remoción rigurosa de stopwords y requerimiento de códigos o pares de keywords específicas. |
| **Bloqueo del bot por Markdown malformado** (caracteres `_`, `*`, `[` en pliegos). | **Bajo** | Sanitización previa con `limpiar_texto_markdown()` y captura preventiva de `BadRequest`. |

---

## 5. Criterios de Aceptación y Validación

1. **Pruebas de Catálogo:**
   * Carga y persistencia correcta del Excel base.
   * Modificación de ítems sin pérdida de registros preexistentes.
2. **Pruebas de Matching:**
   * Caso código exacto $\rightarrow$ Score 100%.
   * Caso descripción con sinónimos / keywords $\rightarrow$ Score $\ge$ 65%.
   * Caso sin relación comercial $\rightarrow$ Score 0%.
3. **Pruebas de Integración y Concurrencia:**
   * Ejecución simultánea de una búsqueda manual y el cron de alertas sin bloqueos ni errores en el log.
   * Generación y apertura correcta del Excel de 2 hojas en lectores compatibles (Excel, LibreOffice, Google Sheets).
4. **Despliegue Contenerizado:**
   * Contenedor Docker estable, montando volúmenes para `/app/resultados`, `/app/configuraciones_busqueda.json`, `/app/alertas_busqueda.json` y `/app/catalogo_empresa.xlsx`.

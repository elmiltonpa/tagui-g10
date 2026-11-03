# 📄 Informe de Nuevas Funcionalidades: Búsqueda de Ítems y Cruce con Catálogo de Empresa

Este documento resume la implementación de la mejora planteada en el apartado *"Información Adicional / Posible mejora a futuro"* del `README.md`: **incorporar la búsqueda de ítems en `comprar.gob.ar`, recolectar los ítems asociados a cada proceso licitatorio y cruzarlos contra un catálogo propio de la empresa**.

> **Nota sobre el código:**  
> Todos los bloques de código nuevos o modificados en el proyecto están delimitados entre los comentarios **`# --- NUEVO ---`** y **`# --- FIN NUEVO ---`** para que cualquier integrante del grupo pueda identificar rápidamente qué se agregó en cada archivo.

---

## 🚀 ¿Qué hace ahora el bot?

### 1. Gestión del Catálogo de la Empresa (`/catalogo` y `catalogo_empresa.xlsx`)
- Se creó el módulo **`catalogo.py`** y el nuevo comando **`/catalogo`** en el bot de Telegram.
- El bot trabaja con un archivo Excel propio (**`catalogo_empresa.xlsx`**) que contiene los productos o servicios que la empresa puede proveer, con las columnas:
  - `Codigo` (código oficial del nomenclador de COMPR.AR, ej. `2.3.2-1571.23`)
  - `Descripcion` (nombre o descripción del ítem)
  - `Palabras_Clave` (palabras clave separadas por coma para el cruce flexible)
  - `Rubro` (rubro asociado)
- Si el archivo no existe al arrancar el bot, se genera automáticamente con ítems de ejemplo y se mantiene sincronizado en `resultados/catalogo_empresa.xlsx` para persistir al usar Docker.
- **Desde el comando `/catalogo` en Telegram se puede:**
  1. **📋 Ver ítems del catálogo:** Lista en el chat todos los ítems cargados actualmente y sus palabras clave.
  2. **🔍 Buscar en COMPR.AR (`BuscarItemCiudadano`):** Permite escribir una palabra (ej. `"resma"`, `"computadora"`, `"chapa"`) o un código, consulta `https://comprar.gob.ar/Item/Item/BuscarItemCiudadano`, muestra los ítems oficiales del Estado con su código exacto y permite **agregarlos al catálogo de la empresa tocando un botón**.
  3. **➕ Agregar ítem manualmente:** Permite cargar un ítem enviando `CODIGO | DESCRIPCION | PALABRAS_CLAVE`.
  4. **📥 Descargar `catalogo_empresa.xlsx`:** Envía el archivo Excel actual por el chat de Telegram.
  5. **📤 Subir/Reemplazar Excel (`.xlsx`):** Permite enviar un archivo Excel editado en la PC para actualizar todo el catálogo de una sola vez.
  6. **🗑️ Eliminar un ítem:** Permite borrar ítems del catálogo desde botones interactivos.

---

### 2. Recolección de Ítems de cada Licitación (`scraper.py`)
- Originalmente el bot sólo descargaba el archivo general de procesos (`ReporteProcesos.xls`), donde únicamente figura el título general de la contratación pero **no qué productos ni qué cantidades se están comprando**.
- Ahora, durante la búsqueda de procesos, el bot también accede a la vista previa del pliego (`VistaPreviaPliegoCiudadano.aspx`) y extrae la tabla **"Detalle de productos o servicios"** (`#ctl00_CPH1_UCVistaPreviaPliego_UC_DetalleProductos_gvLineaPliego`), recolectando para cada renglón:
  - **Número de renglón**
  - **Objeto del gasto**
  - **Código oficial del ítem**
  - **Descripción técnica**
  - **Cantidad** y **Unidad de medida** (separadas automáticamente, ej. `2.000,00` y `UNIDAD`).

---

### 3. Motor de Cruce Inteligente (`catalogo.py`)
Como los distintos organismos públicos redactan las descripciones de maneras diferentes, el cruce de cada ítem licitado contra `catalogo_empresa.xlsx` evalúa **3 criterios combinados**:
1. **Por Código de Catálogo (`100%` exacto / `88%` misma clase):** Verifica si el código oficial de COMPR.AR del renglón licitado coincide con el código cargado en nuestro catálogo.
2. **Por Palabras Clave (`65%` a `95%`):** Normaliza los textos (minúsculas, sin tildes ni signos) y verifica cuántas palabras o frases clave configuradas en el catálogo aparecen en el ítem licitado.
3. **Por Similitud de Texto (`>= 60%`):** Utiliza `difflib.SequenceMatcher` y solapamiento de palabras relevantes (sin *stopwords*) entre la descripción del pliego y la descripción del catálogo.

---

### 4. Reporte Excel Enriquecido (`.xlsx`)
El archivo Excel que envía el bot pasa a ser un libro `.xlsx` con **dos hojas**:
- **Hoja 1 (`Procesos`):** Conserva todas las columnas originales de COMPR.AR y suma **4 columnas nuevas**:
  - `Coincide con Catálogo` (`SÍ` / `NO` / `No inspeccionado`)
  - `Ítems Coincidentes` (ej. `2 de 5`)
  - `Grado de Coincidencia` (porcentaje de renglones del proceso que coinciden con el catálogo, ej. `40.0%`)
  - `Detalle de Coincidencias` (resumen de qué renglones matchearon, cantidad, unidad y bajo qué criterio).
- **Hoja 2 (`Items_Licitados`):** Contiene el desglose renglón por renglón de todos los ítems extraídos de las licitaciones (`Número de Proceso`, `Renglón`, `Código Ítem COMPR.AR`, `Descripción del Ítem`, `Cantidad`, `Unidad de Medida`, `Coincide con Catálogo`, `Ítem del Catálogo`, `Criterio de Coincidencia` y `Puntaje Match (%)`).

Además, en el chat de Telegram el bot envía un resumen destacando cuáles de los procesos encontrados coinciden con el catálogo de la empresa.

---

### 5. Priorización de Alertas Automáticas (`/alertas`)
- Al configurar una nueva alerta en `/alertas`, se agregó un paso que permite elegir entre:
  - **🎯 Solo si incluye ítems de mi catálogo:** La tarea en segundo plano (`tarea_revisar_alertas`) analiza los ítems de las nuevas licitaciones y **sólo envía la notificación si al menos un ítem coincide con el catálogo de la empresa**, detallando en el mensaje de alerta qué ítems coincidieron.
  - **🌐 Notificar todas las del rubro:** Mantiene el comportamiento general pero destacando igualmente en el mensaje si alguna licitación matchea con el catálogo.

---

## 📂 Resumen de Archivos Modificados y Creados

Todos los cambios pueden buscarse en el editor filtrando por el texto `# --- NUEVO ---`:

| Archivo | Estado | Descripción de los cambios |
| :--- | :---: | :--- |
| `catalogo.py` | **Nuevo** | Carga/guardado de `catalogo_empresa.xlsx`, motor de cruce por código/keywords/similitud y generación del Excel enriquecido con 2 hojas. |
| `scraper.py` | Modificado | Función `buscar_items_ciudadano()`, extracción de ítems por pliego (`gvLineaPliego`) e integración del enriquecimiento de Excel. |
| `handlers.py` | Modificado | Flujo completo del comando `/catalogo`, opción de prioridad al crear alertas, filtrado en `tarea_revisar_alertas()` y resumen de coincidencias en `/buscar`. |
| `teclados.py` | Modificado | Teclados inline para el menú `/catalogo`, resultados de `BuscarItemCiudadano` y prioridad de alertas. |
| `alertas.py` | Modificado | Campo `solo_catalogo` al guardar alertas y función `obtener_resumen_coincidencias_excel()`. |
| `bot.py` | Modificado | Registro del comando `/catalogo`, nuevos estados del `ConversationHandler` e inicialización del catálogo al arrancar. |
| `config.py` | Modificado | Constante `CATALOGO_PATH` (`catalogo_empresa.xlsx`). |
| `requirements.txt` | Modificado | Agregada la librería `openpyxl==3.1.5` para soporte de archivos `.xlsx`. |

---

## ⚡ Optimizaciones y Estabilización Técnica

1. **Sincronización y Concurrencia (`TAGUI_LOCK`):**
   - Se incorporó un lock global sobre el motor de TagUI para serializar el acceso a Chrome. Si una alerta periódica y una búsqueda manual coinciden en el tiempo, una espera a la otra en cola en vez de generar un conflicto de procesos o caída de Chrome.
   - En Telegram, el bot le informa al usuario si el navegador está ocupado y pone su consulta en espera automáticamente.
2. **Escrituras Atómicas y Thread-Safety:**
   - La persistencia de alertas, presets y catálogo ahora utiliza guardado atómico (escritura en archivo temporal + `os.replace`) protegido por mutex. Esto previene corrupción de archivos JSON o Excel por lecturas/escrituras simultáneas.
3. **Consulta Rápida HTTP-First en Catálogo (`_GrillaItems`):**
   - La búsqueda ciudadana (`/catalogo` $\rightarrow$ Buscar en COMPR.AR) ahora consulta primero el endpoint AJAX oficial vía `httpx` con headers de navegador. Responde en ~200ms sin necesidad de abrir ni bloquear el navegador TagUI, usando RPA únicamente como respaldo.
4. **Inspección Profunda Configurable:**
   - Se añadió `MAX_PROCESOS_INSPECCION` en `config.py` (por defecto 10, configurable vía `.env`), permitiendo inspeccionar más procesos sin tiempos muertos gracias a la aceleración HTTP.
5. **Resiliencia en Mensajería Telegram:**
   - Se agregaron capturas de `BadRequest` en el envío de mensajes con formato Markdown y reportes adjuntos para garantizar que caracteres especiales en nombres de contrataciones públicas nunca interrumpan el flujo del bot.

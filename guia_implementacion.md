# 🛠️ Guía de Implementación: Búsqueda de Ítems y Cruce con Catálogo

Esta guía detalla el orden recomendado para codificar e integrar el módulo de catálogo y cruce inteligente de ítems en la arquitectura actual del bot de `comprar.gob.ar`.

## Fase 1: Dependencias y Variables Globales
1. **`requirements.txt`:** Agregar la librería `openpyxl==3.1.5` para permitir la creación y manipulación de archivos `.xlsx` de múltiples hojas.
2. **`config.py`:** Definir la ruta donde residirá la base de datos de ítems de la empresa para que persista en el volumen de Docker:
   ```python
   CATALOGO_PATH = os.path.join(BASE_DIR, 'catalogo_empresa.xlsx')
   ```

## Fase 2: Creación del Motor Central (`catalogo.py`)
Crear este archivo desde cero para aislar toda la lógica de negocio relacionada con el cruce de información:
* **Inicialización:** Crear `cargar_catalogo()` para generar el Excel base con las columnas `Codigo`, `Descripcion`, `Palabras_Clave` y `Rubro` si no existe al arrancar.
* **Motor de Matching:** Codificar `cruzar_item_contra_catalogo()` para evaluar 3 niveles de coincidencia:
  * Código exacto de COMPR.AR (100%).
  * Coincidencia parcial iterando sobre las `Palabras_Clave`.
  * Similitud difusa usando `difflib.SequenceMatcher` (umbral >= 60%).
* **Generación de Reportes:** Escribir `enriquecer_reporte_excel()` para que tome el `.xls` original de TagUI, ejecute el matching, y guarde el `.xlsx` final con la hoja original ampliada y la nueva hoja `Items_Licitados`.

## Fase 3: Adaptación del RPA (`scraper.py`)
Intervenir el flujo de automatización existente para recolectar el detalle renglón por renglón:
* **Navegación Profunda:** Luego de listar los procesos, utilizar TagUI o `httpx` para acceder a la `VistaPreviaPliegoCiudadano.aspx` de cada licitación.
* **Extracción de Renglones:** Scrapear el código HTML de la tabla de ítems (`#ctl00_CPH1_UCVistaPreviaPliego_UC_DetalleProductos_gvLineaPliego`).
* **Parseo de Datos:** Aislar el número de renglón, código oficial, descripción técnica, cantidad y unidad de medida.
* **Integración del Motor:** Antes de retornar el archivo en `buscar_procesos_de_un_rubro()`, pasar los datos recolectados por `enriquecer_reporte_excel()` y borrar el `.xls` transitorio.
* **Búsqueda Ciudadana:** Agregar una función `buscar_items_ciudadano()` que ataque el endpoint web para buscar códigos oficiales de ítems.

## Fase 4: Actualización del Motor de Alertas (`alertas.py`)
Para que las alertas automáticas soporten la priorización:
* **Modificar el guardado:** Actualizar `guardar_alerta()` agregando el parámetro booleano `solo_catalogo=False` para guardarlo en el JSON de configuración.
* **Lectura de coincidencias:** Crear `obtener_resumen_coincidencias_excel(ruta_excel)`. Debe leer el `.xlsx` con `pandas` y devolver un diccionario con los IDs que hicieron match y un resumen en texto Markdown.

## Fase 5: Configuración de la App de Telegram (`bot.py`)
Integrar los nuevos flujos conversacionales al Dispatcher:
* **Inicialización:** Ejecutar `asegurar_catalogo_inicial()` de `catalogo.py` antes de iniciar la app.
* **Nuevos comandos y estados:** Agregar `CommandHandler('catalogo', menu_catalogo)` y registrar los nuevos estados (`ELEGIR_PRIORIDAD_ALERTA`, `GESTION_CATALOGO`, `BUSCAR_ITEM_WEB`, `INGRESAR_ITEM_MANUAL`, `SUBIR_CATALOGO_EXCEL`).
* **Manejo de archivos:** Configurar un `MessageHandler` para interceptar documentos Excel en el estado `SUBIR_CATALOGO_EXCEL`.

## Fase 6: Teclados e Interfaz (`teclados.py`)
Construir los menús de botones:
* Armar los `InlineKeyboardMarkup` para el menú principal de `/catalogo` (Botones: Ver ítems, Buscar, Agregar manual, Subir Excel, Borrar).
* Armar el selector de prioridad ("Solo catálogo" vs "Todas") para el alta de nuevas alertas en `/alertas`.

## Fase 7: Handlers y Flujos (`handlers.py`)
Conectar la lógica con los comandos del bot:
* **Búsquedas manuales:** En `ejecutar_busqueda()`, procesar las coincidencias y enviar un texto resumen antes de adjuntar el Excel.
* **Priorización de alertas:** En `tipo_operacion_elegido()`, desviar al usuario al estado de prioridad si está creando una alerta.
* **Filtro en notificaciones (Background):** En `tarea_revisar_alertas()`, si la alerta tiene `solo_catalogo=True` y no hay coincidencias en el Excel, aplicar un `continue` para silenciar el aviso.
* **Módulo `/catalogo`:** Implementar todas las funciones de navegación (`menu_catalogo`, `accion_catalogo`), la conexión con el buscador oficial (`termino_item_web_ingresado`), y la recepción y actualización de archivos (`archivo_catalogo_recibido`).
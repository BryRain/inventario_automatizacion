# Inventario-Final · Automatización inteligente de inventario con n8n

> **Corte final · EAM 2026 II · Automatización inteligente con n8n** (profesor: Robert Garcia Rey)

## 1. Identificación

| | |
|---|---|
| **Proyecto** | Control automático de stock con alertas en Trello, consulta en lenguaje natural y registro de errores |
| **Integrantes** | William Rodriguez, Kevin Rengifo, Bryan Benavides y Michael Menotti |
| **Repositorio** | `n8n-final-rodriguez_rengifo_benavides_menotti-Inventario` |
| **Versión de n8n** | Reciente: los nodos usan If 2.3, Filter 2.3, Switch 3.4, Google Sheets 4.7 y Agent 2.2 |

**Problema.** El encargado de inventario revisaba a mano una hoja de cálculo, creaba tarjetas de reposición sin control de duplicados y no había registro de datos erróneos ni de fallos.

**Objetivo.** Revisar el stock de forma automática (diaria, por solicitud o manual), crear **una sola** tarjeta de reposición por producto bajo el mínimo, interpretar con IA las observaciones del encargado y permitir consultar el stock por chat, dejando constancia localizable de cualquier fallo.

**Alcance de la versión final.** Incluye: revisión de stock, validaciones, alertas en Trello con control de duplicados, clasificación de observaciones con IA, agente de consulta de **solo lectura**, y registro de errores. Excluye: modificar el stock, comprar o reservar, notificaciones por correo o chat, y un servidor público permanente.

## 2. Vista general

Todo está en **un único workflow** (`workflows/Workflow_Final.json`), sin llamadas a otros workflows:

1. **Entradas:** `Webhook` POST `/revisar-inventario` (con Header Auth), `Schedule Trigger Diario` (8:00, zona America/Bogota) o ejecución manual.
2. **Revisión:** lee la hoja *Hoja 1*, valida cada fila, aparta las inválidas a la hoja *Errores*, filtra `cantidad <= stock_minimo` y clasifica por categoría.
3. **Alerta por producto (ciclo uno a uno):** valida la entrada, consulta las tarjetas abiertas de la lista de Trello, omite duplicados, interpreta la observación con IA (si existe), crea la tarjeta y confirma con el `id` devuelto por Trello.
4. **Respuesta:** resumen real (creadas, duplicadas omitidas, fallidas, filas inválidas).
5. **Agente de consulta:** `Chat Inventario` → `Agente Inventario` (Gemini) con una sola herramienta de lectura, `consultar_stock`.
6. **Errores:** `Error Trigger` → clasifica y registra en la hoja *Errores*.

**Función de la IA** (Gemini 2.5 Flash, Google AI Studio): (a) clasificar y resumir observaciones de texto libre (la regla de reposición **no** usa IA: es `cantidad <= stock_minimo`); (b) entender consultas de stock en lenguaje natural. Detalle en la [documentación](documentacion/Documentacion_Proyecto_Final.md).

Diagramas: [Proceso manual](documentacion/diagramas/Proceso_Manual.png) · [Proceso automatizado](documentacion/diagramas/Proceso_Automatizado.png) · [Agente y errores](documentacion/diagramas/Proceso_Automatizado_Agente_y_Errores.png)

## 3. Requisitos

- n8n (autohospedado o cloud) — versión indicada arriba.
- Cuenta de **Google** con Google Sheets (permiso de lectura y escritura sobre su propio libro).
- Cuenta de **Trello** con un tablero y una lista de reposición propios; API Key y Token de Trello.
- Clave de **Google AI Studio** (Gemini). Verifiquen que el modelo `models/gemini-2.5-flash` siga disponible y tenga cuota; si no, cambien el modelo en los nodos `Gemini Chat Model` y `Gemini Chat Model Agente`.
- Para llamar al webhook desde fuera: una URL accesible (túnel temporal como ngrok/cloudflared o la URL de su n8n). No se exige servidor permanente.

## 4. Configuración (con credenciales propias)

**4.1 Libro de Google Sheets** (datos ficticios incluidos en [`recursos/`](recursos/)):
- Hoja **`Hoja 1`** con encabezados `producto, categoria, cantidad, stock_minimo, observacion` → importe [`Inventario_Hoja1_datos_ficticios.csv`](recursos/Inventario_Hoja1_datos_ficticios.csv). Las categorías `Perifericos` y `Monitores` (sin tilde, exactas) tienen mensaje propio; las demás usan el mensaje genérico.
- Hoja **`Errores`** con encabezados `fecha_hora, workflow, nodo, mensaje, ejecucion, tipo, estado, url_ejecucion` ([`Errores_encabezados.csv`](recursos/Errores_encabezados.csv)).
- Copien el **ID del libro** (la parte larga de la URL entre `/d/` y `/edit`).

**4.2 Trello:** cree una lista (p. ej. «Reposición urgente»). Obtengan su **ID de lista** (agregar `.json` a la URL de una tarjeta de esa lista y buscar `idList`, o usar la API).

**4.3 Credenciales en n8n** (crear estas 4 con valores propios; nunca se suben al repositorio):

| Credencial (tipo) | Valor | Se usa en |
|---|---|---|
| Google Sheets OAuth2 | su cuenta de Google | 5 nodos de Google Sheets |
| Header Auth (para Trello) | Name: `Authorization` · Value: `OAuth oauth_consumer_key="SU_API_KEY", oauth_token="SU_TOKEN"` | `Consultar Tarjetas Abiertas`, `Crear tarjeta Trello` |
| Header Auth (para el webhook) | Name: p. ej. `X-Inventario-Key` · Value: una clave inventada por ustedes | `Webhook` |
| Google Gemini (PaLM) API | su clave de Google AI Studio | `Gemini Chat Model`, `Gemini Chat Model Agente` |

**4.4 Parámetros a reemplazar en el JSON importado** (marcadores `REEMPLAZAR…`):

| Marcador | Dónde | Reemplazar por |
|---|---|---|
| `REEMPLAZAR_ID_HOJA_INVENTARIO` | los 5 nodos de Google Sheets (campo *Document*, modo ID) | ID de su libro |
| `REEMPLAZAR_ID_LISTA_TRELLO` | URL de `Consultar Tarjetas Abiertas` y parámetro `idList` de `Crear tarjeta Trello` | ID de su lista de Trello |
| credenciales `REEMPLAZAR` | todos los nodos con credencial | seleccionar la credencial propia |

## 5. Importación y ejecución

1. En n8n: **Workflows → Import from file →** [`workflows/Workflow_Final.json`](workflows/Workflow_Final.json).
2. Abra cada nodo marcado con advertencia roja y seleccione su credencial; reemplace los marcadores de 4.4.
3. **Workflow → Settings → Error workflow:** seleccione **este mismo workflow** (así el registro de errores funciona sin otro workflow). *(Con la importación por CLI `n8n import:workflow` el ajuste ya viene hecho.)*
4. Nodo `consultar_stock`: debe quedar con *Workflow ID* `{{ $workflow.id }}` (llama a este mismo workflow). Si muestra «workflow no encontrado», seleccione el workflow importado por ID.
5. **Guarde** y pruebe con **Execute workflow** (ejecución manual). Para el webhook de producción y el programado, **active** el workflow.

**Ejemplo reproducible (datos ficticios).**
- *Condiciones iniciales:* hoja cargada con el CSV de `recursos/`, lista de Trello vacía, workflow activo.
- *Entrada:*
```bash
curl -X POST "https://SU-HOST/webhook/revisar-inventario" \
  -H "X-Inventario-Key: SU_CLAVE" -H "Content-Type: application/json" \
  -d '{"accion":"revisar_stock","solicitud_id":"DEMO-001"}'
```
- *Resultado esperado (1.ª ejecución):* `HTTP 200` con `productos_bajo_minimo: 4`, `alertas_creadas: 4`, `duplicados_omitidos: 0`, `alertas_fallidas: 0`, `filas_invalidas: 1`; en Trello 4 tarjetas «Reposición urgente: …» (Mouse Óptico, Monitor 24", Disco SSD 1TB, Webcam HD); en la hoja *Errores* una fila `dato_invalido / pendiente_correccion` por *Cable HDMI*.
- *2.ª ejecución idéntica:* `alertas_creadas: 0`, `duplicados_omitidos: 4` (no se crean tarjetas repetidas).
- *Dónde consultarlo:* respuesta HTTP, lista de Trello y hoja *Errores*; la ejecución, en **Executions** de n8n.
- *Chat:* abra `Chat Inventario` en el editor y escriba «¿cuánto stock hay de mouse óptico?».

**Detener las ejecuciones programadas tras las pruebas:** *desactive* el workflow (interruptor *Active*). Con eso se detienen el `Schedule Trigger Diario` (8:00) y el webhook de producción.

> Nota: el `Error Trigger` solo reacciona a ejecuciones **no manuales** (webhook, programadas o ejecutadas por la herramienta del agente), no a las que se lanzan con el botón del editor.

## 6. Orientación

-  [Documentación del proyecto](documentacion/Documentacion_Proyecto_Final.md)
-  [Bitácora de pruebas (clase 11)](pruebas/Bitacora_Clase_11_Pruebas.md) · [Evidencias](pruebas/evidencias/)
- [Presentación (PDF)](presentacion/Presentacion_Proyecto_Final.pdf)
- Diagramas (también [resumen](documentacion/diagramas/Proceso_Automatizado_Resumen.png)): [manual](documentacion/diagramas/Proceso_Manual.png) · [automatizado](documentacion/diagramas/Proceso_Automatizado.png)

**Limitaciones relevantes:** la memoria del agente es del tipo *Simple Memory* (no apta para modo de colas); `Crear tarjeta Trello` no se reintenta a propósito (evita duplicados); el control de duplicados compara por nombre de tarjeta en una sola lista; la salida de la IA es un borrador que requiere contraste humano; el estado real de verificación de cada prueba consta en la bitácora.



video de demostracion: 



https://github.com/user-attachments/assets/2dc45469-9607-459b-aa2b-45d83cb3b304






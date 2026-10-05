# Backlog — Livetrack

## 05-oct-2026 — Hecho: botón Waze/Google Maps + capa de lluvia (RainViewer)

- **Causa:** manejando se necesita ruta óptima, tráfico y alertas de riesgo
  (inundaciones) para no mojar el vehículo. LiveTrack solo compartía el punto
  GPS; sin routing ni meteorología.
- **Cambio 1 — Navegación externa (`openInWaze`, `openInGMaps`):** al fijar
  destino aparece una fila con dos botones (WAZE / MAPS). Delegan routing,
  tráfico e incidentes a Waze y Google Maps vía deep-link (`waze.com/ul?ll=…`
  y `google.com/maps/dir/?api=1&destination=…`). También se muestran en modo
  visor cuando el que comparte envía destino, para que el receptor pueda
  navegar al mismo lugar.
- **Cambio 2 — Capa de lluvia (`toggleRain`, `loadRainLayer`):** botón 🌧 en
  el header. Usa `api.rainviewer.com/public/weather-maps.json` (gratis, sin
  API key); se agarra el último frame disponible y se pinta encima del mapa
  con `L.tileLayer` en un pane propio (`rain`, z-index 350) para que el
  filtro CSS del modo oscuro no decolore el radar. Refresca cada 10 min.
- **Verificación:** `node` smoke test — tags balanceadas (51 divs, 10 botones,
  3 scripts) y el JS inline parsea sin errores. Falta probar en móvil con GPS
  real que el deep-link a Waze abra la app nativa (en escritorio abre la web).
- **Pendiente / siguiente paso:** alertas de inundación por zona (polígonos
  de riesgo + aviso cuando el GPS entra en uno) — fuente oficial varía por
  ciudad; en MX habría que juntar datos de Protección Civil local o CONAGUA.

## 29-sep-2026 — Pendiente: compartir la liga en vivo por LaterWhats + caducidad

- **Repo correcto:** este (`livetrack`) es la versión más completa; `ltrax` es
  un fork más simple (sin campo de número ni aviso de "alguien abrió tu liga").
  Confirmar cuál está desplegado en Vercel antes de tocar el otro.
- **Ya existe:** la liga `?v=<código>` abre el visor sin login, con el mapa y el
  marcador moviéndose (MQTT). `sendWA()` abre `wa.me/<número>` con la liga.
- **Por hacer:**
  1. Botón **🕓 LATER**: abrir `latherwhats.vercel.app/?body=<mensaje con liga>&source=livetrack`
     (LaterWhats ya acepta `?body=`), para elegir contacto y enviar/programar
     desde ahí. Ya está hecho así en `ltrax` (commit 0f43146); portarlo aquí.
  2. **Caducidad de la liga** (30 min / 1 h / 8 h) y código más largo que 6
     caracteres: hoy viaja por brokers MQTT públicos y no caduca.
  3. Aviso en pantalla de que en iPhone el GPS solo actualiza con la pantalla
     encendida (ya usa Wake Lock).

## 29-sep-2026 — Hecho: botón "🕓 COMPARTIR POR LATER"

- `index.html` (`shareLater`): abre `latherwhats.vercel.app/?body=…&source=livetrack`
  con el mensaje y la liga en vivo; el contacto se elige en LaterWhats y se
  manda al momento o se programa. Queda pendiente la caducidad de la liga.

## 29-sep-2026 — Hecho: la liga caduca y el código es de 12 caracteres

- **Código:** `genCode()` ahora saca 12 caracteres (60 bits) con
  `crypto.getRandomValues`. Antes eran 6 con `Math.random`, y como el código es
  el tema MQTT de un broker público, era la única barrera.
- **Caducidad:** selector "⏱ La liga funciona durante" (30 min, 1 h —por
  defecto—, 8 h o sin límite). El que comparte calcula `expAt` al empezar y lo
  manda en cada punto (`exp`). Al vencer, `expireShare()` deja de publicar y
  borra el mensaje retenido. El visor rechaza cualquier punto con `exp` vencido
  y muestra "⏰ Esta liga ya caducó" (revisa cada 5 s).
- **Límite:** es una barrera del lado del cliente. Sin servidor propio, quien
  hable directo con el broker MQTT podría ignorar `exp`; el código largo es lo
  que lo hace impráctico. Las ligas viejas de 6 caracteres siguen abriendo.
- `docs`: el aviso del GPS en iPhone (solo con pantalla encendida) sigue pendiente.

## 29-sep-2026 — Hecho: aviso del GPS en iPhone

- `index.html`: recuadro fijo en la vista de quien comparte ("Deja esta
  pantalla abierta…") y un toast al volver a la app si estuvo fuera más de 10 s
  mientras compartía (`visibilitychange`), para que sepa que el otro vio la
  ubicación congelada. La causa es de iOS: una PWA no recibe GPS en segundo
  plano; el Wake Lock solo evita que la pantalla se apague sola.

## 29-sep-2026 — Fix: el mapa decía "API KEY REQUIRED"

- **Causa:** los mapas de CARTO (`basemaps.cartocdn.com`) ahora exigen API key.
  No devuelven error: dibujan la marca de agua "API KEY REQUIRED" encima, por
  eso el respaldo a OpenStreetMap (`tileerror`) nunca se activaba.
- **Fix:** `setTiles()` usa OpenStreetMap siempre; el modo oscuro se hace con
  un filtro CSS de inversión sobre el panel de mosaicos (`DARK_FILTER`), el
  claro sin filtro. Se quitó el filtro por mosaico de `.leaflet-tile` (se
  aplicaba doble).
- **Ojo:** `ltrax` tiene el mismo problema (mismo proveedor de mapas).

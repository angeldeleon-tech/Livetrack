# Backlog — Livetrack

## 06-oct-2026 — Hecho: gesto de arrastre del handle del panel

- **Cambio:** `initSheetDrag` suscribe `pointerdown/move/up/cancel` al
  `.handle-tap`. Durante el drag se desactiva la transición y se aplica
  `translateY(Npx)` inline en vivo (clamp 0 a `offsetHeight - 34`). Al
  soltar: si `|dy| < 6` se interpreta como tap y se hace toggle; si pasó
  `max(40, maxOffset * 0.25)` cambia de estado, si no snap atrás.
- **CSS:** `touch-action: none` en `.handle-tap` para que iOS no haga
  scroll vertical durante el drag; cursor `grab`/`grabbing`.
- **Verificación:** parse JS OK. Falta probar en el cel que el drag se
  sienta natural y el tap siga funcionando.

## 06-oct-2026 — Hecho: panel inferior colapsable para mapa a pantalla completa

- **Causa:** manejando, el bottom sheet con stats/botones/aviso ocupa ~45 %
  de la pantalla; el mapa queda chico. El `.handle` (barrita visual) ya
  existía pero no hacía nada; el CSS del `.sheet` ya tenía `transition` en
  `transform`, así que la base estaba lista pero sin cablear.
- **Cambio:** el handle ahora vive dentro de `.handle-tap` (área táctil
  cómoda) con `onclick="toggleSheet()"`. En estado colapsado, el sheet se
  baja con `translateY(calc(100% - 34px))` dejando solo la zona del handle
  visible, el mapa gana ~45 % de alto, y aparece la leyenda "TOCA PARA
  EXPANDIR" dentro del propio handle. `pointer-events:none` en los hijos
  del sheet colapsado evita taps accidentales sobre botones ocultos.
- **Persistencia:** estado en `localStorage.lt_sheet_collapsed`. Al cargar
  se restaura. Al final de la animación (450 ms) se llama
  `map.invalidateSize()` para que Leaflet recalcule el centro y los tiles.
- **Verificación:** parse JS OK, 52/52 divs. Falta probar en el cel que
  la animación corra fluida y el toque sobre el handle sea cómodo.

## 05-oct-2026 — Fix: "Zoom Level Not Supported" en el radar + "RECONECTANDO" al volver de Waze

- **Causa 1 (radar):** RainViewer solo publica tiles hasta zoom 10. Como el
  mapa se centra en el GPS a zoom 16, Leaflet pedía `/16/x/y.png` y el
  servidor devolvía un PNG estático con la leyenda "Zoom Level Not Supported"
  encima del mapa.
- **Fix 1:** `loadRainLayer` → `L.tileLayer(..., {maxNativeZoom:10, maxZoom:19})`.
  Leaflet reescala los tiles del zoom nativo 10 para los niveles superiores,
  así el radar se ve (más pixelado, pero visible y correcto) en zoom alto.
- **Causa 2 (MQTT):** `mqtt.connect(..., {reconnectPeriod: 0, ...})` desactiva
  la reconexión automática de la librería. Al salir a Waze/Maps, iOS pausa la
  pestaña y el broker cierra la conexión; el handler `close` solo cambiaba el
  badge a RECONECTANDO pero nunca reintentaba. Quedaba el texto permanente.
- **Fix 2:** en `visibilitychange` al volver visible, si `!connected` se
  cierra el cliente, se resetea `brokerIdx = 0` y se llama a `initMQTT()`
  tras 400 ms. Reconecta solo desde el primer broker.
- **Verificación:** parse JS OK. Pendiente: validar en el cel que al salir a
  Waze y volver, el badge pase de RECONECTANDO a CONECTADO en segundos.

## 05-oct-2026 — Hecho: zonas de riesgo de inundación (geocerca)

- **Causa:** manejando bajo lluvia, el vehículo no debe meterse a
  encharcamientos. La capa de RainViewer te enseña dónde está lloviendo, pero
  no sabe en qué cruces el agua se junta hasta inundar. Faltaba una capa de
  **zonas conocidas** con aviso cuando el GPS entra.
- **Realidad del dato:** no existe hoy una API pública gratuita y estable de
  inundaciones en vivo para MX consumible desde un HTML estático. CENAPRED y
  CONAGUA no exponen JSON; Protección Civil publica PDFs y shapefiles. Lo
  entregable sin backend es un **seed de polígonos** editable a mano.
- **Cambio:** constante `FLOOD_ZONES` inline en `index.html` con 7 polígonos
  seed (5 CDMX, 1 GDL, 1 MTY) de puntos de encharcamiento severo conocidos
  (Viaducto/Churubusco, bajopuente Mixcoac, Río San Joaquín, Fray Servando,
  Taxqueña; Patria/López Mateos; Garza Sada). Cada feature lleva `name`,
  `severity` (1-3) y `city`. Las coordenadas son aproximadas: cuadrados de
  ~300-500 m sobre el cruce. **Se enriquece a mano** con reportes locales.
- **UI:** botón 💧 nuevo en el header. Prendido → dibuja los polígonos como
  capa Leaflet (rojo severidad 3, ámbar severidad 2, click abre popup con
  nombre y ciudad). Independiente del botón 🌧 (RainViewer).
- **Alerta geocerca (`checkFloodZone`):** en cada `onGPS` corre
  point-in-polygon por ray casting contra todos los features; si el punto
  entra en una zona, toast + vibración (`[120,80,120,80,250]`). Un solo
  aviso por entrada: no re-alerta mientras sigas dentro de la misma, pero
  si sales y vuelves a entrar en otra, sí dispara. Cooldown adicional de
  60 s para evitar spam en el borde del polígono.
- **Al fijar destino:** `setDest` verifica si el destino cae en una zona y
  avisa antes de que arranques.
- **Fix de paso:** `toast(msg, ms)` ahora respeta el segundo parámetro — ya
  había 3 usos pasando duración (5-6 s) que la firma antigua ignoraba.
- **Decisión de embeber en lugar de servir archivo:** `vercel.json` reescribe
  todo a `/index.html`, así que `/zonas-inundacion.geojson` devolvería el HTML.
  Para no tocar la config de deploy ni introducir fetch con posibles fallas,
  los datos van inline. Si pasa de ~50 features, migrar a archivo + ajustar
  `routes` con `{handle:"filesystem"}` antes del rewrite.
- **Verificación:** parse JS OK, GeoJSON OK (7 features, todos los anillos
  cerrados), 51/51 divs, 11/11 buttons, 3/3 scripts.
- **Pendiente:** validar coordenadas con fuentes oficiales (Atlas CENAPRED,
  Protección Civil CDMX) y ampliar cobertura por ciudades donde el usuario
  manejará. UI para que el usuario agregue sus propias zonas desde la app.

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


## Sincronización con el Maestro de Drive y backlog-global — 2026-10-06

Pendientes y decisiones registrados hoy en «KA - Backlog Maestro» (sección **Livetrack**), tras auditar este repo contra el buzón:

- [ ] LVT-001 | P2 | Proceder a renombrar a «Ruta-KAi» en repositorio, stack, documentación y referencias del ecosistema (DECISIÓN-ANX confirmada) | Chat | 2026-10-06
- [ ] LVT-002 | P1 | DECISIÓN-ANX: diseño de ruteo propio anti-inundación para Monterrey (capa histórica + lluvia SMN + reportes sociales + umbral en mm + Google Routes API con bloqueos); seguridad sobre tiempo | Chat | 2026-10-06
- [ ] LVT-003 | P1 | Cuello de botella: Livetrack solo envía ubicación actual, no conoce la ruta futura (la calculan Waze/Google); para alertas preventivas hay que integrar API de rutas o predecir ruta; evaluar viabilidad y costos | Chat | 2026-10-06
- [ ] LVT-004 | P1 | DECISIÓN-ANX: «Modo seguridad bajo coacción» con toggles: pregunta señuelo camuflada para verificar recálculo, destino señuelo en pantalla y alias camuflados para destinos frecuentes | Chat | 2026-10-06
- [ ] LVT-005 | P1 | DECISIÓN-ANX: zonas de exclusión manual personalizadas (calles/colonias inseguras) que el ruteo nunca usa; la pantalla de edición solo disponible con el GPS detenido | Chat | 2026-10-06
- [ ] LVT-006 | P2 | Anx: validar en celular que al volver de Waze el badge pasa de RECONECTANDO a CONECTADO y que el deep-link abre la app nativa | Anx | 2026-10-06
- [ ] LVT-007 | P3 | Validar coordenadas de FLOOD_ZONES contra fuentes oficiales (CENAPRED, Protección Civil) y UI para zonas propias | Code | 2026-10-06
- [ ] LVT-008 | P2 | DECISIÓN-ANX: confirmar qué repo está desplegado en Vercel, Livetrack o Ltrax; Ltrax aún tiene el mapa de CARTO sin API key y código de 6 caracteres sin caducidad (portar fixes o archivar) | Anx | 2026-10-06

# Backlog — Livetrack

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

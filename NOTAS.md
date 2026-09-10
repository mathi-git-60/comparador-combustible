# Comparador de combustible — notas de contexto

App de una sola página (HTML + CSS + JS vanilla, sin build ni frameworks) para cargar precios de combustible por emblema (estación de servicio) y compararlos.

## Estructura de datos
Cada emblema:
```
{
  id, nombre,
  combustibles: [{ tipo: 'Nafta'|'Diesel'|'Alcohol', detalle, precio }],
  descuentos: {
    Lunes: { activo, tipo: 'descuento_pct'|'reintegro_pct'|'reintegro_fijo', valor, tarjeta },
    Martes: {...}, ... Domingo: {...}
  }
}
```
El campo `ubicacion` se eliminó del formulario y del guardado. Emblemas viejos que todavía lo tengan en el dato lo conservan en la fila de la base pero ya no se lee ni se muestra en ningún lado.

## Pestañas
- **Inicio**: precio más bajo cargado por tipo de combustible (sin descuentos), resumen de emblemas.
- **Agregar emblema**: alta y edición (nombre, combustibles con precio, descuentos/reintegros por día con la tarjeta que corresponde).
- **Comparar**: elegís combustible + litros + día, y lista los emblemas ordenados de más barato a más caro ya con el descuento/reintegro del día aplicado.

## Persistencia
Usa Supabase (tabla `emblemas`, columnas `id` uuid, `nombre` text, `combustibles` jsonb, `descuentos` jsonb, `created_at`, `updated_at`). El cliente JS de Supabase se carga vía CDN (`@supabase/supabase-js` UMD desde jsdelivr, sin npm ni build) directamente en `index.html`, con la Project URL y la publishable/anon key hardcodeadas en el script — es lo esperado, esa key está pensada para ser pública porque el acceso real lo controla RLS. RLS está activo con políticas públicas de lectura/escritura (sin login, cualquiera con la URL ve y edita los mismos datos, igual que antes con localStorage/window.storage, pero ahora compartido entre dispositivos).

Antes usó `window.storage` (API exclusiva de los artefactos de Claude.ai) y después `localStorage` (por dispositivo). Ambos quedaron reemplazados por Supabase para que los datos se vean iguales desde cualquier dispositivo.

## Decisiones de diseño ya tomadas
- Paleta oscura tipo "cartel de precios de estación de servicio" (asfalto + ámbar), Nafta en rojo, Diesel en verde, Alcohol en violeta.
- Tipografía: JetBrains Mono para precios/números, Manrope para el resto.
- El botón "Guardar emblema" es un `<button type="button">` con click handler manual — se evitó `<form>`/`submit` porque en el entorno de artefactos de Claude.ai el submit nativo no disparaba el evento.
- Los inputs de descuento por día nunca se deshabilitan (`pointer-events`), solo se atenúan visualmente — evita bugs de toque en mobile.
- Ícono de la app: surtidor de combustible en ámbar/rojo/verde sobre fondo casi negro, mismo estilo que la paleta. `icon-192.png` (192x192), `icon-512.png` (512x512), `apple-touch-icon.png` (180x180) y `favicon-32.png` (32x32) son las versiones finales servidas; `index.html` referencia las cuatro (favicon-32 y icon-192 como `rel="icon"` con `sizes`, apple-touch-icon aparte) y `sw.js` las precachea. El cache de service worker está en `v2` porque el contenido de estos PNG cambió sin cambiar de nombre de archivo — si se vuelve a regenerar un ícono manteniendo el mismo filename, bumpear `CACHE_NAME` de nuevo para que no quede la versión vieja cacheada en los dispositivos.

## Pendiente / ideas no implementadas
- Editar el orden o eliminar un emblema desde Inicio (hoy solo desde Comparar).
- Historial de precios por fecha (hoy solo se guarda el precio actual, se sobrescribe).
- Multi-moneda o soporte fuera de Guaraníes.

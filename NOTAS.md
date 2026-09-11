# Comparador de combustible — notas de contexto

App de una sola página (HTML + CSS + JS vanilla, sin build ni frameworks) para cargar precios de combustible por emblema (estación de servicio) y compararlos.

## Estructura de datos
Cada emblema (tabla `emblemas`):
```
{ id, nombre, combustibles: [{ tipo: 'Nafta'|'Diesel'|'Alcohol', detalle, precio }] }
```
Los descuentos viven en su propia tabla relacional `descuentos` (uno-a-muchos con `emblemas`, `on delete cascade`), ya no en una columna jsonb dentro de `emblemas`:
```
{
  id, emblema_id,
  dias: ['Lunes','Martes',...],           // uno o varios días de la semana en la misma fila
  tipo: 'descuento_pct'|'reintegro_pct'|'reintegro_fijo'|'gs_litro',
  valor,
  banco, tipo_tarjeta: ''|'Débito'|'Crédito'|'QR', marca_tarjeta: ''|'Visa'|'Mastercard'|'Amex'|'Cabal'|'Panal'|'Otros',
  requiere_autocarga: bool,
  fecha_limite: date | null
}
```
Un emblema puede tener varios descuentos que se combinan (ej: Enex 20% con tarjeta de crédito + 400 Gs/litro extra si autocargás). Cada descuento se evalúa por separado:
- **Filtro de día**: el día elegido (Comparar) o el día de hoy (Inicio) tiene que estar en `dias`.
- **Filtro de tarjeta**: si el descuento tiene banco/tipo/marca de tarjeta, solo se aplica si esa combinación está marcada como "Tengo esta tarjeta" en Config. Si no requiere tarjeta (banco/tipo/marca todos vacíos), se evalúa siempre, sin pasar por este filtro.
- **Filtro de autocarga**: si `requiere_autocarga` es true, solo se aplica cuando el toggle de autocarga está prendido (Comparar) o nunca en Inicio salvo como nota informativa (ver abajo).

Los ahorros de todos los descuentos que pasan los tres filtros se **suman**: los de tipo % (descuento_pct/reintegro_pct) se calculan sobre el subtotal, `gs_litro` se multiplica por los litros, y `reintegro_fijo` es un monto plano — todo capado para no superar el subtotal.

`fecha_limite` es informativa: si ya pasó, el descuento se sigue aplicando igual (muchas promos se renuevan con el mismo %), pero se muestra un aviso "⚠ Revisá las bases y condiciones..." en Agregar emblema, Comparar e Inicio.

## Pestañas
- **Inicio**: precio más bajo cargado por tipo de combustible (sin descuentos como base), con la línea de descuento de hoy si aplica. Si el usuario configuró alguna tarjeta en Config, arriba aparece una sección "Destacados" automática con los emblemas que tienen algún descuento hoy que coincide con esas tarjetas (no requiere marcar nada a mano, reemplaza a los favoritos viejos). Si todavía no configuró ninguna tarjeta, esa sección no se muestra. Los descuentos que requieren autocarga no se aplican al precio mostrado en Inicio (no hay ese contexto), pero si hay alguno disponible se muestra como nota "+ Gs. X más con autocarga".
- **Agregar**: alta y edición de emblema — nombre, combustibles con precio, y descuentos con selección de días por chips (multi-selección), tipo (%, reintegro %, reintegro fijo, Gs/litro), banco/tipo/marca de tarjeta (opcionales), toggle de autocarga y fecha límite opcional.
- **Comparar**: selector "Cargar por litros" / "Cargar por monto" (invierte el cálculo: litros→Gs o Gs→litros), combustible, día, y toggle "¿Vas a autocargar?" que decide si se suman los descuentos que lo requieren. Resultados ordenados de más conveniente a menos.
- **Historial**: sin cambios, gráfico de precio por emblema+combustible en el tiempo (tabla `precios_historial`).
- **Config** (nueva): lista todas las combinaciones banco+tipo+marca de tarjeta usadas en descuentos de cualquier emblema, con un toggle "Tengo esta tarjeta" por combinación. Es una preferencia **por dispositivo** en `localStorage` (clave `tarjetas-v1`), no se guarda en Supabase. Por defecto todas las tarjetas están **apagadas** (nada configurado) — se decidió así, y no todas prendidas, para que mientras el usuario no visitó esta pestaña la app no asuma que tiene tarjetas que en realidad no tiene (esto es lo que hace que en Inicio no aparezca la sección "Destacados" hasta que se configure al menos una).

## Cálculo Comparar en modo "Cargar por monto" (Gs → litros)
Se invierte algebraicamente la misma fórmula del modo litros→Gs: dado un monto neto deseado, se resuelve el subtotal antes de descuentos y de ahí los litros:
```
subtotal = (monto + reintegroFijoTotal) / (1 - sumaFraccionesPct - gsLitroTotal/precioLitro)
litros = subtotal / precioLitro
```
Esto asume que el monto ingresado es lo que la persona quiere terminar pagando en total, ya neto de todos los descuentos aplicables.

## Persistencia
Supabase, proyecto `aylhlynzglgjtplujkqv` (región sa-east-1). Tablas: `emblemas` (id, nombre, combustibles jsonb, created_at, updated_at), `descuentos` (ver arriba, FK a emblemas con `on delete cascade`), `precios_historial` (sin cambios). RLS activo con políticas públicas de lectura/escritura en las tres tablas (sin login, cualquiera con la URL ve y edita los mismos datos). El cliente JS de Supabase se carga vía CDN (`@supabase/supabase-js` UMD desde jsdelivr) con la Project URL y la publishable/anon key hardcodeadas en `index.html` — es lo esperado, esa key está pensada para ser pública porque el acceso real lo controla RLS.

Al guardar un emblema, los descuentos se sincronizan con un reemplazo completo (`delete` de todos los descuentos de ese `emblema_id` + `insert` de la lista nueva) en vez de diffing fila por fila — más simple y suficiente para la escala de esta app.

La migración `crear_tabla_descuentos_relacional` migró los datos viejos (un descuento jsonb por día dentro de `emblemas.descuentos`) agrupando por emblema+tipo+valor+tarjeta (así los días con la misma promo quedaron en una sola fila con varios días), seteó `tipo_tarjeta='Crédito'` por defecto en los que tenían tarjeta cargada (heurística razonable dado que los textos viejos eran del estilo "... TC") y dejó `marca_tarjeta` vacía para que se complete a mano después. Terminada la migración, se borró la columna jsonb vieja de `emblemas`.

## Decisiones de diseño ya tomadas
- Paleta oscura tipo "cartel de precios de estación de servicio" (asfalto + ámbar), Nafta en rojo, Diesel en verde, Alcohol en violeta.
- Tipografía: JetBrains Mono para precios/números, Manrope para el resto.
- El botón "Guardar emblema" es un `<button type="button">` con click handler manual — se evitó `<form>`/`submit` porque en el entorno de artefactos de Claude.ai el submit nativo no disparaba el evento.
- Los chips de día y los toggles (switch) nunca se deshabilitan (`pointer-events`), solo se atenúan visualmente — evita bugs de toque en mobile.
- Se sacó el sistema de favoritos manuales (estrella) del todo: quedó reemplazado por la combinación Config (tarjetas que tenés) + destacados automáticos en Inicio.
- Ícono de la app: surtidor de combustible en ámbar/rojo/verde sobre fondo casi negro, mismo estilo que la paleta. `icon-192.png`, `icon-512.png`, `apple-touch-icon.png` y `favicon-32.png` son las versiones finales servidas y precacheadas por `sw.js` (`CACHE_NAME` en `v2`; si se regenera un ícono manteniendo el mismo filename, bumpear `CACHE_NAME` de nuevo).

## Pendiente / ideas no implementadas
- Completar `marca_tarjeta` de los 4 descuentos migrados desde la estructura vieja (quedaron con marca vacía, ver arriba).
- Multi-moneda o soporte fuera de Guaraníes.

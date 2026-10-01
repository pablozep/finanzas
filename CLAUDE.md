# Finanzas — app personal de gastos

App web de finanzas personales de uso individual. Registra ingresos, egresos y
transferencias desde el celular, los guarda en una hoja de Google que sigue
siendo auditable a mano, y calcula cuánto debería haber en cada cuenta para
conciliar contra los bancos.

## Arquitectura

```
index.html  ──HTTP──>  Apps Script (Web App)  ──>  Google Sheets
  (PWA)                   backend                 un archivo por año
```

- **Front-end**: un único `index.html` con HTML, CSS y JS embebidos. Sin
  frameworks, sin compilación, sin `node_modules`. GitHub Pages lo sirve tal cual.
- **Back-end**: Google Apps Script publicado como Web App. El código vive en
  Google (proyecto "Finanzas Personales Script"), no en este repositorio.
- **Base de datos**: Google Sheets, un archivo por año (`Finanzas_2026`, ...).

| Archivo | Para qué |
|---|---|
| `index.html` | La app completa |
| `manifest.json` | Metadatos de PWA |
| `apple-touch-icon.png` | Ícono de iOS (180×180, sin esquinas redondeadas) |
| `icono-192.png`, `icono-512.png` | Íconos del manifiesto |

## La hoja de cálculo

**Movimientos** (columnas por posición; no reordenar):

| A | B | C | D | E | F | G | H | I | J | K |
|---|---|---|---|---|---|---|---|---|---|---|
| ID | Fecha | Tipo | Categoría | Monto | Descripción | Medio (histórico) | Origen | Marca de tiempo | Cuenta | Cuenta destino |

- `Tipo`: `Ingreso`, `Egreso` o `Transferencia`.
- `Monto` siempre positivo. El signo lo determina el tipo.
- `Cuenta`: en un Ingreso, donde llega; en Egreso o Transferencia, de donde sale.
- `Cuenta destino`: en una Transferencia, a dónde va. En un Egreso de categoría
  del grupo Ahorro, la cuenta de inversión que recibe (ej. Racional).
- Las Transferencias no tienen categoría y no cuentan como ingreso ni egreso.

**Cuentas**: `Cuenta | Tipo | Saldo inicial | Fecha del saldo | Emoji | En disponible | Activa | Saldo esperado`.
La columna H calcula con fórmulas la misma regla que el backend, para auditar a mano.
`En disponible = NO` saca la cuenta del total principal (Racional, inversiones).

También: `Categorias`, `Presupuesto`, `Config`, `Resumen_Mensual`, `Resumen_Anual`.

## Regla de saldos

Saldo esperado = saldo inicial + movimientos con `fecha > fecha del saldo`.

El saldo inicial es el saldo **al cierre** de su fecha: los movimientos de ese
mismo día ya están incluidos en el número anotado y no se vuelven a contar.
(Antes la regla era `>=`, "saldo al inicio del día", y producía doble conteo
cuando el usuario anotaba el saldo y la fecha de hoy, que es lo natural.)

- **Ingreso**: suma a `Cuenta`.
- **Egreso**: resta de `Cuenta`; si tiene destino, suma al destino.
- **Transferencia**: resta de `Cuenta`, suma a `Cuenta destino`.

Esta regla está implementada en **tres lugares que deben coincidir siempre**:
`calcularSaldos()` en el script, `efectoEnCuentas()` en `index.html`, y las
fórmulas de la columna H de la hoja Cuentas. Si se cambia en uno, se cambia en
los tres.

## Restricciones del proyecto

1. **Un solo archivo.** Todo el front-end vive en `index.html`. No separar en
   `.css` ni `.js`, no agregar dependencias npm, no introducir un paso de build.
2. **Sin frameworks.** JavaScript plano, ES5+ compatible con Safari de iOS
   (sin funciones flecha en el código de la app).
3. **Sin secretos en el código.** El repositorio es público. La clave la
   escribe el usuario una vez y queda en `localStorage`.
4. **Español de Chile** en toda la interfaz. Pesos sin decimales con `fmt()`.

## Invariantes de la capa de red

Cada una resuelve un problema que ya costó depurar. **No modificar sin
entender por qué está.**

- **`Content-Type: text/plain` en los POST.** Apps Script no responde a
  `OPTIONS`; con `application/json` el preflight CORS rompe la petición.
- **`&_=Date.now()` y `cache: 'no-store'` en los GET.** Sin esto Safari sirve
  respuestas viejas.
- **`normalizarUrl()` limpia `/u/1/`** de la dirección del script. Ese segmento
  aparece al copiar la URL con varias cuentas de Google abiertas, convierte la
  ruta en un archivo de Drive, y Safari la rechaza con "The string did not
  match the expected pattern".
- **Corte de 30 s con `AbortController`** en los POST.

## Cola de envío (guardado optimista)

Crear y eliminar **no esperan al servidor**. El movimiento entra a
`localStorage` (`fin.cola`), se muestra al instante con la etiqueta
"enviando", y `procesarCola()` lo manda en segundo plano, de a uno.

- Cada ítem lleva un **`uid`**. El servidor lo recuerda 6 h en `CacheService`:
  si llega dos veces, no duplica. Eliminar un id que ya no existe devuelve ok.
  Sin esto, los reintentos tras un corte duplicaban o atascaban movimientos.
- `aplicarCola()` reconstruye `resumen` a partir de `resumenServidor` más los
  pendientes. Los totales del mes solo usan pendientes de ese mes; **los saldos
  de cuentas usan todos los pendientes**, porque son globales.
- Los POST piden `conResumen: true` y reciben el resumen actualizado en la misma
  respuesta, para no hacer una segunda petición.
- Reintentos: al abrir la app, al tocar ↻, al volver a primer plano, al
  recuperar conexión y cada 20 s mientras haya pendientes.
- Tipos de ítem: alta (sin `op`), `op: 'eliminar'` y `op: 'asignar'` (pone la
  cuenta a un movimiento que no la tenía; viaja como `editar`). `movDeItem()`
  convierte cualquiera a forma de movimiento para saldos y listas.
- Errores de datos (categoría, cuenta, monto, token) marcan el ítem como
  `bloqueado`: no se reintenta y la banda ofrece descartarlo.

## Contrato del backend

Todas las peticiones llevan `token`. Respuestas: `{ok: true, datos}` o
`{ok: false, error}`.

- `GET ?action=inicio&anio=&mes=` → `{catalogo, resumen}`. Lo usa el arranque.
- `GET ?action=anual&anio=` → los 12 meses, con `conDatos` por mes.
- `POST {action:'crear', uid, tipo, categoria, monto, fecha, descripcion, cuenta, cuentaDestino, conResumen, resumenAnio, resumenMes}`
- `POST {action:'eliminar', uid, id, anio, conResumen, resumenAnio, resumenMes}`
- `POST {action:'editar', uid, id, anio, cuenta, conResumen, resumenAnio, resumenMes}`

`catalogo.cuentas`: `[{cuenta, tipo, emoji, disponible}]` (para el selector).

`resumen.cuentas`:
```jsonc
{
  "lista": [{ "cuenta": "Banco Estado", "tipo": "Banco", "emoji": "🏦",
              "disponible": true, "saldoInicial": 500000, "fechaSaldo": "2026-10-02",
              "configurada": true, "saldo": 325000, "entradas": 0, "salidas": 0,
              "ultimos": [{ "id", "fecha", "tipo", "categoria", "monto",
                            "signo": -1, "descripcion", "contraparte" }] }],
  "totalDisponible": 395000, "totalInvertido": 1100000,
  "configurado": true, "desde": "2026-10-02",
  "sinCuenta": { "n": 0, "monto": 0, "lista": [{ "id", "fecha", "anio", "mes",
                 "tipo", "categoria", "monto", "descripcion" }] }
}
```

`resumen.movimientos` incluye las transferencias; `resumen.categorias`,
`grupos`, `ingresos` y `egresos` no.

## Rotación anual

`crearArchivoDelAnio()` duplica el archivo, vacía Movimientos y **escribe los
saldos de cierre como saldo inicial con fecha 31 de diciembre** en la hoja
Cuentas del año nuevo. Sin eso, en enero todas las cuentas volverían a cero.

## Lenguaje visual

- Acento coral `#F2854B`, fondo `#FBFAF8`, texto `#242A38`. Tipografía Nunito.
- Tintes por grupo en el objeto `TINTES`. Transferencias en gris azulado (⇄).
- Elemento distintivo: las barras verticales de la vista Simple (relleno = gasto,
  franja clara = presupuesto).
- Número grande: **disponible en cuentas** en el mes en curso cuando las cuentas
  están configuradas; saldo del mes en meses pasados o sin configurar.
- Avanzada: tarjetas de cuentas (tocar abre la conciliación), desplegable de
  movimientos sin cuenta (tocar uno permite asignarle cuenta), tendencia anual,
  anillo por grupo, avance por categoría, movimientos del mes.

## Cómo probar

Abrir `index.html` en Safari: se conecta al backend real. Verificar después de
cada cambio: arranque desde caché, guardar un gasto, una transferencia y un
ahorro con destino, eliminar, navegar entre meses, y conciliar una cuenta.

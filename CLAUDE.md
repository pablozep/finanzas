# Finanzas — app personal de gastos

App web de finanzas personales de uso individual. Registra ingresos y egresos
desde el celular y los guarda en una hoja de Google, que sigue siendo auditable
a mano.

## Arquitectura

```
index.html  ──HTTP──>  Apps Script (Web App)  ──>  Google Sheets
  (PWA)                   backend                 un archivo por año
```

- **Front-end**: un único `index.html` con HTML, CSS y JS embebidos. Sin
  frameworks, sin compilación, sin `node_modules`. GitHub Pages lo sirve tal cual.
- **Back-end**: Google Apps Script publicado como Web App. El código vive en
  Google, no en este repositorio.
- **Base de datos**: Google Sheets. Una pestaña `Movimientos` como registro
  continuo, más `Categorias`, `Presupuesto`, `Config` y dos hojas de resumen
  con fórmulas.

Archivos del repo:

| Archivo | Para qué |
|---|---|
| `index.html` | La app completa |
| `manifest.json` | Metadatos de PWA |
| `apple-touch-icon.png` | Ícono de iOS (180×180, sin esquinas redondeadas) |
| `icono-192.png`, `icono-512.png` | Íconos del manifiesto |

## Restricciones del proyecto

Estas no son preferencias de estilo, son decisiones tomadas:

1. **Un solo archivo.** Todo el front-end vive en `index.html`. No separar en
   `.css` ni `.js`, no agregar dependencias npm, no introducir un paso de build.
   La app tiene que poder abrirse con doble clic y funcionar.
2. **Sin frameworks.** Nada de React, Vue ni Svelte. JavaScript plano, ES5+
   compatible con Safari de iOS.
3. **Sin secretos en el código.** El repositorio es público. La clave de acceso
   al backend la escribe el usuario una vez y queda en `localStorage`. Nunca
   escribirla en el archivo.
4. **Español de Chile** en toda la interfaz. Montos en pesos chilenos, sin
   decimales, con `Intl.NumberFormat('es-CL')`.

## Invariantes de la capa de red

Cada una de estas líneas resuelve un problema concreto que ya costó depurar.
**No modificarlas sin entender por qué están.**

- **`Content-Type: text/plain` en los POST.** Apps Script no responde a
  peticiones `OPTIONS`. Con `application/json` el navegador dispara un preflight
  CORS y la petición falla entera.
- **`&_=Date.now()` y `cache: 'no-store'` en los GET.** Apps Script no manda
  cabeceras anti-caché. Sin esto, Safari sirve la respuesta anterior y la
  pantalla se queda congelada con datos viejos.
- **El año viaja en `eliminar` y `editar`.** Cada año vive en un archivo
  distinto; sin el año, el backend borraría en el archivo equivocado.
- **La caché de `localStorage` se invalida al guardar y al eliminar.** Si se
  omite, el usuario ve el movimiento en la hoja pero no en la app.
- **Los montos se guardan siempre positivos.** El signo lo determina la columna
  `Tipo` (`Ingreso` / `Egreso`).

## Contrato del backend

Todas las peticiones llevan `token`. Las respuestas tienen la forma
`{ok: true, datos: ...}` o `{ok: false, error: "..."}`.

### `GET ?action=inicio&anio=&mes=`
Catálogo y resumen del mes en una sola llamada. Es lo que usa el arranque.

```jsonc
{
  "catalogo": {
    "anio": 2026,
    "moneda": "CLP",
    "archivo": "Finanzas_2026",
    "aniosDisponibles": ["2026"],
    "categorias": [
      { "categoria": "Supermercado", "tipo": "Egreso", "grupo": "Gastos", "emoji": "🛒" }
    ],
    "medios": ["Efectivo", "Debito", "..."]
  },
  "resumen": {
    "anio": 2026, "mes": 8,
    "ingresos": { "real": 200000, "ppto": 0, "diferencia": 200000 },
    "egresos":  { "real": 5890,   "ppto": 0, "diferencia": 5890 },
    "saldo":    { "real": 194110, "ppto": 0 },
    "tasaAhorro": 0.97,
    "sinCategorizar": 0,
    "grupos": [
      { "grupo": "Gastos", "real": 5890, "ppto": 0, "diferencia": 5890, "porcentaje": 1 }
    ],
    "categorias": [
      { "categoria": "Supermercado", "tipo": "Egreso", "grupo": "Gastos",
        "emoji": "🛒", "real": 5890, "ppto": 0, "diferencia": 5890, "avance": null }
    ],
    "movimientos": [
      { "id": "MOV-0003", "fecha": "2026-08-28", "anio": 2026, "mes": 8,
        "tipo": "Egreso", "categoria": "Supermercado", "monto": 5890,
        "descripcion": "Lider", "medio": "Debito", "fila": 4 }
    ]
  }
}
```

### `GET ?action=anual&anio=`
Los 12 meses del año. Pensado para gráficos de tendencia.

```jsonc
{
  "anio": 2026,
  "meses": [
    { "mes": 8, "ingresos": 200000, "egresos": 5890, "saldo": 194110,
      "acumulado": 194110, "tasaAhorro": 0.97, "conDatos": true }
    // ... los 12, en orden
  ],
  "totales": {
    "ingresos": 380000, "egresos": 225890, "saldo": 154110,
    "tasaAhorro": 0.40, "promedioEgresos": 112945,
    "mesesConDatos": 2, "mejorMes": 8, "peorMes": 9
  }
}
```

`conDatos: false` marca los meses sin movimientos: los que aún no llegan y los
que pasaron vacíos. **Un gráfico no debe dibujar esos meses como saldo cero**,
porque aplanaría la línea hasta diciembre.

### `POST` (cuerpo JSON)
- `{action: "crear", tipo, categoria, monto, fecha, descripcion, medio, origen}`
- `{action: "eliminar", id, anio}`
- `{action: "editar", id, anio, ...campos}`

## Lenguaje visual

- **Acento**: coral `#F2854B`. **Fondo**: `#FBFAF8`. **Texto**: `#242A38`.
- **Tipografía**: Nunito (400/600/800). Números grandes y gruesos.
- **Tintes por grupo**, usados de forma consistente en barras, anillo y listas:
  Servicios amarillo, Gastos rosa, Deudas violeta, Ahorro verde, ingresos azul
  y oliva. Están en el objeto `TINTES`.
- **Elemento distintivo**: las cinco barras verticales de la vista Simple. El
  relleno sólido es el gasto real; la franja más clara detrás es el presupuesto.
  Esa relación es la lectura principal de la app y no debe perderse.
- **Dos vistas**: Simple (saldo, barras, últimos movimientos) y Avanzada
  (anillo por grupo, avance por categoría, todos los movimientos).

## Cómo probar

Abrir `index.html` directamente en Safari. Se conecta al backend real, así que
se ve con datos verdaderos. La clave ya queda guardada entre recargas.

Verificar siempre después de un cambio:
1. El arranque pinta desde caché y luego actualiza (el ↻ gira mientras tanto).
2. Guardar un movimiento lo refleja en la pantalla sin recargar.
3. Navegar entre meses con las flechas funciona en ambas direcciones.
4. Eliminar un movimiento lo quita de la lista.

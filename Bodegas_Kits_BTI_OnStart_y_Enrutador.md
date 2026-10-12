# Bodegas & Kits BTI — `App.OnStart` corregido y el enrutador
### Dos errores, dos causas distintas. Aquí están los dos arreglos.

---

# PARTE 1 — Por qué falló `ThisItem` y dónde va cada botón

## 1.1 La causa

`ThisItem` **solo existe dentro de la plantilla de una galería**. Fuera de ahí Power Apps no sabe a qué fila se refiere, y da exactamente ese error: *«El nombre no es válido. No se reconoce ThisItem»*.

Entonces el problema no es la fórmula, es **dónde la pegó**. Y la culpa es mía: le di el enrutador diciendo «péguelo en `btnNavItem` y en `btnTile`» sin aclarar que esos dos botones viven dentro de galerías que todavía no ha construido.

## 1.2 Dónde vive cada botón — ubicación exacta

| Botón | Vive dentro de | Que a su vez vive en | ¿`ThisItem` funciona? | ¿Ya lo construyó? |
|---|---|---|---|---|
| `btnNavItem` | plantilla de **`galNav`** | `cntNav` | **Sí** | **Sí** (C.5) |
| `btnTile` | plantilla de **`galBotones`** | `cntVMenu` | **Sí** | **No todavía** (es J.1) |
| `btnAbrirArea_Seg` | plantilla de **`galAreas`** | `cntVSeguimiento` | **Sí** | No todavía (J.3) |
| `btnRouter` | **hijo directo de `scrPrincipal`** | — | **No** | nuevo, lo creamos ahora |

```
scrPrincipal
├─ cntCuerpo
│   ├─ cntBarra
│   ├─ cntVMenu                    ← J.1, aún no construido
│   │   ├─ galKpi
│   │   └─ galBotones
│   │       └─ [PLANTILLA]
│   │           ├─ cntBoton
│   │           ├─ lblBtnCta
│   │           └─ btnTile  ←───── AQUÍ va el OnSelect de btnTile
│   └─ … las otras 8 vistas
├─ recScrimNav
├─ cntNav                          ← ya construido (C.4)
│   ├─ cntNavMarca
│   ├─ galNav
│   │   └─ [PLANTILLA]
│   │       ├─ recNavItem
│   │       ├─ icoNavItem
│   │       ├─ lblNavItem
│   │       ├─ lblNavCta
│   │       └─ btnNavItem  ←────── AQUÍ sí funciona ThisItem (ya lo tiene)
│   └─ lblNavUser
├─ … los velos y los popups
├─ tmrVencidos
└─ btnRouter  ←──────────────────── NUEVO · hijo directo de la pantalla · Visible = false
```

## 1.3 El rediseño: el enrutador **ya no usa `ThisItem`**

En lugar de repetir 60 líneas en dos botones, el enrutador vive **una sola vez** en un botón oculto y se dispara con `Select()`. Los botones de las galerías quedan en dos líneas.

> **Sobre `Select()`:** antes le dije «cero `Select()`». Preciso la regla, porque la diferencia importa. El bug del doble clic en `procesos01` fue `Select(btn);; algoQueDependeDeEso` — el `Select` es **asíncrono**, así que lo que sigue corre antes de que el botón termine. Aquí `Select(btnRouter)` es **la última instrucción** y nada depende de su resultado, así que es seguro. Es el mismo patrón de `btnEnviarCorreoEstatus` que ya le funciona en `procesos01`.

### Otro error mío que esto arregla

En `btnNavItem` le había puesto `If(Parent.Width < nfCorteMovil; …)`. **Está mal:** dentro de la plantilla de `galNav`, `Parent` es **la galería**, no la pantalla. `galNav.Width` mide unos 196 px, así que esa condición siempre daba `true`. Ahora esa línea vive en `btnRouter`, que sí es hijo de la pantalla, y usa el nombre de la pantalla para no depender de `Parent`.

---

# PARTE 2 — `btnRouter` — Botón clásico oculto

## Ubicación
Hijo directo de **`scrPrincipal`**, al final de la lista de controles de la pantalla.

## Propiedades

| Propiedad | Valor |
|---|---|
| `X` | `-200` |
| `Y` | `-200` |
| `Width` | `80` |
| `Height` | `40` |
| `Text` | `"router"` |
| `Visible` | `false` |
| `DisplayMode` | `DisplayMode.Edit` ← **no lo deje en Disabled o `Select()` no lo alcanza** |
| `TabIndex` | `-1` |

## `btnRouter` → `OnSelect`

```powerapps
/* ============================================================
   ENRUTADOR ÚNICO
   Se dispara DESPUÉS de que varPantalla ya cambió de valor.
   No usa ThisItem: lee varPantalla.
   ============================================================ */

/* ---------- 0. común a todas las vistas ---------- */
Set(varPopup; "");;
If(scrPrincipal.Width < nfCorteMovil; Set(varNavAbierto; false));;

/* ---------- 1. ASIGNAR: estado inicial + kit completo precargado ---------- */
If(varPantalla = "ASIG"; Set(varModoCarga; "KIT"));;
If(varPantalla = "ASIG"; Set(varTipo; "RETORNABLE"));;
If(varPantalla = "ASIG"; Set(varBodegaFiltro; "TODAS"));;
If(varPantalla = "ASIG"; Set(varBuscar; ""));;
If(varPantalla = "ASIG"; Set(varArea; LookUp(BG_Areas; false)));;
If(varPantalla = "ASIG"; Set(varEncargado; LookUp(CATALOGO_PERSONAL1; false)));;
If(varPantalla = "ASIG"; Set(varRecibe; LookUp(CATALOGO_PERSONAL1; false)));;
If(varPantalla = "ASIG"; Clear(colCatSel));;
If(
    varPantalla = "ASIG";
    ClearCollect(
        colSel;
        ForAll(
            Filter(BG_KitDetalle; Title = varKit.Title) As d;
            With(
                {a: LookUp(nfCatalogo; Clave = d.Clave)};
                Patch(
                    nfLineaVacia;
                    {
                        Clave:       a.Clave;
                        Codigo:      a.Codigo;
                        Descripcion: a.Descripcion;
                        Tipo:        a.Tipo.Value;
                        Categoria:   a.Categoria.Value;
                        Bodega:      a.Bodega.Value;
                        Unidad:      a.Unidad;
                        Existencia:  a.Existencia;
                        Estandar:    d.CantidadEstandar;
                        Cantidad:    Min(d.CantidadEstandar; a.Existencia)
                    }
                )
            )
        )
    )
);;
If(varPantalla = "ASIG"; RemoveIf(colSel; Existencia <= 0));;

/* ---------- 2. SEGUIMIENTO: resumen por área ---------- */
If(varPantalla = "SEG"; ClearCollect(colDetAb; nfDetAbierto));;
If(
    varPantalla = "SEG";
    ClearCollect(
        colResumenArea;
        ForAll(
            GroupBy(colDetAb; 'CodigoArea'; 'Lineas') As g;
            With(
                {a: LookUp(nfAreas; Title = g.CodigoArea)};
                {
                    CodigoArea:  g.CodigoArea;
                    NombreArea:  Coalesce(a.NombreArea; g.CodigoArea);
                    Cultivo:     Coalesce(a.Cultivo.Value; "—");
                    Responsable: Coalesce(a.ResponsableArea.DisplayName; "—");
                    Encargado:   Coalesce(a.EncargadoArea; "—");
                    Renglones:   CountRows(g.Lineas);
                    Afuera:      Sum(g.Lineas; CantEntregada - CantBuena - CantDanada - CantExtraviada);
                    Danadas:     Sum(g.Lineas; CantDanada);
                    Perdidas:    Sum(g.Lineas; CantExtraviada);
                    Abiertas:    CountRows(Distinct(g.Lineas; NoSalida))
                }
            )
        )
    )
);;

/* ---------- 3. DEVOLUCIÓN: limpia la captura anterior ---------- */
If(varPantalla = "DEV"; Set(varAreaDev; LookUp(BG_Areas; false)));;
If(varPantalla = "DEV"; Set(varSalDev; LookUp(BG_Salidas; false)));;
If(varPantalla = "DEV"; Clear(colDev));;

/* ---------- 4. INGRESOS: pendientes de Bodega General ---------- */
If(
    varPantalla = "ING";
    ClearCollect(
        colPorIngresar;
        ForAll(
            nfPorIngresar As r;
            {
                IdGeneral:     r.ID;
                Codigo:        r.Codigo;
                Descripcion:   r.Descripcion;
                Cantidad:      r.Cantidad;
                Unidad:        r.Unidad;
                AreaUso:       r.AreaUso;
                Responsable:   r.Responsable;
                Tipo:          Coalesce(LookUp(BG_Catalogo; Codigo = r.Codigo; Tipo.Value); "UN SOLO USO");
                BodegaDestino: Coalesce(LookUp(BG_Catalogo; Codigo = r.Codigo; Bodega.Value); "BODEGA 1");
                Categoria:     Coalesce(LookUp(BG_Catalogo; Codigo = r.Codigo; Categoria.Value); "INSUMO");
                EsNuevo:       IsBlank(LookUp(BG_Catalogo; Codigo = r.Codigo; Codigo))
            }
        )
    )
);;

/* ---------- 5. CATÁLOGO: filtros en limpio + lo que está afuera ---------- */
If(varPantalla = "CAT"; Set(varTipoCat; ""));;
If(varPantalla = "CAT"; Set(varBodCat; "TODAS"));;
If(varPantalla = "CAT"; Set(varBuscarCat; ""));;
If(varPantalla = "CAT"; ClearCollect(colDetAb; nfDetAbierto));;

/* ---------- 6. BAJAS ---------- */
If(
    varPantalla = "BAJA";
    ClearCollect(colBajas; Sort(BG_Bajas; FechaBaja; SortOrder.Descending))
)
```

---

# PARTE 3 — Los botones que llaman al enrutador

## 3.1 `btnNavItem` → `OnSelect`
**Ubicación:** plantilla de `galNav`, dentro de `cntNav`. Ya lo tiene construido.

```powerapps
Set(varPantalla; ThisItem.Pant);;
Select(btnRouter)
```

## 3.2 `btnTile` → `OnSelect`
**Ubicación:** plantilla de `galBotones`, dentro de `cntVMenu`. **Todavía no existe** — se crea en J.1.

```powerapps
Set(varPantalla; ThisItem.Pant);;
Select(btnRouter)
```

> Es la **misma fórmula** porque tanto `nfMenu` como `nfBotonesMenu` traen el campo `Pant`.

## 3.3 `icoAtras` → `OnSelect`
**Ubicación:** dentro de `cntBarra`. Ya lo tiene. **No** usa `ThisItem` ni necesita el enrutador.

```powerapps
Set(varPantalla; If(varPantalla = "SEGDET"; "SEG"; "MENU"))
```

## 3.4 Desde cualquier otro botón (sin galería)
Si más adelante quiere un botón que lleve a una vista, el patrón es siempre el mismo:

```powerapps
Set(varPantalla; "CAT");;
Select(btnRouter)
```

## 3.5 `btnAbrirArea_Seg` → `OnSelect` (J.3, no usa el enrutador)
Hace su propia carga porque necesita el área de **esa fila**.

```powerapps
Set(varAreaVista; LookUp(nfAreas; Title = ThisItem.CodigoArea));;
ClearCollect(
    colSalArea;
    Sort(Filter(BG_Salidas; CodigoArea = ThisItem.CodigoArea); FechaEntrega; SortOrder.Descending)
);;
Set(varSalVista; First(colSalArea));;
ClearCollect(colDetSal; Filter(BG_SalidaDetalle; NoSalida = First(colSalArea).NoSalida));;
Set(varPantalla; "SEGDET")
```

---

# PARTE 4 — Por qué `Blank()` y `Clear()` daban error

## 4.1 `Set(var; Blank())` → *«No se ha encontrado ningún tipo para la variable»*

Power Fx deduce el tipo de una variable del **valor** que le asigna. `Blank()` no tiene tipo, así que la variable queda sin tipo y cualquier `varAreaVista.Title` falla.

**Lo que usted hizo** — `Set(varSalVista; "")` — sí quita el error, pero **crea uno peor**: tipa la variable como **texto**, y entonces `varSalVista.NoSalida` nunca va a funcionar. El título de la barra y la galería de renglones van a fallar más adelante.

**La solución:** `LookUp(lista; false)` devuelve un registro **en blanco pero con el tipo de la lista**.

```powerapps
Set(varAreaVista; LookUp(BG_Areas; false))
```

- `IsBlank(varAreaVista)` sigue devolviendo `true` → todas las validaciones siguen sirviendo.
- `varAreaVista.NombreArea` ya compila → las etiquetas funcionan.

| Variable | Semilla correcta |
|---|---|
| `varArea` · `varAreaVista` · `varAreaDev` | `LookUp(BG_Areas; false)` |
| `varEncargado` · `varRecibe` | `LookUp(CATALOGO_PERSONAL1; false)` |
| `varSalVista` · `varSalDev` · `varSalida` | `LookUp(BG_Salidas; false)` |
| `varIng` | `LookUp(BG_Ingresos; false)` |
| `varKit` | `First(Filter(BG_Kits; Activo = true))` |

> Si el editor le marca `false` como condición, use la variante equivalente: `First(Filter(BG_Areas; ID < 0))`. Hace lo mismo, delega y nunca devuelve filas.

## 4.2 `Clear(colSel)` → *«No se reconoce colSel»*

Una colección **solo existe** si en algún lugar de la app hay un `Collect` o `ClearCollect` que la cree. `Clear()` sola no la crea, así que el nombre no existe todavía — justo porque los controles que la llenan (`btnKitTodo`, el enrutador) aún no están construidos.

**La solución:** créela con su forma y vacíela enseguida. Dos líneas, y ya queda definida con el esquema correcto para toda la app.

```powerapps
ClearCollect(colSel; nfLineaVacia);;
Clear(colSel);;
```

| Colección | Semilla |
|---|---|
| `colSel` | `ClearCollect(colSel; nfLineaVacia);; Clear(colSel);;` |
| `colDev` | `ClearCollect(colDev; nfDevVacia);; Clear(colDev);;` |
| `colCatSel` | `ClearCollect(colCatSel; {Cat: ""});; Clear(colCatSel);;` |

Las demás (`colDetAb`, `colResumenArea`, `colSalArea`, `colDetSal`, `colPorIngresar`, `colBajas`) **no se siembran**: el enrutador ya las crea con `ClearCollect`, y eso basta para que el nombre exista en tiempo de diseño.

---

# PARTE 5 — `App.OnStart` completo y corregido

Reemplace todo el contenido de `App.OnStart` por esto. Ya no hay nada comentado ni nada que dé error.

```powerapps
/* ============================================================
   App.OnStart  ·  Bodegas & Kits BTI
   Solo estado de interfaz y tipado de variables.
   Ningún dato maestro: eso lo dan las fórmulas con nombre
   (nfCatalogo, nfAreas, nfDetAbierto, nfSalAbiertas…).
   ============================================================ */


/* ------------------------------------------------------------
   1. VARIABLES DE TEXTO
   ------------------------------------------------------------ */
Set(varPantalla;     "MENU");;      /* MENU ASIG SEG SEGDET DEV ING CAT BAJA FLUJO */
Set(varPopup;        "");;          /* "" CARRITO AREA KIT                        */
Set(varModoCarga;    "KIT");;       /* KIT | UNIDAD                               */
Set(varTipo;         "RETORNABLE");;
Set(varBodegaFiltro; "TODAS");;
Set(varBuscar;       "");;
Set(varTipoCat;      "");;
Set(varBodCat;       "TODAS");;
Set(varBuscarCat;    "");;
Set(varNoSalida;     "");;


/* ------------------------------------------------------------
   2. VARIABLES BOOLEANAS
   ------------------------------------------------------------ */
Set(varNavAbierto; false);;
Set(varGuardando;  false);;


/* ------------------------------------------------------------
   3. VARIABLES DE REGISTRO
   Power Fx NO acepta Set(var; Blank()): la variable queda sin
   tipo. LookUp(lista; false) devuelve un registro EN BLANCO
   pero CON EL TIPO de la lista, y IsBlank() sigue dando true.
   ------------------------------------------------------------ */
Set(varArea;      LookUp(BG_Areas; false));;
Set(varAreaVista; LookUp(BG_Areas; false));;
Set(varAreaDev;   LookUp(BG_Areas; false));;

Set(varEncargado; LookUp(CATALOGO_PERSONAL1; false));;
Set(varRecibe;    LookUp(CATALOGO_PERSONAL1; false));;

Set(varSalVista;  LookUp(BG_Salidas; false));;
Set(varSalDev;    LookUp(BG_Salidas; false));;
Set(varSalida;    LookUp(BG_Salidas; false));;

Set(varIng;       LookUp(BG_Ingresos; false));;

Set(varKit;       First(Filter(BG_Kits; Activo = true)));;


/* ------------------------------------------------------------
   4. COLECCIONES
   Clear() sola no crea la colección y da "no se reconoce".
   Se crean con su forma y se vacían enseguida: así quedan
   definidas con el esquema correcto desde el arranque.
   ------------------------------------------------------------ */
ClearCollect(colSel; nfLineaVacia);;
Clear(colSel);;

ClearCollect(colDev; nfDevVacia);;
Clear(colDev);;

ClearCollect(colCatSel; {Cat: ""});;
Clear(colCatSel)
```

### Otras propiedades de `App`

| Propiedad | Valor |
|---|---|
| `StartScreen` | `scrPrincipal` |
| `SizeBreakpoints` | `[600; 760; 1024; 1366]` |
| `OnError` | `Notify("Ocurrió un error: " & FirstError.Message; NotificationType.Error; 4000)` |

> `varNoSalida` e `varIng` se usan al guardar (`btnPopGuardar`, `btnIngresar`). Se siembran aquí para que esas fórmulas compilen antes de que se ejecuten por primera vez.

---

# PARTE 6 — Orden para dejarlo sin errores hoy mismo

| Paso | Qué hacer | Resultado |
|---|---|---|
| 1 | Pegue el `OnStart` de la Parte 5 completo | desaparecen los errores de `Blank()` y de `Clear()` |
| 2 | Inserte `btnRouter` como hijo directo de `scrPrincipal` con las propiedades de la Parte 2 | |
| 3 | Pegue el enrutador en `btnRouter.OnSelect` | **va a marcar error en las líneas de `colResumenArea`, `colPorIngresar` y `colBajas`** si esas colecciones no se usan en ningún control todavía. Es normal: el error se va cuando construya `cntVSeguimiento`, `cntVIngresos` y `cntVBajas`. Si le molesta, comente esos tres bloques y descoméntelos al llegar a J.3, J.6 y J.8 |
| 4 | Cambie `btnNavItem.OnSelect` por las dos líneas de 3.1 | ya puede tocar el menú y cambiar `varPantalla` |
| 5 | Cambie `icoAtras.OnSelect` por la línea de 3.3 | |
| 6 | Verifique con `lblTitulo` que cambia el texto al tocar cada entrada del menú | el menú ya enruta, aunque las vistas estén vacías |
| 7 | Siga con J.0 y J.1 (los contenedores de vista y `cntVMenu`) | ahí nace `btnTile` |

### Prueba rápida sin haber construido las vistas
1. Toque **Seguimiento** en el menú: `lblCrumb` debe decir `MOVIMIENTOS` y `lblTitulo` `Seguimiento por área`.
2. El ítem del menú debe quedar pintado en esmeralda (`recNavItem.Fill` compara `ThisItem.Pant = varPantalla`).
3. En teléfono, el menú debe cerrarse solo al tocar una entrada — eso lo hace `btnRouter`, no el botón de la galería.

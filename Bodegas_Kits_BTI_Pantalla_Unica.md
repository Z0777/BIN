# Bodegas & Kits BTI — Una sola pantalla
### Continuación desde C.6. Lo que ya construyó se queda; aquí están solo los cambios y lo nuevo.

---

# PARTE H — Correcciones a lo que ya construyó

## H.1 `DropShadow` — el valor correcto es `ExtraBold`

Tiene razón, el valor que acepta el sistema es `ExtraBold`, no `ExtraLarge`.

### `cntNav` → `DropShadow`
```powerapps
If(Parent.Width < nfCorteMovil; DropShadow.ExtraBold; DropShadow.None)
```

Los valores válidos del enumerado son: `DropShadow.None` · `DropShadow.Light` · `DropShadow.Regular` · `DropShadow.Semibold` · `DropShadow.Bold` · `DropShadow.ExtraBold`.

**Dónde más lo escribí mal** (cámbielo cuando llegue a esos controles):

| Control | Valor correcto |
|---|---|
| `cntPopCarrito` → `DropShadow` | `DropShadow.ExtraBold` |
| `cntPopArea` → `DropShadow` | `DropShadow.ExtraBold` |
| `cntPopKit` → `DropShadow` | `DropShadow.ExtraBold` |

---

## H.2 `ZIndex` — **no es una propiedad que se escriba.** Es el orden de capas

Aquí me equivoqué yo y le hice perder tiempo. En Power Apps `ZIndex` **es de solo lectura**: le muestra en qué capa quedó un control, pero **no se le puede asignar un número con fórmula**. No existe la casilla para escribir `ZIndex = 80`.

**Lo que sí se hace:** se reordenan los controles en el **árbol de vistas**.

- Clic derecho sobre el control → **Reordenar** → `Traer al frente` / `Traer adelante` / `Enviar atrás` / `Enviar al fondo`
- O desde la cinta: **Inicio → Reordenar**

**Regla clave:** el orden solo compara **hermanos** — controles que comparten el mismo padre. Un control dentro de un contenedor nunca puede quedar encima de algo que esté fuera de ese contenedor.

### Orden de capas que necesita la pantalla — **del fondo al frente**

| Orden | Control | Por qué en esa capa |
|---|---|---|
| 1 (al fondo) | `cntCuerpo` | es el lienzo |
| 2 | `recScrimNav` | velo que oscurece el cuerpo en teléfono |
| 3 | `cntNav` | el menú flota sobre el velo |
| 4 | `recScrimPop` | velo del carrito, tapa también el menú |
| 5 | `cntPopCarrito` | |
| 6 | `recScrimKit` | |
| 7 | `cntPopKit` | |
| 8 | `recScrimArea` | |
| 9 (al frente) | `cntPopArea` | es el popup que se abre **encima** del carrito |

> Cómo lograrlo rápido: inserte los 9 controles **en ese mismo orden**. En Power Apps cada control nuevo se coloca delante del anterior, así que insertándolos de arriba hacia abajo de esta tabla quedan solos. Si ya los tiene desordenados, seleccione `cntPopArea` → Reordenar → **Traer al frente**, luego `recScrimArea` → Traer al frente, y siga hacia arriba en la tabla al revés.

### Orden de capas dentro de la plantilla de `galNav` (del fondo al frente)

| Orden | Control |
|---|---|
| 1 | `recNavItem` |
| 2 | `icoNavItem` |
| 3 | `lblNavItem` |
| 4 | `lblNavCta` |
| 5 (al frente) | `btnNavItem` |

### Orden de capas dentro de la plantilla de `galItems` (del fondo al frente)

| Orden | Control | Nota |
|---|---|---|
| 1 | `cntTarjeta` | |
| 2 | `recFranja` | |
| 3 | `recCheck` | |
| 4 | `icoCheck` | |
| 5 | `btnSelItem` | el botón invisible |
| 6 (al frente) | `cntCantidad` | **tiene que quedar delante del botón**, o los `+` / `−` no responden |

> En todas las tablas de la guía anterior, donde dice `ZIndex = 20` o `ZIndex = 5`, **ignórelo y use estas tablas de orden**. No escriba nada en `ZIndex`.

---

## H.3 Paleta nueva — solo cambia `nfCol` en `App.Formulas`

Como los colores se guardan como **texto hex** dentro de `nfCol` y los controles los leen con `ColorValue(nfCol.…)`, basta reemplazar el bloque en `App.Formulas` y **todo lo que ya construyó cambia solo**. No toque ni un control.

El menú pasa de verde oscuro a un **azul marino profundo**, el lienzo a un **gris azulado claro**, y el acento a un **esmeralda más saturado**. Es una combinación más atractiva y los blancos de las tarjetas resaltan mucho mejor.

| Token | Hex | Dónde se usa |
|---|---|---|
| `Bg` | `#EDF1F4` | fondo del cuerpo (`cntCuerpo.Fill`) |
| `Surface` | `#FFFFFF` | tarjetas, barra, popups |
| `Surface2` | `#F5F8FA` | campos, pie del popup, botones apagados |
| `Surface3` | `#E7EDF1` | campos de solo lectura |
| `Ink` | `#14202B` | texto principal |
| `Muted` | `#55657A` | texto secundario |
| `Faint` | `#8595A8` | etiquetas en mayúsculas, pistas |
| `Line` | `#D3DCE4` | bordes de tarjeta y campo |
| `Line2` | `#E6ECF1` | separadores internos |
| `Nav` | `#16212E` | **fondo del menú lateral** |
| `Nav2` | `#243448` | hover del menú |
| `NavInk` | `#DFE8F1` | texto del menú |
| `NavMut` | `#8294A9` | texto apagado del menú |
| `Accent` | `#157F5B` | **botones de acción, códigos, bordes activos** |
| `Accent2` | `#1FA172` | **ítem seleccionado del menú** (más brillante, resalta sobre el azul) |
| `AccentSoft` | `#D8EFE5` | fondo de pastillas y tarjetas seleccionadas |
| `AccentInk` | `#0A4A35` | texto sobre `AccentSoft` |
| `NavSelTxt` | `#06281D` | texto del ítem seleccionado del menú |
| `Blue` / `BlueSoft` | `#1F6FB2` / `#DBEAF7` | estatus ASIGNADO, etiqueta Bodega 1 |
| `Amber` / `AmberSoft` | `#9A6207` / `#FAEBD0` | dañado, parcial, bajo mínimo |
| `Red` / `RedSoft` | `#A62B2B` / `#F8DEDE` | extraviado, de baja |
| `Violet` / `VioletSoft` | `#5A4B9C` / `#E6E2F5` | consumido, etiqueta Contenedor |
| `Slate` / `SlateSoft` | `#495A6B` / `#E2E8EE` | devuelto, cerrado |

### `App.Formulas` → reemplace el bloque `nfCol` completo
```powerapps
nfCol = {
    /* ---------- Lienzo ---------- */
    Bg:          "#EDF1F4";
    Surface:     "#FFFFFF";
    Surface2:    "#F5F8FA";
    Surface3:    "#E7EDF1";

    /* ---------- Texto ---------- */
    Ink:         "#14202B";
    Muted:       "#55657A";
    Faint:       "#8595A8";

    /* ---------- Líneas ---------- */
    Line:        "#D3DCE4";
    Line2:       "#E6ECF1";

    /* ---------- Menú lateral ---------- */
    Nav:         "#16212E";
    Nav2:        "#243448";
    NavInk:      "#DFE8F1";
    NavMut:      "#8294A9";
    NavSelTxt:   "#06281D";

    /* ---------- Acento ---------- */
    Accent:      "#157F5B";
    Accent2:     "#1FA172";
    AccentSoft:  "#D8EFE5";
    AccentInk:   "#0A4A35";

    /* ---------- Semánticos ---------- */
    Blue:        "#1F6FB2";   BlueSoft:   "#DBEAF7";
    Amber:       "#9A6207";   AmberSoft:  "#FAEBD0";
    Red:         "#A62B2B";   RedSoft:    "#F8DEDE";
    Violet:      "#5A4B9C";   VioletSoft: "#E6E2F5";
    Slate:       "#495A6B";   SlateSoft:  "#E2E8EE"
};
```

### Los únicos **tres** controles ya construidos que hay que retocar

Son los que usaban el verde del menú y ahora deben usar el esmeralda brillante sobre el azul marino.

#### `recNavItem` (plantilla de `galNav`) → `Fill`
```powerapps
If(ThisItem.Pant = varPantalla; ColorValue(nfCol.Accent2); RGBA(0; 0; 0; 0))
```

#### `icoNavItem` (plantilla de `galNav`) → `Color`
```powerapps
If(ThisItem.Pant = varPantalla; ColorValue(nfCol.NavSelTxt); ColorValue(nfCol.NavMut))
```

#### `lblNavItem` (plantilla de `galNav`) → `Color`
```powerapps
If(ThisItem.Pant = varPantalla; ColorValue(nfCol.NavSelTxt); ColorValue(nfCol.NavMut))
```

#### `lblNavCta` (plantilla de `galNav`) → `Color`
```powerapps
If(ThisItem.Pant = varPantalla; ColorValue(nfCol.NavSelTxt); ColorValue(nfCol.NavInk))
```

#### `lblNavLogo` (en `cntNavMarca`) → `Fill` y `Color`
```powerapps
/* Fill */
ColorValue(nfCol.Accent2)
/* Color */
ColorValue(nfCol.NavSelTxt)
```

Todo lo demás (`cntCuerpo.Fill`, `cntNav.Fill`, `cntBarra.Fill`, `lblNavTitulo.Color`, `lblNavSub.Color`, `lblNavUser.Color`, `icoNav.Color`, `icoAtras.Color`, `lblCrumb.Color`, `lblTitulo.Color`) **no se toca**: ya leen de `nfCol` y cambian solos.

---

## H.4 Una sola pantalla — qué cambia de C.1 a C.6

| Lo que construyó | Cambio |
|---|---|
| El nombre de la pantalla | renómbrela **`scrPrincipal`**. Es la única de la app |
| `cntCuerpo` · `recScrimNav` · `cntNav` · `galNav` · `cntNavMarca` · `cntNavTexto` · `lblNavUser` | **sin cambios** |
| `cntBarra` y sus hijos | `lblCrumb.Text`, `lblTitulo.Text`, `icoAtras.Visible` y `icoAtras.OnSelect` pasan a ser dinámicos → ver **I.3** |
| `btnNavItem.OnSelect` | ya no usa `Navigate()`; ahora prende/apaga contenedores → ver **I.4** |
| **C.7 "copiar a las otras pantallas"** | **se elimina.** No hay otras pantallas |
| `App.StartScreen` | `scrPrincipal` |

---

# PARTE I — Arquitectura de una sola pantalla

## I.1 El principio

`cntCuerpo` es un contenedor de diseño **vertical**. Sus hijos son la barra y **nueve contenedores de vista**, uno por cada entrada del menú. Los nueve llevan `FillPortions = 1`, pero solo uno tiene `Visible = true` a la vez.

> **Por qué funciona:** en un contenedor de diseño automático, un hijo con `Visible = false` **no ocupa espacio**. Así que el único visible se come todo el alto sobrante sin que haya que calcular nada. Es el equivalente exacto de cambiar de pantalla, pero sin `Navigate()`.

**Lo que se gana:** el menú, la barra y los popups se construyen **una sola vez**. Nunca más hay que sincronizar ocho copias.

**Lo que hay que cuidar:** un contenedor **no tiene `OnVisible`**. Las cargas de datos que estaban en `scrXxx.OnVisible` se mudan al `OnSelect` del botón del menú (ver I.4).

## I.2 Árbol completo de `scrPrincipal`

```
scrPrincipal                                    ← LA ÚNICA PANTALLA
│
├─ 1  cntCuerpo ·············· Contenedor VERTICAL
│   ├─ 1  cntBarra ··········· Contenedor HORIZONTAL        FillPortions 0
│   │   ├─ icoNav ············ Icono Hamburger
│   │   ├─ icoAtras ·········· Icono ChevronLeft
│   │   └─ cntTitulos ········ Contenedor VERTICAL          FillPortions 1
│   │       ├─ lblCrumb ······ Etiqueta
│   │       └─ lblTitulo ····· Etiqueta
│   │
│   ├─ 2  cntVMenu ·········· Contenedor VERTICAL   FP 1   Visible = varPantalla = "MENU"
│   ├─ 3  cntVAsignar ······· Contenedor VERTICAL   FP 1   Visible = varPantalla = "ASIG"
│   ├─ 4  cntVSeguimiento ··· Contenedor VERTICAL   FP 1   Visible = varPantalla = "SEG"
│   ├─ 5  cntVSegDetalle ···· Contenedor VERTICAL   FP 1   Visible = varPantalla = "SEGDET"
│   ├─ 6  cntVDevolucion ···· Contenedor VERTICAL   FP 1   Visible = varPantalla = "DEV"
│   ├─ 7  cntVIngresos ······ Contenedor VERTICAL   FP 1   Visible = varPantalla = "ING"
│   ├─ 8  cntVCatalogo ······ Contenedor VERTICAL   FP 1   Visible = varPantalla = "CAT"
│   ├─ 9  cntVBajas ········· Contenedor VERTICAL   FP 1   Visible = varPantalla = "BAJA"
│   └─ 10 cntVFlujo ········· Contenedor VERTICAL   FP 1   Visible = varPantalla = "FLUJO"
│
├─ 2  recScrimNav ············ Rectángulo
├─ 3  cntNav ················· Contenedor VERTICAL
│   ├─ cntNavMarca ·········· Contenedor HORIZONTAL   FP 0
│   ├─ galNav ··············· Galería VERTICAL        FP 1
│   └─ lblNavUser ··········· Etiqueta                FP 0
│
├─ 4  recScrimPop ············ Rectángulo
├─ 5  cntPopCarrito ·········· Contenedor MANUAL
├─ 6  recScrimKit ············ Rectángulo
├─ 7  cntPopKit ·············· Contenedor MANUAL
├─ 8  recScrimArea ··········· Rectángulo
├─ 9  cntPopArea ············· Contenedor MANUAL
└─ 10 tmrVencidos ············ Temporizador (Visible = false)
```

## I.3 `cntBarra` dinámico — **reemplaza la versión de C.6**

El contenedor `cntBarra` no cambia. Cambian cuatro propiedades de sus hijos.

### `icoAtras` — Icono `Icon.ChevronLeft`
```powerapps
/* Visible */
varPantalla <> "MENU"

/* OnSelect */
Set(varPantalla; If(varPantalla = "SEGDET"; "SEG"; "MENU"))
```

### `lblCrumb` — Etiqueta
```powerapps
/* Text */
Switch(
    varPantalla;
    "MENU";   "INICIO";
    "ASIG";   "MOVIMIENTOS";
    "SEG";    "MOVIMIENTOS";
    "SEGDET"; "SEGUIMIENTO";
    "DEV";    "MOVIMIENTOS";
    "ING";    "INVENTARIO";
    "CAT";    "INVENTARIO";
    "BAJA";   "INVENTARIO";
    "FLUJO";  "REFERENCIA";
    ""
)
```

### `lblTitulo` — Etiqueta
```powerapps
/* Text */
Switch(
    varPantalla;
    "MENU";   "Menú principal";
    "ASIG";   "Asignar kit o unidad";
    "SEG";    "Seguimiento por área";
    "SEGDET"; varAreaVista.Title & " — " & varAreaVista.NombreArea;
    "DEV";    "Registrar devolución";
    "ING";    "Ingresos de Bodega General";
    "CAT";    "Catálogo por tipo y bodega";
    "BAJA";   "Artículos dados de baja";
    "FLUJO";  "Flujo de estatus";
    ""
)
```

## I.4 El enrutador — `btnNavItem.OnSelect`

Esta es la pieza central: prende el contenedor y hace la carga que antes vivía en `OnVisible`.

> **La misma fórmula sirve para `btnTile`** (los botones del menú principal), porque `nfBotonesMenu` también trae el campo `Pant`. Péguela igual en los dos.

```powerapps
/* ---------- 1. cambiar de vista y cerrar el menú en teléfono ---------- */
Set(varPantalla; ThisItem.Pant);;
Set(varPopup; "");;
If(Parent.Width < nfCorteMovil; Set(varNavAbierto; false));;

/* ---------- 2. ASIGNAR: estado inicial + kit completo precargado ---------- */
If(
    ThisItem.Pant = "ASIG";
    Set(varModoCarga; "KIT")
);;
If(
    ThisItem.Pant = "ASIG";
    Set(varTipo; "RETORNABLE")
);;
If(
    ThisItem.Pant = "ASIG";
    Set(varBodegaFiltro; "TODAS")
);;
If(
    ThisItem.Pant = "ASIG";
    Set(varBuscar; "")
);;
If(
    ThisItem.Pant = "ASIG";
    Set(varArea; Blank())
);;
If(
    ThisItem.Pant = "ASIG";
    Set(varEncargado; Blank())
);;
If(
    ThisItem.Pant = "ASIG";
    Set(varRecibe; Blank())
);;
If(
    ThisItem.Pant = "ASIG";
    Clear(colCatSel)
);;
If(
    ThisItem.Pant = "ASIG";
    Set(varKit; Coalesce(varKit; First(nfKits)))
);;
If(
    ThisItem.Pant = "ASIG";
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
If(
    ThisItem.Pant = "ASIG";
    RemoveIf(colSel; Existencia <= 0)
);;

/* ---------- 3. SEGUIMIENTO: resumen por área ---------- */
If(
    ThisItem.Pant = "SEG";
    ClearCollect(colDetAb; nfDetAbierto)
);;
If(
    ThisItem.Pant = "SEG";
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

/* ---------- 4. DEVOLUCIÓN: limpia la captura anterior ---------- */
If(
    ThisItem.Pant = "DEV";
    Set(varAreaDev; Blank())
);;
If(
    ThisItem.Pant = "DEV";
    Set(varSalDev; Blank())
);;
If(
    ThisItem.Pant = "DEV";
    Clear(colDev)
);;

/* ---------- 5. INGRESOS: pendientes de Bodega General ---------- */
If(
    ThisItem.Pant = "ING";
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

/* ---------- 6. CATÁLOGO: filtros en limpio ---------- */
If(
    ThisItem.Pant = "CAT";
    Set(varTipoCat; "")
);;
If(
    ThisItem.Pant = "CAT";
    Set(varBodCat; "TODAS")
);;
If(
    ThisItem.Pant = "CAT";
    Set(varBuscarCat; "")
);;

/* ---------- 7. BAJAS ---------- */
If(
    ThisItem.Pant = "BAJA";
    ClearCollect(colBajas; Sort(BG_Bajas; FechaBaja; SortOrder.Descending))
)
```

> **Por qué un `If` por instrucción y no un `Switch` con bloques:** así cada línea es una expresión independiente y la fórmula nunca falla por anidamiento. Es más larga de leer y es la que no se rompe. Y **cero `Select()`**: ese fue el bug del doble clic en `procesos01`.

### `App.OnStart` — un solo cambio
```powerapps
Set(varPantalla; "MENU")
```
Ya estaba. Solo confirme que `App.StartScreen = scrPrincipal`.

---

# PARTE J — Los nueve contenedores de vista

## J.0 Propiedades que comparten los nueve

Todos son **Contenedor de diseño vertical**, hijos de `cntCuerpo`, hermanos de `cntBarra`.

```powerapps
FillPortions         = 1
AlignInContainer     = AlignInContainer.Stretch
LayoutDirection      = LayoutDirection.Vertical
LayoutGap            = 12
LayoutAlignItems     = LayoutAlignItems.Stretch
LayoutJustifyContent = LayoutJustifyContent.Start
LayoutOverflowY      = LayoutOverflow.Hide
LayoutWrap           = false
PaddingTop    = 16
PaddingBottom = 16
PaddingLeft   = nfPad
PaddingRight  = nfPad
Fill          = RGBA(0; 0; 0; 0)        /* el color lo da cntCuerpo */
```
Lo único distinto en cada uno es `Visible` y su contenido.

| Orden | Contenedor | `Visible` | `LayoutGap` | Entrada del menú |
|---|---|---|---|---|
| 2 | `cntVMenu` | `varPantalla = "MENU"` | `14` | Menú principal |
| 3 | `cntVAsignar` | `varPantalla = "ASIG"` | `12` | Asignar kit o unidad |
| 4 | `cntVSeguimiento` | `varPantalla = "SEG"` | `12` | Seguimiento por área |
| 5 | `cntVSegDetalle` | `varPantalla = "SEGDET"` | `12` | — (se abre desde Seguimiento) |
| 6 | `cntVDevolucion` | `varPantalla = "DEV"` | `12` | Registrar devolución |
| 7 | `cntVIngresos` | `varPantalla = "ING"` | `12` | Ingresos de Bodega General |
| 8 | `cntVCatalogo` | `varPantalla = "CAT"` | `12` | Catálogo por tipo y bodega |
| 9 | `cntVBajas` | `varPantalla = "BAJA"` | `12` | Artículos dados de baja |
| 10 | `cntVFlujo` | `varPantalla = "FLUJO"` | `14` | Flujo de estatus |

---

## J.1 `cntVMenu` — Menú principal

| Orden | Nivel | Control | Tipo exacto | Propiedades |
|---|---|---|---|---|
| 1 | 3 | `galKpi` | **Galería en blanco HORIZONTAL** | `FillPortions = 0` · `Height = 108` · `AlignInContainer = AlignInContainer.Stretch` · `Layout = Layout.Horizontal` · `TemplateSize = If(Parent.Width < nfCorteMovil; 152; (Self.Width - 8) / 5)` · `TemplatePadding = 4` · `ShowScrollbar = true` |
| 2 | 3 | `galBotones` | **Galería en blanco VERTICAL** (cuadrícula) | `FillPortions = 1` · `AlignInContainer = AlignInContainer.Stretch` · `Layout = Layout.Vertical` · `WrapCount = Max(1; RoundDown(Self.Width / 252; 0))` · `TemplateSize = 142` · `TemplatePadding = 5` · `Items = nfBotonesMenu` |

### `galKpi` → `Items`
```powerapps
Table(
    {Val: nfCtaAbiertas;  Etq: "SALIDAS ABIERTAS";     Col: nfCol.Ink};
    {Val: nfCtaAfuera;    Etq: "PIEZAS AFUERA";        Col: nfCol.Ink};
    {Val: nfCtaDanadas;   Etq: "PIEZAS DAÑADAS";       Col: nfCol.Amber};
    {Val: nfCtaPerdidas;  Etq: "PIEZAS EXTRAVIADAS";   Col: nfCol.Red};
    {Val: nfCtaIngresos;  Etq: "INGRESOS POR APLICAR"; Col: nfCol.Blue}
)
```

### Plantilla de `galKpi` — contenedor automático (no hay botón invisible)

| Orden | Nivel | Control | Tipo exacto | Propiedades |
|---|---|---|---|---|
| 1 | 4 | `cntKpi` | **Contenedor de diseño VERTICAL** | `X = 0` · `Y = 0` · `Width = Parent.TemplateWidth - 8` · `Height = 100` · `LayoutGap = 2` · `LayoutJustifyContent = LayoutJustifyContent.Center` · `PaddingLeft = 14` · `PaddingRight = 14` · `Fill = ColorValue(nfCol.Surface)` · `BorderThickness = 1` · `BorderColor = ColorValue(nfCol.Line)` · `RadiusTopLeft…RadiusBottomRight = nfRadio` |
| 1.1 | 5 | `lblKpiVal` | Etiqueta | `FillPortions = 0` · `Height = 34` · `AlignInContainer = AlignInContainer.Stretch` · `Text = ThisItem.Val` · `Size = 22` · `FontWeight = FontWeight.Bold` · `Color = ColorValue(ThisItem.Col)` |
| 1.2 | 5 | `lblKpiEtq` | Etiqueta | `FillPortions = 0` · `Height = 30` · `AlignInContainer = AlignInContainer.Stretch` · `Text = ThisItem.Etq` · `Size = 9,5` · `FontWeight = FontWeight.Semibold` · `Color = ColorValue(nfCol.Faint)` |

### Plantilla de `galBotones` — isla manual (el botón invisible se enciman)

**Orden de capas, del fondo al frente:** `cntBoton` → `btnTile`

| Orden | Nivel | Control | Tipo exacto | Propiedades |
|---|---|---|---|---|
| 1 | 4 | `cntBoton` | **Contenedor de diseño VERTICAL** | `X = 0` · `Y = 0` · `Width = Parent.TemplateWidth - 10` · `Height = 134` · `LayoutGap = 8` · `PaddingTop = 16` · `PaddingBottom = 14` · `PaddingLeft = 16` · `PaddingRight = 16` · `Fill = ColorValue(nfCol.Surface)` · `BorderThickness = 1` · `BorderColor = ColorValue(nfCol.Line)` · `RadiusTopLeft…RadiusBottomRight = nfRadio` |
| 1.1 | 5 | `cntIcoBtn` | **Contenedor MANUAL** | `FillPortions = 0` · `Width = 38` · `Height = 38` · `AlignInContainer = AlignInContainer.Start` · `Fill = ColorValue(nfCol.AccentSoft)` · `RadiusTopLeft…RadiusBottomRight = 9` |
| 1.1.a | 6 | `icoBtn` | Icono | `X = 10` · `Y = 10` · `Width = 18` · `Height = 18` · `Color = ColorValue(nfCol.AccentInk)` · `Icon = Switch(ThisItem.Ico; "ADD"; Icon.Add; "TREND"; Icon.TrendingUp; "BACK"; Icon.Reload; "DOWN"; Icon.Download; "ITEMS"; Icon.Items; Icon.Trash)` |
| 1.2 | 5 | `lblBtnTit` | Etiqueta | `FillPortions = 0` · `Height = 22` · `AlignInContainer = AlignInContainer.Stretch` · `Text = ThisItem.Titulo` · `Size = 14,5` · `FontWeight = FontWeight.Semibold` · `Color = ColorValue(nfCol.Ink)` |
| 1.3 | 5 | `lblBtnDet` | Etiqueta | `FillPortions = 1` · `AlignInContainer = AlignInContainer.Stretch` · `Text = ThisItem.Detalle` · `Size = 11,5` · `Color = ColorValue(nfCol.Muted)` |
| 2 | 4 | `lblBtnCta` | Etiqueta | `X = Parent.TemplateWidth - 56` · `Y = 16` · `Width = 32` · `Height = 20` · `Text = ThisItem.Cta` · `Visible = ThisItem.Cta > 0` · `Align = Align.Center` · `Size = 10,5` · `Color = ColorValue(nfCol.Accent)` · `BorderThickness = 1` · `BorderColor = ColorValue(nfCol.Accent)` · `RadiusTopLeft…RadiusBottomRight = 5` |
| 3 | 4 | `btnTile` | **Botón clásico invisible** | `X = 0` · `Y = 0` · `Width = Parent.TemplateWidth - 10` · `Height = 134` · `Text = ""` · `Fill = RGBA(0; 0; 0; 0)` · `HoverFill = RGBA(21; 127; 91; 0,06)` · `PressedFill = RGBA(21; 127; 91; 0,12)` · `BorderColor = RGBA(0; 0; 0; 0)` · `HoverBorderColor = RGBA(0; 0; 0; 0)` · `OnSelect` = **la misma fórmula de I.4** |

---

## J.2 `cntVAsignar` — Asignar kit o unidad

| Orden | Nivel | Control | Tipo exacto | Propiedades |
|---|---|---|---|---|
| 1 | 3 | `cntModo` | **Contenedor de diseño HORIZONTAL** | `FillPortions = 0` · `Height = 40` · `AlignInContainer = AlignInContainer.Start` · `LayoutGap = 6` · `LayoutAlignItems = LayoutAlignItems.Center` · `LayoutWrap = true` |
| 1.1 | 4 | `btnModoKit` | Botón clásico | `FillPortions = 0` · `Width = 134` · `Height = 38` · `Text = "Por kit"` · `Fill = If(varModoCarga = "KIT"; ColorValue(nfCol.Accent); ColorValue(nfCol.Surface2))` · `Color = If(varModoCarga = "KIT"; White; ColorValue(nfCol.Muted))` · `BorderColor = ColorValue(nfCol.Line)` · `RadiusTopLeft…RadiusBottomRight = 8` |
| 1.2 | 4 | `btnModoUnidad` | Botón clásico | `FillPortions = 0` · `Width = 134` · `Height = 38` · `Text = "Por unidad"` · `Fill = If(varModoCarga = "UNIDAD"; ColorValue(nfCol.Accent); ColorValue(nfCol.Surface2))` · `Color = If(varModoCarga = "UNIDAD"; White; ColorValue(nfCol.Muted))` |
| 2 | 3 | `cntPanelKit` | **Contenedor de diseño VERTICAL** | `FillPortions = 0` · `Height = 76` · `AlignInContainer = AlignInContainer.Stretch` · `Visible = varModoCarga = "KIT"` · `LayoutGap = 4` · `PaddingTop = 8` · `PaddingBottom = 8` · `PaddingLeft = 14` · `PaddingRight = 14` · `Fill = ColorValue(nfCol.Surface)` · `BorderThickness = 1` · `BorderColor = ColorValue(nfCol.Line)` · `RadiusTopLeft…RadiusBottomRight = nfRadio` |
| 2.1 | 4 | `lblKitEtq` | Etiqueta | `FillPortions = 0` · `Height = 14` · `Text = "KIT DE CICLO"` · `Size = 9` · `FontWeight = FontWeight.Semibold` · `Color = ColorValue(nfCol.Faint)` |
| 2.2 | 4 | `cntKitPicker` | **Contenedor MANUAL** | `FillPortions = 0` · `Height = 38` · `AlignInContainer = AlignInContainer.Stretch` · `Fill = RGBA(0; 0; 0; 0)` |
| 2.2.a | 5 | `lblKitElegido` | Etiqueta | `X = 0` · `Y = 8` · `Width = Parent.Width - 32` · `Height = 22` · `Text = varKit.Title & " · " & varKit.NombreKit & "   (" & CountRows(Filter(BG_KitDetalle; Title = varKit.Title)) & " renglones)"` · `Size = 13,5` · `FontWeight = FontWeight.Semibold` |
| 2.2.b | 5 | `icoKitFlecha` | Icono `Icon.ChevronRight` | `X = Parent.Width - 24` · `Y = 9` · `Width = 20` · `Height = 20` · `Color = ColorValue(nfCol.Faint)` |
| 2.2.c | 5 | `btnElegirKit` | **Botón clásico invisible** | `X = 0` · `Y = 0` · `Width = Parent.Width` · `Height = Parent.Height` · `Text = ""` · `Fill = RGBA(0; 0; 0; 0)` · `HoverFill = RGBA(21; 127; 91; 0,06)` · `BorderColor = RGBA(0; 0; 0; 0)` · `OnSelect = Set(varPopup; "KIT")` |
| 3 | 3 | `cntPanelUnidad` | **Contenedor de diseño VERTICAL** | `FillPortions = 0` · `Height = 152` · `AlignInContainer = AlignInContainer.Stretch` · `Visible = varModoCarga = "UNIDAD"` · `LayoutGap = 10` · mismo `Fill` / borde / padding que `cntPanelKit` |
| 3.1 | 4 | `cntTipo` | **Contenedor de diseño HORIZONTAL** | `FillPortions = 0` · `Height = 38` · `AlignInContainer = AlignInContainer.Start` · `LayoutGap = 6` |
| 3.1.a | 5 | `btnTipoRet` | Botón clásico | `FillPortions = 0` · `Width = 150` · `Height = 38` · `Text = "Retornable"` · `Fill = If(varTipo = "RETORNABLE"; ColorValue(nfCol.Accent); ColorValue(nfCol.Surface2))` · `Color = If(varTipo = "RETORNABLE"; White; ColorValue(nfCol.Muted))` · `OnSelect = Set(varTipo; "RETORNABLE");; Clear(colCatSel)` |
| 3.1.b | 5 | `btnTipoUso` | Botón clásico | `FillPortions = 0` · `Width = 150` · `Height = 38` · `Text = "Un solo uso"` · `Fill = If(varTipo = "UN SOLO USO"; ColorValue(nfCol.Accent); ColorValue(nfCol.Surface2))` · `OnSelect = Set(varTipo; "UN SOLO USO");; Clear(colCatSel)` |
| 3.2 | 4 | `cntBusqueda` | **Contenedor de diseño HORIZONTAL** | `FillPortions = 0` · `Height = 38` · `AlignInContainer = AlignInContainer.Stretch` · `LayoutGap = 10` · `LayoutWrap = true` |
| 3.2.a | 5 | `txtBuscar` | **Text input moderno** | `FillPortions = 1` · `LayoutMinWidth = 180` · `Height = 36` · `AlignInContainer = AlignInContainer.Center` · `Type = TextInputType.Search` · `Placeholder = "Buscar código o descripción"` · `TriggerOutput = TriggerOutput.Delayed` · `Appearance = Appearance.Outline` · `Fill = ColorValue(nfCol.Surface2)` · `BorderColor = ColorValue(nfCol.Line)` · `Size = 13` · `RadiusTopLeft…RadiusBottomRight = 8` · `OnChange = Set(varBuscar; Self.Text)` |
| 3.2.b | 5 | `galBodegaF` | **Galería en blanco HORIZONTAL** | `FillPortions = 0` · `Width = Min(400; Parent.Width * 0,45)` · `Height = 38` · `Layout = Layout.Horizontal` · `TemplateSize = 128` · `Items = nfBodegasF` |
| 3.3 | 4 | `galCatChips` | **Galería en blanco HORIZONTAL** | `FillPortions = 0` · `Height = 38` · `AlignInContainer = AlignInContainer.Stretch` · `Layout = Layout.Horizontal` · `TemplateSize = 136` · `ShowScrollbar = true` · `Items = Distinct(Filter(nfCatalogo; Tipo.Value = varTipo); Categoria.Value)` |
| 4 | 3 | `galItems` | **Galería en blanco VERTICAL** (cuadrícula) | `FillPortions = 1` · `AlignInContainer = AlignInContainer.Stretch` · `Layout = Layout.Vertical` · `WrapCount = Max(1; RoundDown(Self.Width / 270; 0))` · `TemplateSize = 128` · `TemplatePadding = 5` · `DelayItemLoading = true` · `LoadingSpinner = LoadingSpinner.Data` |
| 5 | 3 | `cntAccion` | **Contenedor de diseño HORIZONTAL** | `FillPortions = 0` · `Height = 48` · `AlignInContainer = AlignInContainer.Stretch` · `LayoutJustifyContent = LayoutJustifyContent.End` · `LayoutAlignItems = LayoutAlignItems.Center` |
| 5.1 | 4 | `btnContinuar` | Botón clásico | `FillPortions = 0` · `Width = Min(270; Parent.Width - 20)` · `Height = 42` · `Text = "Continuar  ·  " & CountRows(colSel) & " renglones  ·  " & Sum(colSel; Cantidad) & " pzas"` · `Fill = ColorValue(nfCol.Accent)` · `Color = White` · `BorderColor = ColorValue(nfCol.Accent)` · `RadiusTopLeft…RadiusBottomRight = 8` · `DisplayMode = If(IsEmpty(colSel); DisplayMode.Disabled; DisplayMode.Edit)` |

### Plantilla de `galItems`

**Orden de capas, del fondo al frente:** `cntTarjeta` → `recFranja` → `recCheck` → `icoCheck` → `btnSelItem` → `cntCantidad`

| Orden | Nivel | Control | Tipo exacto | Propiedades |
|---|---|---|---|---|
| 1 | 4 | `cntTarjeta` | **Contenedor de diseño VERTICAL** | `X = 0` · `Y = 0` · `Width = Parent.TemplateWidth - 10` · `Height = 122` · `LayoutGap = 4` · `PaddingTop = 10` · `PaddingBottom = 10` · `PaddingLeft = 40` · `PaddingRight = 12` · `Fill = ColorValue(nfCol.Surface)` · `BorderThickness = 1` · `BorderColor = If(!IsBlank(LookUp(colSel; Clave = ThisItem.Clave; Clave)); ColorValue(nfCol.Accent); ColorValue(nfCol.Line))` · `RadiusTopLeft…RadiusBottomRight = nfRadio` |
| 1.1 | 5 | `lblCod` | Etiqueta | `FillPortions = 0` · `Height = 15` · `Text = ThisItem.Codigo` · `Size = 10,5` · `Color = ColorValue(nfCol.Accent)` |
| 1.2 | 5 | `lblDesc` | Etiqueta | `FillPortions = 1` · `Text = ThisItem.Descripcion` · `Size = 13` · `FontWeight = FontWeight.Semibold` · `Color = ColorValue(nfCol.Ink)` |
| 1.3 | 5 | `cntTags` | **Contenedor de diseño HORIZONTAL** | `FillPortions = 0` · `Height = 18` · `AlignInContainer = AlignInContainer.Stretch` · `LayoutGap = 5` |
| 1.3.a | 6 | `lblTagTipo` | Etiqueta | `FillPortions = 0` · `Width = If(ThisItem.Tipo = "RETORNABLE"; 88; 94)` · `Text = ThisItem.Tipo` · `Size = 9` · `Align = Align.Center` · `Fill = If(ThisItem.Tipo = "RETORNABLE"; ColorValue(nfCol.AccentSoft); ColorValue(nfCol.AmberSoft))` · `Color = If(ThisItem.Tipo = "RETORNABLE"; ColorValue(nfCol.AccentInk); ColorValue(nfCol.Amber))` · `RadiusTopLeft…RadiusBottomRight = 4` |
| 1.3.b | 6 | `lblTagBodega` | Etiqueta | `FillPortions = 0` · `Width = 90` · `Text = ThisItem.Bodega` · `Size = 9` · `Align = Align.Center` · `Color = If(ThisItem.Bodega = "BODEGA 1"; ColorValue(nfCol.Blue); ColorValue(nfCol.Violet))` · `BorderThickness = 1` · `BorderColor = If(ThisItem.Bodega = "BODEGA 1"; ColorValue(nfCol.Blue); ColorValue(nfCol.Violet))` · `RadiusTopLeft…RadiusBottomRight = 4` |
| 1.4 | 5 | `lblStock` | Etiqueta | `FillPortions = 0` · `Height = 18` · `Text = "Existencia " & ThisItem.Existencia & " " & ThisItem.Unidad & If(ThisItem.Existencia <= 0; " · AGOTADO"; "")` · `Size = 10,5` · `Color = If(ThisItem.Existencia <= 0; ColorValue(nfCol.Red); ColorValue(nfCol.Muted))` |
| 2 | 4 | `recFranja` | Rectángulo | `X = 0` · `Y = 0` · `Width = 3` · `Height = 122` · `Fill = ColorValue(nfCol.Accent)` · `Visible = !IsBlank(LookUp(colSel; Clave = ThisItem.Clave; Clave))` |
| 3 | 4 | `recCheck` | Rectángulo | `X = 12` · `Y = 12` · `Width = 20` · `Height = 20` · `Fill = If(!IsBlank(LookUp(colSel; Clave = ThisItem.Clave; Clave)); ColorValue(nfCol.Accent); ColorValue(nfCol.Surface2))` · `BorderThickness = 1` · `BorderColor = If(!IsBlank(LookUp(colSel; Clave = ThisItem.Clave; Clave)); ColorValue(nfCol.Accent); ColorValue(nfCol.Line))` · `RadiusTopLeft…RadiusBottomRight = 5` |
| 4 | 4 | `icoCheck` | Icono `Icon.Check` | `X = 15` · `Y = 15` · `Width = 14` · `Height = 14` · `Color = White` · `Visible = !IsBlank(LookUp(colSel; Clave = ThisItem.Clave; Clave))` |
| 5 | 4 | `btnSelItem` | **Botón clásico invisible** | `X = 0` · `Y = 0` · `Width = Parent.TemplateWidth - 10` · `Height = 122` · `Text = ""` · `Fill = RGBA(0; 0; 0; 0)` · `HoverFill = RGBA(21; 127; 91; 0,06)` · `PressedFill = RGBA(21; 127; 91; 0,12)` · `BorderColor = RGBA(0; 0; 0; 0)` · `HoverBorderColor = RGBA(0; 0; 0; 0)` · `DisabledFill = RGBA(0; 0; 0; 0)` · `DisplayMode = If(ThisItem.Existencia <= 0; DisplayMode.Disabled; DisplayMode.Edit)` |
| 6 | 4 | `cntCantidad` | **Contenedor MANUAL** | `X = Parent.TemplateWidth - 124` · `Y = 86` · `Width = 102` · `Height = 28` · `Visible = !IsBlank(LookUp(colSel; Clave = ThisItem.Clave; Clave))` · `Fill = ColorValue(nfCol.Surface2)` · `BorderThickness = 1` · `BorderColor = ColorValue(nfCol.Line)` · `RadiusTopLeft…RadiusBottomRight = 7` |
| 6.1 | 5 | `btnMenos` | Botón clásico | `X = 0` · `Y = 0` · `Width = 28` · `Height = 28` · `Text = "−"` · `Fill = RGBA(0; 0; 0; 0)` · `BorderColor = RGBA(0; 0; 0; 0)` · `HoverFill = ColorValue(nfCol.AccentSoft)` · `Color = ColorValue(nfCol.Ink)` |
| 6.2 | 5 | `lblCant` | Etiqueta | `X = 28` · `Y = 0` · `Width = 46` · `Height = 28` · `Text = LookUp(colSel; Clave = ThisItem.Clave; Cantidad)` · `Align = Align.Center` · `Size = 13` · `FontWeight = FontWeight.Semibold` |
| 6.3 | 5 | `btnMas` | Botón clásico | `X = 74` · `Y = 0` · `Width = 28` · `Height = 28` · `Text = "+"` · `Fill = RGBA(0; 0; 0; 0)` · `BorderColor = RGBA(0; 0; 0; 0)` · `HoverFill = ColorValue(nfCol.AccentSoft)` · `DisplayMode = If(LookUp(colSel; Clave = ThisItem.Clave; Cantidad) >= ThisItem.Existencia; DisplayMode.Disabled; DisplayMode.Edit)` |

---

## J.3 `cntVSeguimiento` — Seguimiento por área

| Orden | Nivel | Control | Tipo exacto | Propiedades |
|---|---|---|---|---|
| 1 | 3 | `galKpiSeg` | **Galería en blanco HORIZONTAL** | `FillPortions = 0` · `Height = 108` · `AlignInContainer = AlignInContainer.Stretch` · `Layout = Layout.Horizontal` · `TemplateSize = If(Parent.Width < nfCorteMovil; 152; (Self.Width - 6) / 4)` · `TemplatePadding = 4` |
| 2 | 3 | `lblPista` | Etiqueta | `FillPortions = 0` · `Height = 18` · `AlignInContainer = AlignInContainer.Stretch` · `Text = "Deslice para ver todas las áreas  →"` · `Size = 11` · `Color = ColorValue(nfCol.Faint)` |
| 3 | 3 | `galAreas` | **Galería en blanco HORIZONTAL** | `FillPortions = 1` · `AlignInContainer = AlignInContainer.Stretch` · `Layout = Layout.Horizontal` · `TemplateSize = 254` · `TemplatePadding = 5` · `ShowScrollbar = true` · `Items = colResumenArea` |

### `galKpiSeg` → `Items`
```powerapps
Table(
    {Val: nfCtaAbiertas; Etq: "SALIDAS ABIERTAS"; Col: nfCol.Ink};
    {Val: nfCtaAfuera;   Etq: "PIEZAS AFUERA";    Col: nfCol.Ink};
    {Val: nfCtaDanadas;  Etq: "DAÑADAS";          Col: nfCol.Amber};
    {Val: nfCtaPerdidas; Etq: "EXTRAVIADAS";      Col: nfCol.Red}
)
```
La plantilla es la misma de `galKpi` (`cntKpi` + `lblKpiVal` + `lblKpiEtq`): cópiela.

### Plantilla de `galAreas`

**Orden de capas, del fondo al frente:** `cntArea` → `btnAbrirArea_Seg`

| Orden | Nivel | Control | Tipo exacto | Propiedades |
|---|---|---|---|---|
| 1 | 4 | `cntArea` | **Contenedor de diseño VERTICAL** | `X = 0` · `Y = 0` · `Width = Parent.TemplateWidth - 10` · `Height = 202` · `LayoutGap = 3` · `PaddingTop = 13` · `PaddingBottom = 13` · `PaddingLeft = 13` · `PaddingRight = 13` · `Fill = ColorValue(nfCol.Surface)` · `BorderThickness = 1` · `BorderColor = ColorValue(nfCol.Line)` · `RadiusTopLeft…RadiusBottomRight = nfRadio` |
| 1.1 | 5 | `lblAreaCod` | Etiqueta | `FillPortions = 0` · `Height = 15` · `Text = ThisItem.CodigoArea` · `Size = 10,5` · `Color = ColorValue(nfCol.Accent)` |
| 1.2 | 5 | `lblAreaNom` | Etiqueta | `FillPortions = 0` · `Height = 22` · `Text = ThisItem.NombreArea` · `Size = 14,5` · `FontWeight = FontWeight.Semibold` |
| 1.3 | 5 | `lblAreaCul` | Etiqueta | `FillPortions = 0` · `Height = 18` · `Text = ThisItem.Cultivo` · `Size = 11,5` · `Color = ColorValue(nfCol.Muted)` |
| 1.4 | 5 | `cntAreaPills` | **Contenedor de diseño HORIZONTAL** | `FillPortions = 0` · `Height = 46` · `AlignInContainer = AlignInContainer.Stretch` · `LayoutGap = 5` · **`LayoutWrap = true`** |
| 1.4.a | 6 | `lblAreaEstado` | Etiqueta | `FillPortions = 0` · `Width = 112` · `Height = 20` · `Text = If(ThisItem.Abiertas > 0; ThisItem.Abiertas & " abierta(s)"; "Al día")` · `Size = 10` · `Align = Align.Center` · `Fill = If(ThisItem.Abiertas > 0; ColorValue(nfCol.AccentSoft); ColorValue(nfCol.SlateSoft))` · `Color = If(ThisItem.Abiertas > 0; ColorValue(nfCol.AccentInk); ColorValue(nfCol.Slate))` · `RadiusTopLeft…RadiusBottomRight = 999` |
| 1.4.b | 6 | `lblAreaAfuera` | Etiqueta | `FillPortions = 0` · `Width = 100` · `Height = 20` · `Visible = ThisItem.Afuera > 0` · `Text = ThisItem.Afuera & " pzas afuera"` · `Size = 10` · `Align = Align.Center` · `Color = ColorValue(nfCol.Muted)` · `BorderThickness = 1` · `BorderColor = ColorValue(nfCol.Line)` · `RadiusTopLeft…RadiusBottomRight = 999` |
| 1.4.c | 6 | `lblAreaDan` | Etiqueta | `FillPortions = 0` · `Width = 88` · `Height = 20` · `Visible = ThisItem.Danadas > 0` · `Text = ThisItem.Danadas & " dañadas"` · `Size = 10` · `Align = Align.Center` · `Fill = ColorValue(nfCol.AmberSoft)` · `Color = ColorValue(nfCol.Amber)` · `RadiusTopLeft…RadiusBottomRight = 999` |
| 1.4.d | 6 | `lblAreaPer` | Etiqueta | `FillPortions = 0` · `Width = 106` · `Height = 20` · `Visible = ThisItem.Perdidas > 0` · `Text = ThisItem.Perdidas & " extraviadas"` · `Size = 10` · `Align = Align.Center` · `Fill = ColorValue(nfCol.RedSoft)` · `Color = ColorValue(nfCol.Red)` · `RadiusTopLeft…RadiusBottomRight = 999` |
| 1.5 | 5 | `lblAreaGente` | Etiqueta | `FillPortions = 1` · `AlignInContainer = AlignInContainer.Stretch` · `Text = "Responsable " & ThisItem.Responsable & Char(10) & "Encargado " & ThisItem.Encargado` · `Size = 10,5` · `Color = ColorValue(nfCol.Faint)` |
| 2 | 4 | `btnAbrirArea_Seg` | **Botón clásico invisible** | `X = 0` · `Y = 0` · `Width = Parent.TemplateWidth - 10` · `Height = 202` · `Text = ""` · `Fill = RGBA(0; 0; 0; 0)` · `HoverFill = RGBA(21; 127; 91; 0,06)` · `BorderColor = RGBA(0; 0; 0; 0)` |

### `btnAbrirArea_Seg` → `OnSelect` — ya no navega, cambia de contenedor
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

## J.4 `cntVSegDetalle` — Detalle del área

| Orden | Nivel | Control | Tipo exacto | Propiedades |
|---|---|---|---|---|
| 1 | 3 | `cntEncabezado` | **Contenedor de diseño VERTICAL** | `FillPortions = 0` · `Height = 34` · `AlignInContainer = AlignInContainer.Stretch` · `LayoutGap = 0` |
| 1.1 | 4 | `lblDetGente` | Etiqueta | `FillPortions = 0` · `Height = 32` · `AlignInContainer = AlignInContainer.Stretch` · `Size = 12` · `Color = ColorValue(nfCol.Muted)` |
| 2 | 3 | `galSalidasArea` | **Galería en blanco HORIZONTAL** | `FillPortions = 0` · `Height = 176` · `AlignInContainer = AlignInContainer.Stretch` · `Layout = Layout.Horizontal` · `TemplateSize = 222` · `TemplatePadding = 5` · `Items = colSalArea` |
| 3 | 3 | `cntAccionesKit` | **Contenedor de diseño HORIZONTAL** | `FillPortions = 0` · `Height = 44` · `AlignInContainer = AlignInContainer.Stretch` · `LayoutGap = 8` · `LayoutAlignItems = LayoutAlignItems.Center` · `LayoutWrap = true` |
| 3.1 | 4 | `lblSalTtl` | Etiqueta | `FillPortions = 1` · `LayoutMinWidth = 140` · `Height = 22` · `Size = 13` · `FontWeight = FontWeight.Semibold` |
| 3.2 | 4 | `btnTodoEntregado` | Botón clásico | `FillPortions = 0` · `Width = 168` · `Height = 34` · `Text = "Todo ENTREGADO"` · `Fill = ColorValue(nfCol.Surface)` · `Color = ColorValue(nfCol.Ink)` · `BorderThickness = 1` · `BorderColor = ColorValue(nfCol.Line)` · `RadiusTopLeft…RadiusBottomRight = 8` |
| 3.3 | 4 | `btnCerrarTodo` | Botón clásico | `FillPortions = 0` · `Width = 130` · `Height = 34` · `Text = "Cerrar todo"` · mismo estilo |
| 4 | 3 | `galRenglones` | **Galería en blanco VERTICAL de ALTO FLEXIBLE** | `FillPortions = 1` · `AlignInContainer = AlignInContainer.Stretch` · `Layout = Layout.Vertical` · `TemplatePadding = 4` · `Items = colDetSal` · **sin `TemplateSize` ni `WrapCount`** |

### `lblDetGente` → `Text`
```powerapps
"Responsable " & varAreaVista.ResponsableArea.DisplayName &
"   ·   Encargado " & varAreaVista.EncargadoArea &
"   ·   " & varAreaVista.Cultivo.Value &
"   ·   " & CountRows(colSalArea) & " salida(s)"
```

### `lblSalTtl` → `Text`
```powerapps
varSalVista.NoSalida & "  ·  " & CountRows(colDetSal) & " renglones  ·  " &
Text(varSalVista.FechaEntrega; "dd/mm/yyyy")
```

### Plantilla de `galSalidasArea`

**Orden de capas:** `cntSal` → `btnSelSalida`

| Orden | Nivel | Control | Tipo exacto | Propiedades |
|---|---|---|---|---|
| 1 | 4 | `cntSal` | **Contenedor de diseño VERTICAL** | `X = 0` · `Y = 0` · `Width = Parent.TemplateWidth - 10` · `Height = 166` · `LayoutGap = 3` · `PaddingTop = 12` · `PaddingBottom = 12` · `PaddingLeft = 13` · `PaddingRight = 13` · `Fill = ColorValue(nfCol.Surface)` · `BorderThickness = 1` · `BorderColor = If(ThisItem.NoSalida = varSalVista.NoSalida; ColorValue(nfCol.Accent); ColorValue(nfCol.Line))` · `RadiusTopLeft…RadiusBottomRight = nfRadio` |
| 1.1 | 5 | `lblSalNo` | Etiqueta | `FillPortions = 0` · `Height = 16` · `Text = ThisItem.NoSalida` · `Size = 11` · `Color = ColorValue(nfCol.Accent)` |
| 1.2 | 5 | `lblSalFecha` | Etiqueta | `FillPortions = 0` · `Height = 22` · `Text = Text(ThisItem.FechaEntrega; "dd/mm/yyyy")` · `Size = 14` · `FontWeight = FontWeight.Semibold` |
| 1.3 | 5 | `lblSalTipo` | Etiqueta | `FillPortions = 0` · `Height = 18` · `Text = If(ThisItem.TipoCarga.Value = "KIT"; "Kit " & ThisItem.CodigoKit; "Entrega por unidad")` · `Size = 11,5` · `Color = ColorValue(nfCol.Muted)` |
| 1.4 | 5 | `lblSalAvance` | Etiqueta | `FillPortions = 0` · `Height = 20` · `Size = 10` · `Align = Align.Center` · `Width = 120` · `AlignInContainer = AlignInContainer.Start` · `Fill = ColorValue(nfCol.SlateSoft)` · `Color = ColorValue(nfCol.Slate)` · `RadiusTopLeft…RadiusBottomRight = 999` |
| 1.5 | 5 | `lblSalGente` | Etiqueta | `FillPortions = 1` · `AlignInContainer = AlignInContainer.Stretch` · `Size = 10,5` · `Color = ColorValue(nfCol.Faint)` |
| 2 | 4 | `btnSelSalida` | **Botón clásico invisible** | `X = 0` · `Y = 0` · `Width = Parent.TemplateWidth - 10` · `Height = 166` · `Fill = RGBA(0; 0; 0; 0)` · `HoverFill = RGBA(21; 127; 91; 0,06)` · `BorderColor = RGBA(0; 0; 0; 0)` |

```powerapps
/* lblSalAvance.Text */
With(
    {d: Filter(BG_SalidaDetalle; NoSalida = ThisItem.NoSalida)};
    CountRows(Filter(d; Abierta = false)) & "/" & CountRows(d) & " cerrados"
)

/* lblSalGente.Text */
"Responsable " & ThisItem.Responsable.DisplayName & Char(10) &
"Encargado " & ThisItem.Encargado & "  ·  Recibió " & ThisItem.Recibe

/* btnSelSalida.OnSelect */
Set(varSalVista; ThisItem);;
ClearCollect(colDetSal; Filter(BG_SalidaDetalle; NoSalida = ThisItem.NoSalida))
```

### Plantilla de `galRenglones` — **sin isla manual** (los botones son visibles)

| Orden | Nivel | Control | Tipo exacto | Propiedades |
|---|---|---|---|---|
| 1 | 4 | `cntRenglon` | **Contenedor de diseño VERTICAL** | `X = 0` · `Y = 0` · `Width = Parent.TemplateWidth - 8` · `LayoutGap = 6` · `LayoutMinHeight = 96` · `PaddingTop = 11` · `PaddingBottom = 11` · `PaddingLeft = 14` · `PaddingRight = 14` · `Fill = ColorValue(nfCol.Surface)` · `BorderThickness = 1` · `BorderColor = ColorValue(nfCol.Line2)` · `RadiusTopLeft…RadiusBottomRight = nfRadio` |
| 1.1 | 5 | `cntRenTitulo` | **Contenedor de diseño HORIZONTAL** | `FillPortions = 0` · `Height = 20` · `AlignInContainer = AlignInContainer.Stretch` · `LayoutGap = 8` |
| 1.1.a | 6 | `lblRenCod` | Etiqueta | `FillPortions = 0` · `Width = 80` · `Text = ThisItem.Codigo` · `Size = 10,5` · `Color = ColorValue(nfCol.Accent)` |
| 1.1.b | 6 | `lblRenDesc` | Etiqueta | `FillPortions = 1` · `Text = ThisItem.Descripcion` · `Size = 13,5` · `FontWeight = FontWeight.Semibold` |
| 1.2 | 5 | `cntRenPills` | **Contenedor de diseño HORIZONTAL** | `FillPortions = 0` · `Height = 44` · `AlignInContainer = AlignInContainer.Stretch` · `LayoutGap = 5` · **`LayoutWrap = true`** |
| 1.2.a | 6 | `lblRenEstatus` | Etiqueta | `FillPortions = 0` · `Width = 132` · `Height = 20` · `Text = ThisItem.Estatus.Value` · `Size = 10` · `Align = Align.Center` · `Fill = ColorValue(LookUp(nfEstatus; Estatus = ThisItem.Estatus.Value; Bg))` · `Color = ColorValue(LookUp(nfEstatus; Estatus = ThisItem.Estatus.Value; Txt))` · `RadiusTopLeft…RadiusBottomRight = 999` |
| 1.2.b | 6 | `lblRenTipo` | Etiqueta | `FillPortions = 0` · `Width = 96` · `Height = 20` · `Text = ThisItem.Tipo.Value` · `Size = 9` · `Align = Align.Center` |
| 1.2.c | 6 | `lblRenBodega` | Etiqueta | `FillPortions = 0` · `Width = 90` · `Height = 20` · `Text = ThisItem.Bodega.Value` · `Size = 9` · `Align = Align.Center` |
| 1.2.d | 6 | `lblRenCant` | Etiqueta | `FillPortions = 0` · `Width = 320` · `Height = 20` · `Size = 10,5` · `Color = ColorValue(nfCol.Muted)` |
| 1.3 | 5 | `lblRenComentario` | Etiqueta | `FillPortions = 0` · `AlignInContainer = AlignInContainer.Stretch` · `AutoHeight = true` · `Visible = !IsBlank(ThisItem.ComentarioLinea)` · `Text = ThisItem.ComentarioLinea` · `Size = 11,5` · `Italic = true` · `Color = ColorValue(nfCol.Muted)` |
| 1.4 | 5 | `cntRenAcciones` | **Contenedor de diseño HORIZONTAL** | `FillPortions = 0` · `Height = 34` · `AlignInContainer = AlignInContainer.Stretch` · `LayoutGap = 6` · **`LayoutWrap = true`** · `LayoutJustifyContent = LayoutJustifyContent.Start` |
| 1.4.a | 6 | `btnRenEntregar` | Botón clásico | `FillPortions = 0` · `Width = 96` · `Height = 30` · `Text = "Entregar"` · `Visible = ThisItem.Estatus.Value = "ASIGNADO"` |
| 1.4.b | 6 | `btnRenEnUso` | Botón clásico | `FillPortions = 0` · `Width = 118` · `Height = 30` · `Text = "Dejar en uso"` · `Visible = And(ThisItem.Estatus.Value = "ENTREGADO"; ThisItem.Tipo.Value = "RETORNABLE")` |
| 1.4.c | 6 | `btnRenConsumido` | Botón clásico | `FillPortions = 0` · `Width = 110` · `Height = 30` · `Text = "Consumido"` · `Visible = And(ThisItem.Tipo.Value = "UN SOLO USO"; Or(ThisItem.Estatus.Value = "ENTREGADO"; ThisItem.Estatus.Value = "ASIGNADO"))` |
| 1.4.d | 6 | `btnRenBaja` | Botón clásico | `FillPortions = 0` · `Width = 112` · `Height = 30` · `Text = "Dar de baja"` · `Visible = (ThisItem.CantDanada + ThisItem.CantExtraviada) > 0` · `Fill = RGBA(0; 0; 0; 0)` · `Color = ColorValue(nfCol.Red)` · `BorderThickness = 1` · `BorderColor = ColorValue(nfCol.Red)` |

```powerapps
/* lblRenCant.Text */
"Entregado " & ThisItem.CantEntregada & " " & ThisItem.Unidad &
If(ThisItem.CantBuena > 0;      "  ·  Bueno " & ThisItem.CantBuena; "") &
If(ThisItem.CantDanada > 0;     "  ·  Dañado " & ThisItem.CantDanada; "") &
If(ThisItem.CantExtraviada > 0; "  ·  Extraviado " & ThisItem.CantExtraviada; "") &
With(
    {p: ThisItem.CantEntregada - ThisItem.CantBuena - ThisItem.CantDanada - ThisItem.CantExtraviada};
    If(p > 0; "  ·  Pendiente " & p; "")
)
```

---

## J.5 `cntVDevolucion` — Registrar devolución

| Orden | Nivel | Control | Tipo exacto | Propiedades |
|---|---|---|---|---|
| 1 | 3 | `cntFiltrosDev` | **Contenedor de diseño VERTICAL** | `FillPortions = 0` · `Height = 118` · `AlignInContainer = AlignInContainer.Stretch` · `LayoutGap = 8` · `PaddingTop = 12` · `PaddingBottom = 12` · `PaddingLeft = 14` · `PaddingRight = 14` · `Fill = ColorValue(nfCol.Surface)` · `BorderThickness = 1` · `BorderColor = ColorValue(nfCol.Line)` · `RadiusTopLeft…RadiusBottomRight = nfRadio` |
| 1.1 | 4 | `cntAreaDevPicker` | **Contenedor MANUAL** | `FillPortions = 0` · `Height = 44` · `AlignInContainer = AlignInContainer.Stretch` · `Fill = ColorValue(nfCol.Surface2)` · `BorderThickness = 1` · `BorderColor = If(IsBlank(varAreaDev); ColorValue(nfCol.Red); ColorValue(nfCol.Line))` · `RadiusTopLeft…RadiusBottomRight = 8` |
| 1.1.a | 5 | `lblAreaDevEtq` | Etiqueta | `X = 12` · `Y = 5` · `Width = Parent.Width - 48` · `Height = 13` · `Text = "ÁREA"` · `Size = 9` · `FontWeight = FontWeight.Semibold` · `Color = ColorValue(nfCol.Faint)` |
| 1.1.b | 5 | `lblAreaDevNom` | Etiqueta | `X = 12` · `Y = 18` · `Width = Parent.Width - 48` · `Height = 20` · `Text = Coalesce(varAreaDev.Title & " · " & varAreaDev.NombreArea; "Toque para seleccionar")` · `Size = 13,5` · `FontWeight = FontWeight.Semibold` |
| 1.1.c | 5 | `icoAreaDevFlecha` | Icono `Icon.ChevronRight` | `X = Parent.Width - 30` · `Y = 12` · `Width = 20` · `Height = 20` · `Color = ColorValue(nfCol.Faint)` |
| 1.1.d | 5 | `btnAbrirAreaDev` | **Botón clásico invisible** | `X = 0` · `Y = 0` · `Width = Parent.Width` · `Height = Parent.Height` · `Fill = RGBA(0; 0; 0; 0)` · `HoverFill = RGBA(21; 127; 91; 0,06)` · `BorderColor = RGBA(0; 0; 0; 0)` · `OnSelect = Set(varPopup; "AREA")` |
| 1.2 | 4 | `cntAutoDev` | **Contenedor de diseño HORIZONTAL** | `FillPortions = 0` · `Height = 22` · `AlignInContainer = AlignInContainer.Stretch` · `LayoutGap = 16` · `LayoutWrap = true` |
| 1.2.a | 5 | `lblDevFechaAuto` | Etiqueta | `FillPortions = 1` · `Height = 18` · `Text = "Fecha de devolución: " & Text(Now(); "dd/mm/yyyy hh:mm") & " · automática"` · `Size = 11` · `Color = ColorValue(nfCol.Muted)` |
| 1.2.b | 5 | `lblDevRecibeAuto` | Etiqueta | `FillPortions = 1` · `Height = 18` · `Text = "Recibe (User): " & nfUser.Nombre` · `Size = 11` · `Color = ColorValue(nfCol.Muted)` |
| 2 | 3 | `galSalDev` | **Galería en blanco HORIZONTAL** | `FillPortions = 0` · `Height = 150` · `AlignInContainer = AlignInContainer.Stretch` · `Layout = Layout.Horizontal` · `TemplateSize = 222` · `TemplatePadding = 5` |
| 3 | 3 | `cntAccionesDev` | **Contenedor de diseño HORIZONTAL** | `FillPortions = 0` · `Height = 40` · `AlignInContainer = AlignInContainer.Stretch` · `LayoutGap = 8` · `LayoutAlignItems = LayoutAlignItems.Center` · `LayoutWrap = true` |
| 3.1 | 4 | `btnMarcarTodo` | Botón clásico | `FillPortions = 0` · `Width = 126` · `Height = 34` · `Text = "Marcar todo"` |
| 3.2 | 4 | `btnDesmarcarTodo` | Botón clásico | `FillPortions = 0` · `Width = 146` · `Height = 34` · `Text = "Desmarcar todo"` |
| 3.3 | 4 | `lblContadorDev` | Etiqueta | `FillPortions = 1` · `Height = 20` · `Align = Align.Right` · `Size = 11` · `Color = ColorValue(nfCol.Faint)` |
| 4 | 3 | `galDev` | **Galería en blanco VERTICAL** (cuadrícula) | `FillPortions = 1` · `AlignInContainer = AlignInContainer.Stretch` · `Layout = Layout.Vertical` · `WrapCount = Max(1; RoundDown(Self.Width / 330; 0))` · `TemplateSize = If(Self.Width < 720; 272; 250)` · `TemplatePadding = 5` · `Items = colDev` |
| 5 | 3 | `cntGuardarDev` | **Contenedor de diseño HORIZONTAL** | `FillPortions = 0` · `Height = 52` · `AlignInContainer = AlignInContainer.Stretch` · `LayoutJustifyContent = LayoutJustifyContent.End` · `LayoutAlignItems = LayoutAlignItems.Center` |
| 5.1 | 4 | `btnGuardarDev` | Botón clásico | `FillPortions = 0` · `Width = Min(260; Parent.Width - 20)` · `Height = 42` · `Text = "Registrar devolución"` · `Fill = ColorValue(nfCol.Accent)` · `Color = White` |

### `galSalDev` → `Items`
```powerapps
If(
    IsBlank(varAreaDev);
    Blank();
    Sort(
        Filter(BG_Salidas; CodigoArea = varAreaDev.Title; EstadoSalida.Value = "ABIERTA");
        FechaEntrega;
        SortOrder.Descending
    )
)
```

### Plantilla de `galDev`

**Orden de capas, del fondo al frente:** `cntDev` → `recCheckDev` → `icoCheckDev` → `btnMarcarLinea`

| Orden | Nivel | Control | Tipo exacto | Propiedades |
|---|---|---|---|---|
| 1 | 4 | `cntDev` | **Contenedor de diseño VERTICAL** | `X = 0` · `Y = 0` · `Width = Parent.TemplateWidth - 10` · `Height = Parent.TemplateHeight - 8` · `LayoutGap = 6` · `PaddingTop = 10` · `PaddingBottom = 10` · `PaddingLeft = 40` · `PaddingRight = 12` · `Fill = ColorValue(nfCol.Surface)` · `BorderThickness = 1` · `BorderColor = If(ThisItem.Marcado; ColorValue(nfCol.Accent); ColorValue(nfCol.Line))` · `RadiusTopLeft…RadiusBottomRight = nfRadio` |
| 1.1 | 5 | `lblDevCod` | Etiqueta | `FillPortions = 0` · `Height = 15` · `Text = ThisItem.Codigo` · `Size = 10,5` · `Color = ColorValue(nfCol.Accent)` |
| 1.2 | 5 | `lblDevDesc` | Etiqueta | `FillPortions = 0` · `Height = 20` · `Text = ThisItem.Descripcion` · `Size = 13` · `FontWeight = FontWeight.Semibold` |
| 1.3 | 5 | `cntDevPills` | **Contenedor de diseño HORIZONTAL** | `FillPortions = 0` · `Height = 20` · `AlignInContainer = AlignInContainer.Stretch` · `LayoutGap = 5` · `LayoutWrap = false` |
| 1.3.a | 6 | `lblDevEstatus` | Etiqueta | `FillPortions = 0` · `Width = 124` · `Height = 20` · `Text = ThisItem.Estatus` · `Size = 9,5` · `Align = Align.Center` |
| 1.3.b | 6 | `lblDevPend` | Etiqueta | `FillPortions = 1` · `Height = 20` · `Text = "Pendiente " & ThisItem.Pendiente & " de " & ThisItem.Entregada & " " & ThisItem.Unidad` · `Size = 10` · `Color = ColorValue(nfCol.Muted)` |
| 1.4 | 5 | `cntDevCampos` | **Contenedor de diseño VERTICAL** | `FillPortions = 1` · `AlignInContainer = AlignInContainer.Stretch` · `LayoutGap = 8` · `Visible = ThisItem.Marcado` |
| 1.4.1 | 6 | `cntDevNumeros` | **Contenedor de diseño HORIZONTAL** | `FillPortions = 0` · `Height = 54` · `AlignInContainer = AlignInContainer.Stretch` · `LayoutGap = 7` · `Visible = ThisItem.Tipo = "RETORNABLE"` |
| 1.4.1.a | 7 | `cntBuenas` | **Contenedor de diseño VERTICAL** | `FillPortions = 1` · `LayoutGap = 2` |
| | 8 | `lblEtqBuenas` | Etiqueta | `FillPortions = 0` · `Height = 14` · `Text = "BUENAS"` · `Size = 9` · `Align = Align.Center` · `Color = ColorValue(nfCol.Accent)` |
| | 8 | `txtBuenas` | **Text input moderno** | `FillPortions = 0` · `Height = 36` · `AlignInContainer = AlignInContainer.Stretch` · `Default = ThisItem.Buenas` · `Type = TextInputType.SingleLine` · `TriggerOutput = TriggerOutput.FocusOut` · `Align = Align.Center` · `Appearance = Appearance.Outline` · `BorderColor = ColorValue(nfCol.Accent)` · `Color = ColorValue(nfCol.AccentInk)` · `Size = 14` · `FontWeight = FontWeight.Semibold` · `RadiusTopLeft…RadiusBottomRight = 7` |
| 1.4.1.b | 7 | `cntDanadas` | **Contenedor de diseño VERTICAL** | `FillPortions = 1` · `LayoutGap = 2` → `lblEtqDanadas` (`Text = "DAÑADAS"`, `Color = ColorValue(nfCol.Amber)`) y `txtDanadas` (`BorderColor = ColorValue(nfCol.Amber)`, `Color = ColorValue(nfCol.Amber)`) |
| 1.4.1.c | 7 | `cntPerdidas` | **Contenedor de diseño VERTICAL** | `FillPortions = 1` · `LayoutGap = 2` → `lblEtqPerdidas` (`Text = "EXTRAVIADAS"`, `Color = ColorValue(nfCol.Red)`) y `txtPerdidas` (`BorderColor = ColorValue(nfCol.Red)`, `Color = ColorValue(nfCol.Red)`) |
| 1.4.2 | 6 | `lblDevSuma` | Etiqueta | `FillPortions = 0` · `Height = 18` · `AlignInContainer = AlignInContainer.Stretch` · `Size = 10,5` · `Visible = And(ThisItem.Marcado; ThisItem.Tipo = "RETORNABLE")` |
| 1.4.3 | 6 | `lblDevConsumo` | Etiqueta | `FillPortions = 0` · `Height = 24` · `Width = 260` · `AlignInContainer = AlignInContainer.Start` · `Visible = And(ThisItem.Marcado; ThisItem.Tipo = "UN SOLO USO")` · `Text = "Se cierra como CONSUMIDO · " & ThisItem.Pendiente & " " & ThisItem.Unidad` · `Size = 11` · `Align = Align.Center` · `Fill = ColorValue(nfCol.VioletSoft)` · `Color = ColorValue(nfCol.Violet)` · `RadiusTopLeft…RadiusBottomRight = 999` |
| 1.4.4 | 6 | `chkDarBaja` | Casilla | `FillPortions = 0` · `Height = 26` · `AlignInContainer = AlignInContainer.Stretch` · `Text = "Dar de baja las dañadas y extraviadas"` · `Color = ColorValue(nfCol.Red)` · `Visible = And(ThisItem.Marcado; (ThisItem.Danadas + ThisItem.Perdidas) > 0)` · `Default = ThisItem.DarBaja` |
| 1.4.5 | 6 | `txtComentDev` | **Text input moderno** | `FillPortions = 1` · `AlignInContainer = AlignInContainer.Stretch` · `Type = TextInputType.Multiline` · `Default = ThisItem.Comentario` · `MaxLength = 500` · `Appearance = Appearance.Outline` · `Size = 12` · `RadiusTopLeft…RadiusBottomRight = 7` |
| 2 | 4 | `recCheckDev` | Rectángulo | `X = 12` · `Y = 12` · `Width = 20` · `Height = 20` · `Fill = If(ThisItem.Marcado; ColorValue(nfCol.Accent); ColorValue(nfCol.Surface2))` · `BorderThickness = 1` · `BorderColor = If(ThisItem.Marcado; ColorValue(nfCol.Accent); ColorValue(nfCol.Line))` · `RadiusTopLeft…RadiusBottomRight = 5` |
| 3 | 4 | `icoCheckDev` | Icono `Icon.Check` | `X = 15` · `Y = 15` · `Width = 14` · `Height = 14` · `Color = White` · `Visible = ThisItem.Marcado` |
| 4 | 4 | `btnMarcarLinea` | **Botón clásico invisible** | `X = 0` · `Y = 0` · `Width = Parent.TemplateWidth - 10` · **`Height = 62`** · `Fill = RGBA(0; 0; 0; 0)` · `HoverFill = RGBA(21; 127; 91; 0,06)` · `BorderColor = RGBA(0; 0; 0; 0)` |

> `btnMarcarLinea` mide **62 px**, no toda la tarjeta: así `cntDevCampos` queda libre y las tres cajas reciben el toque.

---

## J.6 `cntVIngresos` — Ingresos de Bodega General

| Orden | Nivel | Control | Tipo exacto | Propiedades |
|---|---|---|---|---|
| 1 | 3 | `cntIngEncabezado` | **Contenedor de diseño HORIZONTAL** | `FillPortions = 0` · `Height = 44` · `AlignInContainer = AlignInContainer.Stretch` · `LayoutGap = 10` · `LayoutAlignItems = LayoutAlignItems.Center` · `LayoutWrap = true` |
| 1.1 | 4 | `lblIngTtl` | Etiqueta | `FillPortions = 1` · `Height = 22` · `Text = CountRows(colPorIngresar) & " renglón(es) pendientes de ingresar"` · `Size = 13` · `FontWeight = FontWeight.Semibold` |
| 1.2 | 4 | `btnIngresarTodo` | Botón clásico | `FillPortions = 0` · `Width = 236` · `Height = 36` · `Text = "Ingresar todos los pendientes"` · `Fill = ColorValue(nfCol.Surface)` · `Color = ColorValue(nfCol.Ink)` · `BorderThickness = 1` · `BorderColor = ColorValue(nfCol.Line)` · `DisplayMode = If(IsEmpty(colPorIngresar); DisplayMode.Disabled; DisplayMode.Edit)` |
| 2 | 3 | `galIngresos` | **Galería en blanco VERTICAL de ALTO FLEXIBLE** | `FillPortions = 1` · `AlignInContainer = AlignInContainer.Stretch` · `Layout = Layout.Vertical` · `TemplatePadding = 4` · `Items = colPorIngresar` |

### Plantilla de `galIngresos` — contenedor automático puro

| Orden | Nivel | Control | Tipo exacto | Propiedades |
|---|---|---|---|---|
| 1 | 4 | `cntIng` | **Contenedor de diseño VERTICAL** | `X = 0` · `Y = 0` · `Width = Parent.TemplateWidth - 8` · `LayoutGap = 8` · `LayoutMinHeight = 118` · `PaddingTop = 12` · `PaddingBottom = 12` · `PaddingLeft = 14` · `PaddingRight = 14` · `Fill = ColorValue(nfCol.Surface)` · `BorderThickness = 1` · `BorderColor = ColorValue(nfCol.Line)` · `RadiusTopLeft…RadiusBottomRight = nfRadio` |
| 1.1 | 5 | `lblIngCod` | Etiqueta | `FillPortions = 0` · `Height = 16` · `Text = ThisItem.Codigo & If(ThisItem.EsNuevo; "   ·   CÓDIGO NUEVO"; "")` · `Size = 10,5` · `Color = If(ThisItem.EsNuevo; ColorValue(nfCol.Blue); ColorValue(nfCol.Accent))` |
| 1.2 | 5 | `lblIngDesc` | Etiqueta | `FillPortions = 0` · `Height = 20` · `Text = ThisItem.Descripcion` · `Size = 13,5` · `FontWeight = FontWeight.Semibold` |
| 1.3 | 5 | `lblIngDatos` | Etiqueta | `FillPortions = 0` · `Height = 18` · `Size = 11` · `Color = ColorValue(nfCol.Muted)` |
| 1.4 | 5 | `cntIngSelectores` | **Contenedor de diseño HORIZONTAL** | `FillPortions = 0` · `Height = 40` · `AlignInContainer = AlignInContainer.Stretch` · `LayoutGap = 8` · **`LayoutWrap = true`** |
| 1.4.a | 6 | `galIngTipo` | **Galería en blanco HORIZONTAL** | `FillPortions = 0` · `Width = 272` · `Height = 38` · `Layout = Layout.Horizontal` · `TemplateSize = 134` |
| 1.4.b | 6 | `galIngBodega` | **Galería en blanco HORIZONTAL** | `FillPortions = 0` · `Width = 252` · `Height = 38` · `Layout = Layout.Horizontal` · `TemplateSize = 124` |
| 1.4.c | 6 | `btnIngresar` | Botón clásico | `FillPortions = 0` · `Width = 120` · `Height = 36` · `Text = "Ingresar"` · `Fill = ColorValue(nfCol.Accent)` · `Color = White` · `RadiusTopLeft…RadiusBottomRight = 8` |

```powerapps
/* lblIngDatos.Text */
"SALIDAS #" & ThisItem.IdGeneral & "   ·   " & ThisItem.Cantidad & " " & ThisItem.Unidad &
"   ·   Área " & ThisItem.AreaUso & "   ·   " & ThisItem.Responsable

/* galIngTipo.Items  — se pasa el Id de la fila padre para no usar ThisItem anidado (regla E11) */
ForAll(nfTipos As t; {Valor: t.Value; Fila: ThisItem.IdGeneral})

/* galIngBodega.Items */
ForAll(nfBodegas As b; {Valor: b.Value; Fila: ThisItem.IdGeneral})
```

Plantilla de `galIngTipo` y `galIngBodega` (idéntica, cambian solo los nombres):

| Orden | Nivel | Control | Tipo | Propiedades |
|---|---|---|---|---|
| 1 | 7 | `recIngChip` | Rectángulo | `X = 0` · `Y = 0` · `Width = Parent.TemplateWidth - 6` · `Height = 36` · `Fill = If(ThisItem.Valor = LookUp(colPorIngresar; IdGeneral = ThisItem.Fila; Tipo); ColorValue(nfCol.AccentSoft); ColorValue(nfCol.Surface2))` · `BorderThickness = 1` · `BorderColor = If(ThisItem.Valor = LookUp(colPorIngresar; IdGeneral = ThisItem.Fila; Tipo); ColorValue(nfCol.Accent); ColorValue(nfCol.Line))` · `RadiusTopLeft…RadiusBottomRight = 999` |
| 2 | 7 | `lblIngChip` | Etiqueta | `X = 0` · `Y = 8` · `Width = Parent.TemplateWidth - 6` · `Height = 20` · `Text = ThisItem.Valor` · `Size = 10` · `Align = Align.Center` |
| 3 | 7 | `btnIngChip` | **Botón clásico invisible** | `X = 0` · `Y = 0` · `Width = Parent.TemplateWidth - 6` · `Height = 36` · `Fill = RGBA(0; 0; 0; 0)` · `BorderColor = RGBA(0; 0; 0; 0)` · `OnSelect = UpdateIf(colPorIngresar; IdGeneral = ThisItem.Fila; {Tipo: ThisItem.Valor})` |

> En `galIngBodega` el `OnSelect` del chip es `UpdateIf(colPorIngresar; IdGeneral = ThisItem.Fila; {BodegaDestino: ThisItem.Valor})` y el `Fill` compara contra `BodegaDestino`.

---

## J.7 `cntVCatalogo` — Catálogo por tipo y bodega

| Orden | Nivel | Control | Tipo exacto | Propiedades |
|---|---|---|---|---|
| 1 | 3 | `cntFiltrosCat` | **Contenedor de diseño VERTICAL** | `FillPortions = 0` · `Height = 100` · `AlignInContainer = AlignInContainer.Stretch` · `LayoutGap = 10` · `PaddingTop = 12` · `PaddingBottom = 12` · `PaddingLeft = 14` · `PaddingRight = 14` · `Fill = ColorValue(nfCol.Surface)` · `BorderThickness = 1` · `BorderColor = ColorValue(nfCol.Line)` · `RadiusTopLeft…RadiusBottomRight = nfRadio` |
| 1.1 | 4 | `cntCatTipos` | **Contenedor de diseño HORIZONTAL** | `FillPortions = 0` · `Height = 36` · `AlignInContainer = AlignInContainer.Start` · `LayoutGap = 6` · `LayoutWrap = true` |
| 1.1.a | 5 | `btnCatTodos` | Botón clásico | `FillPortions = 0` · `Width = 110` · `Height = 34` · `Text = "Todos"` · `Fill = If(varTipoCat = ""; ColorValue(nfCol.Accent); ColorValue(nfCol.Surface2))` · `Color = If(varTipoCat = ""; White; ColorValue(nfCol.Muted))` · `OnSelect = Set(varTipoCat; "")` |
| 1.1.b | 5 | `btnCatRet` | Botón clásico | `FillPortions = 0` · `Width = 136` · `Height = 34` · `Text = "Retornable"` · `Fill = If(varTipoCat = "RETORNABLE"; ColorValue(nfCol.Accent); ColorValue(nfCol.Surface2))` · `OnSelect = Set(varTipoCat; "RETORNABLE")` |
| 1.1.c | 5 | `btnCatUso` | Botón clásico | `FillPortions = 0` · `Width = 142` · `Height = 34` · `Text = "Un solo uso"` · `Fill = If(varTipoCat = "UN SOLO USO"; ColorValue(nfCol.Accent); ColorValue(nfCol.Surface2))` · `OnSelect = Set(varTipoCat; "UN SOLO USO")` |
| 1.2 | 4 | `cntCatBusqueda` | **Contenedor de diseño HORIZONTAL** | `FillPortions = 0` · `Height = 38` · `AlignInContainer = AlignInContainer.Stretch` · `LayoutGap = 10` · `LayoutWrap = true` |
| 1.2.a | 5 | `txtCatBuscar` | **Text input moderno** | `FillPortions = 1` · `LayoutMinWidth = 170` · `Height = 36` · `AlignInContainer = AlignInContainer.Center` · `Type = TextInputType.Search` · `Placeholder = "Código o descripción"` · `TriggerOutput = TriggerOutput.Delayed` · `OnChange = Set(varBuscarCat; Self.Text)` |
| 1.2.b | 5 | `galCatBodega` | **Galería en blanco HORIZONTAL** | `FillPortions = 0` · `Width = 380` · `Height = 38` · `Layout = Layout.Horizontal` · `TemplateSize = 126` · `Items = nfBodegasF` |
| 2 | 3 | `galCatalogo` | **Galería en blanco VERTICAL** | `FillPortions = 1` · `AlignInContainer = AlignInContainer.Stretch` · `Layout = Layout.Vertical` · `TemplateSize = 76` · `TemplatePadding = 4` · `WrapCount = 1` |

### Plantilla de `galCatalogo` — contenedor automático puro

| Orden | Nivel | Control | Tipo exacto | Propiedades |
|---|---|---|---|---|
| 1 | 4 | `cntCat` | **Contenedor de diseño HORIZONTAL** | `X = 0` · `Y = 0` · `Width = Parent.TemplateWidth - 8` · `Height = 70` · `LayoutGap = 10` · `LayoutAlignItems = LayoutAlignItems.Center` · `PaddingLeft = 14` · `PaddingRight = 14` · `Fill = ColorValue(nfCol.Surface)` · `BorderThickness = 1` · `BorderColor = ColorValue(nfCol.Line)` · `RadiusTopLeft…RadiusBottomRight = nfRadio` |
| 1.1 | 5 | `cntCatTexto` | **Contenedor de diseño VERTICAL** | `FillPortions = 1` · `LayoutMinWidth = 160` · `LayoutGap = 2` · `AlignInContainer = AlignInContainer.Center` |
| 1.1.a | 6 | `lblCatCod` | Etiqueta | `FillPortions = 0` · `Height = 16` · `Text = ThisItem.Codigo` · `Size = 10,5` · `Color = ColorValue(nfCol.Accent)` |
| 1.1.b | 6 | `lblCatDesc` | Etiqueta | `FillPortions = 0` · `Height = 20` · `Text = ThisItem.Descripcion` · `Size = 13` · `FontWeight = FontWeight.Semibold` |
| 1.2 | 5 | `lblCatTipo` | Etiqueta | `FillPortions = 0` · `Width = 94` · `Height = 20` · `AlignInContainer = AlignInContainer.Center` · `Text = ThisItem.Tipo` · `Size = 9` · `Align = Align.Center` |
| 1.3 | 5 | `lblCatBodega` | Etiqueta | `FillPortions = 0` · `Width = 90` · `Height = 20` · `AlignInContainer = AlignInContainer.Center` · `Text = ThisItem.Bodega` · `Size = 9` · `Align = Align.Center` |
| 1.4 | 5 | `lblCatExistencia` | Etiqueta | `FillPortions = 0` · `Width = 96` · `Height = 24` · `AlignInContainer = AlignInContainer.Center` · `Text = ThisItem.Existencia & " " & ThisItem.Unidad` · `Size = 13` · `Align = Align.Right` · `FontWeight = FontWeight.Semibold` · `Color = If(ThisItem.Existencia <= ThisItem.Minimo; ColorValue(nfCol.Amber); ColorValue(nfCol.Ink))` |
| 1.5 | 5 | `lblCatAfuera` | Etiqueta | `FillPortions = 0` · `Width = 86` · `Height = 20` · `AlignInContainer = AlignInContainer.Center` · `Text = If(ThisItem.Afuera > 0; ThisItem.Afuera & " afuera"; "—")` · `Size = 10,5` · `Align = Align.Right` · `Color = ColorValue(nfCol.Muted)` |
| 1.6 | 5 | `lblCatEstado` | Etiqueta | `FillPortions = 0` · `Width = 86` · `Height = 20` · `AlignInContainer = AlignInContainer.Center` · `Text = ThisItem.Estado` · `Size = 9,5` · `Align = Align.Center` · `Fill = ColorValue(LookUp(nfEstatus; Estatus = ThisItem.Estado; Bg))` · `Color = ColorValue(LookUp(nfEstatus; Estatus = ThisItem.Estado; Txt))` · `RadiusTopLeft…RadiusBottomRight = 999` |

### `galCatalogo` → `Items`
```powerapps
ForAll(
    Filter(
        BG_Catalogo;
        Or(varTipoCat = ""; Tipo.Value = varTipoCat);
        Or(varBodCat = "TODAS"; Bodega.Value = varBodCat);
        Or(
            IsBlank(varBuscarCat);
            StartsWith(Codigo; varBuscarCat);
            StartsWith(Descripcion; varBuscarCat)
        )
    ) As a;
    {
        Clave:       a.Clave;
        Codigo:      a.Codigo;
        Descripcion: a.Descripcion;
        Tipo:        a.Tipo.Value;
        Categoria:   a.Categoria.Value;
        Bodega:      a.Bodega.Value;
        Unidad:      a.Unidad;
        Existencia:  a.Existencia;
        Minimo:      a.Minimo;
        Estado:      a.EstadoArticulo.Value;
        Afuera: Sum(
            Filter(colDetAb; Clave = a.Clave);
            CantEntregada - CantBuena - CantDanada - CantExtraviada
        )
    }
)
```
> `colDetAb` se llena al entrar a Seguimiento. Para que Catálogo funcione sin pasar por ahí, agregue al enrutador de I.4: `If(ThisItem.Pant = "CAT"; ClearCollect(colDetAb; nfDetAbierto));;`

---

## J.8 `cntVBajas` — Artículos dados de baja

| Orden | Nivel | Control | Tipo exacto | Propiedades |
|---|---|---|---|---|
| 1 | 3 | `lblBajasTtl` | Etiqueta | `FillPortions = 0` · `Height = 24` · `AlignInContainer = AlignInContainer.Stretch` · `Text = CountRows(colBajas) & " artículo(s) fuera del inventario"` · `Size = 13` · `FontWeight = FontWeight.Semibold` |
| 2 | 3 | `galBajas` | **Galería en blanco VERTICAL de ALTO FLEXIBLE** | `FillPortions = 1` · `AlignInContainer = AlignInContainer.Stretch` · `Layout = Layout.Vertical` · `TemplatePadding = 4` · `Items = colBajas` |

### Plantilla de `galBajas`

| Orden | Nivel | Control | Tipo exacto | Propiedades |
|---|---|---|---|---|
| 1 | 4 | `cntBaja` | **Contenedor de diseño VERTICAL** | `X = 0` · `Y = 0` · `Width = Parent.TemplateWidth - 8` · `LayoutGap = 5` · `LayoutMinHeight = 110` · `PaddingTop = 12` · `PaddingBottom = 12` · `PaddingLeft = 14` · `PaddingRight = 14` · `Fill = ColorValue(nfCol.Surface)` · `BorderThickness = 1` · `BorderColor = ColorValue(nfCol.Line)` · `RadiusTopLeft…RadiusBottomRight = nfRadio` |
| 1.1 | 5 | `lblBajaTitulo` | Etiqueta | `FillPortions = 0` · `Height = 20` · `Text = ThisItem.Title & "   ·   " & ThisItem.Descripcion` · `Size = 13,5` · `FontWeight = FontWeight.Semibold` |
| 1.2 | 5 | `lblBajaDatos` | Etiqueta | `FillPortions = 0` · `Height = 18` · `Size = 11` · `Color = ColorValue(nfCol.Muted)` |
| 1.3 | 5 | `lblBajaMotivo` | Etiqueta | `FillPortions = 0` · `AlignInContainer = AlignInContainer.Stretch` · `AutoHeight = true` · `Text = ThisItem.Motivo` · `Size = 11,5` · `Italic = true` · `Color = ColorValue(nfCol.Muted)` |
| 1.4 | 5 | `cntBajaPie` | **Contenedor de diseño HORIZONTAL** | `FillPortions = 0` · `Height = 32` · `AlignInContainer = AlignInContainer.Stretch` · `LayoutGap = 10` · `LayoutAlignItems = LayoutAlignItems.Center` · `LayoutWrap = true` |
| 1.4.a | 6 | `lblBajaQuien` | Etiqueta | `FillPortions = 1` · `Height = 18` · `Text = "Reportó " & ThisItem.Reporto & "   ·   Registró " & ThisItem.RegistroPor.DisplayName` · `Size = 10,5` · `Color = ColorValue(nfCol.Faint)` |
| 1.4.b | 6 | `btnReactivar` | Botón clásico | `FillPortions = 0` · `Width = 160` · `Height = 30` · `Text = "Reactivar artículo"` · `Fill = RGBA(0; 0; 0; 0)` · `Color = ColorValue(nfCol.Accent)` · `BorderThickness = 1` · `BorderColor = ColorValue(nfCol.Accent)` |

```powerapps
/* lblBajaDatos.Text */
Text(ThisItem.FechaBaja; "dd/mm/yyyy") & "   ·   " & ThisItem.Cantidad & " pza(s)   ·   " &
ThisItem.Bodega.Value & "   ·   " & ThisItem.NoSalida & "   ·   " & ThisItem.CodigoArea
```

---

## J.9 `cntVFlujo` — Flujo de estatus

| Orden | Nivel | Control | Tipo exacto | Propiedades |
|---|---|---|---|---|
| 1 | 3 | `cntFlujoRet` | **Contenedor de diseño VERTICAL** | `FillPortions = 0` · `Height = 108` · `AlignInContainer = AlignInContainer.Stretch` · `LayoutGap = 8` · `PaddingTop = 14` · `PaddingBottom = 14` · `PaddingLeft = 14` · `PaddingRight = 14` · `Fill = ColorValue(nfCol.Surface)` · `BorderThickness = 1` · `BorderColor = ColorValue(nfCol.Line)` · `RadiusTopLeft…RadiusBottomRight = nfRadio` |
| 1.1 | 4 | `lblFlujoRetTtl` | Etiqueta | `FillPortions = 0` · `Height = 22` · `Text = "RETORNABLE — herramienta, equipo, caja, basurero, bandeja"` · `Size = 11` · `FontWeight = FontWeight.Semibold` · `Color = ColorValue(nfCol.Faint)` |
| 1.2 | 4 | `galFlujoRet` | **Galería en blanco HORIZONTAL** | `FillPortions = 1` · `AlignInContainer = AlignInContainer.Stretch` · `Layout = Layout.Horizontal` · `TemplateSize = 158` · `ShowScrollbar = true` |
| 2 | 3 | `cntFlujoUso` | **Contenedor de diseño VERTICAL** | igual que `cntFlujoRet`, `Height = 108` |
| 2.1 | 4 | `lblFlujoUsoTtl` | Etiqueta | `FillPortions = 0` · `Height = 22` · `Text = "UN SOLO USO — insumo, guantes, cloro, bolsa de polinización"` · `Size = 11` · `FontWeight = FontWeight.Semibold` · `Color = ColorValue(nfCol.Faint)` |
| 2.2 | 4 | `galFlujoUso` | **Galería en blanco HORIZONTAL** | `FillPortions = 1` · `Layout = Layout.Horizontal` · `TemplateSize = 158` |
| 3 | 3 | `galLeyenda` | **Galería en blanco VERTICAL de ALTO FLEXIBLE** | `FillPortions = 1` · `AlignInContainer = AlignInContainer.Stretch` · `Layout = Layout.Vertical` · `WrapCount = Max(1; RoundDown(Self.Width / 300; 0))` · `TemplateSize = 72` · `Items = nfLeyenda` |

### `galFlujoRet` → `Items`
```powerapps
Table(
    {Paso: 1; Est: "ASIGNADO"};
    {Paso: 2; Est: "ENTREGADO"};
    {Paso: 3; Est: "EN USO"};
    {Paso: 4; Est: "DEVUELTO PARCIAL"};
    {Paso: 5; Est: "DEVUELTO"}
)
```

### `galFlujoUso` → `Items`
```powerapps
Table(
    {Paso: 1; Est: "ASIGNADO"};
    {Paso: 2; Est: "ENTREGADO"};
    {Paso: 3; Est: "CONSUMIDO"}
)
```

### Plantilla de `galFlujoRet` y `galFlujoUso` (la misma)

| Orden | Nivel | Control | Tipo exacto | Propiedades |
|---|---|---|---|---|
| 1 | 5 | `cntPaso` | **Contenedor de diseño HORIZONTAL** | `X = 0` · `Y = 0` · `Width = Parent.TemplateWidth` · `Height = 34` · `LayoutGap = 6` · `LayoutAlignItems = LayoutAlignItems.Center` |
| 1.1 | 6 | `lblPasoEst` | Etiqueta | `FillPortions = 1` · `Height = 24` · `Text = ThisItem.Est` · `Size = 10` · `Align = Align.Center` · `Fill = ColorValue(LookUp(nfEstatus; Estatus = ThisItem.Est; Bg))` · `Color = ColorValue(LookUp(nfEstatus; Estatus = ThisItem.Est; Txt))` · `RadiusTopLeft…RadiusBottomRight = 999` |
| 1.2 | 6 | `lblPasoFlecha` | Etiqueta | `FillPortions = 0` · `Width = 16` · `Height = 24` · `Text = "→"` · `Align = Align.Center` · `Color = ColorValue(nfCol.Faint)` · `Visible = ThisItem.Paso < CountRows(Parent.AllItems)` |

### `App.Formulas` → agregue `nfLeyenda`
```powerapps
nfLeyenda = Table(
    {Est: "ASIGNADO";         Desc: "Guardado en la salida pero todavía no sale físicamente. Ya descuenta existencia."};
    {Est: "ENTREGADO";        Desc: "Salió de la bodega y lo recibió quien lo fue a traer."};
    {Est: "EN USO";           Desc: "Queda en el área todo el ciclo, no se entrega a diario."};
    {Est: "DEVUELTO PARCIAL"; Desc: "Regresaron menos piezas de las entregadas. El saldo sigue pendiente."};
    {Est: "DEVUELTO";         Desc: "Completo y bueno. Regresa a la existencia de su bodega."};
    {Est: "DAÑADO";           Desc: "Regresó con daño. Cantidad propia, comentario obligatorio."};
    {Est: "EXTRAVIADO";       Desc: "No regresó y no se localiza. Comentario y responsable obligatorios."};
    {Est: "DE BAJA";          Desc: "Sale del inventario. Queda registrado en Bajas."};
    {Est: "CONSUMIDO";        Desc: "Un solo uso, gastado en la labor. Cierra sin devolución."};
    {Est: "VENCIDO";          Desc: "Pasaron los días de préstamo y sigue afuera. Lo calcula la app."}
);
```

### Plantilla de `galLeyenda`

| Orden | Nivel | Control | Tipo exacto | Propiedades |
|---|---|---|---|---|
| 1 | 4 | `cntLeyenda` | **Contenedor de diseño HORIZONTAL** | `X = 0` · `Y = 0` · `Width = Parent.TemplateWidth - 8` · `Height = 66` · `LayoutGap = 10` · `LayoutAlignItems = LayoutAlignItems.Start` · `PaddingTop = 10` · `PaddingBottom = 10` |
| 1.1 | 5 | `lblLeyEst` | Etiqueta | `FillPortions = 0` · `Width = 132` · `Height = 22` · `Text = ThisItem.Est` · `Size = 9,5` · `Align = Align.Center` · `Fill = ColorValue(LookUp(nfEstatus; Estatus = ThisItem.Est; Bg))` · `Color = ColorValue(LookUp(nfEstatus; Estatus = ThisItem.Est; Txt))` · `RadiusTopLeft…RadiusBottomRight = 999` |
| 1.2 | 5 | `lblLeyDesc` | Etiqueta | `FillPortions = 1` · `AlignInContainer = AlignInContainer.Stretch` · `Text = ThisItem.Desc` · `Size = 11,5` · `Color = ColorValue(nfCol.Muted)` |

---

# PARTE K — Orden de construcción desde donde va

| Paso | Qué hacer |
|---|---|
| 1 | Reemplace el bloque `nfCol` en `App.Formulas` (H.3) y agregue `nfLeyenda` (J.9) |
| 2 | Retoque los 5 controles de color del menú (H.3) |
| 3 | Corrija `cntNav.DropShadow` a `ExtraBold` (H.1) |
| 4 | Renombre la pantalla a `scrPrincipal` y ponga `App.StartScreen = scrPrincipal` |
| 5 | Haga dinámicos `lblCrumb`, `lblTitulo`, `icoAtras` (I.3) |
| 6 | Pegue el enrutador en `btnNavItem.OnSelect` (I.4) |
| 7 | Inserte los **nueve** contenedores de vista como hijos de `cntCuerpo`, después de `cntBarra`, con las propiedades de J.0 |
| 8 | Llene `cntVMenu` (J.1) y pegue el mismo enrutador en `btnTile.OnSelect` |
| 9 | Llene `cntVAsignar` (J.2), luego los tres popups |
| 10 | Llene `cntVSeguimiento` (J.3) y `cntVSegDetalle` (J.4) |
| 11 | Llene `cntVDevolucion` (J.5) |
| 12 | Llene `cntVIngresos` (J.6), `cntVCatalogo` (J.7), `cntVBajas` (J.8), `cntVFlujo` (J.9) |
| 13 | Ordene las capas de la pantalla y de cada plantilla según las tablas de H.2 |

### Prueba de que el cambio de vista funciona
1. Toque cada entrada del menú: debe cambiar el contenido, el título y la miga, **sin parpadeo de pantalla** (ya no hay `Navigate`).
2. En Seguimiento, toque un área: entra a `SEGDET` y la flecha de atrás regresa a `SEG`, no al menú.
3. En teléfono, toque una entrada del menú: el menú se cierra solo y el contenido cambia.
4. Entre a Ingresos, luego a Catálogo y de vuelta a Ingresos: la lista de pendientes debe recargarse cada vez (lo hace el enrutador).

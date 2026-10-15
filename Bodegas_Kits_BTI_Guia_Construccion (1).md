# BODEGAS & KITS BTI — Guía de construcción
### Power Apps Canvas · localización español (`;` separador · `;;` encadenado · coma decimal) · SharePoint

---

# PARTE 0 — Auditoría: qué se hizo mal en Mantenimiento Preventivo y cómo lo evitamos aquí

Estos son los errores reales que costaron retrabajo en `DBASIG` / `REVISIONMANT` / `procesos01`. Cada uno se convierte en una regla obligatoria para esta app.

| # | Error cometido | Por qué pasó | Regla para Bodegas & Kits |
|---|---|---|---|
| **E1** | `Patch(DBASIG; Defaults(DBASIG); …)` en el botón Guardar → **un registro duplicado cada vez que se guardaba** el mismo activo | El `Patch` nunca supo si era alta o edición | **Bandera de modo obligatoria.** `varModo` = `"NUEVO"` / `"EDITAR"` y `If(varModo = "NUEVO"; Defaults(L); LookUp(L; ID = varReg.ID))`. En esta app la cabecera se crea **una sola vez** con `Set(varSalida; Patch(...))` y de ahí en adelante todo es `Patch(BG_Salidas; varSalida; …)` |
| **E2** | `Set(varRegMant; {…})` con **forma de registro distinta** según el tipo (PREVENTIVO no traía `COMENTARIO`) → `htmlInfoMant` reventaba | No había una plantilla única de registro | **Plantilla única en `App.Formulas`.** `nfLineaVacia` declara los 14 campos; todo `Collect`/`Set` nace de ella. Si un campo no aplica, va en blanco pero **existe** |
| **E3** | Al reabrir la tarjeta, `varRegMant` se reconstruía con `ESTADO: {Value:"PENDIENTE"}` fijo → **se perdía el seguimiento** | Literales en lugar de lectura del registro vivo | **`Coalesce(existente.Campo; valorInicial)`** siempre. Nunca un literal al reabrir algo que ya existe |
| **E4** | Las tarjetas KPI leían `DB` (activos) en lugar de `DBASIG` (tareas), y dejaban fuera `REVISIONMANT` | Cada tarjeta consultaba su propia fuente | **Una sola colección de verdad.** Los KPI y las galerías de resumen leen `colDetAbierto`, nunca la lista directo |
| **E5** | `Filter(DBASIG; Year(CREADO) = varAnio)` → **no delega** | Función sobre la columna | **Siempre `Campo >= ini And Campo <= fin`** sobre columna Fecha **indexada**. Nunca `Year()`, `Month()`, `Text()` dentro de `Filter` |
| **E6** | `ForAll(Filter(...))` truncado en 500 filas **sin ningún aviso** | Límite de filas de datos por defecto | **Límite de filas = 2000** + columna bandera `Abierta` (Sí/No, indexada) para que el filtro traiga decenas, no miles |
| **E7** | `Select(btnOculto)` usado para orquestar → **bug del doble clic en `procesos01`** | `Select()` se ejecuta **asíncrono**: lo que sigue corre antes de que termine | **Cero `Select()` para orquestar.** Si una rutina se repite, se repite el código (ver *Rutinas* en la Parte 9). Es más largo y es correcto |
| **E8** | `FechaFin` guardada como texto `"dd/mm/yyyy"` → fechas corruptas al leerlas | Se guardó `Text(fecha; formato)` en columna de texto | **Columnas Fecha y hora** en SharePoint; se escribe `Now()` o `.SelectedDate`. `Text()` **solo** para mostrar en una etiqueta |
| **E9** | Popup con adjuntos desplazados y `ManualLayout` mezclado con `AutoLayout` | El popup era hijo del contenedor del cuerpo | **El popup es hijo de la PANTALLA**, nunca de `cntCuerpo`. **Todos** los contenedores de esta app son `LayoutMode = Manual` |
| **E10** | `GroupBy` / `Ungroup` con **comillas dobles** → error de fórmula | En Power Fx moderno los nombres de columna de estas dos funciones son **identificadores**, no texto | **Comillas simples siempre:** `GroupBy(tabla; 'CodigoArea'; 'Lineas')` · `Ungroup(tabla; 'Lineas')` |
| **E11** | `ThisItem` dentro de `ForAll` anidado apuntaba al registro equivocado | Ámbito implícito | **Operador `As` obligatorio** en todo `ForAll` y en todo `Filter` que se anide: `ForAll(colSel As s; …)` |
| **E12** | `AddColumns` / `ShowColumns` sobre colecciones → tipos inestables | Proyección implícita | **`ForAll` con construcción explícita de registro.** Ya es su estándar; aquí no hay una sola excepción |

### Dos errores de diseño que usted detectó en mi propuesta anterior y también quedan como regla

| # | Error | Regla |
|---|---|---|
| **E13** | Área en un **Combo box** | Con 20+ áreas el desplegable se corta dentro del popup. **Área = botón + popup propio con galería y buscador.** Nunca un desplegable |
| **E14** | Controles clásicos de entrada | Toda **entrada de datos** usa controles **modernos** (Combo box, Text input). Todo lo **pintado** (contenedores, rectángulos, etiquetas, botones invisibles) sigue siendo **clásico**, porque el botón moderno no tiene `Fill` transparente ni `HoverFill` |

---

# PARTE 1 — SharePoint: estructura y los 3 cambios que habilitan la delegación

### 1.1 `BG_Catalogo` — maestro por **código + bodega**

| Columna interna | Tipo | Índice | Nota |
|---|---|---|---|
| `Title` | Texto | ✔ | el **Código** |
| `Clave` | Texto | ✔ | `Codigo & "|" & Bodega` — **la escribe siempre la app** |
| `Codigo` | Texto | ✔ | |
| `Descripcion` | Texto | | |
| `Tipo` | Elección | | `RETORNABLE` · `UN SOLO USO` |
| `Categoria` | Elección | | HERRAMIENTA · EQUIPO · MATERIAL · INSUMO · MOBILIARIO · LIMPIEZA |
| `Bodega` | Elección | | `BODEGA 1` · `CONTENEDOR` |
| `Unidad` | Texto | | |
| `Existencia` | Número | | |
| `Minimo` | Número | | |
| `EstadoArticulo` | Elección | ✔ | `ACTIVO` · `DE BAJA` |

> `HER-001` existe dos veces: `HER-001|BODEGA 1` y `HER-001|CONTENEDOR`. Así siempre se sabe **dónde está** y el filtro por bodega delega.

### 1.2 `BG_Areas`
`Title` (CodigoArea ✔) · `NombreArea` · `Cultivo` (Elección) · `KitAsignado` (Texto) · `ResponsableArea` (Persona) · `EncargadoArea` (Persona) · `Activa` (Sí/No ✔)

### 1.3 `BG_Personal` — **lista nueva** (reemplaza al conector Office365 en el combo)
`Title` (nombre ✔) · `Correo` (Texto) · `Puesto` (Texto) · `AreaHabitual` (Texto) · `Activo` (Sí/No ✔)

> Con lista propia el Combo box moderno carga instantáneo, funciona sin conector y `DefaultSelectedItems` sí puede devolver la selección guardada. Con `Office365Usuarios.SearchUser` nada de eso funciona bien.

### 1.4 `BG_Kits` / `BG_KitDetalle`
`BG_Kits`: `Title` (CodigoKit ✔) · `NombreKit` · `Cultivo` · `Activo` (Sí/No ✔)
`BG_KitDetalle`: `Title` (CodigoKit ✔) · `Clave` (Texto ✔) · `CantidadEstandar` (Número)

### 1.5 `BG_Salidas` — cabecera
`Title` · `NoSalida` (Texto ✔) · `FechaEntrega` (**Fecha y hora** ✔) · `TipoCarga` (Elección: KIT · UNIDAD) · `CodigoKit` · `CodigoArea` (Texto ✔) · `ResponsableArea` (Persona) · `EncargadoArea` (Persona) · `Bodeguero` (Persona) · `Recoge` (Persona) · `Observacion` (multilínea) · `EstadoSalida` (Elección: ABIERTA · CERRADA ✔)

### 1.6 `BG_SalidaDetalle` — renglones (aquí vive el estatus)
`Title` · `NoSalida` (Texto ✔) · `CodigoArea` (Texto ✔) · `Clave` (Texto ✔) · `Codigo` · `Descripcion` · `Tipo` (Elección) · `Categoria` (Elección) · `Bodega` (Elección) · `Unidad` · `CantEntregada` (Nº) · `CantBuena` (Nº) · `CantDanada` (Nº) · `CantExtraviada` (Nº) · `Estatus` (Elección) · `ComentarioLinea` (multilínea) · `FechaDevolucion` (Fecha y hora) · `RecibeDevolucion` (Persona) · **`Abierta` (Sí/No ✔)** ← **CAMBIO CLAVE (E6)**

> **`Abierta` es la columna que salva la app.** La escribe la app: `true` al crear el renglón, `false` cuando pasa a DEVUELTO / CONSUMIDO / DE BAJA. `Filter(BG_SalidaDetalle; Abierta = true)` delega y trae decenas de filas en lugar de miles. Es el mismo truco que `FECHAPROXIMA` en Mantenimiento.

### 1.7 `BG_Ingresos`
`Title` (NoIngreso) · `IdSalidaGeneral` (Número ✔) · `FechaIngreso` (Fecha y hora) · `Codigo` · `Descripcion` · `Cantidad` (Nº) · `Unidad` · `Tipo` (Elección) · `BodegaDestino` (Elección) · `AreaUso` · `Responsable` · `IngresadoPor` (Persona)

### 1.8 `SALIDAS` — lista existente de Bodega General
Se le agrega **una sola columna**: `ProcesadoBodega` (Sí/No, predeterminado No, **indexada**).

### 1.9 `BG_Bajas`
`Title` (Codigo) · `Clave` · `Descripcion` · `Bodega` (Elección) · `Cantidad` (Nº) · `FechaBaja` (Fecha y hora) · `NoSalida` · `CodigoArea` · `Motivo` (multilínea) · `Reporto` (Persona) · `RegistroPor` (Persona)

### 1.10 Configuración de la app
`Configuración → General → Límite de filas de datos = 2000`
`Configuración → Próximas características → Fórmulas con nombre (Named formulas) = Activado`

---

# PARTE 2 — `App.Formulas`: aquí vive el 80 % de la lógica

**Por qué esto optimiza la app:** una fórmula con nombre **no se ejecuta al abrir**, se evalúa cuando alguien la lee y **se recalcula sola** cuando cambian sus insumos. Resultado: `OnStart` baja de 11 `ClearCollect` a cero, la app abre de inmediato, y **desaparecen los datos rancios** — ya no hace falta ningún botón de refrescar ni repetir `ClearCollect(colCatalogo; …)` en cada `OnSelect` (ese fue el origen del retrabajo en Mantenimiento).

> Restricciones reales: una fórmula con nombre **no puede** usar `Set`, `Patch`, `Collect`, `Notify`, `Navigate`, ni `Self`/`Parent`/`ThisItem`. Solo valores y consultas.

```powerapps
/* ===================== 1. TEMA (hex como TEXTO) ===================== */
nfCol = {
    Bg:          "#EDF0EC";
    Surface:     "#FFFFFF";
    Surface2:    "#F6F8F5";
    Surface3:    "#EAEEE9";
    Ink:         "#151E1A";
    Muted:       "#5B6862";
    Faint:       "#8B968F";
    Line:        "#D4DBD5";
    Line2:       "#E6EAE4";
    Nav:         "#14241D";
    Nav2:        "#1D3329";
    NavInk:      "#DCE8E1";
    NavMut:      "#8AA398";
    Accent:      "#1E6B4C";
    AccentSoft:  "#DDEBE3";
    AccentInk:   "#0F4331";
    Blue:        "#2A5C88";
    BlueSoft:    "#DBE7F1";
    Amber:       "#A4630D";
    AmberSoft:   "#F6E6CD";
    Red:         "#9C2A25";
    RedSoft:     "#F4DBD9";
    Violet:      "#57468A";
    VioletSoft:  "#E5E0F0";
    Slate:       "#4A585F";
    SlateSoft:   "#E1E7E9"
};

/* ===================== 2. MEDIDAS Y RESPONSIVO ===================== */
nfCorteMovil  = 760;      // debajo de esto el menú se vuelve superpuesto
nfAnchoNav    = 212;
nfAltoBarra   = 54;
nfRadio       = 10;
nfPad         = 16;

/* ===================== 3. USUARIO EN SESIÓN = BODEGUERO ===================== */
nfUser = {
    Nombre: User().FullName;
    Correo: Lower(User().Email);
    Claim:  "i:0#.f|membership|" & Lower(User().Email)
};

/* ===================== 4. COLORES POR ESTATUS ===================== */
nfEstatus = Table(
    {Estatus: "ASIGNADO";         Txt: "#2A5C88"; Bg: "#DBE7F1"; Orden: 1};
    {Estatus: "ENTREGADO";        Txt: "#0F4331"; Bg: "#DDEBE3"; Orden: 2};
    {Estatus: "EN USO";           Txt: "#0F4331"; Bg: "#DDEBE3"; Orden: 3};
    {Estatus: "DEVUELTO PARCIAL"; Txt: "#A4630D"; Bg: "#F6E6CD"; Orden: 4};
    {Estatus: "DAÑADO";           Txt: "#A4630D"; Bg: "#F6E6CD"; Orden: 5};
    {Estatus: "EXTRAVIADO";       Txt: "#9C2A25"; Bg: "#F4DBD9"; Orden: 6};
    {Estatus: "VENCIDO";          Txt: "#A4630D"; Bg: "#F6E6CD"; Orden: 7};
    {Estatus: "DEVUELTO";         Txt: "#4A585F"; Bg: "#E1E7E9"; Orden: 8};
    {Estatus: "CONSUMIDO";        Txt: "#57468A"; Bg: "#E5E0F0"; Orden: 9};
    {Estatus: "DE BAJA";          Txt: "#9C2A25"; Bg: "#F4DBD9"; Orden: 10};
    {Estatus: "ACTIVO";           Txt: "#0F4331"; Bg: "#DDEBE3"; Orden: 11}
);

/* ===================== 5. PLANTILLA ÚNICA DE RENGLÓN (regla E2) =====================
   Todo Collect/Set de una línea del carrito nace de aquí con Patch(nfLineaVacia; {…}).
   Así ninguna rama puede quedarse sin un campo.                                      */
nfLineaVacia = {
    Clave:       "";
    Codigo:      "";
    Descripcion: "";
    Tipo:        "";
    Categoria:   "";
    Bodega:      "";
    Unidad:      "";
    Existencia:  0;
    Estandar:    1;
    Cantidad:    1
};

/* ===================== 6. PLANTILLA ÚNICA DE LÍNEA DE DEVOLUCIÓN ===================== */
nfDevVacia = {
    IdLinea:     0;
    Clave:       "";
    Codigo:      "";
    Descripcion: "";
    Tipo:        "";
    Bodega:      "";
    Unidad:      "";
    Estatus:     "";
    Entregada:   0;
    Pendiente:   0;
    Marcado:     false;
    Buenas:      0;
    Danadas:     0;
    Perdidas:    0;
    Comentario:  "";
    DarBaja:     false
};

/* ===================== 7. DATOS MAESTROS — se recalculan solos =====================
   Ninguno de estos necesita ClearCollect ni botón de refrescar.                      */
nfCatalogo = Filter(BG_Catalogo; EstadoArticulo.Value = "ACTIVO");
nfAreas    = Sort(Filter(BG_Areas; Activa = true); Title; SortOrder.Ascending);
nfPersonal = Sort(Filter(BG_Personal; Activo = true); Title; SortOrder.Ascending);
nfKits     = Filter(BG_Kits; Activo = true);

/* ===================== 8. TRABAJO ABIERTO — delega por la columna Abierta ===================== */
nfDetAbierto    = Filter(BG_SalidaDetalle; Abierta = true);
nfSalAbiertas   = Sort(Filter(BG_Salidas; EstadoSalida.Value = "ABIERTA"); FechaEntrega; SortOrder.Descending);
nfPorIngresar   = Filter(SALIDAS; ProcesadoBodega = false);

/* ===================== 9. CONTADORES PARA EL MENÚ Y LAS INSIGNIAS ===================== */
nfCtaAbiertas   = CountRows(nfSalAbiertas);
nfCtaIngresos   = CountRows(nfPorIngresar);
nfCtaBajas      = CountRows(BG_Bajas);
nfCtaAfuera     = Sum(nfDetAbierto; CantEntregada - CantBuena - CantDanada - CantExtraviada);
nfCtaDanadas    = Sum(nfDetAbierto; CantDanada);
nfCtaPerdidas   = Sum(nfDetAbierto; CantExtraviada);

/* ===================== 10. MENÚ LATERAL (una sola definición) ===================== */
nfMenu = Table(
    {Pant: "MENU";  Etiqueta: "Menú principal";       Ico: "HOME";  Cta: 0};
    {Pant: "ASIG";  Etiqueta: "Asignar kit / unidad"; Ico: "ADD";   Cta: 0};
    {Pant: "SEG";   Etiqueta: "Seguimiento";          Ico: "TREND"; Cta: nfCtaAbiertas};
    {Pant: "DEV";   Etiqueta: "Devolución";           Ico: "BACK";  Cta: nfCtaAbiertas};
    {Pant: "ING";   Etiqueta: "Ingresos B. General";  Ico: "DOWN";  Cta: nfCtaIngresos};
    {Pant: "CAT";   Etiqueta: "Catálogo";             Ico: "ITEMS"; Cta: 0};
    {Pant: "BAJA";  Etiqueta: "Bajas";                Ico: "TRASH"; Cta: nfCtaBajas};
    {Pant: "FLUJO"; Etiqueta: "Flujo de estatus";     Ico: "FLOW";  Cta: 0}
);

/* ===================== 11. BOTONES DEL MENÚ PRINCIPAL ===================== */
nfBotonesMenu = Table(
    {
        Pant: "ASIG"; Ico: "ADD"; Titulo: "Asignar kit o unidad";
        Detalle: "Kit completo o artículo por artículo. El área y quien recoge se piden una sola vez.";
        Cta: 0
    };
    {
        Pant: "SEG"; Ico: "TREND"; Titulo: "Seguimiento por área";
        Detalle: "Galería horizontal de invernaderos con todo lo que tienen afuera.";
        Cta: nfCtaAbiertas
    };
    {
        Pant: "DEV"; Ico: "BACK"; Titulo: "Registrar devolución";
        Detalle: "Separe piezas buenas, dañadas y extraviadas del mismo renglón.";
        Cta: nfCtaAbiertas
    };
    {
        Pant: "ING"; Ico: "DOWN"; Titulo: "Ingresos de Bodega General";
        Detalle: "Aplique los renglones de SALIDAS y acumule existencia por código.";
        Cta: nfCtaIngresos
    };
    {
        Pant: "CAT"; Ico: "ITEMS"; Titulo: "Catálogo";
        Detalle: "Retornable o un solo uso, en Bodega 1 o en el Contenedor.";
        Cta: 0
    };
    {
        Pant: "BAJA"; Ico: "TRASH"; Titulo: "Artículos de baja";
        Detalle: "Lo dañado o extraviado que ya salió del inventario.";
        Cta: nfCtaBajas
    }
);

/* ===================== 12. CONSTANTES DE LISTAS ===================== */
nfTipos      = ["RETORNABLE"; "UN SOLO USO"];
nfBodegas    = ["BODEGA 1"; "CONTENEDOR"];
nfBodegasF   = ["TODAS"; "BODEGA 1"; "CONTENEDOR"];
nfCategorias = ["HERRAMIENTA"; "EQUIPO"; "MATERIAL"; "INSUMO"; "MOBILIARIO"; "LIMPIEZA"];
nfEstFisico  = ["BUENO"; "DAÑADO"; "EXTRAVIADO"];
nfDiasPrestamo = 30;      // a partir de aquí el renglón se marca VENCIDO
```

---

# PARTE 3 — `App.OnStart`: solo estado, nada de datos

```powerapps
/* Solo variables de estado de la interfaz. Ni un ClearCollect de datos maestros:
   eso lo resuelven las fórmulas con nombre de la Parte 2.                        */

Set(varPantalla;     "MENU");;
Set(varNavAbierto;   false);;
Set(varPopup;        "");;          /* ""  |  "CARRITO"  |  "AREA"  |  "KIT" */
Set(varModoCarga;    "KIT");;
Set(varTipo;         "RETORNABLE");;
Set(varBodegaFiltro; "TODAS");;
Set(varBuscar;       "");;
Set(varGuardando;    false);;

Set(varKit;       First(nfKits));;
Set(varArea;      Blank());;        /* lo llena el popup de área, no un combo */
Set(varRecoge;    Blank());;
Set(varAreaVista; Blank());;
Set(varSalVista;  Blank());;
Set(varAreaDev;   Blank());;
Set(varSalDev;    Blank());;

Clear(colSel);;                     /* carrito de entrega */
Clear(colCatSel);;                  /* chips de categoría activos */
Clear(colDev)                       /* líneas de devolución en edición */
```

### Otras propiedades de `App`

| Propiedad | Valor |
|---|---|
| `StartScreen` | `scrMenu` ← **nunca `Navigate` en `OnStart`** |
| `SizeBreakpoints` | `[600; 760; 1024; 1366]` |
| `OnError` | `Notify("Ocurrió un error: " & FirstError.Message; NotificationType.Error; 4000)` |

---

# PARTE 4 — Catálogo de controles: cuál usar y cuál NO

| Necesidad | Control **correcto** | **No usar** | Por qué |
|---|---|---|---|
| Posicionar cualquier bloque | **Contenedor** clásico con `LayoutMode = Manual` | Contenedor de diseño automático | mezclarlos desplaza el popup (E9) |
| Elegir **área / invernadero** | `cntPopArea` → `galAreaPop` + botón oculto | Combo box, Dropdown | 20+ áreas; el desplegable se corta dentro del popup (E13) |
| Elegir **kit** | `cntPopKit` → `galKitPop` horizontal | Dropdown | se ve nombre, cultivo y # de renglones |
| Elegir **quien recoge** | **Combo box moderno** `cmbRecoge` | Combo box clásico | búsqueda incremental, `ItemDisplayText`, `ValidationState` |
| Buscar texto | **Text input moderno** `txtBuscar` | — | salida sigue siendo `.Text` |
| Comentario largo | **Text input moderno** con `Type: TextInputType.Multiline` | — | |
| Cantidad + / − | **Botón clásico** + Etiqueta | Number input | el clásico acepta `Fill` transparente |
| Selección de tarjeta | **Botón clásico invisible** encima de la plantilla | Botón moderno | el moderno no tiene `Fill` transparente, `HoverFill` ni `PressedFill` (E14) |
| Check de selección | **Rectángulo** + **Icono** `Icon.Check` | Casilla (Checkbox) | la selección la gobierna el botón oculto, no dos controles peleándose |
| Pastilla de estatus | **Etiqueta** con `Fill` + `RadiusTopLeft…` | — | |
| Fecha de entrega / devolución | **ninguno**: se escribe `Now()` | Date picker | la fecha es automática por definición |
| Cifras y barras de resumen | **Etiqueta** y **Rectángulo** dentro de galería | Html text | menos puntos de falla que el HTML (E2) |

### Propiedades reales del **Combo box moderno** (las verifiqué en la documentación)

| Propiedad | Para qué | Cuidado |
|---|---|---|
| `Items` | origen | |
| **`ItemDisplayText`** | texto visible de cada fila, p. ej. `ThisItem.Title` | **`Fields` ya no existe** — fue eliminada |
| `DefaultSelectedItems` | selección inicial; **tiene que ser registros de `Items`** | |
| `InputTextPlaceholder` | texto guía | **no** es `HintText` |
| `SelectMultiple` | `false` para selección única — **el valor por defecto es `true`** | si no lo pone, deja escoger varios |
| `IsSearchable` | `true` activa el buscador | |
| `Selected` / `SelectedItems` / `SearchText` | **salidas de solo lectura** | |
| `Required` / `ValidationState` | validación visual | `ValidationState.Error` / `.None` |
| `OnChange` | se dispara en cada selección | |
| `Appearance` | `Appearance.Outline` / `.FilledLighter` / `.FilledDarker` | |
| `Fill` · `BorderColor` · `Color` · `BorderThickness` | sí se puede pintar | |
| `RadiusTopLeft` · `RadiusTopRight` · `RadiusBottomLeft` · `RadiusBottomRight` | esquinas | **`BorderRadius` ya no existe** |
| `Size` | tamaño de letra en puntos | **`FontSize` fue renombrada a `Size`** |
| `Color` | color de letra | **`FontColor` fue renombrada a `Color`** |

### Propiedades reales del **Text input moderno**

| Propiedad | Valor |
|---|---|
| `Default` | valor inicial |
| `Text` | **la salida sigue siendo `.Text`**, no `.Value` |
| `Placeholder` | texto guía |
| `Type` | `TextInputType.SingleLine` · `.Multiline` · `.Password` · `.Search` ← **no hay propiedad `Mode`** |
| `TriggerOutput` | `TriggerOutput.Keypress` (predeterminado) · `.FocusOut` · `.Delayed` |
| `MaxLength` · `Required` · `ValidationState` · `DisplayMode` | |
| `Appearance` · `Fill` · `BorderColor` · `Color` · `Size` · `RadiusTopLeft…` | |

> **Dropdown moderno**, si llega a usarlo: la selección inicial se llama **`Default`**, no `DefaultSelectedItems`.

---

# PARTE 5 — Jerarquía de contenedores: el esqueleto de TODAS las pantallas

Este árbol se construye una vez y se copia a cada pantalla. Los seis hijos de la pantalla son **hermanos**, no anidados, y su orden lo decide el `ZIndex`. Así el popup siempre tapa al menú y el menú siempre tapa al cuerpo.

```
scrXxxx                                    ← Pantalla   Fill = ColorValue(nfCol.Bg)
│
├── NIVEL 1 · cntCuerpo          Contenedor Manual      ZIndex 1
│   │                            X = If(Parent.Width < nfCorteMovil; 0; nfAnchoNav)
│   │                            Y = 0
│   │                            Width = Parent.Width - Self.X
│   │                            Height = Parent.Height
│   │
│   ├── NIVEL 2 · cntBarra       Contenedor Manual      ZIndex 10
│   │   │                        X=0 · Y=0 · Width=Parent.Width · Height=nfAltoBarra
│   │   ├── NIVEL 3 · icoNav         Icono Hamburger
│   │   ├── NIVEL 3 · icoAtras       Icono ChevronLeft
│   │   ├── NIVEL 3 · lblCrumb       Etiqueta
│   │   └── NIVEL 3 · lblTitulo      Etiqueta
│   │
│   ├── NIVEL 2 · cntFiltros     Contenedor Manual      ZIndex 5
│   │   │                        X=nfPad · Y=nfAltoBarra+14
│   │   │                        Width=Parent.Width-(nfPad*2) · Height=(variable por pantalla)
│   │   └── NIVEL 3 · (botones de modo, chips, buscador, botón de área…)
│   │
│   └── NIVEL 2 · galPrincipal   Galería                ZIndex 4
│       │                        X=nfPad
│       │                        Y=cntFiltros.Y + cntFiltros.Height + 12
│       │                        Width=Parent.Width-(nfPad*2)
│       │                        Height=Parent.Height - Self.Y - nfPad
│       └── NIVEL 3 · cntItem    Contenedor Manual  ← la plantilla de la galería
│           ├── NIVEL 4 · recFranja      Rectángulo        ZIndex 1
│           ├── NIVEL 4 · recCheck       Rectángulo        ZIndex 2
│           ├── NIVEL 4 · icoCheck       Icono Check       ZIndex 3
│           ├── NIVEL 4 · lblCod         Etiqueta          ZIndex 3
│           ├── NIVEL 4 · lblDesc        Etiqueta          ZIndex 3
│           ├── NIVEL 4 · lblTagTipo     Etiqueta          ZIndex 3
│           ├── NIVEL 4 · lblTagBodega   Etiqueta          ZIndex 3
│           ├── NIVEL 4 · lblStock       Etiqueta          ZIndex 3
│           ├── NIVEL 4 · btnSelItem     Botón invisible   ZIndex 5   ← tapa todo lo de arriba
│           └── NIVEL 4 · cntCantidad    Contenedor Manual ZIndex 10  ← SOBRE el botón oculto
│               ├── NIVEL 5 · btnMenos   Botón
│               ├── NIVEL 5 · lblCant    Etiqueta
│               └── NIVEL 5 · btnMas     Botón
│
├── NIVEL 1 · recScrimNav        Rectángulo             ZIndex 80
├── NIVEL 1 · cntNav             Contenedor Manual      ZIndex 90
│   ├── NIVEL 2 · recNavLogo / lblNavTitulo / lblNavSub
│   ├── NIVEL 2 · galNav         Galería
│   │   └── NIVEL 3 · recNavItem / icoNavItem / lblNavItem / lblNavCta / btnNavItem (ZIndex 20)
│   └── NIVEL 2 · lblNavUser     Etiqueta
│
├── NIVEL 1 · recScrimPop        Rectángulo             ZIndex 100
├── NIVEL 1 · cntPopCarrito      Contenedor Manual      ZIndex 110
│   ├── NIVEL 2 · lblPopTit / icoPopX
│   ├── NIVEL 2 · galPopLineas   Galería
│   │   └── NIVEL 3 · cntPopLinea → NIVEL 4 · etiquetas + cntPopCant + icoPopQuitar
│   └── NIVEL 2 · cntPopPie      Contenedor Manual
│       └── NIVEL 3 · lblPopTotal / btnAbrirArea / cmbRecoge / lblAuto* / txtObs / btnPopMas / btnPopGuardar
│
├── NIVEL 1 · recScrimArea       Rectángulo             ZIndex 120
└── NIVEL 1 · cntPopArea         Contenedor Manual      ZIndex 130   ← popup SOBRE popup
    ├── NIVEL 2 · lblAreaTit / icoAreaX
    ├── NIVEL 2 · txtBuscarArea  Text input moderno
    └── NIVEL 2 · galAreaPop     Galería
        └── NIVEL 3 · cntAreaFila → NIVEL 4 · etiquetas + icoAreaCheck + btnElegirArea (ZIndex 20)
```

### Las 4 reglas de oro de este esqueleto

1. **`cntCuerpo`, los scrims y los popups son hermanos** — hijos directos de la pantalla. Si mete el popup dentro de `cntCuerpo`, el velo no cubre el menú y los controles se desplazan (E9).
2. **El `ZIndex` del botón oculto es 5; el del contenedor de cantidad es 10.** Si se invierte, el botón se come los `+` / `−`. Esto aplica igual en `galDev` y en `galNav`.
3. **Nadie usa `Y` absoluta después del primer control.** Siempre `ControlAnterior.Y + ControlAnterior.Height + separación`. Así cambiar la altura de los filtros no rompe la galería.
4. **Todo contenedor tiene `LayoutMode = Manual`.** Sin excepción.

### Esqueleto común — fórmulas

#### `cntCuerpo` — Contenedor Manual (Nivel 1)
```powerapps
X       = If(Parent.Width < nfCorteMovil; 0; nfAnchoNav)
Y       = 0
Width   = Parent.Width - Self.X
Height  = Parent.Height
Fill    = ColorValue(nfCol.Bg)
ZIndex  = 1
```

#### `recScrimNav` — Rectángulo (Nivel 1)
```powerapps
X = 0 · Y = 0 · Width = Parent.Width · Height = Parent.Height
Fill     = RGBA(10; 20; 15; 0,55)
Visible  = And(Parent.Width < nfCorteMovil; varNavAbierto)
OnSelect = Set(varNavAbierto; false)
ZIndex   = 80
```

#### `cntNav` — Contenedor Manual (Nivel 1)
```powerapps
X       = 0
Y       = 0
Width   = If(Parent.Width < nfCorteMovil; Min(268; Parent.Width * 0,86); nfAnchoNav)
Height  = Parent.Height
Fill    = ColorValue(nfCol.Nav)
Visible = Or(Parent.Width >= nfCorteMovil; varNavAbierto)
ZIndex  = 90
```

#### `galNav` — Galería en blanco **vertical** (Nivel 2, dentro de `cntNav`)
```powerapps
Items         = nfMenu
X             = 8
Y             = 68
Width         = Parent.Width - 16
Height        = Parent.Height - 140
TemplateSize  = 40
TemplatePadding = 2
ShowScrollbar = false
```

#### `recNavItem` — Rectángulo (Nivel 3)
```powerapps
X = 0 · Y = 0 · Width = Parent.TemplateWidth · Height = 38
Fill = If(ThisItem.Pant = varPantalla; ColorValue(nfCol.Accent); RGBA(0; 0; 0; 0))
RadiusTopLeft = 8 · RadiusTopRight = 8 · RadiusBottomLeft = 8 · RadiusBottomRight = 8
```

#### `icoNavItem` — Icono (Nivel 3)
```powerapps
Icon = Switch(
    ThisItem.Ico;
    "HOME";  Icon.Home;
    "ADD";   Icon.Add;
    "TREND"; Icon.TrendingUp;
    "BACK";  Icon.Reload;
    "DOWN";  Icon.Download;
    "ITEMS"; Icon.Items;
    "TRASH"; Icon.Trash;
    Icon.Hierarchy
)
X = 10 · Y = 11 · Width = 17 · Height = 17
Color = If(ThisItem.Pant = varPantalla; ColorValue("#06140E"); ColorValue(nfCol.NavMut))
```

#### `lblNavItem` — Etiqueta (Nivel 3)
```powerapps
Text  = ThisItem.Etiqueta
X = 34 · Y = 9 · Width = Parent.TemplateWidth - 76 · Height = 20
Size  = 12,5
Color = If(ThisItem.Pant = varPantalla; ColorValue("#06140E"); ColorValue(nfCol.NavMut))
FontWeight = If(ThisItem.Pant = varPantalla; FontWeight.Semibold; FontWeight.Normal)
```

#### `lblNavCta` — Etiqueta (Nivel 3)
```powerapps
Text    = ThisItem.Cta
Visible = ThisItem.Cta > 0
X = Parent.TemplateWidth - 40 · Y = 11 · Width = 30 · Height = 16
Align = Align.Center · Size = 10
Fill  = If(ThisItem.Pant = varPantalla; RGBA(0; 0; 0; 0,18); RGBA(255; 255; 255; 0,14))
Color = If(ThisItem.Pant = varPantalla; ColorValue("#06140E"); ColorValue(nfCol.NavInk))
RadiusTopLeft = 4 · RadiusTopRight = 4 · RadiusBottomLeft = 4 · RadiusBottomRight = 4
```

#### `btnNavItem` — **Botón clásico invisible** (Nivel 3)
```powerapps
X = 0 · Y = 0 · Width = Parent.TemplateWidth · Height = 38
Text               = ""
Fill               = RGBA(0; 0; 0; 0)
HoverFill          = RGBA(255; 255; 255; 0,08)
PressedFill        = RGBA(255; 255; 255; 0,14)
BorderColor        = RGBA(0; 0; 0; 0)
HoverBorderColor   = RGBA(0; 0; 0; 0)
PressedBorderColor = RGBA(0; 0; 0; 0)
ZIndex             = 20

OnSelect =
If(Parent.Width < nfCorteMovil; Set(varNavAbierto; false));;
Switch(
    ThisItem.Pant;
    "MENU";  Navigate(scrMenu;                ScreenTransition.None);
    "ASIG";  Navigate(scrAsignar;             ScreenTransition.None);
    "SEG";   Navigate(scrSeguimiento;         ScreenTransition.None);
    "DEV";   Navigate(scrDevolucion;          ScreenTransition.None);
    "ING";   Navigate(scrIngresos;            ScreenTransition.None);
    "CAT";   Navigate(scrCatalogo;            ScreenTransition.None);
    "BAJA";  Navigate(scrBajas;               ScreenTransition.None);
    "FLUJO"; Navigate(scrFlujo;               ScreenTransition.None)
)
```

#### `cntBarra` y sus hijos (Nivel 2 y 3)
```powerapps
/* cntBarra */
X = 0 · Y = 0 · Width = Parent.Width · Height = nfAltoBarra
Fill = ColorValue(nfCol.Surface)
BorderThickness = 1 · BorderColor = ColorValue(nfCol.Line) · BorderStyle = BorderStyle.Solid

/* icoNav — Icono Hamburger */
X = 12 · Y = 12 · Width = 30 · Height = 30
Color    = ColorValue(nfCol.Muted)
OnSelect = Set(varNavAbierto; !varNavAbierto)

/* icoAtras — Icono ChevronLeft */
X = 48 · Y = 12 · Width = 30 · Height = 30
Visible  = varPantalla <> "MENU"
Color    = ColorValue(nfCol.Muted)
OnSelect = If(
    varPantalla = "SEGDET";
    Navigate(scrSeguimiento; ScreenTransition.None);
    Navigate(scrMenu; ScreenTransition.None)
)

/* lblCrumb — cambie solo el literal en cada pantalla */
Text = "MOVIMIENTOS"
X = If(varPantalla = "MENU"; 50; 84) · Y = 9 · Width = Parent.Width - 200 · Height = 14
Size = 9,5 · Color = ColorValue(nfCol.Faint)

/* lblTitulo — cambie solo el literal en cada pantalla */
Text = "Asignar kit o unidad"
X = If(varPantalla = "MENU"; 50; 84) · Y = 24 · Width = Parent.Width - 200 · Height = 22
Size = 15 · FontWeight = FontWeight.Semibold · Color = ColorValue(nfCol.Ink)
```

---

# PARTE 6 — `scrMenu`

### `scrMenu.OnVisible`
```powerapps
Set(varPantalla; "MENU")
```
> Nada más. Los KPI son fórmulas con nombre (`nfCtaAfuera`, `nfCtaDanadas`…) y se recalculan solos. Esto es lo que en Mantenimiento exigía cinco `ClearCollect` por pantalla.

### `galKpi` — Galería en blanco **horizontal** (Nivel 2 de `cntCuerpo`)
```powerapps
Items = Table(
    {Val: nfCtaAbiertas;  Etq: "SALIDAS ABIERTAS";     Col: nfCol.Ink};
    {Val: nfCtaAfuera;    Etq: "PIEZAS AFUERA";        Col: nfCol.Ink};
    {Val: nfCtaDanadas;   Etq: "PIEZAS DAÑADAS";       Col: nfCol.Amber};
    {Val: nfCtaPerdidas;  Etq: "PIEZAS EXTRAVIADAS";   Col: nfCol.Red};
    {Val: nfCtaIngresos;  Etq: "INGRESOS POR APLICAR"; Col: nfCol.Blue}
)
X = nfPad · Y = nfAltoBarra + 16
Width  = Parent.Width - (nfPad * 2)
Height = 94
TemplateSize = If(Parent.Width < nfCorteMovil; 152; (Self.Width - 8) / 5)
TemplatePadding = 4
```
Plantilla (Nivel 3 y 4):
```powerapps
/* recKpi — Rectángulo */
X = 0 · Y = 0 · Width = Parent.TemplateWidth - 8 · Height = 86
Fill = ColorValue(nfCol.Surface) · BorderThickness = 1 · BorderColor = ColorValue(nfCol.Line)
RadiusTopLeft = nfRadio · RadiusTopRight = nfRadio · RadiusBottomLeft = nfRadio · RadiusBottomRight = nfRadio

/* lblKpiVal — Etiqueta */
Text = ThisItem.Val
X = 14 · Y = 14 · Width = Parent.TemplateWidth - 28 · Height = 34
Size = 22 · FontWeight = FontWeight.Bold · Color = ColorValue(ThisItem.Col)

/* lblKpiEtq — Etiqueta */
Text = ThisItem.Etq
X = 14 · Y = 50 · Width = Parent.TemplateWidth - 28 · Height = 28
Size = 9,5 · Color = ColorValue(nfCol.Faint) · FontWeight = FontWeight.Semibold
```

### `galBotones` — Galería en blanco **vertical con ajuste** (Nivel 2)
```powerapps
Items  = nfBotonesMenu
X      = nfPad
Y      = galKpi.Y + galKpi.Height + 14
Width  = Parent.Width - (nfPad * 2)
Height = Parent.Height - Self.Y - nfPad
WrapCount       = Max(1; RoundDown(Self.Width / 252; 0))
TemplateSize    = 142
TemplatePadding = 5
```

Plantilla:

| Control | Tipo | Nivel | Propiedad → fórmula |
|---|---|---|---|
| `cntBoton` | Contenedor Manual | 3 | `X=0` · `Y=0` · `Width=Parent.TemplateWidth-10` · `Height=134` · `Fill=ColorValue(nfCol.Surface)` · `BorderThickness=1` · `BorderColor=ColorValue(nfCol.Line)` · `RadiusTopLeft…=nfRadio` |
| `recIcoBtn` | Rectángulo | 4 | `X=16` · `Y=16` · `Width=38` · `Height=38` · `Fill=ColorValue(nfCol.AccentSoft)` · `RadiusTopLeft…=9` |
| `icoBtn` | Icono | 4 | `Icon=Switch(ThisItem.Ico; "ADD"; Icon.Add; "TREND"; Icon.TrendingUp; "BACK"; Icon.Reload; "DOWN"; Icon.Download; "ITEMS"; Icon.Items; Icon.Trash)` · `X=26` · `Y=26` · `Width=18` · `Height=18` · `Color=ColorValue(nfCol.AccentInk)` |
| `lblBtnTit` | Etiqueta | 4 | `Text=ThisItem.Titulo` · `X=16` · `Y=62` · `Width=Parent.Width-32` · `Height=22` · `Size=14,5` · `FontWeight=FontWeight.Semibold` |
| `lblBtnDet` | Etiqueta | 4 | `Text=ThisItem.Detalle` · `X=16` · `Y=84` · `Width=Parent.Width-32` · `Height=42` · `Size=11,5` · `Color=ColorValue(nfCol.Muted)` |
| `lblBtnCta` | Etiqueta | 4 | `Text=ThisItem.Cta` · `Visible=ThisItem.Cta>0` · `X=Parent.Width-48` · `Y=16` · `Width=32` · `Height=20` · `Align=Align.Center` · `Size=10,5` · `Color=ColorValue(nfCol.Accent)` · `BorderThickness=1` · `BorderColor=ColorValue(nfCol.Accent)` · `RadiusTopLeft…=5` |
| `btnTile` | **Botón invisible** | 4 | `X=0` · `Y=0` · `Width=Parent.Width` · `Height=Parent.Height` · `Text=""` · `Fill=RGBA(0;0;0;0)` · `HoverFill=RGBA(30;107;76;0,05)` · `PressedFill=RGBA(30;107;76;0,10)` · `BorderColor=RGBA(0;0;0;0)` · `ZIndex=20` |

```powerapps
/* btnTile.OnSelect */
Switch(
    ThisItem.Pant;
    "ASIG"; Navigate(scrAsignar;    ScreenTransition.None);
    "SEG";  Navigate(scrSeguimiento; ScreenTransition.None);
    "DEV";  Navigate(scrDevolucion;  ScreenTransition.None);
    "ING";  Navigate(scrIngresos;    ScreenTransition.None);
    "CAT";  Navigate(scrCatalogo;    ScreenTransition.None);
    Navigate(scrBajas; ScreenTransition.None)
)
```

### `tmrVencidos` — Temporizador (Nivel 2, `Visible = false`)
```powerapps
Duration  = 300000
AutoStart = true
Repeat    = true

OnTimerEnd =
ForAll(
    Filter(
        nfDetAbierto;
        Estatus.Value = "ENTREGADO" || Estatus.Value = "EN USO"
    ) As r;
    If(
        DateDiff(
            LookUp(BG_Salidas; NoSalida = r.NoSalida; FechaEntrega);
            Now();
            TimeUnit.Days
        ) > nfDiasPrestamo;
        Patch(
            BG_SalidaDetalle;
            LookUp(BG_SalidaDetalle; ID = r.ID);
            {Estatus: {Value: "VENCIDO"}}
        )
    )
)
```
> Delega porque `nfDetAbierto` filtra por `Abierta = true`. Sin esa columna, este `ForAll` barría toda la lista y se truncaría en silencio (E6).

---

# PARTE 7 — `scrAsignar`

### `scrAsignar.OnVisible`
```powerapps
Set(varPantalla; "ASIG");;
Set(varPopup; "");;
Set(varModoCarga; "KIT");;
Set(varTipo; "RETORNABLE");;
Set(varBodegaFiltro; "TODAS");;
Set(varBuscar; "");;
Set(varArea; Blank());;
Set(varRecoge; Blank());;
Clear(colCatSel);;
Set(varKit; Coalesce(varKit; First(nfKits)));;

/* el kit arranca con TODO seleccionado — se puede destildar lo que no va.
   Sin Select(): el código va aquí mismo (regla E7).                       */
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
);;
RemoveIf(colSel; Existencia <= 0)
```
> **`Patch(nfLineaVacia; {…})`** es la aplicación directa de la regla E2: aunque falte un dato, el registro trae los 10 campos con el tipo correcto. Esto es exactamente lo que hacía fallar a `htmlInfoMant`.

### 7.1 `cntFiltros` (Nivel 2) — altura variable según el modo
```powerapps
X      = nfPad
Y      = nfAltoBarra + 14
Width  = Parent.Width - (nfPad * 2)
Height = If(varModoCarga = "KIT"; 132; 206)
Fill   = ColorValue(nfCol.Surface)
BorderThickness = 1 · BorderColor = ColorValue(nfCol.Line)
RadiusTopLeft… = nfRadio
```

#### `btnModoKit` / `btnModoUnidad` — Botones clásicos (Nivel 3)
```powerapps
/* btnModoKit */
X = 14 · Y = 14 · Width = 134 · Height = 38
Text  = "Por kit"
Fill  = If(varModoCarga = "KIT"; ColorValue(nfCol.Accent); ColorValue(nfCol.Surface2))
Color = If(varModoCarga = "KIT"; White; ColorValue(nfCol.Muted))
BorderColor = ColorValue(nfCol.Line)
RadiusTopLeft… = 8

OnSelect =
Set(varModoCarga; "KIT");;
Clear(colCatSel);;
ClearCollect(
    colSel;
    ForAll(
        Filter(BG_KitDetalle; Title = varKit.Title) As d;
        With(
            {a: LookUp(nfCatalogo; Clave = d.Clave)};
            Patch(
                nfLineaVacia;
                {
                    Clave: a.Clave; Codigo: a.Codigo; Descripcion: a.Descripcion;
                    Tipo: a.Tipo.Value; Categoria: a.Categoria.Value; Bodega: a.Bodega.Value;
                    Unidad: a.Unidad; Existencia: a.Existencia;
                    Estandar: d.CantidadEstandar;
                    Cantidad: Min(d.CantidadEstandar; a.Existencia)
                }
            )
        )
    )
);;
RemoveIf(colSel; Existencia <= 0)
```
```powerapps
/* btnModoUnidad */
X = 154 · Y = 14 · Width = 134 · Height = 38
Text  = "Por unidad"
Fill  = If(varModoCarga = "UNIDAD"; ColorValue(nfCol.Accent); ColorValue(nfCol.Surface2))
Color = If(varModoCarga = "UNIDAD"; White; ColorValue(nfCol.Muted))

OnSelect = Set(varModoCarga; "UNIDAD");; Clear(colSel);; Clear(colCatSel)
```

#### `btnElegirKit` — Botón que abre el popup de kit (Nivel 3) · **modo KIT**
```powerapps
X = 14 · Y = 62 · Width = Parent.Width - 28 · Height = 44
Visible = varModoCarga = "KIT"
Text    = ""                                    /* el texto lo pone lblKitElegido */
Fill    = ColorValue(nfCol.Surface2)
HoverFill = ColorValue(nfCol.AccentSoft)
BorderThickness = 1 · BorderColor = ColorValue(nfCol.Line)
RadiusTopLeft… = 8
OnSelect = Set(varPopup; "KIT")
```
```powerapps
/* lblKitElegido — Etiqueta encima del botón, ZIndex menor */
Text = "KIT · " & varKit.Title & "  ·  " & varKit.NombreKit &
       "   (" & CountRows(Filter(BG_KitDetalle; Title = varKit.Title)) & " renglones)"
X = 26 · Y = 72 · Width = Parent.Width - 70 · Height = 24
Size = 13 · FontWeight = FontWeight.Semibold · Color = ColorValue(nfCol.Ink)
Visible = varModoCarga = "KIT"
ZIndex  = 2                                     /* el botón va en ZIndex 5 */
```
```powerapps
/* icoKitFlecha — Icono ChevronRight */
X = Parent.Width - 44 · Y = 74 · Width = 20 · Height = 20
Color = ColorValue(nfCol.Faint) · Visible = varModoCarga = "KIT" · ZIndex = 2
```

#### `btnTipoRet` / `btnTipoUso` — primer nivel del modo UNIDAD (Nivel 3)
```powerapps
/* btnTipoRet */
X = 14 · Y = 62 · Width = 150 · Height = 38
Visible = varModoCarga = "UNIDAD"
Text  = "Retornable"
Fill  = If(varTipo = "RETORNABLE"; ColorValue(nfCol.Accent); ColorValue(nfCol.Surface2))
Color = If(varTipo = "RETORNABLE"; White; ColorValue(nfCol.Muted))
OnSelect = Set(varTipo; "RETORNABLE");; Clear(colCatSel)

/* btnTipoUso */
X = 170 · Y = 62 · Width = 150 · Height = 38
Visible = varModoCarga = "UNIDAD"
Text  = "Un solo uso"
Fill  = If(varTipo = "UN SOLO USO"; ColorValue(nfCol.Accent); ColorValue(nfCol.Surface2))
Color = If(varTipo = "UN SOLO USO"; White; ColorValue(nfCol.Muted))
OnSelect = Set(varTipo; "UN SOLO USO");; Clear(colCatSel)
```

#### `txtBuscar` — **Text input moderno** (Nivel 3)
```powerapps
X = 14 · Y = 110 · Width = Min(320; Parent.Width - 28) · Height = 36
Visible       = varModoCarga = "UNIDAD"
Default       = ""
Placeholder   = "Buscar código o descripción"
Type          = TextInputType.Search
TriggerOutput = TriggerOutput.Delayed
Appearance    = Appearance.Outline
Fill          = ColorValue(nfCol.Surface2)
BorderColor   = ColorValue(nfCol.Line)
Color         = ColorValue(nfCol.Ink)
Size          = 13
RadiusTopLeft = 8 · RadiusTopRight = 8 · RadiusBottomLeft = 8 · RadiusBottomRight = 8
OnChange      = Set(varBuscar; Self.Text)
```
> `TriggerOutput.Delayed` evita que la galería se recalcule en cada tecla. `varBuscar` existe para que la galería no dependa de un control y pueda delegar sin sorpresas.

#### `galBodegaF` — chips de bodega, Galería horizontal (Nivel 3)
```powerapps
Items = nfBodegasF
X = Min(344; Parent.Width - 28) · Y = 110 · Width = Parent.Width - Self.X - 14 · Height = 40
TemplateSize = 128

/* recChipBod — Rectángulo */
Fill = If(ThisItem.Value = varBodegaFiltro; ColorValue(nfCol.AccentSoft); ColorValue(nfCol.Surface))
BorderThickness = 1
BorderColor = If(ThisItem.Value = varBodegaFiltro; ColorValue(nfCol.Accent); ColorValue(nfCol.Line))
RadiusTopLeft… = 999

/* lblChipBod — Etiqueta */
Text  = ThisItem.Value
Size  = 11,5 · Align = Align.Center
Color = If(ThisItem.Value = varBodegaFiltro; ColorValue(nfCol.AccentInk); ColorValue(nfCol.Muted))

/* btnChipBod — Botón invisible · ZIndex 20 */
OnSelect = Set(varBodegaFiltro; ThisItem.Value)
```

#### `galCatChips` — chips de categoría, Galería horizontal (Nivel 3)
```powerapps
Items = Distinct(Filter(nfCatalogo; Tipo.Value = varTipo); Categoria.Value)
X = 14 · Y = 156 · Width = Parent.Width - 28 · Height = 40
Visible = varModoCarga = "UNIDAD"
TemplateSize = 136

/* btnChipCat — Botón invisible · ZIndex 20 */
OnSelect =
If(
    ThisItem.Value in colCatSel.Cat;
    RemoveIf(colCatSel; Cat = ThisItem.Value);
    Collect(colCatSel; {Cat: ThisItem.Value})
)

/* recChipCat.Fill */
If(ThisItem.Value in colCatSel.Cat; ColorValue(nfCol.AccentSoft); ColorValue(nfCol.Surface))
```

### 7.2 `galItems` — Galería en blanco **vertical con ajuste** (Nivel 2)
```powerapps
X      = nfPad
Y      = cntFiltros.Y + cntFiltros.Height + 12
Width  = Parent.Width - (nfPad * 2)
Height = Parent.Height - Self.Y - 72          /* 72 = espacio del botón Continuar */
WrapCount       = Max(1; RoundDown(Self.Width / 270; 0))
TemplateSize    = 128
TemplatePadding = 5
```

```powerapps
/* galItems.Items */
If(
    varModoCarga = "KIT";

    /* ---------- KIT: solo los artículos del kit elegido ---------- */
    ForAll(
        Filter(BG_KitDetalle; Title = varKit.Title) As d;
        With(
            {a: LookUp(nfCatalogo; Clave = d.Clave)};
            Patch(
                nfLineaVacia;
                {
                    Clave: a.Clave; Codigo: a.Codigo; Descripcion: a.Descripcion;
                    Tipo: a.Tipo.Value; Categoria: a.Categoria.Value; Bodega: a.Bodega.Value;
                    Unidad: a.Unidad; Existencia: a.Existencia;
                    Estandar: d.CantidadEstandar; Cantidad: d.CantidadEstandar
                }
            )
        )
    );

    /* ---------- UNIDAD: tipo → bodega → categoría → búsqueda ---------- */
    ForAll(
        Filter(
            nfCatalogo;
            Tipo.Value = varTipo;
            Or(varBodegaFiltro = "TODAS"; Bodega.Value = varBodegaFiltro);
            Or(IsEmpty(colCatSel); Categoria.Value in colCatSel.Cat);
            Or(
                IsBlank(varBuscar);
                StartsWith(Codigo; varBuscar);
                StartsWith(Descripcion; varBuscar)
            )
        ) As a;
        Patch(
            nfLineaVacia;
            {
                Clave: a.Clave; Codigo: a.Codigo; Descripcion: a.Descripcion;
                Tipo: a.Tipo.Value; Categoria: a.Categoria.Value; Bodega: a.Bodega.Value;
                Unidad: a.Unidad; Existencia: a.Existencia;
                Estandar: 1; Cantidad: 1
            }
        )
    )
)
```

**Plantilla de `galItems`** — Nivel 3 y 4 (`ZIndex` entre paréntesis):

```powerapps
/* ---- cntItem · Contenedor Manual (Nivel 3) ---- */
X = 0 · Y = 0 · Width = Parent.TemplateWidth - 10 · Height = 122
Fill = ColorValue(nfCol.Surface)
BorderThickness = 1
BorderColor = If(
    !IsBlank(LookUp(colSel; Clave = ThisItem.Clave; Clave));
    ColorValue(nfCol.Accent);
    ColorValue(nfCol.Line)
)
RadiusTopLeft = nfRadio · RadiusTopRight = nfRadio
RadiusBottomLeft = nfRadio · RadiusBottomRight = nfRadio

/* ---- recFranja · Rectángulo (1) ---- */
X = 0 · Y = 0 · Width = 3 · Height = Parent.Height
Fill    = ColorValue(nfCol.Accent)
Visible = !IsBlank(LookUp(colSel; Clave = ThisItem.Clave; Clave))

/* ---- recCheck · Rectángulo (2) ---- */
X = 12 · Y = 12 · Width = 20 · Height = 20
Fill = If(
    !IsBlank(LookUp(colSel; Clave = ThisItem.Clave; Clave));
    ColorValue(nfCol.Accent);
    ColorValue(nfCol.Surface2)
)
BorderThickness = 1
BorderColor = If(
    !IsBlank(LookUp(colSel; Clave = ThisItem.Clave; Clave));
    ColorValue(nfCol.Accent);
    ColorValue(nfCol.Line)
)
RadiusTopLeft… = 5

/* ---- icoCheck · Icono Check (3) ---- */
X = 15 · Y = 15 · Width = 14 · Height = 14
Color   = White
Visible = !IsBlank(LookUp(colSel; Clave = ThisItem.Clave; Clave))

/* ---- lblCod · Etiqueta (3) ---- */
Text = ThisItem.Codigo
X = 40 · Y = 10 · Width = Parent.Width - 52 · Height = 15
Size = 10,5 · Color = ColorValue(nfCol.Accent)

/* ---- lblDesc · Etiqueta (3) ---- */
Text = ThisItem.Descripcion
X = 40 · Y = 26 · Width = Parent.Width - 52 · Height = 34
Size = 13 · FontWeight = FontWeight.Semibold · Color = ColorValue(nfCol.Ink)

/* ---- lblTagTipo · Etiqueta (3) ---- */
Text  = ThisItem.Tipo
X = 40 · Y = 64 · Width = If(ThisItem.Tipo = "RETORNABLE"; 88; 94) · Height = 18
Size  = 9 · Align = Align.Center
Fill  = If(ThisItem.Tipo = "RETORNABLE"; ColorValue(nfCol.AccentSoft); ColorValue(nfCol.AmberSoft))
Color = If(ThisItem.Tipo = "RETORNABLE"; ColorValue(nfCol.AccentInk); ColorValue(nfCol.Amber))
RadiusTopLeft… = 4

/* ---- lblTagBodega · Etiqueta (3) ---- */
Text = ThisItem.Bodega
X = lblTagTipo.X + lblTagTipo.Width + 5 · Y = 64 · Width = 90 · Height = 18
Size = 9 · Align = Align.Center
Color       = If(ThisItem.Bodega = "BODEGA 1"; ColorValue(nfCol.Blue); ColorValue(nfCol.Violet))
BorderThickness = 1
BorderColor = If(ThisItem.Bodega = "BODEGA 1"; ColorValue(nfCol.Blue); ColorValue(nfCol.Violet))
RadiusTopLeft… = 4

/* ---- lblStock · Etiqueta (3) ---- */
Text = "Existencia " & ThisItem.Existencia & " " & ThisItem.Unidad &
       If(ThisItem.Existencia <= 0; " · AGOTADO"; "")
X = 40 · Y = 90 · Width = Parent.Width - 160 · Height = 18
Size  = 10,5
Color = If(ThisItem.Existencia <= 0; ColorValue(nfCol.Red); ColorValue(nfCol.Muted))
```

#### `btnSelItem` — **Botón clásico invisible (ZIndex 5)**
```powerapps
X = 0 · Y = 0 · Width = Parent.Width · Height = Parent.Height
Text               = ""
Fill               = RGBA(0; 0; 0; 0)
HoverFill          = RGBA(30; 107; 76; 0,05)
PressedFill        = RGBA(30; 107; 76; 0,11)
BorderColor        = RGBA(0; 0; 0; 0)
HoverBorderColor   = RGBA(0; 0; 0; 0)
PressedBorderColor = RGBA(0; 0; 0; 0)
DisabledFill       = RGBA(0; 0; 0; 0)
DisplayMode        = If(ThisItem.Existencia <= 0; DisplayMode.Disabled; DisplayMode.Edit)
ZIndex             = 5

OnSelect =
If(
    IsBlank(LookUp(colSel; Clave = ThisItem.Clave; Clave));
    Collect(
        colSel;
        Patch(
            nfLineaVacia;
            {
                Clave:       ThisItem.Clave;
                Codigo:      ThisItem.Codigo;
                Descripcion: ThisItem.Descripcion;
                Tipo:        ThisItem.Tipo;
                Categoria:   ThisItem.Categoria;
                Bodega:      ThisItem.Bodega;
                Unidad:      ThisItem.Unidad;
                Existencia:  ThisItem.Existencia;
                Estandar:    ThisItem.Estandar;
                Cantidad:    Min(Max(1; ThisItem.Estandar); ThisItem.Existencia)
            }
        )
    );
    RemoveIf(colSel; Clave = ThisItem.Clave)
)
```

#### `cntCantidad` — Contenedor Manual (**ZIndex 10**, por encima del botón oculto)
```powerapps
X = Parent.Width - 114 · Y = 86 · Width = 102 · Height = 28
Visible = !IsBlank(LookUp(colSel; Clave = ThisItem.Clave; Clave))
Fill    = ColorValue(nfCol.Surface2)
BorderThickness = 1 · BorderColor = ColorValue(nfCol.Line)
RadiusTopLeft… = 7
ZIndex  = 10
```
```powerapps
/* btnMenos · Botón (Nivel 5) */
X = 0 · Y = 0 · Width = 28 · Height = 28
Text = "−" · Fill = RGBA(0; 0; 0; 0) · BorderColor = RGBA(0; 0; 0; 0)
HoverFill = ColorValue(nfCol.AccentSoft) · Color = ColorValue(nfCol.Ink)
OnSelect = UpdateIf(
    colSel;
    Clave = ThisItem.Clave;
    {Cantidad: Max(1; LookUp(colSel; Clave = ThisItem.Clave; Cantidad) - 1)}
)

/* lblCant · Etiqueta (Nivel 5) */
Text = LookUp(colSel; Clave = ThisItem.Clave; Cantidad)
X = 28 · Y = 0 · Width = 46 · Height = 28
Align = Align.Center · Size = 13 · FontWeight = FontWeight.Semibold

/* btnMas · Botón (Nivel 5) */
X = 74 · Y = 0 · Width = 28 · Height = 28
Text = "+" · Fill = RGBA(0; 0; 0; 0) · BorderColor = RGBA(0; 0; 0; 0)
HoverFill = ColorValue(nfCol.AccentSoft)
DisplayMode = If(
    LookUp(colSel; Clave = ThisItem.Clave; Cantidad) >= ThisItem.Existencia;
    DisplayMode.Disabled;
    DisplayMode.Edit
)
OnSelect = UpdateIf(
    colSel;
    Clave = ThisItem.Clave;
    {Cantidad: Min(ThisItem.Existencia; LookUp(colSel; Clave = ThisItem.Clave; Cantidad) + 1)}
)
```

### 7.3 `btnContinuar` — abre el carrito (Nivel 2)
```powerapps
X = Parent.Width - 268 · Y = Parent.Height - 58 · Width = 252 · Height = 42
Text = "Continuar  ·  " & CountRows(colSel) & " renglones  ·  " & Sum(colSel; Cantidad) & " pzas"
DisplayMode = If(IsEmpty(colSel); DisplayMode.Disabled; DisplayMode.Edit)
Fill  = ColorValue(nfCol.Accent) · Color = White · BorderColor = ColorValue(nfCol.Accent)
RadiusTopLeft… = 8
ZIndex = 20

OnSelect =
/* precarga el área sugerida por el kit, sin sobreescribir una elección previa */
Set(
    varArea;
    Coalesce(
        varArea;
        If(
            varModoCarga = "KIT";
            LookUp(nfAreas; KitAsignado = varKit.Title);
            Blank()
        )
    )
);;
Set(varPopup; "CARRITO")
```

---

# PARTE 8 — Los tres popups

## 8.1 `cntPopCarrito` (ZIndex 110) + `recScrimPop` (100)

### `recScrimPop` — Rectángulo (Nivel 1)
```powerapps
X = 0 · Y = 0 · Width = Parent.Width · Height = Parent.Height
Fill     = RGBA(10; 20; 15; 0,58)
Visible  = varPopup = "CARRITO"
OnSelect = Set(varPopup; "")
ZIndex   = 100
```

### `cntPopCarrito` — Contenedor Manual (Nivel 1)
```powerapps
Width  = Min(580; Parent.Width - 24)
Height = Min(740; Parent.Height - 24)
X      = (Parent.Width - Self.Width) / 2
Y      = (Parent.Height - Self.Height) / 2
Fill   = ColorValue(nfCol.Surface)
BorderThickness = 1 · BorderColor = ColorValue(nfCol.Line)
RadiusTopLeft = 12 · RadiusTopRight = 12 · RadiusBottomLeft = 12 · RadiusBottomRight = 12
DropShadow = DropShadow.ExtraLarge
Visible    = varPopup = "CARRITO"
ZIndex     = 110
```

### `lblPopTit` · `icoPopX` (Nivel 2)
```powerapps
/* lblPopTit */
Text = If(varModoCarga = "KIT"; "Confirmar " & varKit.NombreKit; "Confirmar entrega por unidad")
X = 16 · Y = 14 · Width = Parent.Width - 64 · Height = 24
Size = 15 · FontWeight = FontWeight.Semibold

/* icoPopX — Icono Cancel */
X = Parent.Width - 42 · Y = 14 · Width = 26 · Height = 26
Color = ColorValue(nfCol.Muted) · OnSelect = Set(varPopup; "")
```

### `galPopLineas` — Galería vertical, **el carrito** (Nivel 2)
```powerapps
Items  = colSel
X = 0 · Y = 50
Width  = Parent.Width
Height = Parent.Height - 50 - cntPopPie.Height
TemplateSize  = 78
ShowScrollbar = true
```
Plantilla (Nivel 3 y 4):
```powerapps
/* cntPopLinea · Contenedor Manual */
X = 0 · Y = 0 · Width = Parent.TemplateWidth · Height = 76
Fill = ColorValue(nfCol.Surface)
BorderThickness = 1 · BorderColor = ColorValue(nfCol.Line2)

/* lblPLDesc · Etiqueta */
Text = ThisItem.Codigo & "   " & ThisItem.Descripcion
X = 16 · Y = 10 · Width = Parent.Width - 190 · Height = 20
Size = 13 · FontWeight = FontWeight.Semibold

/* lblPLTags · Etiqueta */
Text = ThisItem.Tipo & "   ·   " & ThisItem.Bodega &
       "   ·   existencia " & ThisItem.Existencia & " " & ThisItem.Unidad
X = 16 · Y = 32 · Width = Parent.Width - 190 · Height = 18
Size = 10 · Color = ColorValue(nfCol.Muted)

/* lblPLAviso · Etiqueta */
Text    = "Solo hay " & ThisItem.Existencia & " " & ThisItem.Unidad & " en " & ThisItem.Bodega
Visible = ThisItem.Cantidad > ThisItem.Existencia
X = 16 · Y = 52 · Width = Parent.Width - 190 · Height = 18
Size = 10 · Color = ColorValue(nfCol.Red) · FontWeight = FontWeight.Semibold

/* cntPopCant · Contenedor Manual (ZIndex 10) → btnPopMenos / lblPopCant / btnPopMas */
X = Parent.Width - 158 · Y = 24 · Width = 102 · Height = 28

/* btnPopMenos.OnSelect */
UpdateIf(colSel; Clave = ThisItem.Clave; {Cantidad: Max(1; ThisItem.Cantidad - 1)})

/* lblPopCant.Text */
ThisItem.Cantidad

/* btnPopMas.DisplayMode */
If(ThisItem.Cantidad >= ThisItem.Existencia; DisplayMode.Disabled; DisplayMode.Edit)
/* btnPopMas.OnSelect */
UpdateIf(colSel; Clave = ThisItem.Clave; {Cantidad: Min(ThisItem.Existencia; ThisItem.Cantidad + 1)})

/* icoPopQuitar · Icono Cancel */
X = Parent.Width - 44 · Y = 26 · Width = 22 · Height = 22
Color = ColorValue(nfCol.Faint)
OnSelect = RemoveIf(colSel; Clave = ThisItem.Clave)
```

### `cntPopPie` — Contenedor Manual, datos **en conjunto** (Nivel 2)
```powerapps
X      = 0
Height = 312
Y      = Parent.Height - Self.Height
Width  = Parent.Width
Fill   = ColorValue(nfCol.Surface2)
BorderThickness = 1 · BorderColor = ColorValue(nfCol.Line2)
```

#### `lblPopTotal` (Nivel 3)
```powerapps
Text = Sum(colSel; Cantidad) & " pieza(s) en " & CountRows(colSel) & " renglón(es)"
X = 16 · Y = 10 · Width = Parent.Width - 32 · Height = 22
Size = 13 · FontWeight = FontWeight.Semibold
```

#### `btnAbrirArea` — **el selector de área que pidió** (Nivel 3)
```powerapps
X = 16 · Y = 42 · Width = (Parent.Width - 44) / 2 · Height = 44
Text      = ""
Fill      = ColorValue(nfCol.Surface)
HoverFill = ColorValue(nfCol.AccentSoft)
BorderThickness = 1
BorderColor = If(IsBlank(varArea); ColorValue(nfCol.Red); ColorValue(nfCol.Line))
RadiusTopLeft… = 8
ZIndex    = 5
OnSelect  = Set(varPopup; "AREA")
```
```powerapps
/* lblAreaEtq · Etiqueta (ZIndex 2) */
Text = "ÁREA / INVERNADERO"
X = 28 · Y = 48 · Width = btnAbrirArea.Width - 50 · Height = 14
Size = 9 · Color = ColorValue(nfCol.Faint) · FontWeight = FontWeight.Semibold

/* lblAreaElegida · Etiqueta (ZIndex 2) */
Text = If(
    IsBlank(varArea);
    "Toque para seleccionar";
    varArea.Title & " · " & varArea.NombreArea
)
X = 28 · Y = 62 · Width = btnAbrirArea.Width - 50 · Height = 20
Size = 13 · FontWeight = FontWeight.Semibold
Color = If(IsBlank(varArea); ColorValue(nfCol.Faint); ColorValue(nfCol.Ink))

/* icoAreaFlecha · Icono ChevronRight (ZIndex 2) */
X = btnAbrirArea.Width - 10 · Y = 54 · Width = 20 · Height = 20
Color = ColorValue(nfCol.Faint)
```

#### `cmbRecoge` — **Combo box moderno** (Nivel 3) · el único combo de la app
```powerapps
X      = btnAbrirArea.X + btnAbrirArea.Width + 12
Y      = 42
Width  = btnAbrirArea.Width
Height = 44

Items                = nfPersonal
ItemDisplayText      = ThisItem.Title
DefaultSelectedItems = Filter(nfPersonal; ID = Coalesce(varArea.ID * 0 + varRecoge.ID; -1))
InputTextPlaceholder = "¿Quién lo llega a traer?"
SelectMultiple       = false
IsSearchable          = true
Required              = true
ValidationState       = If(IsBlank(varRecoge); ValidationState.Error; ValidationState.None)
Appearance            = Appearance.Outline
Fill                  = ColorValue(nfCol.Surface)
BorderColor           = ColorValue(nfCol.Line)
BorderThickness       = 1
Color                 = ColorValue(nfCol.Ink)
Size                  = 13
RadiusTopLeft = 8 · RadiusTopRight = 8 · RadiusBottomLeft = 8 · RadiusBottomRight = 8
AccessibleLabel       = "Persona que recoge los materiales"

OnChange = Set(varRecoge; Self.Selected)
```
> `DefaultSelectedItems` se simplifica a `Filter(nfPersonal; ID = Coalesce(varRecoge.ID; -1))`; el `varArea.ID * 0` sobra — bórrelo. Lo dejo señalado para que no lo copie por inercia.

#### Etiquetas automáticas (Nivel 3) — lo que **no** se pregunta
```powerapps
/* lblAutoFecha */
Text = "Fecha de entrega:  " & Text(Now(); "dd/mm/yyyy hh:mm") & "   ·   automática"

/* lblAutoBodeguero */
Text = "Bodeguero (User):  " & nfUser.Nombre

/* lblAutoResp */
Text = "Responsable de área:  " & Coalesce(varArea.ResponsableArea.DisplayName; "—")

/* lblAutoEnc */
Text = "Encargado de invernadero:  " & Coalesce(varArea.EncargadoArea.DisplayName; "—")

/* las cuatro:  X = 16 · Width = Parent.Width - 32 · Height = 18 · Size = 11
   Color = ColorValue(nfCol.Muted) · Y = 96 / 116 / 136 / 156 */
```

#### `txtObs` — **Text input moderno** (Nivel 3)
```powerapps
X = 16 · Y = 182 · Width = Parent.Width - 32 · Height = 58
Default       = ""
Placeholder   = "Observación de la entrega (opcional)"
Type          = TextInputType.Multiline
MaxLength     = 500
Appearance    = Appearance.Outline
Fill          = ColorValue(nfCol.Surface)
BorderColor   = ColorValue(nfCol.Line)
Size          = 12,5
RadiusTopLeft… = 8
```

#### `btnPopMas` — regresa a la galería sin perder nada (Nivel 3)
```powerapps
X = 16 · Y = 250 · Width = (Parent.Width - 44) / 2 · Height = 44
Text  = "Agregar más artículos"
Fill  = ColorValue(nfCol.Surface) · Color = ColorValue(nfCol.Ink)
BorderThickness = 1 · BorderColor = ColorValue(nfCol.Line)
OnSelect = Set(varPopup; "")
```

#### `btnPopGuardar` — **el guardado completo** (Nivel 3)
```powerapps
X = btnPopMas.X + btnPopMas.Width + 12 · Y = 250 · Width = btnPopMas.Width · Height = 44
Text = "Guardar entrega"
Fill = ColorValue(nfCol.Accent) · Color = White
RadiusTopLeft… = 8

DisplayMode = If(
    Or(
        varGuardando;
        IsEmpty(colSel);
        IsBlank(varArea);
        IsBlank(varRecoge);
        CountRows(Filter(colSel; Cantidad > Existencia)) > 0
    );
    DisplayMode.Disabled;
    DisplayMode.Edit
)
```

```powerapps
/* btnPopGuardar.OnSelect */

Set(varGuardando; true);;

/* ---------- 1. CABECERA: se crea UNA sola vez (regla E1) ----------
   El ID de SharePoint genera el consecutivo: cero colisiones, cero duplicados. */
Set(
    varSalida;
    Patch(
        BG_Salidas;
        Defaults(BG_Salidas);
        {
            FechaEntrega: Now();
            TipoCarga:    {Value: varModoCarga};
            CodigoKit:    If(varModoCarga = "KIT"; varKit.Title; "");
            CodigoArea:   varArea.Title;
            ResponsableArea: {
                '@odata.type': "#Microsoft.Azure.Connectors.SharePoint.SPListExpandedUser";
                Claims:      varArea.ResponsableArea.Claims;
                DisplayName: varArea.ResponsableArea.DisplayName;
                Email:       varArea.ResponsableArea.Email;
                Department:  ""; JobTitle: ""; Picture: ""
            };
            EncargadoArea: {
                '@odata.type': "#Microsoft.Azure.Connectors.SharePoint.SPListExpandedUser";
                Claims:      varArea.EncargadoArea.Claims;
                DisplayName: varArea.EncargadoArea.DisplayName;
                Email:       varArea.EncargadoArea.Email;
                Department:  ""; JobTitle: ""; Picture: ""
            };
            Bodeguero: {
                '@odata.type': "#Microsoft.Azure.Connectors.SharePoint.SPListExpandedUser";
                Claims:      nfUser.Claim;
                DisplayName: nfUser.Nombre;
                Email:       nfUser.Correo;
                Department:  ""; JobTitle: ""; Picture: ""
            };
            Recoge: {
                '@odata.type': "#Microsoft.Azure.Connectors.SharePoint.SPListExpandedUser";
                Claims:      "i:0#.f|membership|" & Lower(varRecoge.Correo);
                DisplayName: varRecoge.Title;
                Email:       Lower(varRecoge.Correo);
                Department:  ""; JobTitle: ""; Picture: ""
            };
            Observacion:  txtObs.Text;
            EstadoSalida: {Value: "ABIERTA"}
        }
    )
);;

Set(varNoSalida; "SAL-" & Text(varSalida.ID; "0000"));;
Patch(BG_Salidas; varSalida; {Title: varNoSalida; NoSalida: varNoSalida});;

/* ---------- 2. RENGLONES: Abierta = true (regla E6) ---------- */
ForAll(
    colSel As s;
    Patch(
        BG_SalidaDetalle;
        Defaults(BG_SalidaDetalle);
        {
            Title:           varNoSalida;
            NoSalida:        varNoSalida;
            CodigoArea:      varArea.Title;
            Clave:           s.Clave;
            Codigo:          s.Codigo;
            Descripcion:     s.Descripcion;
            Tipo:            {Value: s.Tipo};
            Categoria:       {Value: s.Categoria};
            Bodega:          {Value: s.Bodega};
            Unidad:          s.Unidad;
            CantEntregada:   s.Cantidad;
            CantBuena:       0;
            CantDanada:      0;
            CantExtraviada:  0;
            Estatus:         {Value: "ASIGNADO"};
            ComentarioLinea: "";
            Abierta:         true
        }
    )
);;

/* ---------- 3. DESCUENTO DE EXISTENCIA en la bodega de origen ---------- */
ForAll(
    colSel As s;
    With(
        {a: LookUp(BG_Catalogo; Clave = s.Clave)};
        Patch(BG_Catalogo; a; {Existencia: a.Existencia - s.Cantidad})
    )
);;

/* ---------- 4. CIERRE ----------
   Sin ClearCollect de catálogo: nfCatalogo se recalcula sola.               */
Set(varAreaVista; varArea);;
Set(varSalVista;  LookUp(BG_Salidas; NoSalida = varNoSalida));;
Clear(colSel);;
Reset(txtObs);;
Set(varArea; Blank());;
Set(varRecoge; Blank());;
Set(varPopup; "");;
Set(varGuardando; false);;
Notify(varNoSalida & " guardada · " & varAreaVista.Title; NotificationType.Success; 3000);;
Navigate(scrSeguimientoDetalle; ScreenTransition.None)
```
> `varGuardando` bloquea el botón durante el guardado: es el antídoto al **doble clic** que generaba registros duplicados en `procesos01` (E7).

## 8.2 `cntPopArea` — popup de área (ZIndex 130) · **reemplaza al combo box**

### `recScrimArea` — Rectángulo (Nivel 1)
```powerapps
X = 0 · Y = 0 · Width = Parent.Width · Height = Parent.Height
Fill     = RGBA(10; 20; 15; 0,45)
Visible  = varPopup = "AREA"
OnSelect = Set(varPopup; "CARRITO")      /* regresa al carrito, no lo cierra */
ZIndex   = 120
```

### `cntPopArea` — Contenedor Manual (Nivel 1)
```powerapps
Width  = Min(460; Parent.Width - 24)
Height = Min(560, Parent.Height - 40)
X      = (Parent.Width - Self.Width) / 2
Y      = (Parent.Height - Self.Height) / 2
Fill   = ColorValue(nfCol.Surface)
BorderThickness = 1 · BorderColor = ColorValue(nfCol.Line)
RadiusTopLeft = 12 · RadiusTopRight = 12 · RadiusBottomLeft = 12 · RadiusBottomRight = 12
DropShadow = DropShadow.ExtraLarge
Visible    = varPopup = "AREA"
ZIndex     = 130
```

### `lblAreaTit` · `icoAreaX` (Nivel 2)
```powerapps
/* lblAreaTit */
Text = "Seleccione el área o invernadero"
X = 16 · Y = 14 · Width = Parent.Width - 64 · Height = 24
Size = 15 · FontWeight = FontWeight.Semibold

/* icoAreaX — Icono Cancel */
X = Parent.Width - 42 · Y = 14 · Width = 26 · Height = 26
Color = ColorValue(nfCol.Muted)
OnSelect = Set(varPopup; "CARRITO")
```

### `txtBuscarArea` — **Text input moderno** (Nivel 2)
```powerapps
X = 16 · Y = 48 · Width = Parent.Width - 32 · Height = 38
Default       = ""
Placeholder   = "Buscar INV-01, Invernadero 3, tomate…"
Type          = TextInputType.Search
TriggerOutput = TriggerOutput.Delayed
Appearance    = Appearance.Outline
Fill          = ColorValue(nfCol.Surface2)
BorderColor   = ColorValue(nfCol.Line)
Size          = 13
RadiusTopLeft… = 8
```

### `galAreaPop` — Galería vertical (Nivel 2)
```powerapps
Items = Filter(
    nfAreas;
    Or(
        IsBlank(txtBuscarArea.Text);
        StartsWith(Title; txtBuscarArea.Text);
        StartsWith(NombreArea; txtBuscarArea.Text)
    )
)
X = 8 · Y = 96
Width  = Parent.Width - 16
Height = Parent.Height - 108
TemplateSize  = 70
ShowScrollbar = true
```
Plantilla (Nivel 3 y 4):
```powerapps
/* cntAreaFila · Contenedor Manual */
X = 0 · Y = 0 · Width = Parent.TemplateWidth - 4 · Height = 66
Fill = If(
    ThisItem.Title = varArea.Title;
    ColorValue(nfCol.AccentSoft);
    ColorValue(nfCol.Surface)
)
BorderThickness = 1
BorderColor = If(
    ThisItem.Title = varArea.Title;
    ColorValue(nfCol.Accent);
    ColorValue(nfCol.Line2)
)
RadiusTopLeft… = 8

/* lblAreaFilaCod · Etiqueta */
Text = ThisItem.Title
X = 14 · Y = 9 · Width = 90 · Height = 16
Size = 10,5 · Color = ColorValue(nfCol.Accent)

/* lblAreaFilaNom · Etiqueta */
Text = ThisItem.NombreArea & "   ·   " & ThisItem.Cultivo.Value
X = 14 · Y = 25 · Width = Parent.Width - 60 · Height = 20
Size = 13,5 · FontWeight = FontWeight.Semibold

/* lblAreaFilaGente · Etiqueta */
Text = "Resp. " & ThisItem.ResponsableArea.DisplayName &
       "   ·   Enc. " & ThisItem.EncargadoArea.DisplayName
X = 14 · Y = 45 · Width = Parent.Width - 60 · Height = 16
Size = 10,5 · Color = ColorValue(nfCol.Faint)

/* icoAreaCheck · Icono CheckBadge */
X = Parent.Width - 40 · Y = 22 · Width = 22 · Height = 22
Color   = ColorValue(nfCol.Accent)
Visible = ThisItem.Title = varArea.Title
```

### `btnElegirArea` — **Botón invisible (ZIndex 20)**
```powerapps
X = 0 · Y = 0 · Width = Parent.Width · Height = Parent.Height
Text               = ""
Fill               = RGBA(0; 0; 0; 0)
HoverFill          = RGBA(30; 107; 76; 0,06)
PressedFill        = RGBA(30; 107; 76; 0,12)
BorderColor        = RGBA(0; 0; 0; 0)
HoverBorderColor   = RGBA(0; 0; 0; 0)
ZIndex             = 20

OnSelect =
Set(varArea; ThisItem);;
Reset(txtBuscarArea);;
Set(varPopup; "CARRITO")
```

## 8.3 `cntPopKit` — popup de kit (mismo patrón)

```powerapps
/* recScrimKit · Rectángulo — ZIndex 120 */
Visible = varPopup = "KIT" · OnSelect = Set(varPopup; "")

/* cntPopKit · Contenedor Manual — ZIndex 130 */
Width = Min(520; Parent.Width - 24) · Height = Min(480; Parent.Height - 40)
X = (Parent.Width - Self.Width) / 2 · Y = (Parent.Height - Self.Height) / 2
Visible = varPopup = "KIT"

/* galKitPop · Galería vertical */
Items = nfKits
TemplateSize = 86

/* lblKitPopNom.Text */
ThisItem.Title & "  ·  " & ThisItem.NombreKit

/* lblKitPopDet.Text */
ThisItem.Cultivo.Value & "   ·   " &
CountRows(Filter(BG_KitDetalle; Title = ThisItem.Title)) & " renglones"

/* btnElegirKit · Botón invisible — ZIndex 20 */
OnSelect =
Set(varKit; ThisItem);;
Set(varPopup; "");;
Clear(colCatSel);;
ClearCollect(
    colSel;
    ForAll(
        Filter(BG_KitDetalle; Title = ThisItem.Title) As d;
        With(
            {a: LookUp(nfCatalogo; Clave = d.Clave)};
            Patch(
                nfLineaVacia;
                {
                    Clave: a.Clave; Codigo: a.Codigo; Descripcion: a.Descripcion;
                    Tipo: a.Tipo.Value; Categoria: a.Categoria.Value; Bodega: a.Bodega.Value;
                    Unidad: a.Unidad; Existencia: a.Existencia;
                    Estandar: d.CantidadEstandar;
                    Cantidad: Min(d.CantidadEstandar; a.Existencia)
                }
            )
        )
    )
);;
RemoveIf(colSel; Existencia <= 0)
```

---

# PARTE 9 — `scrSeguimiento` y `scrSeguimientoDetalle`

## 9.1 `scrSeguimiento` — galería **horizontal** de áreas

### `scrSeguimiento.OnVisible`
```powerapps
Set(varPantalla; "SEG");;

/* Una sola lectura delegada por Abierta = true, y el resumen se arma en memoria.
   Aquí es donde GroupBy/Ungroup van con COMILLAS SIMPLES (regla E10).            */
ClearCollect(colDetAb; nfDetAbierto);;

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
                Encargado:   Coalesce(a.EncargadoArea.DisplayName; "—");
                Renglones:   CountRows(g.Lineas);
                Afuera:      Sum(g.Lineas; CantEntregada - CantBuena - CantDanada - CantExtraviada);
                Danadas:     Sum(g.Lineas; CantDanada);
                Perdidas:    Sum(g.Lineas; CantExtraviada);
                Abiertas:    CountRows(Distinct(g.Lineas; NoSalida))
            }
        )
    )
)
```
> Si alguna vez necesita volver a la lista plana: `Ungroup(colResumenArea; 'Lineas')` — **comillas simples**, nunca `Ungroup(colResumenArea; "Lineas")`.

### `galAreas` — Galería en blanco **horizontal** (Nivel 2)
```powerapps
Items  = colResumenArea
X      = nfPad
Y      = galKpi.Y + galKpi.Height + 24
Width  = Parent.Width - (nfPad * 2)
Height = 216
TemplateSize    = 254
TemplatePadding = 5
ShowScrollbar   = true
```
Plantilla:
```powerapps
/* cntArea · Contenedor Manual (Nivel 3) */
X = 0 · Y = 0 · Width = Parent.TemplateWidth - 10 · Height = 206
Fill = ColorValue(nfCol.Surface)
BorderThickness = 1 · BorderColor = ColorValue(nfCol.Line)
RadiusTopLeft… = nfRadio

/* lblAreaCod.Text */    ThisItem.CodigoArea
/* lblAreaNom.Text */    ThisItem.NombreArea
/* lblAreaCul.Text */    ThisItem.Cultivo

/* lblAreaEstado · Etiqueta */
Text  = If(ThisItem.Abiertas > 0; ThisItem.Abiertas & " salida(s) abierta(s)"; "Al día")
Fill  = If(ThisItem.Abiertas > 0; ColorValue(nfCol.AccentSoft); ColorValue(nfCol.SlateSoft))
Color = If(ThisItem.Abiertas > 0; ColorValue(nfCol.AccentInk); ColorValue(nfCol.Slate))
Size  = 10 · Align = Align.Center · RadiusTopLeft… = 999

/* lblAreaAfuera.Text / .Visible */
ThisItem.Afuera & " pzas afuera"  /  ThisItem.Afuera > 0

/* lblAreaDan · Etiqueta */
Text = ThisItem.Danadas & " dañadas" · Visible = ThisItem.Danadas > 0
Fill = ColorValue(nfCol.AmberSoft) · Color = ColorValue(nfCol.Amber)

/* lblAreaPer · Etiqueta */
Text = ThisItem.Perdidas & " extraviadas" · Visible = ThisItem.Perdidas > 0
Fill = ColorValue(nfCol.RedSoft) · Color = ColorValue(nfCol.Red)

/* lblAreaGente · Etiqueta */
Text = "Responsable " & ThisItem.Responsable & Char(10) & "Encargado " & ThisItem.Encargado
Size = 10,5 · Color = ColorValue(nfCol.Faint)
```

### `btnAbrirArea_Seg` — **Botón invisible (ZIndex 20)**
```powerapps
X = 0 · Y = 0 · Width = Parent.Width · Height = Parent.Height
Text = "" · Fill = RGBA(0; 0; 0; 0) · HoverFill = RGBA(30; 107; 76; 0,05)
BorderColor = RGBA(0; 0; 0; 0) · ZIndex = 20

OnSelect =
Set(varAreaVista; LookUp(nfAreas; Title = ThisItem.CodigoArea));;
Set(
    varSalVista;
    First(
        Sort(
            Filter(BG_Salidas; CodigoArea = ThisItem.CodigoArea);
            FechaEntrega;
            SortOrder.Descending
        )
    )
);;
Navigate(scrSeguimientoDetalle; ScreenTransition.None)
```

## 9.2 `scrSeguimientoDetalle`

### `OnVisible`
```powerapps
Set(varPantalla; "SEGDET");;
ClearCollect(
    colSalArea;
    Sort(Filter(BG_Salidas; CodigoArea = varAreaVista.Title); FechaEntrega; SortOrder.Descending)
);;
Set(varSalVista; Coalesce(varSalVista; First(colSalArea)));;
ClearCollect(colDetSal; Filter(BG_SalidaDetalle; NoSalida = varSalVista.NoSalida))
```

### Niveles de esta pantalla
```
cntCuerpo
├── cntBarra                     Nivel 2
├── cntEncabezadoArea            Nivel 2   Y = nfAltoBarra + 14 · Height = 76
│   ├── lblDetArea               Nivel 3
│   └── lblDetGente              Nivel 3
├── galSalidasArea  (horizontal) Nivel 2   Y = cntEncabezadoArea.Y + 90 · Height = 176
├── cntAccionesKit               Nivel 2   Y = galSalidasArea.Y + galSalidasArea.Height + 10 · Height = 46
│   ├── lblSalTtl / btnTodoEntregado / btnCerrarTodo     Nivel 3
└── galRenglones    (vertical)   Nivel 2   Y = cntAccionesKit.Y + 56
```

```powerapps
/* lblDetArea.Text */
varAreaVista.Title & " — " & varAreaVista.NombreArea & "   ·   " & varAreaVista.Cultivo.Value

/* lblDetGente.Text */
"Responsable " & varAreaVista.ResponsableArea.DisplayName &
"   ·   Encargado " & varAreaVista.EncargadoArea.DisplayName &
"   ·   " & CountRows(colSalArea) & " salida(s)"
```

### `galSalidasArea` — Galería **horizontal** (Nivel 2)
```powerapps
Items = colSalArea
TemplateSize = 222

/* cntSal.BorderColor */
If(ThisItem.NoSalida = varSalVista.NoSalida; ColorValue(nfCol.Accent); ColorValue(nfCol.Line))

/* lblSalNo.Text */     ThisItem.NoSalida
/* lblSalFecha.Text */  Text(ThisItem.FechaEntrega; "dd/mm/yyyy hh:mm")
/* lblSalTipo.Text */   If(ThisItem.TipoCarga.Value = "KIT"; "Kit " & ThisItem.CodigoKit; "Entrega por unidad")

/* lblSalAvance.Text */
With(
    {d: Filter(BG_SalidaDetalle; NoSalida = ThisItem.NoSalida)};
    CountRows(Filter(d; Abierta = false)) & "/" & CountRows(d) & " cerrados"
)

/* lblSalGente.Text */
"Entregó " & ThisItem.Bodeguero.DisplayName & Char(10) & "Recogió " & ThisItem.Recoge.DisplayName

/* btnSelSalida · Botón invisible — ZIndex 20 */
OnSelect =
Set(varSalVista; ThisItem);;
ClearCollect(colDetSal; Filter(BG_SalidaDetalle; NoSalida = ThisItem.NoSalida))
```

### `galRenglones` — Galería vertical (Nivel 2)
```powerapps
Items = colDetSal
TemplateSize = If(Self.Width < 720; 148; 104)
```
```powerapps
/* lblRenCod.Text */   ThisItem.Codigo
/* lblRenDesc.Text */  ThisItem.Descripcion

/* lblRenEstatus · Etiqueta */
Text  = ThisItem.Estatus.Value
Fill  = ColorValue(LookUp(nfEstatus; Estatus = ThisItem.Estatus.Value; Bg))
Color = ColorValue(LookUp(nfEstatus; Estatus = ThisItem.Estatus.Value; Txt))
Size  = 10 · Align = Align.Center · RadiusTopLeft… = 999

/* lblRenCant · Etiqueta — aquí se distingue bueno / dañado / extraviado */
Text =
"Entregado " & ThisItem.CantEntregada & " " & ThisItem.Unidad &
If(ThisItem.CantBuena > 0;      "   ·   Bueno " & ThisItem.CantBuena; "") &
If(ThisItem.CantDanada > 0;     "   ·   Dañado " & ThisItem.CantDanada; "") &
If(ThisItem.CantExtraviada > 0; "   ·   Extraviado " & ThisItem.CantExtraviada; "") &
With(
    {p: ThisItem.CantEntregada - ThisItem.CantBuena - ThisItem.CantDanada - ThisItem.CantExtraviada};
    If(p > 0; "   ·   Pendiente " & p; "")
)

/* lblRenTags.Text */
ThisItem.Tipo.Value & "   ·   " & ThisItem.Bodega.Value

/* lblRenComentario.Text / .Visible */
ThisItem.ComentarioLinea  /  !IsBlank(ThisItem.ComentarioLinea)
```

### Botones de la línea (Nivel 3 de `galRenglones`)
```powerapps
/* btnRenEntregar — "Entregar" */
Visible = ThisItem.Estatus.Value = "ASIGNADO"
OnSelect =
Patch(BG_SalidaDetalle; ThisItem; {Estatus: {Value: "ENTREGADO"}});;
ClearCollect(colDetSal; Filter(BG_SalidaDetalle; NoSalida = varSalVista.NoSalida))

/* btnRenEnUso — "Dejar en uso" */
Visible = And(ThisItem.Estatus.Value = "ENTREGADO"; ThisItem.Tipo.Value = "RETORNABLE")
OnSelect =
Patch(BG_SalidaDetalle; ThisItem; {Estatus: {Value: "EN USO"}});;
ClearCollect(colDetSal; Filter(BG_SalidaDetalle; NoSalida = varSalVista.NoSalida))

/* btnRenConsumido — "Consumido" (solo un solo uso) */
Visible = And(
    ThisItem.Tipo.Value = "UN SOLO USO";
    Or(ThisItem.Estatus.Value = "ENTREGADO"; ThisItem.Estatus.Value = "ASIGNADO")
)
OnSelect =
Patch(
    BG_SalidaDetalle;
    ThisItem;
    {
        Estatus:         {Value: "CONSUMIDO"};
        CantBuena:       ThisItem.CantEntregada;
        FechaDevolucion: Now();
        Abierta:         false
    }
);;
ClearCollect(colDetSal; Filter(BG_SalidaDetalle; NoSalida = varSalVista.NoSalida));;
/* ===== RUTINA R1 · cierre de cabecera — pegada, NO llamada con Select ===== */
With(
    {pend: CountRows(Filter(BG_SalidaDetalle; NoSalida = varSalVista.NoSalida; Abierta = true))};
    Patch(
        BG_Salidas;
        LookUp(BG_Salidas; NoSalida = varSalVista.NoSalida);
        {EstadoSalida: {Value: If(pend = 0; "CERRADA"; "ABIERTA")}}
    )
)
```

### `btnTodoEntregado` (Nivel 3 de `cntAccionesKit`)
```powerapps
Text = "Todo ENTREGADO"
OnSelect =
ForAll(
    Filter(colDetSal; Estatus.Value = "ASIGNADO") As r;
    Patch(
        BG_SalidaDetalle;
        LookUp(BG_SalidaDetalle; ID = r.ID);
        {Estatus: {Value: "ENTREGADO"}}
    )
);;
ClearCollect(colDetSal; Filter(BG_SalidaDetalle; NoSalida = varSalVista.NoSalida));;
Notify(varSalVista.NoSalida & ": kit entregado completo"; NotificationType.Success; 2500)
```

### `btnCerrarTodo` (Nivel 3)
```powerapps
Text = "Cerrar todo"
OnSelect =
/* 1. lo retornable pendiente regresa a existencia */
ForAll(
    Filter(colDetSal; Abierta = true; Tipo.Value = "RETORNABLE") As r;
    With(
        {
            p: r.CantEntregada - r.CantBuena - r.CantDanada - r.CantExtraviada;
            a: LookUp(BG_Catalogo; Clave = r.Clave)
        };
        If(p > 0; Patch(BG_Catalogo; a; {Existencia: a.Existencia + p}))
    )
);;
/* 2. se cierra cada renglón */
ForAll(
    Filter(colDetSal; Abierta = true) As r;
    With(
        {p: r.CantEntregada - r.CantBuena - r.CantDanada - r.CantExtraviada};
        Patch(
            BG_SalidaDetalle;
            LookUp(BG_SalidaDetalle; ID = r.ID);
            {
                CantBuena:       r.CantBuena + p;
                FechaDevolucion: Now();
                Abierta:         false;
                Estatus:         {Value: If(r.Tipo.Value = "RETORNABLE"; "DEVUELTO"; "CONSUMIDO")}
            }
        )
    )
);;
ClearCollect(colDetSal; Filter(BG_SalidaDetalle; NoSalida = varSalVista.NoSalida));;
/* ===== RUTINA R1 ===== */
Patch(
    BG_Salidas;
    LookUp(BG_Salidas; NoSalida = varSalVista.NoSalida);
    {EstadoSalida: {Value: "CERRADA"}}
);;
Notify(varSalVista.NoSalida & " cerrada"; NotificationType.Success; 2500)
```

---

# PARTE 10 — `scrDevolucion`

**Sin combo box de área y sin combo de salida:** el área se elige con el mismo popup (`varPopup = "AREA"`), y la salida con una **galería horizontal**. Al tocar la salida se carga `colDev` en ese mismo `OnSelect` — un solo lugar, cero `Select()` (E7).

### Niveles
```
cntCuerpo
├── cntBarra                      Nivel 2
├── cntFiltrosDev                 Nivel 2   Height = 118
│   ├── btnAbrirAreaDev + lblAreaDevEtq + lblAreaDevNom + icoAreaDevFlecha   Nivel 3
│   ├── lblDevFechaAuto / lblDevRecibeAuto                                   Nivel 3
├── galSalDev   (horizontal)      Nivel 2   Height = 150
├── cntAccionesDev                Nivel 2   Height = 46
│   ├── btnMarcarTodo / btnDesmarcarTodo / lblContadorDev                    Nivel 3
├── galDev      (vertical, wrap)  Nivel 2
└── btnGuardarDev                 Nivel 2   Y = Parent.Height - 58
```

### `scrDevolucion.OnVisible`
```powerapps
Set(varPantalla; "DEV");;
Set(varPopup; "");;
Set(varAreaDev; Blank());;
Set(varSalDev; Blank());;
Clear(colDev)
```

### `btnAbrirAreaDev` (Nivel 3) — mismo popup de área
```powerapps
X = 14 · Y = 14 · Width = Min(340; Parent.Width - 28) · Height = 44
Text = "" · Fill = ColorValue(nfCol.Surface) · HoverFill = ColorValue(nfCol.AccentSoft)
BorderThickness = 1
BorderColor = If(IsBlank(varAreaDev); ColorValue(nfCol.Red); ColorValue(nfCol.Line))
ZIndex = 5
OnSelect = Set(varPopup; "AREA")
```
> En el popup de área, `btnElegirArea.OnSelect` debe distinguir de dónde se abrió. Cámbielo a:
```powerapps
/* btnElegirArea.OnSelect — versión final que sirve a las dos pantallas */
If(
    varPantalla = "DEV";
    Set(varAreaDev; ThisItem);
    Set(varArea; ThisItem)
);;
Reset(txtBuscarArea);;
Set(varPopup; If(varPantalla = "DEV"; ""; "CARRITO"))
```

### `lblDevFechaAuto` · `lblDevRecibeAuto` (Nivel 3)
```powerapps
/* lblDevFechaAuto.Text */
"Fecha de devolución:  " & Text(Now(); "dd/mm/yyyy hh:mm") & "   ·   automática"

/* lblDevRecibeAuto.Text */
"Recibe (User):  " & nfUser.Nombre
```

### `galSalDev` — Galería **horizontal** de salidas abiertas del área (Nivel 2)
```powerapps
Items = If(
    IsBlank(varAreaDev);
    Blank();
    Sort(
        Filter(BG_Salidas; CodigoArea = varAreaDev.Title; EstadoSalida.Value = "ABIERTA");
        FechaEntrega;
        SortOrder.Descending
    )
)
X = nfPad · Y = cntFiltrosDev.Y + cntFiltrosDev.Height + 10
Width = Parent.Width - (nfPad * 2) · Height = 150
TemplateSize = 222
```
```powerapps
/* lblSalDevNo.Text */    ThisItem.NoSalida
/* lblSalDevFecha.Text */ Text(ThisItem.FechaEntrega; "dd/mm/yyyy")
/* lblSalDevPend.Text */
CountRows(Filter(BG_SalidaDetalle; NoSalida = ThisItem.NoSalida; Abierta = true)) & " pendientes"

/* cntSalDev.BorderColor */
If(ThisItem.NoSalida = varSalDev.NoSalida; ColorValue(nfCol.Accent); ColorValue(nfCol.Line))
```

### `btnSelSalDev` — **Botón invisible (ZIndex 20)** · aquí se arma `colDev`
```powerapps
X = 0 · Y = 0 · Width = Parent.Width · Height = Parent.Height
Text = "" · Fill = RGBA(0; 0; 0; 0) · HoverFill = RGBA(30; 107; 76; 0,05)
BorderColor = RGBA(0; 0; 0; 0) · ZIndex = 20

OnSelect =
Set(varSalDev; ThisItem);;
ClearCollect(
    colDev;
    ForAll(
        Filter(BG_SalidaDetalle; NoSalida = ThisItem.NoSalida; Abierta = true) As r;
        Patch(
            nfDevVacia;
            {
                IdLinea:     r.ID;
                Clave:       r.Clave;
                Codigo:      r.Codigo;
                Descripcion: r.Descripcion;
                Tipo:        r.Tipo.Value;
                Bodega:      r.Bodega.Value;
                Unidad:      r.Unidad;
                Estatus:     r.Estatus.Value;
                Entregada:   r.CantEntregada;
                Pendiente:   r.CantEntregada - r.CantBuena - r.CantDanada - r.CantExtraviada;
                Marcado:     false;
                Buenas:      0;
                Danadas:     0;
                Perdidas:    0;
                Comentario:  "";
                DarBaja:     false
            }
        )
    )
)
```

### `btnMarcarTodo` / `btnDesmarcarTodo` (Nivel 3)
```powerapps
/* btnMarcarTodo */
Text = "Marcar todo"
DisplayMode = If(IsEmpty(colDev); DisplayMode.Disabled; DisplayMode.Edit)
OnSelect = ForAll(
    colDev As d;
    UpdateIf(
        colDev;
        IdLinea = d.IdLinea;
        {
            Marcado:  true;
            Buenas:   If(d.Tipo = "RETORNABLE"; d.Pendiente; 0);
            Danadas:  0;
            Perdidas: 0
        }
    )
)

/* btnDesmarcarTodo */
Text = "Desmarcar todo"
OnSelect = UpdateIf(
    colDev;
    true;
    {Marcado: false; Buenas: 0; Danadas: 0; Perdidas: 0; Comentario: ""; DarBaja: false}
)

/* lblContadorDev.Text */
CountRows(Filter(colDev; Marcado)) & " de " & CountRows(colDev) & " renglones marcados"
```

### `galDev` — Galería vertical con ajuste (Nivel 2)
```powerapps
Items  = colDev
X      = nfPad
Y      = cntAccionesDev.Y + cntAccionesDev.Height + 10
Width  = Parent.Width - (nfPad * 2)
Height = Parent.Height - Self.Y - 68
WrapCount    = Max(1; RoundDown(Self.Width / 330; 0))
TemplateSize = If(Self.Width < 720; 272; 250)
```

Plantilla de `galDev` — niveles y ZIndex:
```
cntDev                Contenedor Manual   Nivel 3
├── recCheckDev       Rectángulo          (2)
├── icoCheckDev       Icono Check         (3)
├── lblDevCod         Etiqueta            (3)
├── lblDevDesc        Etiqueta            (3)
├── lblDevEstatus     Etiqueta            (3)
├── lblDevPend        Etiqueta            (3)
├── btnMarcarLinea    Botón invisible     (5)  ← SOLO la franja superior, 62 px
└── cntDevCampos      Contenedor Manual   (10) ← los 3 números + casilla + comentario
    ├── txtBuenas     Text input moderno
    ├── txtDanadas    Text input moderno
    ├── txtPerdidas   Text input moderno
    ├── lblDevSuma    Etiqueta
    ├── lblDevConsumo Etiqueta
    ├── chkDarBaja    Casilla
    └── txtComentDev  Text input moderno
```
> **`btnMarcarLinea` mide 62 px de alto, no toda la tarjeta.** Si cubriera la tarjeta entera, taparía las tres cajas de cantidad aun con `ZIndex` menor en algunos renderizados. Es la corrección directa del error de superposición del popup (E9).

```powerapps
/* cntDev.BorderColor */
If(ThisItem.Marcado; ColorValue(nfCol.Accent); ColorValue(nfCol.Line))

/* recCheckDev.Fill */
If(ThisItem.Marcado; ColorValue(nfCol.Accent); ColorValue(nfCol.Surface2))

/* icoCheckDev.Visible */
ThisItem.Marcado

/* lblDevPend.Text */
"Pendiente " & ThisItem.Pendiente & " de " & ThisItem.Entregada & " " & ThisItem.Unidad
```

### `btnMarcarLinea` — **marcar / desmarcar individual (ZIndex 5)**
```powerapps
X = 0 · Y = 0 · Width = Parent.Width · Height = 62
Text = "" · Fill = RGBA(0; 0; 0; 0)
HoverFill = RGBA(30; 107; 76; 0,05) · PressedFill = RGBA(30; 107; 76; 0,11)
BorderColor = RGBA(0; 0; 0; 0) · HoverBorderColor = RGBA(0; 0; 0; 0)
ZIndex = 5

OnSelect =
If(
    ThisItem.Marcado;
    UpdateIf(
        colDev;
        IdLinea = ThisItem.IdLinea;
        {Marcado: false; Buenas: 0; Danadas: 0; Perdidas: 0; Comentario: ""; DarBaja: false}
    );
    UpdateIf(
        colDev;
        IdLinea = ThisItem.IdLinea;
        {
            Marcado: true;
            Buenas:  If(ThisItem.Tipo = "RETORNABLE"; ThisItem.Pendiente; 0)
        }
    )
)
```

### Las tres cantidades — **Text input moderno** dentro de `cntDevCampos` (ZIndex 10)
```powerapps
/* txtBuenas */
X = 0 · Y = 18 · Width = (Parent.Width - 16) / 3 · Height = 36
Visible       = And(ThisItem.Marcado; ThisItem.Tipo = "RETORNABLE")
Default       = ThisItem.Buenas
Type          = TextInputType.SingleLine
TriggerOutput = TriggerOutput.FocusOut
Appearance    = Appearance.Outline
Align         = Align.Center
Fill          = ColorValue(nfCol.Surface)
BorderColor   = ColorValue(nfCol.Accent)
Color         = ColorValue(nfCol.AccentInk)
Size          = 14 · FontWeight = FontWeight.Semibold
RadiusTopLeft… = 7
AccessibleLabel = "Piezas en buen estado"
OnChange = UpdateIf(
    colDev;
    IdLinea = ThisItem.IdLinea;
    {Buenas: Min(ThisItem.Pendiente; Max(0; Value(Self.Text)))}
)

/* txtDanadas */
X = txtBuenas.Width + 8 · Y = 18 · Width = txtBuenas.Width · Height = 36
Visible     = And(ThisItem.Marcado; ThisItem.Tipo = "RETORNABLE")
Default     = ThisItem.Danadas
BorderColor = ColorValue(nfCol.Amber) · Color = ColorValue(nfCol.Amber)
OnChange = UpdateIf(
    colDev;
    IdLinea = ThisItem.IdLinea;
    {Danadas: Min(ThisItem.Pendiente; Max(0; Value(Self.Text)))}
)

/* txtPerdidas */
X = (txtBuenas.Width + 8) * 2 · Y = 18 · Width = txtBuenas.Width · Height = 36
Visible     = And(ThisItem.Marcado; ThisItem.Tipo = "RETORNABLE")
Default     = ThisItem.Perdidas
BorderColor = ColorValue(nfCol.Red) · Color = ColorValue(nfCol.Red)
OnChange = UpdateIf(
    colDev;
    IdLinea = ThisItem.IdLinea;
    {Perdidas: Min(ThisItem.Pendiente; Max(0; Value(Self.Text)))}
)
```
```powerapps
/* lblEtqBuenas / lblEtqDanadas / lblEtqPerdidas — Etiquetas sobre cada caja, Y = 0, Height = 16 */
"BUENAS"      Color = ColorValue(nfCol.Accent)
"DAÑADAS"     Color = ColorValue(nfCol.Amber)
"EXTRAVIADAS" Color = ColorValue(nfCol.Red)
Size = 9 · Align = Align.Center · FontWeight = FontWeight.Semibold
Visible = And(ThisItem.Marcado; ThisItem.Tipo = "RETORNABLE")
```

### `lblDevSuma` — el control de cuadre (Nivel 4)
```powerapps
Text = With(
    {s: ThisItem.Buenas + ThisItem.Danadas + ThisItem.Perdidas};
    s & " de " & ThisItem.Pendiente & " distribuidas" &
    Switch(
        true;
        s > ThisItem.Pendiente; "   ·   se pasó del pendiente";
        s < ThisItem.Pendiente; "   ·   el resto queda pendiente";
        "   ·   renglón completo"
    )
)
Visible = And(ThisItem.Marcado; ThisItem.Tipo = "RETORNABLE")
Size = 10,5
Color = If(
    (ThisItem.Buenas + ThisItem.Danadas + ThisItem.Perdidas) > ThisItem.Pendiente;
    ColorValue(nfCol.Red);
    ColorValue(nfCol.Faint)
)
```

### `lblDevConsumo` — para un solo uso (Nivel 4)
```powerapps
Text    = "Se cierra como CONSUMIDO  ·  " & ThisItem.Pendiente & " " & ThisItem.Unidad
Visible = And(ThisItem.Marcado; ThisItem.Tipo = "UN SOLO USO")
Fill    = ColorValue(nfCol.VioletSoft) · Color = ColorValue(nfCol.Violet)
Size    = 11 · Align = Align.Center · RadiusTopLeft… = 999
```

### `chkDarBaja` — Casilla (Nivel 4)
```powerapps
Text    = "Dar de baja las dañadas y extraviadas"
Visible = And(ThisItem.Marcado; (ThisItem.Danadas + ThisItem.Perdidas) > 0)
Default = ThisItem.DarBaja
Color   = ColorValue(nfCol.Red)
OnCheck   = UpdateIf(colDev; IdLinea = ThisItem.IdLinea; {DarBaja: true})
OnUncheck = UpdateIf(colDev; IdLinea = ThisItem.IdLinea; {DarBaja: false})
```

### `txtComentDev` — **Text input moderno** (Nivel 4)
```powerapps
Visible     = ThisItem.Marcado
Default     = ThisItem.Comentario
Type        = TextInputType.Multiline
MaxLength   = 500
Appearance  = Appearance.Outline
Placeholder = If(
    (ThisItem.Danadas + ThisItem.Perdidas) > 0;
    "Comentario obligatorio: qué le pasó";
    "Comentario (opcional)"
)
ValidationState = If(
    And((ThisItem.Danadas + ThisItem.Perdidas) > 0; IsBlank(ThisItem.Comentario));
    ValidationState.Error;
    ValidationState.None
)
Required = (ThisItem.Danadas + ThisItem.Perdidas) > 0
Size = 12 · RadiusTopLeft… = 7
OnChange = UpdateIf(colDev; IdLinea = ThisItem.IdLinea; {Comentario: Self.Text})
```

### `btnGuardarDev` (Nivel 2)
```powerapps
X = Parent.Width - 268 · Y = Parent.Height - 58 · Width = 252 · Height = 42
Text = "Registrar devolución"
Fill = ColorValue(nfCol.Accent) · Color = White

DisplayMode = If(
    Or(
        varGuardando;
        CountRows(Filter(colDev; Marcado)) = 0;
        CountRows(Filter(colDev; Marcado; (Buenas + Danadas + Perdidas) > Pendiente)) > 0;
        CountRows(Filter(colDev; Marcado; (Danadas + Perdidas) > 0; IsBlank(Comentario))) > 0;
        CountRows(Filter(colDev; Marcado; Tipo = "RETORNABLE"; (Buenas + Danadas + Perdidas) = 0)) > 0
    );
    DisplayMode.Disabled;
    DisplayMode.Edit
)
```

```powerapps
/* btnGuardarDev.OnSelect */

Set(varGuardando; true);;

/* ---------- 1. UN SOLO USO: se consume completo ---------- */
ForAll(
    Filter(colDev; Marcado; Tipo = "UN SOLO USO") As d;
    With(
        {r: LookUp(BG_SalidaDetalle; ID = d.IdLinea)};
        Patch(
            BG_SalidaDetalle;
            r;
            {
                CantBuena:       r.CantBuena + d.Pendiente;
                Estatus:         {Value: "CONSUMIDO"};
                FechaDevolucion: Now();
                ComentarioLinea: d.Comentario;
                Abierta:         false;
                RecibeDevolucion: {
                    '@odata.type': "#Microsoft.Azure.Connectors.SharePoint.SPListExpandedUser";
                    Claims: nfUser.Claim; DisplayName: nfUser.Nombre; Email: nfUser.Correo;
                    Department: ""; JobTitle: ""; Picture: ""
                }
            }
        )
    )
);;

/* ---------- 2. RETORNABLE: buenas / dañadas / extraviadas por separado ---------- */
ForAll(
    Filter(colDev; Marcado; Tipo = "RETORNABLE") As d;
    With(
        {
            r:    LookUp(BG_SalidaDetalle; ID = d.IdLinea);
            baja: If(d.DarBaja; d.Danadas + d.Perdidas; 0);
            rest: d.Pendiente - d.Buenas - d.Danadas - d.Perdidas
        };
        Patch(
            BG_SalidaDetalle;
            r;
            {
                CantEntregada:   r.CantEntregada - baja;
                CantBuena:       r.CantBuena + d.Buenas;
                CantDanada:      r.CantDanada + If(d.DarBaja; 0; d.Danadas);
                CantExtraviada:  r.CantExtraviada + If(d.DarBaja; 0; d.Perdidas);
                FechaDevolucion: Now();
                ComentarioLinea: d.Comentario;
                Abierta:         rest > 0;
                Estatus: {
                    Value: Switch(
                        true;
                        rest > 0;                                    "DEVUELTO PARCIAL";
                        (d.Danadas + d.Perdidas) > 0 And !d.DarBaja; "DAÑADO";
                        "DEVUELTO"
                    )
                };
                RecibeDevolucion: {
                    '@odata.type': "#Microsoft.Azure.Connectors.SharePoint.SPListExpandedUser";
                    Claims: nfUser.Claim; DisplayName: nfUser.Nombre; Email: nfUser.Correo;
                    Department: ""; JobTitle: ""; Picture: ""
                }
            }
        )
    )
);;

/* ---------- 3. LO BUENO regresa a la existencia de su bodega ---------- */
ForAll(
    Filter(colDev; Marcado; Tipo = "RETORNABLE"; Buenas > 0) As d;
    With(
        {a: LookUp(BG_Catalogo; Clave = d.Clave)};
        Patch(BG_Catalogo; a; {Existencia: a.Existencia + d.Buenas})
    )
);;

/* ---------- 4. BAJAS en el acto ---------- */
ForAll(
    Filter(colDev; Marcado; DarBaja; (Danadas + Perdidas) > 0) As d;
    Patch(
        BG_Bajas;
        Defaults(BG_Bajas);
        {
            Title:       d.Codigo;
            Clave:       d.Clave;
            Descripcion: d.Descripcion;
            Bodega:      {Value: d.Bodega};
            Cantidad:    d.Danadas + d.Perdidas;
            FechaBaja:   Now();
            NoSalida:    varSalDev.NoSalida;
            CodigoArea:  varAreaDev.Title;
            Motivo:      d.Comentario;
            Reporto: {
                '@odata.type': "#Microsoft.Azure.Connectors.SharePoint.SPListExpandedUser";
                Claims:      varSalDev.Recoge.Claims;
                DisplayName: varSalDev.Recoge.DisplayName;
                Email:       varSalDev.Recoge.Email;
                Department: ""; JobTitle: ""; Picture: ""
            };
            RegistroPor: {
                '@odata.type': "#Microsoft.Azure.Connectors.SharePoint.SPListExpandedUser";
                Claims: nfUser.Claim; DisplayName: nfUser.Nombre; Email: nfUser.Correo;
                Department: ""; JobTitle: ""; Picture: ""
            }
        }
    )
);;

/* ---------- 5. RUTINA R1 · cierre de cabecera ---------- */
With(
    {pend: CountRows(Filter(BG_SalidaDetalle; NoSalida = varSalDev.NoSalida; Abierta = true))};
    Patch(
        BG_Salidas;
        LookUp(BG_Salidas; NoSalida = varSalDev.NoSalida);
        {EstadoSalida: {Value: If(pend = 0; "CERRADA"; "ABIERTA")}}
    )
);;

/* ---------- 6. RECARGA de la misma salida ---------- */
ClearCollect(
    colDev;
    ForAll(
        Filter(BG_SalidaDetalle; NoSalida = varSalDev.NoSalida; Abierta = true) As r;
        Patch(
            nfDevVacia;
            {
                IdLinea: r.ID; Clave: r.Clave; Codigo: r.Codigo; Descripcion: r.Descripcion;
                Tipo: r.Tipo.Value; Bodega: r.Bodega.Value; Unidad: r.Unidad;
                Estatus: r.Estatus.Value; Entregada: r.CantEntregada;
                Pendiente: r.CantEntregada - r.CantBuena - r.CantDanada - r.CantExtraviada
            }
        )
    )
);;
Set(varGuardando; false);;
Notify("Devolución registrada"; NotificationType.Success; 2500)
```

---

# PARTE 11 — `scrIngresos` · el puente con `SALIDAS` de Bodega General

### `OnVisible`
```powerapps
Set(varPantalla; "ING");;
ClearCollect(
    colPorIngresar;
    ForAll(
        nfPorIngresar As r;
        {
            IdGeneral:   r.ID;
            Codigo:      r.Codigo;
            Descripcion: r.Descripcion;
            Cantidad:    r.Cantidad;
            Unidad:      r.Unidad;
            AreaUso:     r.AreaUso;
            Responsable: r.Responsable;
            /* propone lo que ya existe en el catálogo; el bodeguero puede cambiarlo */
            Tipo:          Coalesce(LookUp(BG_Catalogo; Codigo = r.Codigo; Tipo.Value); "UN SOLO USO");
            BodegaDestino: Coalesce(LookUp(BG_Catalogo; Codigo = r.Codigo; Bodega.Value); "BODEGA 1");
            Categoria:     Coalesce(LookUp(BG_Catalogo; Codigo = r.Codigo; Categoria.Value); "INSUMO");
            EsNuevo:       IsBlank(LookUp(BG_Catalogo; Codigo = r.Codigo; Codigo))
        }
    )
)
```

### `galIngresos` — Galería vertical (Nivel 2)
```powerapps
Items = colPorIngresar
TemplateSize = If(Self.Width < 820; 268; 128)
```
```powerapps
/* lblIngCod.Text */
ThisItem.Codigo & If(ThisItem.EsNuevo; "   ·   CÓDIGO NUEVO"; "")
/* lblIngCod.Color */
If(ThisItem.EsNuevo; ColorValue(nfCol.Blue); ColorValue(nfCol.Accent))

/* lblIngDesc.Text */
ThisItem.Descripcion

/* lblIngDatos.Text */
"SALIDAS #" & ThisItem.IdGeneral & "   ·   " & ThisItem.Cantidad & " " & ThisItem.Unidad &
"   ·   Área " & ThisItem.AreaUso & "   ·   " & ThisItem.Responsable
```

### Los tres selectores: **galerías horizontales de chips**, no desplegables (Nivel 4, ZIndex 10)
```powerapps
/* galIngTipo — Galería horizontal, TemplateSize 132 */
Items = nfTipos
/* recIngTipo.Fill */
If(ThisItem.Value = ThisItem_Padre.Tipo; ColorValue(nfCol.AccentSoft); ColorValue(nfCol.Surface))
```
> Dentro de una galería anidada, `ThisItem` del padre se alcanza renombrando el ámbito: ponga en la galería padre `galIngresos` y use `galIngresos.Selected.Tipo`, o más seguro, declare la galería interna con `Items = ForAll(nfTipos As t; {Valor: t.Value; Fila: ThisItem.IdGeneral})`. **Nunca `ThisItem` a secas en galería anidada (E11).**

```powerapps
/* btnIngTipo · Botón invisible — ZIndex 20 */
OnSelect = UpdateIf(colPorIngresar; IdGeneral = ThisItem.Fila; {Tipo: ThisItem.Valor})

/* galIngBodega — igual, con Items = nfBodegas */
OnSelect = UpdateIf(colPorIngresar; IdGeneral = ThisItem.Fila; {BodegaDestino: ThisItem.Valor})
```

### `btnIngresar` — **la regla de acumulación** (Nivel 4)
```powerapps
Text = "Ingresar"
DisplayMode = If(varGuardando; DisplayMode.Disabled; DisplayMode.Edit)

OnSelect =
Set(varGuardando; true);;

With(
    {
        art: LookUp(
            BG_Catalogo;
            Codigo = ThisItem.Codigo And Bodega.Value = ThisItem.BodegaDestino
        )
    };
    If(
        /* ---- 1. el código YA existe en esa bodega → SOLO SE SUMA ---- */
        !IsBlank(art);
        Patch(
            BG_Catalogo;
            art;
            {
                Existencia:     art.Existencia + ThisItem.Cantidad;
                Tipo:           {Value: ThisItem.Tipo};
                EstadoArticulo: {Value: "ACTIVO"}
            }
        );

        /* ---- 2. no existe → se crea el renglón del catálogo ---- */
        Patch(
            BG_Catalogo;
            Defaults(BG_Catalogo);
            {
                Title:          ThisItem.Codigo;
                Clave:          ThisItem.Codigo & "|" & ThisItem.BodegaDestino;
                Codigo:         ThisItem.Codigo;
                Descripcion:    ThisItem.Descripcion;
                Tipo:           {Value: ThisItem.Tipo};
                Categoria:      {Value: ThisItem.Categoria};
                Bodega:         {Value: ThisItem.BodegaDestino};
                Unidad:         ThisItem.Unidad;
                Existencia:     ThisItem.Cantidad;
                Minimo:         Max(1; RoundUp(ThisItem.Cantidad * 0,2; 0));
                EstadoArticulo: {Value: "ACTIVO"}
            }
        )
    )
);;

/* ---- 3. bitácora del ingreso ---- */
Set(
    varIng;
    Patch(
        BG_Ingresos;
        Defaults(BG_Ingresos);
        {
            IdSalidaGeneral: ThisItem.IdGeneral;
            FechaIngreso:    Now();
            Codigo:          ThisItem.Codigo;
            Descripcion:     ThisItem.Descripcion;
            Cantidad:        ThisItem.Cantidad;
            Unidad:          ThisItem.Unidad;
            Tipo:            {Value: ThisItem.Tipo};
            BodegaDestino:   {Value: ThisItem.BodegaDestino};
            AreaUso:         ThisItem.AreaUso;
            Responsable:     ThisItem.Responsable;
            IngresadoPor: {
                '@odata.type': "#Microsoft.Azure.Connectors.SharePoint.SPListExpandedUser";
                Claims: nfUser.Claim; DisplayName: nfUser.Nombre; Email: nfUser.Correo;
                Department: ""; JobTitle: ""; Picture: ""
            }
        }
    )
);;
Patch(BG_Ingresos; varIng; {Title: "ING-" & Text(varIng.ID; "0000")});;

/* ---- 4. bandera para que NUNCA vuelva a ingresar ---- */
Patch(SALIDAS; LookUp(SALIDAS; ID = ThisItem.IdGeneral); {ProcesadoBodega: true});;

/* ---- 5. refresco local ---- */
RemoveIf(colPorIngresar; IdGeneral = ThisItem.IdGeneral);;
Set(varGuardando; false);;
Notify(
    ThisItem.Codigo & ": existencia acumulada en " & ThisItem.BodegaDestino &
    "  (+" & ThisItem.Cantidad & " " & ThisItem.Unidad & ")";
    NotificationType.Success;
    3000
)
```

---

# PARTE 12 — `scrCatalogo` y `scrBajas`

### `scrCatalogo.OnVisible`
```powerapps
Set(varPantalla; "CAT");;
Set(varTipoCat; "");;
Set(varBodCat; "TODAS");;
Set(varBuscarCat; "")
```

### `galCatalogo` — Galería vertical (Nivel 2)
```powerapps
Items = ForAll(
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
> `colDetAb` se carga en `scrSeguimiento.OnVisible`. Si entra directo a Catálogo sin pasar por Seguimiento, agregue `ClearCollect(colDetAb; nfDetAbierto);;` al `OnVisible` de esta pantalla. **No** use `Sum(Filter(BG_SalidaDetalle; …))` dentro de la galería: `Sum` no delega en SharePoint y era exactamente lo que truncaba los KPI de Mantenimiento (E4, E6).

```powerapps
/* lblCatExistencia.Color */
If(ThisItem.Existencia <= ThisItem.Minimo; ColorValue(nfCol.Amber); ColorValue(nfCol.Ink))

/* lblCatAfuera.Text */
If(ThisItem.Afuera > 0; ThisItem.Afuera & " afuera"; "—")

/* lblCatEstado — pastilla */
Text  = ThisItem.Estado
Fill  = ColorValue(LookUp(nfEstatus; Estatus = ThisItem.Estado; Bg))
Color = ColorValue(LookUp(nfEstatus; Estatus = ThisItem.Estado; Txt))
```

### `scrBajas`
```powerapps
/* OnVisible */
Set(varPantalla; "BAJA");;
ClearCollect(colBajas; Sort(BG_Bajas; FechaBaja; SortOrder.Descending))

/* galBajas.Items */   colBajas

/* lblBajaTitulo.Text */
ThisItem.Title & "   ·   " & ThisItem.Descripcion

/* lblBajaDatos.Text */
Text(ThisItem.FechaBaja; "dd/mm/yyyy") & "   ·   " & ThisItem.Cantidad & " pza(s)   ·   " &
ThisItem.Bodega.Value & "   ·   " & ThisItem.NoSalida & "   ·   " & ThisItem.CodigoArea

/* lblBajaMotivo.Text */    ThisItem.Motivo

/* lblBajaQuien.Text */
"Reportó " & ThisItem.Reporto.DisplayName & "   ·   Registró " & ThisItem.RegistroPor.DisplayName

/* btnReactivar — "Reactivar artículo" (si la baja fue un error) */
OnSelect =
With(
    {a: LookUp(BG_Catalogo; Clave = ThisItem.Clave)};
    Patch(
        BG_Catalogo;
        a;
        {
            EstadoArticulo: {Value: "ACTIVO"};
            Existencia:     a.Existencia + ThisItem.Cantidad
        }
    )
);;
Remove(BG_Bajas; ThisItem);;
ClearCollect(colBajas; Sort(BG_Bajas; FechaBaja; SortOrder.Descending));;
Notify("Artículo reactivado y devuelto a existencia"; NotificationType.Success; 2500)
```

---

# PARTE 13 — Orden de construcción (siga este orden, no otro)

| Paso | Qué hacer | Por qué en este orden |
|---|---|---|
| 1 | Crear las 9 listas de SharePoint con los **índices** y la columna `Abierta` | sin índices nada delega y habría que rehacer fórmulas |
| 2 | `Configuración → Límite de filas = 2000` y **activar Fórmulas con nombre** | `App.Formulas` no existe si no se activa |
| 3 | Pegar **`App.Formulas`** completo (Parte 2) | todo lo demás lo referencia |
| 4 | Pegar **`App.OnStart`** y poner `StartScreen = scrMenu` | |
| 5 | Construir el **esqueleto** en `scrMenu`: `cntCuerpo`, `cntBarra`, `recScrimNav`, `cntNav`, `galNav` | |
| 6 | **Agrupar** `recScrimNav` + `cntNav` + `cntBarra` y copiarlos a las otras 7 pantallas | se construye una vez |
| 7 | `galKpi` y `galBotones` de `scrMenu` + `tmrVencidos` | ya se puede navegar |
| 8 | `scrAsignar`: filtros → `galItems` → `btnContinuar` | |
| 9 | Los **tres popups** como hijos de `scrAsignar`: `cntPopArea`, `cntPopKit`, `cntPopCarrito` | el de área primero: los otros dependen de `varArea` |
| 10 | `scrSeguimiento` → `scrSeguimientoDetalle` | |
| 11 | `scrDevolucion` (copiar `cntPopArea` aquí también) | |
| 12 | `scrIngresos`, `scrCatalogo`, `scrBajas`, `scrFlujo` | |

### Prueba de aceptación antes de publicar
1. Guardar **dos veces seguidas** el mismo kit → deben salir `SAL-0001` y `SAL-0002`, **nunca un duplicado del mismo número** (E1).
2. Doble clic rápido en **Guardar entrega** → un solo registro (E7, `varGuardando`).
3. Entregar 2 tijeras, devolver **1 buena y 1 dañada** → estatus `DAÑADO`, existencia +1, `Abierta` sigue en `false` solo si no quedó pendiente.
4. Marcar la dañada con **Dar de baja** → aparece en `scrBajas`, `CantEntregada` baja a 1, el catálogo no la cuenta.
5. Ingresar dos veces el mismo código de `SALIDAS` → **la segunda vez ya no aparece** en la lista de pendientes (`ProcesadoBodega`).
6. Abrir en teléfono (ancho < 760) → el menú se superpone, el velo lo cierra, el popup del carrito tapa el menú y los `+`/`−` responden al toque.

---

**Fuentes de las propiedades de los controles modernos:**
[Combo box modern control in canvas apps](https://learn.microsoft.com/en-us/power-apps/maker/canvas-apps/controls/modern-controls/modern-control-combobox) ·
[Text input modern control](https://learn.microsoft.com/en-us/power-apps/maker/canvas-apps/controls/modern-controls/modern-control-text-input) ·
[Recent modern control updates in canvas apps](https://learn.microsoft.com/en-us/power-apps/maker/canvas-apps/controls/modern-controls/modern-control-updates)

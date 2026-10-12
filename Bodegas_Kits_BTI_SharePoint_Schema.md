# Esquema SharePoint — Bodegas & Kits BTI
### Lista · Columna · Tipo de dato · Índice · Observación

> **Leyenda**
> - ✔ en Índice: la columna debe marcarse como **columna indexada** en Configuración de la lista → Columnas indexadas.
> - **Persona**: columna de tipo "Persona o grupo" (solo cuando el valor tiene correo de Microsoft).
> - **Texto**: línea única de texto (si no se dice otra cosa).
> - **Elección**: desplegable de SharePoint; los valores exactos se listan en Observación.
> - **Sí/No**: real (switch); **Texto "SI"/"NO"** cuando se almacena como texto plano (caso de `CATALOGO_PERSONAL1`).

---

## 1. `BG_Catalogo`  — maestro de artículos por código + bodega

| Columna | Tipo | Índice | Observación |
|---|---|:---:|---|
| `Title` | Texto | ✔ | el `Codigo` del artículo |
| `Clave` | Texto | ✔ | `Codigo & "|" & Bodega` — la escribe la app, nunca a mano |
| `Codigo` | Texto | ✔ | HER-001, EQP-001, MOB-001… |
| `Descripcion` | Texto | | |
| `Tipo` | Elección | | `RETORNABLE` · `UN SOLO USO` |
| `Categoria` | Elección | | `HERRAMIENTA` · `EQUIPO` · `MATERIAL` · `INSUMO` · `MOBILIARIO` · `LIMPIEZA` |
| `Bodega` | Elección | | `BODEGA 1` · `CONTENEDOR` |
| `Unidad` | Texto | | UND · PAR · KG · GAL · ROL · BLS · LT |
| `Existencia` | Número | | entero, sin decimales |
| `Minimo` | Número | | entero; stock mínimo para alerta |
| `EstadoArticulo` | Elección | ✔ | `ACTIVO` · `DE BAJA` |

---

## 2. `BG_Areas`  — invernaderos y semilleros

| Columna | Tipo | Índice | Observación |
|---|---|:---:|---|
| `Title` | Texto | ✔ | `CodigoArea` — INV-01, SEM-01… |
| `NombreArea` | Texto | | Invernadero 1, Semillero central |
| `Cultivo` | Elección | | `TOMATE` · `CHILE` · `PEPINO` · `CUCURBITÁCEAS` · `MIXTO` |
| `KitAsignado` | Texto | | código del kit sugerido (KIT-TOM, KIT-CHI…) |
| `ResponsableArea` | **Persona** | | tiene correo de Microsoft (`User()` del jefe) |
| `EncargadoArea` | **Texto** | | sin correo — se copia tal cual del `Nombre` de `CATALOGO_PERSONAL1` |
| `EncargadoCodigo` | Texto | | el `CODIGO` del encargado |
| `Activa` | Sí/No | ✔ | |

---

## 3. `BG_Kits`  — cabecera de los kits de ciclo

| Columna | Tipo | Índice | Observación |
|---|---|:---:|---|
| `Title` | Texto | ✔ | `CodigoKit` — KIT-TOM, KIT-CHI, KIT-PEP, KIT-CUC, KIT-SAN |
| `NombreKit` | Texto | | "Kit Ciclo Tomate", "Kit Sanitación"… |
| `Cultivo` | Elección | | mismos valores que `BG_Areas.Cultivo` |
| `Activo` | Sí/No | ✔ | |

---

## 4. `BG_KitDetalle`  — renglones de cada kit

| Columna | Tipo | Índice | Observación |
|---|---|:---:|---|
| `Title` | Texto | ✔ | `CodigoKit` (vuelve a aparecer en cada renglón) |
| `Clave` | Texto | ✔ | la misma `Clave` de `BG_Catalogo` (Codigo+Bodega) |
| `CantidadEstandar` | Número | | entero |

---

## 5. `BG_Salidas`  — cabecera (una por entrega)

| Columna | Tipo | Índice | Observación |
|---|---|:---:|---|
| `Title` | Texto | ✔ | igual a `NoSalida` |
| `NoSalida` | Texto | ✔ | `"SAL-" & Text(ID; "0000")` lo pone la app con el ID autogenerado |
| `FechaEntrega` | **Fecha y hora** | ✔ | `Now()` automático — nadie la escribe |
| `TipoCarga` | Elección | | `KIT` · `UNIDAD` |
| `CodigoKit` | Texto | | solo si `TipoCarga = "KIT"` |
| `CodigoArea` | Texto | ✔ | del popup de área |
| `Responsable` | **Persona** | | **la única columna Persona** — es `User()`, el que entrega en bodega |
| `Encargado` | **Texto** | | encargado del área, elegido del combo (sin correo) |
| `EncargadoCodigo` | Texto | | `CODIGO` del encargado desde `CATALOGO_PERSONAL1` |
| `Recibe` | **Texto** | | persona que llegó a recoger el material (sin correo) |
| `RecibeCodigo` | Texto | | `CODIGO` de quien recibe |
| `Observacion` | Texto multilínea | | |
| `EstadoSalida` | Elección | ✔ | `ABIERTA` · `CERRADA` |

---

## 6. `BG_SalidaDetalle`  — renglones de la salida (aquí vive el estatus)

| Columna | Tipo | Índice | Observación |
|---|---|:---:|---|
| `Title` | Texto | ✔ | igual a `NoSalida` |
| `NoSalida` | Texto | ✔ | enlaza con `BG_Salidas` |
| `CodigoArea` | Texto | ✔ | para filtrar sin unir listas |
| `Clave` | Texto | ✔ | enlaza con `BG_Catalogo` |
| `Codigo` | Texto | | se duplica para evitar `LookUp` en galerías |
| `Descripcion` | Texto | | igual, snapshot al momento de entregar |
| `Tipo` | Elección | | `RETORNABLE` · `UN SOLO USO` |
| `Categoria` | Elección | | mismos valores que `BG_Catalogo` |
| `Bodega` | Elección | | `BODEGA 1` · `CONTENEDOR` |
| `Unidad` | Texto | | |
| `CantEntregada` | Número | | entero |
| `CantBuena` | Número | | entero; piezas devueltas en buen estado |
| `CantDanada` | Número | | entero; piezas devueltas dañadas |
| `CantExtraviada` | Número | | entero; piezas no devueltas y no localizadas |
| `Estatus` | Elección | ✔ | `ASIGNADO` · `ENTREGADO` · `EN USO` · `DEVUELTO PARCIAL` · `DEVUELTO` · `DAÑADO` · `EXTRAVIADO` · `DE BAJA` · `CONSUMIDO` · `VENCIDO` |
| `ComentarioLinea` | Texto multilínea | | obligatorio cuando hay piezas dañadas o extraviadas |
| `FechaDevolucion` | **Fecha y hora** | | `Now()` al cerrar el renglón |
| `RecibeDevolucion` | **Persona** | | `User()` del bodeguero que recibe |
| `DevueltoPor` | **Texto** | | el nombre del encargado o de quien recibió, de `CATALOGO_PERSONAL1` |
| `Abierta` | **Sí/No** | ✔ | **CLAVE** · `true` mientras siga afuera · `false` al cerrar — permite delegación |

---

## 7. `BG_Ingresos`  — bitácora de lo que entró desde Bodega General

| Columna | Tipo | Índice | Observación |
|---|---|:---:|---|
| `Title` | Texto | ✔ | `"ING-" & Text(ID; "0000")` |
| `IdSalidaGeneral` | Número | ✔ | el `ID` del renglón original en `SALIDAS` |
| `FechaIngreso` | **Fecha y hora** | | `Now()` |
| `Codigo` | Texto | | |
| `Descripcion` | Texto | | |
| `Cantidad` | Número | | entero |
| `Unidad` | Texto | | |
| `Tipo` | Elección | | `RETORNABLE` · `UN SOLO USO` |
| `BodegaDestino` | Elección | | `BODEGA 1` · `CONTENEDOR` |
| `AreaUso` | Texto | | copiado de `SALIDAS.AreaUso` |
| `Responsable` | Texto | | copiado de `SALIDAS.Responsable` (como viene) |
| `IngresadoPor` | **Persona** | | `User()` del bodeguero |

---

## 8. `BG_Bajas`  — artículos sacados del inventario

| Columna | Tipo | Índice | Observación |
|---|---|:---:|---|
| `Title` | Texto | ✔ | el `Codigo` del artículo |
| `Clave` | Texto | ✔ | enlace a `BG_Catalogo` |
| `Descripcion` | Texto | | |
| `Bodega` | Elección | | `BODEGA 1` · `CONTENEDOR` |
| `Cantidad` | Número | | entero |
| `FechaBaja` | **Fecha y hora** | | `Now()` |
| `NoSalida` | Texto | | de dónde salió esta pieza |
| `CodigoArea` | Texto | | de qué área se dañó / perdió |
| `Motivo` | Texto multilínea | | comentario obligatorio |
| `Reporto` | **Texto** | | encargado o quien recibió, de `CATALOGO_PERSONAL1` |
| `RegistroPor` | **Persona** | | `User()` del bodeguero que registra |

---

## 9. `SALIDAS`  — **lista EXISTENTE** de Bodega General  _(solo se agrega 1 columna)_

| Columna | Tipo | Índice | Observación |
|---|---|:---:|---|
| `ID` | Número | ✔ | nativo de SharePoint |
| `Codigo` (o `IdCategoria`) | Texto | | **ya existe** |
| `Descripcion` | Texto | | **ya existe** |
| `Cantidad` | Número | | **ya existe** |
| `Unidad` | Texto | | **ya existe** |
| `AreaUso` | Texto | | **ya existe** |
| `Responsable` | Texto | | **ya existe** |
| **`ProcesadoBodega`** | **Sí/No** | ✔ | **NUEVA** · predeterminado `No` · la app la pone en `true` tras ingresar |

---

## 10. `CATALOGO_PERSONAL1`  — **lista EXISTENTE** de personal  _(no se modifica el esquema, solo se recomienda indexar 2 columnas)_

| Columna | Tipo | Índice | Observación |
|---|---|:---:|---|
| `Nombre` | Texto | | el valor que se guarda en `Encargado`, `Recibe`, `DevueltoPor`, `Reporto` |
| `Activo` | **Texto `"SI"` / `"NO"`** | ✔ *(recomendado)* | **ojo: es texto, no Sí/No** · filtrar con `Activo = "SI"` |
| `Rol` | Elección | ✔ *(recomendado)* | sus valores actuales — ajustar las constantes `nfRolEncargado` y `nfRolRecibe` en `App.Formulas` |
| `Categoria` | Elección | | filtro secundario si hace falta |
| `CODIGO` | Texto | | se guarda junto al nombre para trazabilidad |
| `DEPARTAMENTO` | Texto | | informativo en el combo |
| `INGRESO` | Fecha | | no se usa en esta app |

---

## Resumen de las listas

| # | Lista | ¿Es nueva? | Para qué |
|---|---|:---:|---|
| 1 | `BG_Catalogo` | NUEVA | maestro de artículos por código y bodega |
| 2 | `BG_Areas` | NUEVA | invernaderos y semilleros |
| 3 | `BG_Kits` | NUEVA | cabecera de kits de ciclo |
| 4 | `BG_KitDetalle` | NUEVA | renglones de cada kit |
| 5 | `BG_Salidas` | NUEVA | cabecera de cada entrega |
| 6 | `BG_SalidaDetalle` | NUEVA | renglones con estatus y cantidades |
| 7 | `BG_Ingresos` | NUEVA | bitácora de ingresos desde Bodega General |
| 8 | `BG_Bajas` | NUEVA | artículos sacados del inventario |
| 9 | `SALIDAS` | **existente** · agregar `ProcesadoBodega` | puente con Bodega General |
| 10 | `CATALOGO_PERSONAL1` | **existente** · indexar `Activo` y `Rol` | personal sin correo (encargados y quien recibe) |

**Total: 8 listas nuevas · 2 listas existentes tocadas (1 columna nueva + 2 índices sugeridos).**

---

## Configuración de la app en Power Apps

| Opción | Valor |
|---|---|
| Configuración → General → Límite de filas de datos | **2000** |
| Configuración → Próximas características → Fórmulas con nombre | **Activado** |
| `App.StartScreen` | `scrMenu` |
| `App.SizeBreakpoints` | `[600; 760; 1024; 1366]` |

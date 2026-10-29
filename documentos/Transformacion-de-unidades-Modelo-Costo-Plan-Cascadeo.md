# Transformación de unidades — Modelo Costo Plan con Cascadeo

Fecha: 2026-10-29

## 1. Propósito y alcance

El modelo convierte cada cantidad de la lista de materiales (LM) a una unidad común antes de calcular `LM_RENDIMIENTO` y los costos; si un factor de conversión no se aplica, la cantidad de un componente queda mal por el factor completo (por ejemplo, 3,785 al pasar de galón a litro).

Este documento explica, paso a paso y con ejemplos, cómo el notebook `MODELO_COSTO_PLAN_CASCADEO` (rama `main`, productiva, y rama `dev`, de pruebas) transforma las unidades:

- de dónde salen las unidades y los factores (tablas SAP);
- cómo se elige el enfoque **Maestro** o **Componente** según el centro;
- cómo se calcula la cantidad de cada componente en cada nivel del árbol de la LM y en la cascada;
- qué se encontró hasta ahora en el caso BA021738 del centro RP04.

Los números de celda citados corresponden a la rama `dev`; en `main` las celdas están desplazadas una o dos posiciones. Los ejemplos con números inventados se marcan como tales; el caso BA021738 usa datos reales del modelo.

## 2. Nomenclatura

Se usa la misma nomenclatura de la documentación *Modelo Costos Plan Cascadeo – Databricks v1.4*: campos SAP en mayúsculas, columnas y variables del notebook en `código`, y JT = Jarabe Terminado. Una diferencia a tener presente: la documentación llama **IDNRK** al componente de la LM (nombre SAP), mientras que el notebook usa la columna `IDKNR`; son el mismo dato.

| Concepto | Nombre SAP | Nombre en el notebook | Qué es |
| --- | --- | --- | --- |
| Centro | WERKS | `Codigo_de_Centro_Productivo_codigo_de_planta`, `vFiltroCentro` | Planta que se calcula; es el parámetro `FILTRO_CENTRO` |
| Material explosionado | MATNR | `Codigo_de_Material`, `MATNR` | Material cuya LM se explota (nivel 1 = el que se produce) |
| Componente | IDNRK | `IDKNR` | Material que consume una línea de la LM |
| Padre | — | `padre`, `IDKNR_PADRE` | Material al que pertenece la línea; en el nivel 1 es el material explosionado |
| Alternativa de LM | STLAL | `Alternativa` | Alternativa de la lista de materiales |
| Puesto de trabajo | ARBPL | `Puesto_de_Trabajo`, `ARBPL` | Puesto donde se produce |
| Cantidad del componente | MENGE | `MENGE` | Cantidad del componente por cada `BMENG` de la LM |
| Cantidad base de la LM | BMENG | `BMENG` | Cantidad base de la LM (por ejemplo, 450 GLL) |
| Unidad de la línea (componente) | MEINS (tabla ECC_BOM) | `UM` | Unidad en que se escribe el componente en la LM |
| Unidad base de la LM | BMEIN | `UM_MAT_BASE` | Unidad en que está expresada `BMENG` |
| Unidad base del material | MEINS (maestro de materiales) | `MEINS`, `UMB_Material` | Unidad base del material en SAP |
| Unidad de fabricación | FRTME | `FRTME` | Unidad en que se fabrica por planta |
| Unidad de producción | — | `UMB_PROD` | `COALESCE(FRTME, MEINS)`: unidad de la `Producción` |
| Unidad alternativa | MEINH | `MEINH` | Unidad para la que existe un factor de conversión |
| Numerador / denominador | UMREZ / UMREN | `UMREZ`, `UMREN` | 1 MEINH = UMREZ / UMREN de la unidad base del material |
| Unidad con que entra el padre | — | `UM_MAT_BASE_PADRE` | Unidad (`UM`) con que el padre aparece en la línea del nivel anterior |
| Unidad base del padre | — | `MEINS_PADRE` | `MEINS` del padre |
| Nivel y ruta | — | `Nivel`, `RUTA_ARBOL` | Profundidad en el árbol y camino `material/padre/…` |
| Producción y rendimiento | — | `Producción`, `Rendimiento` | Volumen y rendimiento de la hoja A_Volumen |
| Factores aplicados | — | `FACTOR_CONVERSION_UMB` (= UMREZ), `FACTOR_CONVERSION_UMB_UMREN` (= UMREN) | Factores elegidos para la línea |
| Factores de la línea | — | `FACTOR_CONVERSION_UMB_LM`, `FACTOR_CONVERSION_UMB_UMREN_LM` | Factores de la unidad de la línea a la base de la LM |
| Resultados | — | `CANTIDAD_LM`, `LM_RENDIMIENTO`, `PRODUCCION_EN_UNIDADES` | Cantidad de LM, su valor entre `Rendimiento`, y producción convertida |
| Enfoque | — | `vReglaConversionMaestro_o_Componente` | `'Maestro'` o `'Componente'` según el centro |

## 3. Fuentes de datos

La conversión usa cuatro tablas del catálogo `{cat_brn}.sap_s4hana` (capa bronze) y la hoja de volumen cargada desde Excel.

| Tabla / fuente | Variable en el notebook | Campos que se usan | Para qué sirve |
| --- | --- | --- | --- |
| `tbl_pp_ypp_extract_ecc_bom` | `df_lista_mat` | MATNR, STLAL, WERKS, IDNRK, MENGE, BMENG, MEINS (→ `UM`), BMEIN (→ `UM_MAT_BASE`), ARBPL | Lista de materiales; se filtra por `WERKS = FILTRO_CENTRO`, `STLAN = '1'` y `PLNTY = 'N'` |
| `tbl_lo_0material_attr_new` | se une a `df_lista_mat` por IDNRK | MATNR, MEINS | Unidad base del componente (`MEINS`) y del material (`UMB_Material`) |
| `tbl_lo_y_marc_attr` | `df_unit_base_prod` | WERKS, MATNR, FRTME | Unidad de fabricación por planta; con `MEINS` forma `UMB_PROD = COALESCE(FRTME, MEINS)` |
| `tbl_mm_0mat_unit_attr_new` | `df_unit_convert` | MATNR, MEINH, UMREZ, UMREN | **Tabla de factores**: unidades alternativas de cada material. Por sus campos corresponde a la tabla SAP MARM |
| Hoja `A_Volumen` (Excel) | `df_A_volumen` | Producción, Alternativa, Puesto_de_Trabajo, Periodo_Mes, Año, Versión_de_simulación | Volumen a producir por material, puesto y mes |

En todas las tablas el código de material se normaliza quitando los ceros a la izquierda (`regexp_replace(MATNR, '^0+', '')`), para que los cruces coincidan.

**Cómo se lee la tabla de factores.** Cada fila dice que 1 unidad `MEINH` equivale a `UMREZ / UMREN` unidades base del material. La unidad base misma aparece con 1 / 1. Ejemplo real del material 607352: GLL 1/1, ST 1/1 y L 15.625 / 59.146 (= 0,26417), es decir, 1 L = 0,26417 GLL.

## 4. Selección del enfoque: Maestro o Componente

El enfoque se decide una sola vez por ejecución, según el valor del parámetro `FILTRO_CENTRO` (celda de parámetros: celda 5 en `main`, celda 6 en `dev`). La comparación es **exacta**: un código escrito distinto (`RP04AA` en vez de `RP04_AA`) cae en Componente.

```python
if vFiltroCentro in [...]:
    vReglaConversionMaestro_o_Componente = 'Maestro'
else:
    vReglaConversionMaestro_o_Componente = 'Componente'
```

| Rama | Centros con enfoque Maestro |
| --- | --- |
| `main` (productiva) | EP01, EP00, RP00, RP04, SP01, SP00 |
| `dev` (pruebas) | EP01, EP00, RP00, RP04, RP04_AA |

Cualquier otro centro usa Componente. El criterio, según el comentario del propio código, es el siguiente:

- **Componente:** se convierten todos los componentes de la lista de materiales a la unidad del material explosionado.
- **Maestro:** se convierte el material principal explosionado y no se convierten los componentes de ese material; el factor se toma del padre de cada línea.

Las diferencias exactas en el código están en la sección 7. Para comprobar qué enfoque está activo, en una celda de `dev` se imprime `print(repr(vFiltroCentro), vReglaConversionMaestro_o_Componente)`.

## 5. Proceso paso a paso

Premisa del código: la unidad base de la LM (`BMEIN`, columna `UM_MAT_BASE`) se asume igual a la unidad base del material principal (`MEINS` del maestro de materiales).

1. **Leer la lista de materiales.** `df_lista_mat` toma la LM del centro (`WERKS = FILTRO_CENTRO`, `STLAN = '1'`, `PLNTY = 'N'`), quita los ceros a la izquierda de MATNR e IDNRK y trae la unidad base del componente (`MEINS`) desde `tbl_lo_0material_attr_new`.
2. **Leer el volumen.** `df_A_volumen` (hoja `A_Volumen`) aporta `Producción`, `Alternativa`, `Puesto_de_Trabajo`, mes, año y versión de simulación por material.
3. **Preparar las unidades.** `df_unit_base_prod` calcula `UMB_PROD = COALESCE(FRTME, MEINS)` por planta y material. `df_unit_convert` carga la tabla de factores (MEINH, UMREZ, UMREN).
4. **Explosionar el árbol.** El nivel 1 une el volumen con la LM por material, alternativa, centro y puesto de trabajo. Los niveles 2 en adelante abren la LM de cada componente que tiene la suya (alternativa `'01'`), hasta 20 niveles. En cada fila quedan `Nivel`, `padre`, `RUTA_ARBOL`, `UM`, `UM_MAT_BASE`, `MEINS`, `UM_MAT_BASE_PADRE` y `MEINS_PADRE`; `STLKZ = 'X'` marca los componentes que tienen LM propia.
5. **Agregar las unidades de producción.** A cada fila se le añaden `UMB_PROD` y `UMB_Material` desde `df_unit_base_prod`.
6. **Elegir los factores y calcular `CANTIDAD_LM`.** Se hace por nivel y según el enfoque (secciones 6 y 7). El nivel 1 convierte la `Producción` de `UMB_PROD` a la unidad base; los niveles siguientes buscan el factor del padre (Maestro) o del componente (Componente). Un factor ausente en los niveles 2 en adelante se rellena con 1; en el nivel 1 queda vacío y el resultado puede salir como "error en conversion".
7. **Calcular `LM_RENDIMIENTO`.** `LM_RENDIMIENTO = CANTIDAD_LM / Rendimiento`.
8. **Cascadear.** Para cada componente con `STLKZ = 'X'` se abren sus hijos y se recalcula su `LM_RENDIMIENTO` a partir del `LM_RENDIMIENTO` del padre, hasta 30 niveles. Estas cantidades de cascada son las que alimentan los cálculos de componentes posteriores.

## 6. Fórmulas de conversión

En todas, UMREZ y UMREN son los de la tabla de factores (`FACTOR_CONVERSION_UMB` y `FACTOR_CONVERSION_UMB_UMREN`). Un factor "directo" es UMREZ / UMREN y uno "inverso" es UMREN / UMREZ.

**Nivel 1** (material que se produce; igual en Maestro y Componente). El factor se busca con el propio material y `MEINH = UMB_PROD`:

```
CANTIDAD_LM = Producción × (UMREZ / UMREN) × (MENGE / BMENG)
```

**Niveles 2 en adelante.** Primero se elige el factor:

- Si `UM_MAT_BASE = UM_MAT_BASE_PADRE`, UMREZ = 1 y UMREN = 1 (no hay conversión).
- Si no, se busca en la tabla de factores con `MEINH = UM_MAT_BASE_PADRE`. En Maestro la fila es la del **padre**; en Componente, la del **componente**.
- Si no hay fila, UMREZ = 1 y UMREN = 1.

Luego se calcula la cantidad:

```
CANTIDAD_LM = Producción × F × (MENGE / BMENG)
```

donde F es el factor inverso (UMREN / UMREZ) cuando se cumple la condición de la tabla siguiente y el factor directo (UMREZ / UMREN) en caso contrario.

| Enfoque | Condición para usar el factor inverso (niveles 2 en adelante) |
| --- | --- |
| Maestro | `UM_MAT_BASE = MEINS_PADRE` |
| Componente | `UM_MAT_BASE = UM_MAT_BASE_PADRE` |

**Rendimiento.** En cualquier nivel: `LM_RENDIMIENTO = CANTIDAD_LM / Rendimiento`.

**Cascada** (cada hijo se calcula desde su padre ya calculado):

```
LM_RENDIMIENTO_hijo = LM_RENDIMIENTO_padre × G × (MENGE / BMENG) / Rendimiento
```

donde G es el factor inverso si `UM_MAT_BASE = UM_MAT_BASE_PADRE` y el factor directo en caso contrario (misma condición en los dos enfoques). La primera cantidad de la cascada (`CANTIDAD_LM`) usa la condición `MEINS_PADRE = UM_MAT_BASE_PADRE` en Maestro y `UM_MAT_BASE = UM_MAT_BASE_PADRE` en Componente.

Nota sobre la documentación v1.4: el glosario define `LM_RENDIMIENTO` como (Producción × MENGE / BMENG) × Rendimiento, y el código divide entre `Rendimiento`. Es la inconsistencia que ya estaba señalada en los comentarios de esa versión.

## 7. Diferencias entre Maestro y Componente

Comparando celda por celda las dos ramas de `dev` (celdas 17 y 19 para Componente frente a 18 y 20 para Maestro), solo cambian tres condiciones; el nivel 1, el árbol y el resto de la cascada son iguales.

| Punto | Componente | Maestro |
| --- | --- | --- |
| Material con que se busca el factor (niveles 2 en adelante) | El componente (`a.IDKNR = b.MATNR`) | El padre (`a.padre = b.MATNR`); cambio del 23.07.2026 por el caso EP01 |
| Condición factor inverso en `CANTIDAD_LM` de los niveles 2 en adelante | `UM_MAT_BASE = UM_MAT_BASE_PADRE` | `UM_MAT_BASE = MEINS_PADRE` |
| Condición factor inverso en la primera cantidad de la cascada | `UM_MAT_BASE = UM_MAT_BASE_PADRE` | `MEINS_PADRE = UM_MAT_BASE_PADRE` |

**Por qué importa la primera fila.** En el caso BA021738 (sección 9), el componente 436527 solo tiene filas de unidad ST en la tabla de factores, mientras que su padre 607352 tiene la fila en litros (L). Con Maestro se encuentra el factor de litro a galón del padre; con Componente no se encontraría ninguno y quedaría en 1 / 1, sin conversión.

## 8. Ejemplos resueltos

Los números de producción y cantidades de estos ejemplos son inventados para ilustrar el cálculo; los factores de la tabla en el ejemplo B son los reales del material 607352.

### Ejemplo A: nivel 1 (conversión de la producción)

Material PT-X, `Producción` = 1.000 CS, `UMB_PROD` = CS, unidad base GLL. En la tabla de factores, CS: UMREZ = 5 y UMREN = 2 (1 CS = 2,5 GLL). La LM tiene `BMENG` = 1.000 GLL y el jarabe JT con `MENGE` = 120 GLL.

```
CANTIDAD_LM = 1000 × (5 / 2) × (120 / 1000) = 300 GLL de JT
```

Es lo mismo que convertir primero la producción (2.500 GLL) y tomar 12 % de ella.

### Ejemplo B: nivel 2 con cambio de unidad (enfoque Maestro)

Ruta `BA021738 / 607352 / 436527`. El 607352 entra a BA021738 en **litros** (`UM_MAT_BASE_PADRE` = L), pero su propia LM está en **galones**: `BMENG` = 450 GLL y el componente 436527 con `MENGE` = 1 ST. Factores reales del 607352:

| MATNR | MEINH | UMREZ | UMREN | Significado |
| --- | --- | --- | --- | --- |
| 607352 | GLL | 1 | 1 | Unidad base |
| 607352 | L | 15.625 | 59.146 | 1 L = 0,26417 GLL |
| 607352 | ST | 1 | 1 | Unidad |

Supongamos (dato inventado) que el padre 607352 tiene `LM_RENDIMIENTO` = 100.000 L y `Rendimiento` = 1.

1. Como `UM_MAT_BASE` (GLL) es distinto de `UM_MAT_BASE_PADRE` (L), el modelo busca la fila del padre con `MEINH = L`: UMREZ = 15.625 y UMREN = 59.146.
2. La cascada usa el factor directo: G = 15.625 / 59.146 = 0,26417.
3. Cantidad del hijo:

```
LM_RENDIMIENTO_436527 = (100000 × 0,26417 × 1 / 450) / 1 = 58,7 ST
```

**Si el cruce no encuentra la fila**, el factor queda en 1 / 1 y el resultado es 100.000 × 1 / 450 = 222,2 ST: 3,785 veces más (222,2 / 58,7). Es exactamente la diferencia que reportan los usuarios en el caso BA021738.

## 9. Caso BA021738 en RP04: hallazgos

**Reporte del usuario.** En RP04, mes 1, para el componente 436527 (Concentrado) del material BA021738 el cálculo esperado es 238,32 UN y el reporte muestra 902,14. La relación es 902,14 / 238,32 = 3,7854, el factor de galón a litro: una conversión que no se aplicó. Es el mismo problema ya visto en SP00 y SP01.

**Lo comprobado en `dev` (enfoque Maestro, datos reales):**

| Dato | Valor |
| --- | --- |
| Fila de `df_A_volumen` | RP04, puesto 6404, alternativa 1, año 2027, versión CBC_AOP_2027_VF; el Excel muestra `Producción` = 0,00 |
| Fila de `df_result` | Nivel 2, padre 607352, componente 436527, `Producción` = 452.960,45 |
| Unidades de la fila | `UM` = ST, `UM_MAT_BASE` = GLL, `UM_MAT_BASE_PADRE` = L, `MEINS_PADRE` = GLL, `UMB_PROD` = CS |
| LM | `MENGE` = 1, `BMENG` = 450 |
| Factores que mostró la fila | `FACTOR_CONVERSION_UMB` = 1 y `FACTOR_CONVERSION_UMB_UMREN` = 1 |
| `CANTIDAD_LM` de la fila | 452.960,45 × 1 × 1 / 450 = 1.006,58 |
| Tabla de factores del 607352 | L: 15.625 / 59.146 (= 0,26417); GLL y ST: 1 / 1 |

**Conclusiones hasta ahora.**

1. La fila del 607352 en litros **existe** en la tabla de factores, así que con Maestro el modelo debería aplicar 0,26417 y no 1 / 1.
2. Se descartó una diferencia de texto en la clave del cruce: `padre` tiene 6 caracteres y la unidad es `L` (hex 4C) en los dos lados.
3. Un factor 1 / 1 es justo lo que daría el enfoque Componente, que buscaría la unidad L en el componente 436527, el cual solo tiene ST.
4. La cifra de la fila (1.006,58) no coincide con la del reporte (902,14), porque el reporte final sale de la cascada, que parte de la cantidad del padre.

**Próximas comprobaciones.**

- Imprimir `print(repr(vFiltroCentro), vReglaConversionMaestro_o_Componente)` para confirmar qué enfoque estaba activo. Ojo: la lista de `dev` incluye `RP04_AA`, con guión bajo, y la fila de volumen muestra el centro RP04.
- Reproducir el cruce solo para esta fila (padre 607352, `MEINH = L`) y ver si devuelve 15.625 / 59.146.
- Si el cruce funciona, volver a correr las celdas 16 y 18 y comparar con `df_result`, por si el resultado estaba desactualizado.

## 10. Puntos abiertos y validaciones pendientes

- [ ] **Caso BA021738 (RP04):** confirmar el enfoque activo y por qué el factor del padre 607352 en litros no se aplicó (sección 9).
- [ ] **Dirección del factor en Maestro:** para los niveles 2 en adelante, `CANTIDAD_LM` usa el factor inverso cuando `UM_MAT_BASE = MEINS_PADRE`, mientras que la cascada usa el inverso cuando `UM_MAT_BASE = UM_MAT_BASE_PADRE`. En BA021738 (GLL, GLL y L) esas dos condiciones dan direcciones contrarias una vez que el factor se encuentra (0,26417 frente a 3,785). Validar cuál de las dos cifras alimenta el reporte.
- [ ] **Nivel 1 sin fila de factor:** si el material no tiene fila para `UMB_PROD`, el factor queda vacío (no se rellena con 1) y la cantidad puede salir como "error en conversion".
- [ ] **Producción de la fila:** el Excel muestra 0,00 para BA021738 en el mes 1 y `df_result` muestra 452.960,45; confirmar de dónde sale esa producción.
- [ ] **Nombre del centro:** `dev` usa `RP04_AA` y `main` no lo incluye en la lista de Maestro; confirmar el valor real de `FILTRO_CENTRO` y alinear las dos ramas.
- [ ] **SP00 y SP01:** el usuario reporta el mismo síntoma; la nota de la celda 0 de `dev` indica que SP00 "no cuadra".
- [ ] **Documentación v1.4:** unificar la fórmula de `LM_RENDIMIENTO` entre el glosario y el código, y agregar a la v1.4 esta explicación de la transformación de unidades.

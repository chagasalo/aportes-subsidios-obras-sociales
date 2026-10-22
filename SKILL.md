---
name: aportes-subsidios-procesador
description: "Procesa y concilia aportes, contribuciones y subsidios de una obra social argentina a partir de los archivos de AFIP/ARCA y la SSSalud (DDJJ, transferencias OS, SUMA/SUMA65/SUMA70/SUMARTE/SANO y expedientes): lectura por posiciones, imputación por CUIL con prorrateo de doble peso y estimación de DDJJ en cero."
---

# Procesador de aportes y subsidios de obras sociales

Especificación para leer, imputar y conciliar los archivos que una obra social del Sistema Nacional del Seguro de Salud recibe de AFIP/ARCA y de la Superintendencia de Servicios de Salud. Es genérica: sirve para cualquier obra social, identificada por su código RNOS.

## Convenciones

- **Código de obra social:** `<CODIGO_OS>` es el código RNOS de 4 dígitos que aparece en los nombres de archivo; `<CODOSOC>` es el código de 6 dígitos que aparece dentro de los registros. En los ejemplos se usa el código ficticio `9999` / `999905`.
- **Posiciones:** cada campo se indica en posiciones 1-based (como en los diseños de registro oficiales) y como slice de Python 0-based (`line[a:b]`).
- **Importes:** campos numéricos sin separador decimal; los 2 últimos dígitos son decimales (`int(campo) / 100`). Campos vacíos o en blanco = 0.
- **Períodos:** se informan como `AAMM` y se normalizan a `AAAAMM`. Regla de siglo: `AA >= 90` → `19AA`; si no → `20AA`.
- **CUIT/CUIL:** 11 dígitos, sin guiones. `00000000000` significa "sin CUIL informado".
- **Datos personales:** los archivos contienen CUIL, remuneraciones y datos de salud. Procesarlos solo en el equipo del usuario, no copiarlos a resúmenes compartidos y no versionarlos.

---

## 1. Diseños de registro

### 1.1. Declaraciones juradas (nómina DDJJ)

- **Archivo:** `DDJJ<CODIGO_OS>-*.txt` (ej. `DDJJ9999-20260310.txt`).
- **Registros:** ignorar cabecera (empieza con `HF`) y pie (empieza con `TF`). Procesar solo los registros de detalle, que empiezan con el `<CODOSOC>` de la obra social.
- **Última declaración vigente:** para no duplicar deuda, quedarse con una sola fila por CUIT + CUIL + Período, ordenando de forma descendente por `FECPRESENT`, `SECOBLIG` y `ORIGRECT` (la rectificativa más reciente prevalece).

Diseño del registro de detalle (longitud 230):

| Campo | Pos. (1-based) | Slice | Largo | Notas |
|---|---|---|---|---|
| Código OS (`CODOSOC`) | 1-6 | `[0:6]` | 6 | |
| Período | 7-10 | `[6:10]` | 4 | AAMM |
| CUIT empresa | 11-21 | `[10:21]` | 11 | |
| CUIL empleado | 22-32 | `[21:32]` | 11 | |
| Remuneración imponible (`REMOSIMP`) | 33-46 | `[32:46]` | 14 | importe |
| Importe adicional OS (`IMPOSAD`) | 47-60 | `[46:60]` | 14 | importe |
| Zona geográfica | 61-62 | `[60:62]` | 2 | |
| Cantidad grupo familiar (`GRPFAM`) | 63-64 | `[62:64]` | 2 | |
| Cantidad adherentes (`NOGRPFAM`) | 65-66 | `[64:66]` | 2 | |
| Secuencia obligación (`SECOBLIG`) | 67-69 | `[66:69]` | 3 | |
| Condición CUIL (`CONDCUIL`) | 70-71 | `[69:71]` | 2 | tabla SICOSS 05 |
| Situación de revista (`SITCUIL`) | 72-73 | `[71:73]` | 2 | tabla SICOSS 12 |
| Actividad (`ACTIVDES`) | 74-76 | `[73:76]` | 3 | tabla SICOSS 01 |
| Modalidad de contratación (`MODALIDAD`) | 77-79 | `[76:79]` | 3 | tabla SICOSS 08 |
| Código de siniestrado (`CODSINI`) | 80-81 | `[79:81]` | 2 | tabla SICOSS 17 |
| Aporte adicional OS (`APADIOS`) | 82-95 | `[81:95]` | 14 | importe |
| Versión de aplicativo (`VERSION`) | 96-97 | `[95:97]` | 2 | |
| Rem. Dec. 1273/02 (`REM5`) | 98-111 | `[97:111]` | 14 | importe |
| Esposa (`ESPOSA`) | 112 | `[111:112]` | 1 | |
| Excedente aportes (`EXCOSAPO`) | 113-124 | `[112:124]` | 12 | importe |
| Declaró retenciones (`INDRETAPO`) | 125 | `[124:125]` | 1 | S/N |
| Declaró excedente contrib. (`INDEXCCON`) | 126 | `[125:126]` | 1 | S/N |
| Fecha de presentación (`FECPRESENT`) | 127-136 | `[126:136]` | 10 | AAAA-MM-DD |
| Fecha de proceso (`FECPROC`) | 137-146 | `[136:146]` | 10 | AAAA-MM-DD |
| Original/Rectificativa (`ORIGRECT`) | 147 | `[146:147]` | 1 | 0 = original, 1 = rectificativa |
| Aporte OS (`APOBSOC`) | 148-158 | `[147:158]` | 11 | importe |
| Contribución OS (`CONOS`) | 159-172 | `[158:172]` | 14 | importe |
| Remuneración total (`REMTOT`) | 173-186 | `[172:186]` | 14 | importe |
| Uso futuro | 187 | `[186:187]` | 1 | |
| OS informada en DDJJ (`CODOSOC-INFORM`) | 188-193 | `[187:193]` | 6 | |
| Rem. base contribuciones (`REMBASE-COS`) | 194-207 | `[193:207]` | 14 | importe |
| Release versión (`RELEASE-VER`) | 208-209 | `[207:209]` | 2 | |
| Uso futuro | 210-230 | `[209:230]` | 21 | |

### 1.2. Transferencias (archivo OS)

- **Archivo:** `OS<CODIGO_OS>-*.txt` (ej. `OS9999-20260310.txt`).

| Campo | Pos. (1-based) | Slice | Largo | Notas |
|---|---|---|---|---|
| Organismo destino (`ORGAN`) | 1-4 | `[0:4]` | 4 | |
| Concepto (`CODCONC`) | 5-7 | `[4:7]` | 3 | ver sección 3 |
| Importe | 8-22 | `[7:22]` | 15 | importe |
| Débito/Crédito (`INDDBCR`) | 23 | `[22:23]` | 1 | C = suma, D = reversión (resta) |
| Fecha de transferencia (`FECPROC`) | 24-33 | `[23:33]` | 10 | AAAA-MM-DD |
| Fecha de recaudación (`FECREC`) | 34-43 | `[33:43]` | 10 | AAAA-MM-DD |
| CUIT empresa | 44-54 | `[43:54]` | 11 | |
| Período | 55-58 | `[54:58]` | 4 | AAMM |
| Identificación de transferencia/pago | 59-73 | `[58:73]` | 15 | |
| CUIL/CUIT del aportante | 74-84 | `[73:84]` | 11 | ver abajo |
| Banco | 85-87 | `[84:87]` | 3 | |
| Sucursal | 88-90 | `[87:90]` | 3 | |
| Zona | 91-92 | `[90:92]` | 2 | |
| Uso futuro | 93-98 | `[92:98]` | 6 | |
| Tipo de pago | 99 | `[98:99]` | 1 | P = pago direccionado |
| Marca de apropiación | 100 | `[99:100]` | 1 | P = parcial, C = cumplida |

Clasificación según el CUIL del aportante:
- **Nominado:** CUIL informado y distinto de `00000000000` → se imputa directo a ese CUIL.
- **Global:** CUIL `00000000000` → pago a nivel CUIT; se distribuye entre los empleados declarados por esa empresa en ese período (sección 2).

### 1.3. Subsidios (SUMA, SUMA65, SUMARTE, SUMA70, SANO)

**Formato estándar (SUMAGF, SUMA65GF, SUMARTEGF), registros `DT`/`DE`:**

| Campo | Pos. (1-based) | Slice | Largo | Notas |
|---|---|---|---|---|
| Tipo de registro (`TIPOREG`) | 1-2 | `[0:2]` | 2 | DT, DE, TO |
| CUIT empresa | 3-13 | `[2:13]` | 11 | |
| CUIL titular | 14-24 | `[13:24]` | 11 | |
| Código OS (`CODOSOC`) | 25-30 | `[24:30]` | 6 | |
| Período | 31-34 | `[30:34]` | 4 | AAMM |
| Remuneración imponible | 35-46 | `[34:46]` | 12 | importe |
| Aporte OS | 47-58 | `[46:58]` | 12 | importe |
| Contribución OS | 59-70 | `[58:70]` | 12 | importe |
| Subsidio (`SUBSIDIO`) | 71-82 | `[70:82]` | 12 | importe |
| OS relacionada | 83-88 | `[82:88]` | 6 | |
| Indicador parcial/total | 89 | `[88:89]` | 1 | |
| Débito/Crédito | 90 | `[89:90]` | 1 | |
| Motivo de excepción | 91 | `[90:91]` | 1 | |
| Cápitas por grupo de edad | 92-119 | `[91:119]` | 28 | 14 bloques de 2 dígitos |

Cápitas = suma de los 14 bloques (`line[91:93]`, `line[93:95]`, …, `line[117:119]`). Si la suma da 0, se toma 1 (el titular).

Algunos archivos simplificados o preprocesados usan otro diseño: período `[0:6]` (AAAAMM), CUIL `[12:23]`, importe `[26:41]`. Detectar cuál aplica mirando las primeras líneas antes de procesar.

**SUMA70:** período `[6:12]` (AAAAMM), CUIL `[12:23]`, importe `[33:48]` (largo 15). Cápitas = 1.

**SANO / patologías (`SO<CODIGO_OS>-AAAAMMDD.txt`):** mismo diseño que el formato estándar (CUIL `[13:24]`, subsidio `[70:82]`). Cápitas = 1. El período no viene en el registro: es el mes del nombre del archivo menos uno (`SO9999-20260327.txt` → `202602`). No usar `[15:26]` ni `[50:65]`: son offsets viejos que desalinean los campos.

Tipos de registro en el archivo SO:
- `DE` (detalle simple): afiliados con pluriempleo, sin importe de subsidio. Se ignoran para el subsidio.
- `DT` (detalle con subsidio): afiliados con un único empleador. Traen CUIT y subsidio.
- `TO` (total por CUIL): afiliados con pluriempleo, consolidado por CUIL con CUIT en blanco.

Como el análisis es por CUIL: tomar los `DT` (monoempleo) y los `TO` con subsidio mayor a 0 (pluriempleo), y cargarlos bajo el CUIL del afiliado. Sumar `DE` y `TO` del mismo afiliado duplicaría el subsidio.

### 1.4. Expedientes judiciales (EX)

| Campo | Slice | Notas |
|---|---|---|
| Fecha de registro | `[13:23]` | |
| Importe | `[35:50]` | importe |
| Tipo | `[51:52]` | C / D |
| Detalle / carátula | `[52:]` | número de expediente y nombre del amparista |

---

## 2. Imputación por CUIL

### 2.1. Segmentos

Cada concepto de transferencia se asigna a un segmento:

1. **APORTE:** aportes del trabajador (ej. 381, 383, 881).
2. **CONTRIB:** contribuciones patronales (ej. 401, 041, 406).
3. **MONO:** monotributo (ej. C14, C27). Los monotributistas no tienen DDJJ: su análisis sale íntegramente del archivo OS y no se prorratea.

### 2.2. Prorrateo de doble peso para pagos globales

Un pago global (sin CUIL) de un CUIT y período se reparte entre los CUIL que esa empresa declaró en la DDJJ de ese período:

$$\text{Imputado}_{CUIL} = \text{Importe global} \times \frac{\text{Peso}_{CUIL}}{\sum \text{Pesos de los CUIL de la empresa en el período}}$$

- Segmento **APORTE**: peso = `APOBSOC`.
- Segmento **CONTRIB**: peso = `CONOS`.

Si el CUIT no tiene DDJJ para ese período, el importe queda como "global sin distribuir" a nivel CUIT; no se inventa una nómina.

### 2.3. Pesos cuando la DDJJ viene en cero

Si todos los pesos de la empresa en el período son 0, recalcularlos en cascada y usar el primer nivel que dé una suma mayor a 0:

1. **Teórico neto:** aporte y contribución calculados según la sección 2.4.
2. **Remuneración:** `REMTOT` (o `REMOSIMP` si `REMTOT` es 0).
3. **Partes iguales:** peso 1 por cada CUIL declarado.

Registrar en el resultado qué nivel se usó para cada CUIT y período, para poder auditarlo.

### 2.4. Cálculo teórico con tope

El aporte del trabajador tiene tope; la contribución patronal no:

$$\text{Aporte}_{teórico} = \min(REMOSIMP,\ \text{Tope del período}) \times \text{alícuota neta de aporte}$$

$$\text{Contribución}_{teórica} = REMOSIMP \times \text{alícuota neta de contribución}$$

- **Tope:** remuneración máxima sobre la que se calculan los aportes. La actualiza ANSES y AFIP/ARCA la publica en las tablas SICOSS (`resources/sicoss_21_topes_remuneratorios.csv`, columna `os_tope_sueldo_vacaciones`, con la resolución de ANSES en `norma`).
  - Buscar el período con `vigencia_desde`–`vigencia_hasta` (AAAAMM; `999999` = vigente), no con `desde`–`hasta`: en 2008 hay una fila partida en dos vigencias con topes distintos.
  - Usar las columnas `os_*`, no las `ss_*`: hasta 2008-10 el tope de obra social (4.800) era distinto del de seguridad social.
  - Si dos filas incluyen el mismo período, prevalece la de `vigencia_desde` más reciente. Pasa en la planilla oficial con 202610, que figura en la fila de 202609 (hasta 202610) y en la de 202610.
- **Períodos con SAC:** cuando el período incluye sueldo anual complementario, el tope es 1,5 veces el tope del mes (columna `os_tope_sueldo_mas_aguinaldo`). Por defecto son los períodos AAAA06 y AAAA12. El archivo de DDJJ no informa si hubo SAC, así que un SAC proporcional pagado en otro mes (por ejemplo, por una baja) no se detecta: si el aporte declarado supera al teórico en un mes que no es 06 ni 12, revisar esa posibilidad antes de marcarlo como error.
- **Alícuotas netas:** 3 % de aporte y 6 % de contribución, menos el 15 % que va al Fondo Solidario de Redistribución: 2,55 % y 5,10 %. Por período están en `resources/serie_topes_previsionales.csv`.
- **Sin contribución topeada:** no aplicar ningún tope a la contribución, aunque la tabla tenga columnas de tope para la obra social: esos topes son para el aporte.

El mismo cálculo sirve para auditar: un aporte declarado mayor al teórico con tope, o una contribución menor a la teórica, merece revisión.

---

## 3. Conceptos de transferencia

Frecuentes:
- `381`: Aportes de obra social
- `401` / `041`: Contribución obra social
- `383`: Aportes de obra social / intereses resarcitorios
- `406`: Contribución obra social / intereses resarcitorios

Cada concepto tiene un contraconcepto de reversión (ej. `381` ↔ `881`) que viene con indicador `D` y resta. La tabla completa está en `resources/conceptos_transferencia_os.csv` (columnas `codigo`, `descripcion`, `contraconcepto`, `debito_credito`). Ante un código desconocido, buscarlo ahí antes de asignarle un segmento.

---

## 4. Tablas SICOSS

Las tablas de AFIP en `resources/` cumplen dos funciones: interpretar los códigos de la DDJJ (condición, situación de revista, actividad, modalidad, siniestrado) y aplicar el tope de aportes (sección 2.4).

| Archivo | Contenido |
|---|---|
| `sicoss_01_actividades.csv` | Actividades |
| `sicoss_05_condiciones.csv` | Condiciones del CUIL |
| `sicoss_08_modalidades_contratacion.csv` | Modalidades de contratación |
| `sicoss_12_situaciones_revista.csv` | Situaciones de revista |
| `sicoss_17_incapacidades_lrt.csv` | Código de siniestrado / incapacidades |
| `sicoss_19a_perfil_pami.csv` | Perfil de contribuyentes especiales |
| `sicoss_21_topes_remuneratorios.csv` | Topes remuneratorios para aportes (sección 2.4) |
| `sicoss_minimos_previsionales.csv` | Mínimos previsionales |

Notas de la tabla 12 (situaciones de revista), tomadas de la versión anterior de la planilla, que ya no las trae:
- Las SR 5, 6, 10, 11, 12, 13, 14, 15, 21, 33, 42, 44, 45 y 51 admiten remuneración cero. Un afiliado con alguna de ellas y aportes en 0 no es, de por sí, un error.
- Las SR 44, 45 y 49 modifican la compensación desde la versión 44.7.
- La SR 51 se incorpora en la versión 44.11.
- La SR 46 y la 47 se usaron para el retiro voluntario.
- La SR 52 se usó para el Art. 64 del DNU 70/2023: extinción por mutuo acuerdo de las partes (Art. 241, Ley 20.744).
- La SR 53 se usó para la baja de oficio por denuncia.
- La SR 54 se usó para la baja por retiro anticipado o voluntario.
- La SR 55 se usó para el traspaso de personal / RIGI (Ley 27.742).

En la tabla 08, las modalidades marcadas con asteriscos asumen una obra social fija: (*) 119302, (**) 122609, (***) 108506, (****) 111001.

Las tablas 01, 05, 08, 12, 17 y 19a tienen `Código;Desde;Hasta;Descripción;APORTES O.S.;CONTRIBUCIONES O.S.`. Para saber si un código obliga a aportar o contribuir, buscar la fila cuyo rango `Desde`–`Hasta` (AAAAMM) incluya el período: `1` = obligado, `0` = exento. Un mismo código puede tener varias filas con vigencias distintas. Es la primera explicación a revisar cuando un afiliado tiene aportes o contribuciones en 0 en su DDJJ.

`serie_topes_previsionales.csv` resume, por mes desde 2009, las alícuotas netas y los topes salariales (sueldo y sueldo + aguinaldo = 1,5 × tope), con los mismos topes que la tabla oficial. Usa coma decimal. Si las dos fuentes difieren, vale `sicoss_21_topes_remuneratorios.csv`.

**Actualizar los topes:** la versión vigente está en la hoja *Topes_Aportes* de `TablasSICOSSPaginaWebOrganismos.xls`, que AFIP/ARCA publica en https://www.afip.gob.ar/genericos/organismosSegSocial/archivos/TablasSICOSSPaginaWebOrganismos.xls. Antes de calcular, verificar que `sicoss_21_topes_remuneratorios.csv` tenga una fila vigente para el último período a procesar; si la última fila es de un período anterior, bajar la planilla y regenerar el CSV.

---

## 5. Flujo sugerido

1. Inventariar los archivos recibidos y el rango de períodos; detectar el diseño de cada uno con las primeras líneas.
2. Cargar las DDJJ y quedarse con la última declaración vigente (1.1).
3. Cargar transferencias, clasificar cada concepto por segmento y signo, y separar nominados de globales.
4. Distribuir los globales (2.2 y 2.3).
5. Cargar subsidios con la regla de cada formato (1.3).
6. Conciliar por CUIT + CUIL + período: devengado según DDJJ vs. transferido + subsidios. Informar diferencias, globales sin distribuir y conceptos sin clasificar.

## Fuentes de las tablas

- Conceptos de transferencia: listado de AFIP de conceptos para obras sociales.
- Tablas SICOSS y topes de aportes: planilla `TablasSICOSSPaginaWebOrganismos.xls` de AFIP/ARCA (Sistemas de generación de DDJJ - F931), versión consultada el 22-10-2026, con topes vigentes hasta la Resolución 284/2026 de ANSES (período 202610).
- Topes y alícuotas: serie armada a partir de las resoluciones y comunicaciones de AFIP/ARCA. Verificar la vigencia de los últimos meses antes de usarla.

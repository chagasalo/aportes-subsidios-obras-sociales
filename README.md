# Procesador de aportes y subsidios de obras sociales

Skill para agentes de IA (formato `SKILL.md`) que procesa los archivos que una obra social argentina recibe de ARCA y de la Superintendencia de Servicios de Salud, y concilia lo declarado con lo efectivamente transferido, a nivel de cada afiliado.

## Qué problema resuelve

Las obras sociales reciben la recaudación en archivos de texto de ancho fijo: declaraciones juradas por empleado, transferencias por concepto, subsidios (SUMA, SUMA65, SUMA70, SUMARTE, SANO) y expedientes judiciales. Cada formato tiene su diseño de registro, sus excepciones y sus trampas: pagos sin CUIL que hay que prorratear, DDJJ con aportes en cero, rectificativas, reversiones, registros de pluriempleo que duplican importes.

Esta skill documenta esas reglas para que un agente pueda leer los archivos, imputar cada peso al afiliado que corresponde y señalar las diferencias, sin que una persona tenga que recordar cada offset.

## Qué incluye

- `SKILL.md`: diseños de registro, reglas de selección e imputación (prorrateo de doble peso y estimación en cascada) y flujo de conciliación.
- `resources/`: tablas públicas de AFIP normalizadas en CSV (conceptos de transferencia, tablas SICOSS, topes y alícuotas).

## Uso

Copiar la carpeta en el directorio de skills del agente (por ejemplo, `~/.claude/skills/aportes-subsidios-procesador/`) y pedirle que procese una carpeta con los archivos recibidos.

## Datos

El repositorio no contiene datos reales: solo especificaciones y tablas públicas. Los archivos de nómina tienen CUIL, remuneraciones y datos de salud; no deben subirse a ningún repositorio (el `.gitignore` excluye los patrones habituales).

Las tablas se tomaron de publicaciones de ARCA. Verificar su vigencia antes de usarlas en producción.

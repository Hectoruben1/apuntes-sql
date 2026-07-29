# Trampas SQL — Semanas 1 a 5

> Regla de oro: los bugs peligrosos son los que **corren sin error**. Verificar = registrar el número, no solo ejecutar.

---

## Semanas 1–3 · Fundamentos, JOINs, subqueries

### 1. Condiciones imposibles con AND
```sql
WHERE cantidad < 10 AND cantidad > 50   -- imposible: devuelve 0 filas
WHERE cantidad < 10 AND precio > 50     -- correcto: son dos columnas distintas
```

### 2. AND se ejecuta antes que OR (paréntesis)
```sql
WHERE categoria = 'A' AND cantidad > 5 OR precio < 50
-- SQL lo lee así:
WHERE (categoria = 'A' AND cantidad > 5) OR (precio < 50)
-- Lo que quería pedir:
WHERE categoria = 'A' AND (cantidad > 5 OR precio < 50)
```
Igual que en AppSheet: sin paréntesis, AND siempre gana y el resultado cambia por completo.

### 3. COUNT(columna) vs COUNT(*)
- `COUNT(categoria)` → cuenta filas donde categoria **NO es NULL**.
- `COUNT(*)` → cuenta todas las filas sin excepción.

### 4. HAVING solo acepta dos cosas
Agregados (COUNT, SUM, AVG…) o columnas que están en el GROUP BY.
```sql
HAVING vendedor > 2        -- ❌ texto comparado con número
HAVING monto > 200         -- ❌ el monto individual ya no existe tras GROUP BY
HAVING COUNT(*) > 2        -- ✅
HAVING SUM(monto) > 200    -- ✅
HAVING total > 200         -- ✅ (alias en SQLite; ojo: Postgres no acepta alias en HAVING)
```

### 5. Alias explícitos con AS
```sql
COUNT(*) producto           -- "producto" queda como alias confuso
COUNT(*) AS veces_vendido   -- ✅ claro
```

### 6. Fechas en formato ISO
`'2026-06-05'` — año-mes-día, siempre.

### 7. Regla del GROUP BY (la trampa nº 1 de todas)
**Toda columna del SELECT debe estar en el GROUP BY o dentro de un agregado. Sin excepción.**
SQLite lo permite en silencio; **PostgreSQL lo rechaza**. Apareció disfrazada de mil formas: sin SUM, dimensión faltante, GROUP BY por alias en vez de por clave, GROUP BY decorativo sobre un grano ya colapsado.

### 8. GROUP BY con varias dimensiones
Si seleccionas categoría y ciudad, agrupas por **ambas**. GROUP BY crea un grupo por cada combinación única de sus columnas.

### 9. Cada lado del AND necesita su columna completa
```sql
WHERE fecha >= '2026-06-01' AND <= '2026-06-30'   -- ❌ sintaxis
WHERE fecha >= '2026-06-01' AND fecha <= '2026-06-30'  -- ✅
```
SQL **no hereda el sujeto** de la comparación: `categoria != 'A' AND categoria != 'B'` — se repite completo.

### 10. JOIN por las columnas de relación correctas
```sql
INNER JOIN pedidos ON clientes.cliente_id = pedidos.pedido_id  -- ❌ columnas sin relación
INNER JOIN pedidos ON clientes.cliente_id = pedidos.cliente_id -- ✅
```

### 11. Contar por IDs, no por nombres
`COUNT(DISTINCT cliente_id)`, no `COUNT(DISTINCT nombre)` — los nombres se repiten.

### 12. Subconsultas con IN: mismo tipo de significado
```sql
WHERE producto_id IN (SELECT pedido_id FROM pedidos)            -- ❌ compara peras con manzanas
WHERE producto_id IN (SELECT producto_id FROM detalle_pedidos)  -- ✅
```

### 13. Un CTE solo ve las columnas que el CTE anterior produjo
Dentro del CTE2 ya no existe `clientes.cliente_id`, solo `cliente_id` (la columna que salió del CTE1).

### 14. LIMIT 1 sin ORDER BY no significa "el top"
```sql
ORDER BY total_categoria DESC
LIMIT 1
```
Sin ORDER BY, LIMIT devuelve una fila arbitraria.

### 15. COUNT(*) cuenta líneas, no pedidos
En `detalle_pedidos`, un pedido tiene varias líneas: `COUNT(DISTINCT pedido_id)`.

---

## Semana 4 · Window functions

### 16. Ranking: paréntesis vacíos y desempates
- `ROW_NUMBER()`, `RANK()`, `DENSE_RANK()` — las tres llevan paréntesis vacíos.
- **DENSE_RANK no garantiza una sola fila por grupo** ante empates. Para "quedarme con exactamente uno", usar ROW_NUMBER (números únicos).

### 17. OVER() convierte un agregado en window function
- `SUM(x) OVER()` — aunque vaya vacío, es window function: **respeta las filas**, no colapsa.
- `SUM(x)` sin OVER — agregado clásico: colapsa.
- ⚠️ El OVER **se me suelta bajo presión** — pasó varias veces en drills cronometrados. Revisarlo siempre antes de ejecutar.

### 18. LAG/LEAD exigen ORDER BY dentro del OVER
Sin ORDER BY explícito, el orden es indefinido → bug silencioso: corre, pero compara meses al azar.

### 19. Añadir ORDER BY al OVER cambia el marco en silencio
`SUM(x) OVER(ORDER BY mes)` deja de ser "total" y se vuelve **acumulado** (marco por defecto: UNBOUNDED PRECEDING → CURRENT ROW). Escribir el marco explícito comunica intención y evita sorpresas.

### 20. LAST_VALUE con marco por defecto devuelve la fila actual
No el último de la partición. Requiere marco explícito: `ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING`.

### 21. GROUP BY por columna que no identifica el grano
Agrupar por `fecha` en vez de `pedido_id`: si dos pedidos caen el mismo día, se fusionan en silencio.

### 22. CTEs en cadena vs. ramas paralelas
Granos distintos que solo se encuentran en el WHERE final → **ramas paralelas**, no cadena secuencial.

---

## Semana 5 · CASE, drills y simulacro

### 23. El fantasma silencioso (6 apariciones en la semana)
La trampa nº 7 volvió disfrazada: `(cantidad * precio_unitario)` sin SUM tras un GROUP BY, dimensiones incompletas, HAVING fantasma. SQLite calla; Postgres avisará.

### 24. La window nace en un piso; su filtro vive en el piso de arriba
WHERE corre **antes** que SELECT → la columna de ranking aún no existe al filtrar. Filtrar sobre una window function = envolver en CTE y filtrar arriba.
```sql
WITH ranked AS (SELECT ..., ROW_NUMBER() OVER(...) AS rn FROM ...)
SELECT * FROM ranked WHERE rn = 1
```

### 25. ROW_NUMBER + WHERE rn=1 vs FIRST_VALUE (reflejo de entrevista)
- "Quédate solo con el ganador por grupo" → **ROW_NUMBER + filtro rn = 1** (colapsa a una fila).
- "Pega el valor del líder en todas las filas del grupo" → **FIRST_VALUE** (no colapsa nada).

### 26. Un INNER JOIN rompe la cadena de LEFT JOINs
Un solo INNER en cualquier punto de la cadena anula la garantía de conservar todas las filas de la tabla ancla.

### 27. Anti-joins: filtrar por la clave del JOIN
`WHERE pedidos.pedido_id IS NULL` — por la clave, no por una columna de valores.

### 28. Un resultado vacío también es un resultado
El drill de "productos nunca vendidos" devolvió 0 filas y pasó desapercibido. Verificar y decir: "revisé, es correcto: no hay ninguno".

### 29. Una predicción vale lo que el dato que la sostiene
Predecir filas "de memoria" no sirve; anclarla a un dato real (¿cuántas categorías HAY?). Verificar = **registrar el número**.

### Las 5 plantillas de memoria muscular
```sql
ROW_NUMBER() OVER(PARTITION BY grupo ORDER BY x DESC)
SUM(x) OVER(ORDER BY t ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW)   -- acumulado
AVG(x) OVER(ORDER BY t ROWS BETWEEN 2 PRECEDING AND CURRENT ROW)           -- media móvil 3
LAG(x) OVER(PARTITION BY grupo ORDER BY t)                                  -- comparar con anterior
SUM(CASE WHEN cond THEN x ELSE 0 END)                                       -- pivote / agregación condicional
```

---

## Lo bueno y lo aprendido (S4–S5)

**Victorias concretas:**
- Proyecto "2024 Annual Sales Report" en GitHub — el tercero y el más completo, cerrado sin quedarse en "casi".
- Drills 2 y 4 del Día 4 (S5): acumulado en 8 min y media móvil en **6 min**, bajo meta, al primer intento, con marco explícito.
- Simulacro completo con esquema desconocido (PlayZone): para la tercera sesión, resolvía solo con arquitectura correcta.
- Hábito de verificación consolidado: cacé el total anual (13,439.58) por dos caminos independientes.

**El dato duro de la S5 (la lección más importante):**
> Cuando el plan sale completo **antes** de escribir → 5–8 min al primer intento.
> Cuando el plan tiene huecos → el tiempo se **triplica**.

El ritual de 60 segundos ES el atajo, no el peaje:
1. Grano final + predicción de filas **anclada a un dato real**
2. Plantilla completa de la window (con todo el interior del OVER)
3. Releer el enunciado antes de declarar terminado

**Cuellos de botella identificados (no es el SQL, es el proceso):**
- Lectura del esquema desconocido antes de escribir
- Gestión del reloj: si un problema atasca, saltar al siguiente y volver
- Hacer el ritual de planificación solo, sin apoyarme en el diálogo

**De cara a la Semana 6 (PostgreSQL):**
- Postgres rechazará el fantasma silencioso (trampa 7/23) — es un aliado, no un obstáculo.
- Cambian: `strftime` → `EXTRACT` / `DATE_TRUNC` / `TO_CHAR`, y alias en HAVING.

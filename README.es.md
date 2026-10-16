# Hoja de referencia SQL · SQL Cheatsheet

[![Stars](https://img.shields.io/github/stars/Ezra-Zhao/sql-cheatsheet?style=social)](https://github.com/Ezra-Zhao/sql-cheatsheet/stargazers) [![License: MIT](https://img.shields.io/badge/License-MIT-green.svg)](LICENSE) ![Languages](https://img.shields.io/badge/languages-3-blue)

> 🌍 [中文](README.md) · [Português](README.pt.md)

Una **hoja de referencia SQL de una sola página**: 30 consultas y operaciones de uso diario en SQL estándar, que cubren consultas, agregación, JOIN, subconsultas/CTE, escritura con transacciones, e índices y estructura de tablas. Cada entrada responde solo tres preguntas: **qué hace, una sentencia lista para copiar y pegar, y etiquetas de uso**. **Las 30 sentencias se ejecutaron y verificaron de verdad en SQLite 3.45** (resultados incluidos), y las diferencias entre MySQL y PostgreSQL están anotadas en cada entrada. Sin funciones inventadas.

## Para quién es

Desarrolladores backend, analistas de datos, SRE y cualquiera que repase SQL antes de una entrevista. Si cada vez que escribes un JOIN tienes que buscar dónde va el ON, o qué distingue HAVING de WHERE, esta hoja te deja las 30 sentencias más usadas grabadas de una vez.

## Cómo usarla

1. Ve directo a **[CHEATSHEET.md](CHEATSHEET.md)**: la versión en inglés, entrada por entrada igual que esta página.
2. Crea primero las tablas de prueba de abajo (funcionan en SQLite, MySQL y PostgreSQL), ejecuta cada sentencia a mano y compara con el resultado esperado de los comentarios.
3. El orden es la ruta de aprendizaje: fundamentos de consulta → agregación y agrupación → JOIN → subconsultas/CTE → escritura y transacciones → índices y estructura.

## Pruébalo en 30 segundos (tablas de prueba)

Tres tablas pequeñas: departamentos `departments`, empleados `employees` y pedidos `orders`. Todos los ejemplos usan estos datos, así que los resultados son reproducibles.

```sql
CREATE TABLE departments (id INTEGER PRIMARY KEY, name TEXT NOT NULL);
CREATE TABLE employees (id INTEGER PRIMARY KEY, name TEXT NOT NULL, dept_id INTEGER, salary INTEGER, manager_id INTEGER, hired_on TEXT);
CREATE TABLE orders (id INTEGER PRIMARY KEY, emp_id INTEGER, amount REAL, ordered_on TEXT);

INSERT INTO departments VALUES (1, 'Engineering'), (2, 'Sales'), (3, 'HR');
INSERT INTO employees VALUES
  (1, 'Alice', 1, 120000, NULL, '2019-03-01'),
  (2, 'Bob',   1,  95000, 1,    '2020-07-15'),
  (3, 'Carol', 1,  88000, 1,    '2021-01-10'),
  (4, 'David', 2,  72000, NULL, '2018-05-20'),
  (5, 'Eve',   2,  68000, 4,    '2022-09-01'),
  (6, 'Frank', 3,  54000, NULL, '2021-11-30'),
  (7, 'Grace', NULL, 60000, NULL, '2023-02-14');
INSERT INTO orders VALUES
  (101, 2, 250.00, '2024-01-05'),
  (102, 2, 120.50, '2024-01-20'),
  (103, 4, 980.00, '2024-02-02'),
  (104, 5, 430.75, '2024-02-14'),
  (105, 4, 150.00, '2024-03-01'),
  (106, 1, 2000.00, '2024-03-11'),
  (107, 5, 89.99,  '2024-03-15');
```

Fíjate: Grace no tiene departamento (`dept_id` es NULL) y Carol, Frank y Grace no tienen pedidos; varios ejemplos existen justo para que eso se note.

## Qué incluye

| Categoría | Contenido |
|---|---|
| 🔍 Fundamentos de consulta | Columnas concretas / WHERE / ORDER BY+LIMIT / DISTINCT / LIKE / IN·BETWEEN (6) |
| 📊 Agregación y agrupación | COUNT·SUM·AVG / GROUP BY / HAVING / COUNT(DISTINCT) / agregación con CASE (5) |
| 🔗 Familia JOIN | INNER JOIN / LEFT JOIN / anti-join / auto-join / JOIN de tres tablas (5) |
| 🧩 Subconsultas y CTE | Subconsulta escalar / IN / EXISTS / WITH / función ventana RANK (5) |
| ✏️ Escritura y transacciones | INSERT / UPDATE / DELETE / transacciones / UPSERT (5) |
| 🗂️ Índices y estructura | CREATE TABLE / ALTER TABLE / CREATE INDEX / DROP (4) |

---

## 🔍 Fundamentos de consulta

### `SELECT` — elige las columnas que necesitas
```sql
SELECT name, salary FROM employees;
-- 7 filas: nombre y salario de todos. Evita SELECT * por costumbre: nombrar columnas es más claro y más rápido
```
🏷️ `#consulta` `#básicos`

### `WHERE` — filtra filas por condición
```sql
SELECT name, salary FROM employees WHERE salary > 80000 ORDER BY salary DESC;
-- Alice 120000 / Bob 95000 / Carol 88000
```
🏷️ `#consulta` `#filtro`

### `ORDER BY ... LIMIT` — ordena y toma las primeras N
```sql
SELECT name, salary FROM employees ORDER BY salary DESC LIMIT 3;
-- Los tres salarios más altos: Alice / Bob / Carol
-- LIMIT funciona en MySQL / PostgreSQL / SQLite; SQL Server usa TOP 3 y Oracle FETCH FIRST 3 ROWS ONLY
```
🏷️ `#consulta` `#orden` `#paginación`

### `DISTINCT` — elimina duplicados
```sql
SELECT DISTINCT dept_id FROM employees;
-- 1 / 2 / 3 / NULL (Grace no tiene departamento; NULL también cuenta como valor)
```
🏷️ `#consulta` `#sin-duplicados`

### `LIKE` — coincidencia por patrón
```sql
SELECT name FROM employees WHERE name LIKE 'A%';
-- Alice. % coincide con cualquier secuencia de caracteres y _ con exactamente uno
-- Dialecto: LIKE no distingue mayúsculas en ASCII en SQLite / MySQL, pero sí en PostgreSQL;
--           PostgreSQL tiene además ILIKE para ignorar mayúsculas (solo PG; MySQL / SQLite no lo tienen)
```
🏷️ `#consulta` `#filtro` `#texto`

### `IN` / `BETWEEN` — pertenencia a un conjunto y rangos
```sql
SELECT name, salary FROM employees
WHERE dept_id IN (1, 2) AND salary BETWEEN 70000 AND 100000
ORDER BY salary DESC;
-- Bob 95000 / Carol 88000 / David 72000 (BETWEEN incluye ambos extremos)
```
🏷️ `#consulta` `#filtro`

## 📊 Agregación y agrupación

### `COUNT` / `SUM` / `AVG` — agregados básicos
```sql
SELECT COUNT(*) AS headcount, SUM(salary) AS total_payroll, AVG(salary) AS avg_salary FROM employees;
-- 7 personas / suma de salarios 557000 / promedio 79571.43
```
🏷️ `#agregación`

### `GROUP BY` — estadísticas por grupo
```sql
SELECT dept_id, COUNT(*) AS n, AVG(salary) AS avg_salary FROM employees GROUP BY dept_id;
-- Depto. 1: 3 personas, promedio 101000; depto. 2: 2, 70000; depto. 3: 1, 54000; grupo NULL: 1, 60000
```
🏷️ `#agregación` `#agrupación`

### `HAVING` — filtra los grupos ya formados
```sql
SELECT dept_id, AVG(salary) AS avg_salary FROM employees GROUP BY dept_id HAVING AVG(salary) > 60000;
-- Quedan solo el depto. 1 (101000) y el 2 (70000)
-- WHERE filtra filas antes de agrupar; HAVING filtra grupos después. No los confundas
```
🏷️ `#agregación` `#agrupación` `#filtro`

### `COUNT(DISTINCT ...)` — cuenta valores únicos
```sql
SELECT COUNT(DISTINCT dept_id) AS dept_count FROM employees;
-- 3. COUNT ignora los NULL, así que el departamento vacío de Grace no cuenta
```
🏷️ `#agregación` `#sin-duplicados`

### `CASE WHEN` — agregación condicional
```sql
SELECT COUNT(*) AS total,
       SUM(CASE WHEN salary >= 90000 THEN 1 ELSE 0 END) AS high_earners
FROM employees;
-- Total 7 / salarios altos 2 (Alice y Bob). Varias métricas con reglas distintas en una sola consulta
```
🏷️ `#agregación` `#condicional`

## 🔗 Familia JOIN

### `INNER JOIN` — filas que coinciden en ambas tablas
```sql
SELECT e.name, d.name AS dept FROM employees e JOIN departments d ON e.dept_id = d.id;
-- 6 filas: Grace no tiene departamento y queda fuera
```
🏷️ `#JOIN`

### `LEFT JOIN` — conserva todas las filas de la izquierda
```sql
SELECT e.name, d.name AS dept FROM employees e LEFT JOIN departments d ON e.dept_id = d.id;
-- 7 filas: Grace aparece con dept en NULL. Nómina completa → LEFT JOIN; solo coincidencias → INNER JOIN
```
🏷️ `#JOIN`

### `LEFT JOIN ... IS NULL` — filas sin correspondencia
```sql
SELECT e.name FROM employees e LEFT JOIN orders o ON o.emp_id = e.id WHERE o.id IS NULL;
-- Carol / Frank / Grace: empleados que nunca hicieron un pedido. El clásico para cazar datos huérfanos
```
🏷️ `#JOIN` `#depuración`

### Auto-join — una tabla consigo misma
```sql
SELECT e.name AS employee, m.name AS manager
FROM employees e LEFT JOIN employees m ON e.manager_id = m.id;
-- Bob→Alice, Carol→Alice, Eve→David; quien no tiene jefe queda con manager en NULL
-- Organigramas, comentarios con respuestas y cualquier jerarquía en una sola tabla usan este patrón
```
🏷️ `#JOIN` `#jerarquía`

### JOIN de tres tablas + agregación — pedidos por departamento
```sql
SELECT d.name AS dept, COUNT(o.id) AS order_count, SUM(o.amount) AS revenue
FROM departments d
JOIN employees e ON e.dept_id = d.id
JOIN orders o ON o.emp_id = e.id
GROUP BY d.id, d.name;
-- Engineering: 3 pedidos, 2370.50; Sales: 4 pedidos, 1650.74; HR no tiene pedidos y no aparece (INNER JOIN)
```
🏷️ `#JOIN` `#agregación` `#práctica`

## 🧩 Subconsultas y CTE

### Subconsulta escalar — comparar con el promedio global
```sql
SELECT name, salary FROM employees WHERE salary > (SELECT AVG(salary) FROM employees);
-- Alice / Bob / Carol: cobran más que el promedio de la empresa (79571.43)
```
🏷️ `#subconsulta`

### Subconsulta con `IN` — filtrar con el resultado de otra tabla
```sql
SELECT name FROM employees WHERE id IN (SELECT emp_id FROM orders WHERE amount > 500);
-- Alice / David: empleados con algún pedido de más de 500
```
🏷️ `#subconsulta`

### `EXISTS` — comprobar existencia
```sql
SELECT d.name FROM departments d
WHERE EXISTS (SELECT 1 FROM employees e WHERE e.dept_id = d.id AND e.salary > 100000);
-- Engineering: departamentos con al menos un empleado que gana más de 100000.
-- EXISTS solo pregunta "¿existe alguna fila?"; muy usado para filtrar tablas grandes
```
🏷️ `#subconsulta`

### `WITH` (CTE) — nombra un resultado intermedio
```sql
WITH dept_avg AS (
  SELECT dept_id, AVG(salary) AS avg_salary FROM employees GROUP BY dept_id
)
SELECT e.name, e.salary FROM employees e
JOIN dept_avg a ON e.dept_id = a.dept_id
WHERE e.salary > a.avg_salary;
-- Alice (supera el promedio de su depto., 101000) / David (supera 70000): cobran más que el promedio de su departamento
-- Las CTE hacen que una consulta compleja se lea paso a paso; las soportan MySQL 8.0+ / PostgreSQL / SQLite 3.8.8+
```
🏷️ `#CTE` `#subconsulta`

### Función ventana `RANK()` — ranking dentro de cada grupo
```sql
SELECT name, dept_id, salary,
       RANK() OVER (PARTITION BY dept_id ORDER BY salary DESC) AS rnk
FROM employees;
-- En el depto. 1: Alice 1, Bob 2, Carol 3; en el 2: David 1, Eve 2
-- GROUP BY fusiona filas; las funciones ventana no: cada fila conserva su posición. Esa es la diferencia clave
-- Dialecto: MySQL 8.0+ / PostgreSQL / SQLite 3.25+. Las versiones antiguas de MySQL no pueden
```
🏷️ `#ventana` `#ranking`

## ✏️ Escritura y transacciones

### `INSERT INTO` — añade una fila
```sql
INSERT INTO employees (id, name, dept_id, salary, manager_id, hired_on)
VALUES (8, 'Henry', 2, 71000, 4, '2024-06-01');
-- Escribe siempre los nombres de columna; para varias filas, encadena más grupos (...) tras VALUES
```
🏷️ `#escritura`

### `UPDATE ... WHERE` — modifica filas
```sql
UPDATE employees SET salary = salary + 5000 WHERE name = 'Eve';
-- Eve: 68000 → 73000, exactamente 1 fila afectada
-- ⚠️ Sin WHERE actualizas toda la tabla. Antes, ejecuta un SELECT con la misma condición y cuenta las filas
```
🏷️ `#escritura` `#peligro`

### `DELETE ... WHERE` — elimina filas
```sql
DELETE FROM orders WHERE amount < 100;
-- Elimina 1 fila (el pedido de 89.99). Sin WHERE vacía la tabla entera
-- En datos de negocio se suele "borrar suave" con una marca deleted_at en lugar de DELETE real
```
🏷️ `#escritura` `#peligro`

### Transacciones — todo o nada
```sql
BEGIN;
UPDATE employees SET dept_id = 2 WHERE name = 'Frank';
COMMIT;
-- Frank pasa de HR a Sales. Si algo falla a mitad, ROLLBACK deshace todo como si nada hubiera pasado
-- Transferencias y descuentos de inventario deben ir en transacción. En MySQL, START TRANSACTION equivale a BEGIN
```
🏷️ `#transacción` `#escritura`

### `UPSERT` — actualiza si existe, inserta si no
```sql
INSERT INTO departments (id, name) VALUES (3, 'People')
ON CONFLICT(id) DO UPDATE SET name = excluded.name;
-- id=3 ya existe → el departamento pasa de HR a People; si no existiera, se insertaría
-- Esta es la sintaxis de SQLite / PostgreSQL; MySQL usa INSERT ... ON DUPLICATE KEY UPDATE name = VALUES(name)
```
🏷️ `#escritura` `#dialecto`

## 🗂️ Índices y estructura

### `CREATE TABLE` — define una tabla
```sql
CREATE TABLE projects (
  id INTEGER PRIMARY KEY,
  title TEXT NOT NULL,
  lead_id INTEGER REFERENCES employees(id)
);
-- Claves primarias, NOT NULL y claves foráneas se definen en el esquema, no en el código de la aplicación
-- Dialecto: INTEGER PRIMARY KEY es el id autoincremental de SQLite;
--           MySQL escribe INT AUTO_INCREMENT PRIMARY KEY;
--           PostgreSQL escribe INT GENERATED ALWAYS AS IDENTITY PRIMARY KEY (o el clásico SERIAL)
```
🏷️ `#DDL` `#esquema`

### `ALTER TABLE` — añade una columna
```sql
ALTER TABLE employees ADD COLUMN email TEXT;
-- La columna nueva empieza en NULL. Rellena los datos primero y añade NOT NULL después: pasos pequeños, menos bloqueos
```
🏷️ `#DDL`

### `CREATE INDEX` — acelera las búsquedas frecuentes
```sql
CREATE INDEX idx_employees_dept ON employees(dept_id);
-- A partir de ahí, WHERE dept_id = ? usa el índice en vez de recorrer toda la tabla
-- Verifícalo: EXPLAIN QUERY PLAN en SQLite; EXPLAIN en MySQL / PostgreSQL
-- Cada índice frena un poco las escrituras: crea índices solo en las columnas por las que filtras de verdad
```
🏷️ `#DDL` `#índice` `#rendimiento`

### `DROP` — elimina un índice o una tabla
```sql
DROP INDEX idx_employees_dept;
-- MySQL spelling: DROP INDEX idx_employees_dept ON employees;
DROP TABLE projects;
-- ⚠️ Irreversible. Revisa antes que ninguna clave foránea apunte a la tabla y haz copia de seguridad en producción
```
🏷️ `#DDL` `#peligro`

---

## Por qué estas 30

- **Cubren el 90% del día a día**: consultar, calcular, unir, modificar. El SQL de cada día es este, una y otra vez; la sintaxis exótica queda fuera.
- **Cada una trae su resultado esperado**: si lo que obtienes no coincide, algo entendiste mal, y lo corriges en el momento.
- **Los dialectos, sin disimulo**: LIMIT, ILIKE, UPSERT y las claves autoincrementales cambian entre motores; cada entrada aclara qué sintaxis pertenece a cada motor.

## Contribuciones bienvenidas

¿Una sentencia incorrecta, una nota de dialecto equivocada o una traducción mejorable? Los *pull requests* son bienvenidos. Mantén el formato (descripción en una línea / bloque sql / etiquetas) y **ejecuta la sentencia en una base de datos real antes de enviarla**: lo que no coincida con el resultado esperado no entra.

## 📚 Más hojas de referencia

Otras hojas del mismo autor, también en chino, español y portugués:

- [Hoja de referencia FFmpeg](https://github.com/Ezra-Zhao/ffmpeg-cheatsheet) — 28 comandos de audio y vídeo
- [Hoja de referencia Git](https://github.com/Ezra-Zhao/git-cheatsheet) — 30 comandos diarios y de rescate
- [Hoja de referencia Docker](https://github.com/Ezra-Zhao/docker-cheatsheet) — 28 comandos de contenedores
- [Hoja de referencia curl](https://github.com/Ezra-Zhao/curl-cheatsheet) — 29 comandos para depurar APIs
- [Hoja de referencia de expresiones regulares](https://github.com/Ezra-Zhao/regex-cheatsheet) — 30 patrones reales
- [Hoja de referencia jq](https://github.com/Ezra-Zhao/jq-cheatsheet) — 30 filtros para JSON
- [Hoja de referencia de herramientas de terminal](https://github.com/Ezra-Zhao/terminal-superpowers-cheatsheet) — 14 herramientas modernas de terminal
- [Alternativas gratuitas a Adobe](https://github.com/Ezra-Zhao/adobe-free-alternatives-cheatsheet) — 16 alternativas gratuitas
- [Hoja para cancelar suscripciones](https://github.com/Ezra-Zhao/desubscribe-cheatsheet) — 10 cambios que ahorran ≈ $1,260 al año
- [Hoja de referencia kubectl](https://github.com/Ezra-Zhao/kubectl-cheatsheet) — comandos frecuentes de Kubernetes
- [Hoja de referencia SSH](https://github.com/Ezra-Zhao/ssh-cheatsheet) — acceso, túneles y transferencia de archivos

## Licencia

MIT © 2026 Guangyi Zhao. Ver [LICENSE](LICENSE).

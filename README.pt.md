# Folha de referência SQL · SQL Cheatsheet

[![Stars](https://img.shields.io/github/stars/Ezra-Zhao/sql-cheatsheet?style=social)](https://github.com/Ezra-Zhao/sql-cheatsheet/stargazers) [![License: MIT](https://img.shields.io/badge/License-MIT-green.svg)](LICENSE) ![Languages](https://img.shields.io/badge/languages-3-blue)

> 🌍 [中文](README.md) · [Español](README.es.md)

Uma **folha de referência SQL de uma página só**: 30 consultas e operações do dia a dia em SQL padrão, cobrindo consultas, agregação, JOIN, subconsultas/CTE, escrita com transações, e índices e estrutura de tabelas. Cada entrada responde só três coisas: **o que ela faz, uma instrução pronta para copiar e colar, e etiquetas de uso**. **As 30 instruções foram executadas e verificadas de verdade no SQLite 3.45** (resultados conferidos um a um), e as diferenças entre MySQL e PostgreSQL estão anotadas em cada entrada. Nada de funções inventadas.

## Para quem é

Desenvolvedores backend, analistas de dados, SRE e quem está revisando SQL antes de uma entrevista. Se toda vez que escreve um JOIN você precisa procurar onde vai o ON, ou qual é a diferença entre HAVING e WHERE, esta folha fixa as 30 instruções mais usadas de uma vez.

## Como usar

1. Vá direto ao **[CHEATSHEET.md](CHEATSHEET.md)**: a versão em inglês, entrada por entrada igual a esta página.
2. Crie primeiro as tabelas de teste abaixo (funcionam em SQLite, MySQL e PostgreSQL), execute cada instrução à mão e compare com o resultado esperado nos comentários.
3. A ordem é o caminho de aprendizado: fundamentos de consulta → agregação e agrupamento → JOIN → subconsultas/CTE → escrita e transações → índices e estrutura.

## Rode em 30 segundos (tabelas de teste)

Três tabelas pequenas: departamentos `departments`, funcionários `employees` e pedidos `orders`. Todos os exemplos usam esses dados, então os resultados são reproduzíveis.

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

Repare: a Grace não tem departamento (`dept_id` é NULL) e Carol, Frank e Grace não têm pedidos; vários exemplos existem justamente para isso aparecer.

## O que está incluído

| Categoria | Conteúdo |
|---|---|
| 🔍 Fundamentos de consulta | Colunas específicas / WHERE / ORDER BY+LIMIT / DISTINCT / LIKE / IN·BETWEEN (6) |
| 📊 Agregação e agrupamento | COUNT·SUM·AVG / GROUP BY / HAVING / COUNT(DISTINCT) / agregação com CASE (5) |
| 🔗 Família JOIN | INNER JOIN / LEFT JOIN / anti-join / auto-join / JOIN de três tabelas (5) |
| 🧩 Subconsultas e CTE | Subconsulta escalar / IN / EXISTS / WITH / função de janela RANK (5) |
| ✏️ Escrita e transações | INSERT / UPDATE / DELETE / transações / UPSERT (5) |
| 🗂️ Índices e estrutura | CREATE TABLE / ALTER TABLE / CREATE INDEX / DROP (4) |

---

## 🔍 Fundamentos de consulta

### `SELECT` — escolha as colunas de que precisa
```sql
SELECT name, salary FROM employees;
-- 7 linhas: nome e salário de todos. Não use SELECT * por hábito: nomear as colunas é mais claro e mais rápido
```
🏷️ `#consulta` `#básicos`

### `WHERE` — filtra linhas por condição
```sql
SELECT name, salary FROM employees WHERE salary > 80000 ORDER BY salary DESC;
-- Alice 120000 / Bob 95000 / Carol 88000
```
🏷️ `#consulta` `#filtro`

### `ORDER BY ... LIMIT` — ordena e pega as N primeiras
```sql
SELECT name, salary FROM employees ORDER BY salary DESC LIMIT 3;
-- Os três maiores salários: Alice / Bob / Carol
-- LIMIT funciona em MySQL / PostgreSQL / SQLite; SQL Server usa TOP 3 e Oracle, FETCH FIRST 3 ROWS ONLY
```
🏷️ `#consulta` `#ordenação` `#paginação`

### `DISTINCT` — remove duplicados
```sql
SELECT DISTINCT dept_id FROM employees;
-- 1 / 2 / 3 / NULL (a Grace não tem departamento; NULL também conta como valor)
```
🏷️ `#consulta` `#sem-duplicados`

### `LIKE` — correspondência por padrão
```sql
SELECT name FROM employees WHERE name LIKE 'A%';
-- Alice. % casa com qualquer sequência de caracteres e _ com exatamente um
-- Dialeto: o LIKE não diferencia maiúsculas em ASCII no SQLite / MySQL, mas diferencia no PostgreSQL;
--           o PostgreSQL tem ainda o ILIKE, que ignora maiúsculas (só no PG; MySQL / SQLite não têm)
```
🏷️ `#consulta` `#filtro` `#texto`

### `IN` / `BETWEEN` — pertinência a conjunto e intervalos
```sql
SELECT name, salary FROM employees
WHERE dept_id IN (1, 2) AND salary BETWEEN 70000 AND 100000
ORDER BY salary DESC;
-- Bob 95000 / Carol 88000 / David 72000 (BETWEEN inclui os dois extremos)
```
🏷️ `#consulta` `#filtro`

## 📊 Agregação e agrupamento

### `COUNT` / `SUM` / `AVG` — agregados básicos
```sql
SELECT COUNT(*) AS headcount, SUM(salary) AS total_payroll, AVG(salary) AS avg_salary FROM employees;
-- 7 pessoas / soma dos salários 557000 / média 79571.43
```
🏷️ `#agregação`

### `GROUP BY` — estatísticas por grupo
```sql
SELECT dept_id, COUNT(*) AS n, AVG(salary) AS avg_salary FROM employees GROUP BY dept_id;
-- Depto. 1: 3 pessoas, média 101000; depto. 2: 2, 70000; depto. 3: 1, 54000; grupo NULL: 1, 60000
```
🏷️ `#agregação` `#agrupamento`

### `HAVING` — filtra os grupos já formados
```sql
SELECT dept_id, AVG(salary) AS avg_salary FROM employees GROUP BY dept_id HAVING AVG(salary) > 60000;
-- Sobram só o depto. 1 (101000) e o 2 (70000)
-- WHERE filtra linhas antes de agrupar; HAVING filtra grupos depois. Não confunda os dois
```
🏷️ `#agregação` `#agrupamento` `#filtro`

### `COUNT(DISTINCT ...)` — conta valores únicos
```sql
SELECT COUNT(DISTINCT dept_id) AS dept_count FROM employees;
-- 3. COUNT ignora NULLs, então o departamento vazio da Grace não entra na conta
```
🏷️ `#agregação` `#sem-duplicados`

### `CASE WHEN` — agregação condicional
```sql
SELECT COUNT(*) AS total,
       SUM(CASE WHEN salary >= 90000 THEN 1 ELSE 0 END) AS high_earners
FROM employees;
-- Total 7 / salários altos 2 (Alice e Bob). Várias métricas com regras diferentes numa consulta só
```
🏷️ `#agregação` `#condicional`

## 🔗 Família JOIN

### `INNER JOIN` — linhas que combinam nas duas tabelas
```sql
SELECT e.name, d.name AS dept FROM employees e JOIN departments d ON e.dept_id = d.id;
-- 6 linhas: a Grace não tem departamento e fica de fora
```
🏷️ `#JOIN`

### `LEFT JOIN` — mantém todas as linhas da tabela da esquerda
```sql
SELECT e.name, d.name AS dept FROM employees e LEFT JOIN departments d ON e.dept_id = d.id;
-- 7 linhas: a Grace aparece com dept NULL. Lista completa → LEFT JOIN; só combinações → INNER JOIN
```
🏷️ `#JOIN`

### `LEFT JOIN ... IS NULL` — linhas sem correspondência
```sql
SELECT e.name FROM employees e LEFT JOIN orders o ON o.emp_id = e.id WHERE o.id IS NULL;
-- Carol / Frank / Grace: funcionários que nunca fizeram um pedido. O clássico para caçar dados órfãos
```
🏷️ `#JOIN` `#depuração`

### Auto-join — a tabela ligada a ela mesma
```sql
SELECT e.name AS employee, m.name AS manager
FROM employees e LEFT JOIN employees m ON e.manager_id = m.id;
-- Bob→Alice, Carol→Alice, Eve→David; quem não tem chefe fica com manager NULL
-- Organogramas, comentários com respostas e qualquer hierarquia numa tabela só usam esse padrão
```
🏷️ `#JOIN` `#hierarquia`

### JOIN de três tabelas + agregação — pedidos por departamento
```sql
SELECT d.name AS dept, COUNT(o.id) AS order_count, SUM(o.amount) AS revenue
FROM departments d
JOIN employees e ON e.dept_id = d.id
JOIN orders o ON o.emp_id = e.id
GROUP BY d.id, d.name;
-- Engineering: 3 pedidos, 2370.50; Sales: 4 pedidos, 1650.74; HR não tem pedidos e não aparece (INNER JOIN)
```
🏷️ `#JOIN` `#agregação` `#prática`

## 🧩 Subconsultas e CTE

### Subconsulta escalar — comparar com a média geral
```sql
SELECT name, salary FROM employees WHERE salary > (SELECT AVG(salary) FROM employees);
-- Alice / Bob / Carol: ganham acima da média da empresa (79571.43)
```
🏷️ `#subconsulta`

### Subconsulta com `IN` — filtrar pelo resultado de outra tabela
```sql
SELECT name FROM employees WHERE id IN (SELECT emp_id FROM orders WHERE amount > 500);
-- Alice / David: funcionários com algum pedido acima de 500
```
🏷️ `#subconsulta`

### `EXISTS` — teste de existência
```sql
SELECT d.name FROM departments d
WHERE EXISTS (SELECT 1 FROM employees e WHERE e.dept_id = d.id AND e.salary > 100000);
-- Engineering: departamentos com pelo menos um funcionário ganhando mais de 100000.
-- EXISTS só pergunta "existe alguma linha?"; muito usado para filtrar tabelas grandes
```
🏷️ `#subconsulta`

### `WITH` (CTE) — dá nome a um resultado intermediário
```sql
WITH dept_avg AS (
  SELECT dept_id, AVG(salary) AS avg_salary FROM employees GROUP BY dept_id
)
SELECT e.name, e.salary FROM employees e
JOIN dept_avg a ON e.dept_id = a.dept_id
WHERE e.salary > a.avg_salary;
-- Alice (acima da média do depto., 101000) / David (acima de 70000): ganham mais que a média do próprio departamento
-- CTEs fazem uma consulta complexa se ler passo a passo; MySQL 8.0+ / PostgreSQL / SQLite 3.8.8+ dão suporte
```
🏷️ `#CTE` `#subconsulta`

### Função de janela `RANK()` — ranking dentro de cada grupo
```sql
SELECT name, dept_id, salary,
       RANK() OVER (PARTITION BY dept_id ORDER BY salary DESC) AS rnk
FROM employees;
-- No depto. 1: Alice 1, Bob 2, Carol 3; no 2: David 1, Eve 2
-- GROUP BY funde as linhas; funções de janela não: cada linha fica e ganha sua posição. Essa é a diferença essencial
-- Dialeto: MySQL 8.0+ / PostgreSQL / SQLite 3.25+. Versões antigas do MySQL não conseguem
```
🏷️ `#janela` `#ranking`

## ✏️ Escrita e transações

### `INSERT INTO` — adiciona uma linha
```sql
INSERT INTO employees (id, name, dept_id, salary, manager_id, hired_on)
VALUES (8, 'Henry', 2, 71000, 4, '2024-06-01');
-- Escreva sempre os nomes das colunas; para várias linhas, encadeie mais grupos (...) depois de VALUES
```
🏷️ `#escrita`

### `UPDATE ... WHERE` — altera linhas
```sql
UPDATE employees SET salary = salary + 5000 WHERE name = 'Eve';
-- Eve: 68000 → 73000, exatamente 1 linha afetada
-- ⚠️ Sem o WHERE você atualiza a tabela inteira. Antes, rode um SELECT com a mesma condição e conte as linhas
```
🏷️ `#escrita` `#perigo`

### `DELETE ... WHERE` — remove linhas
```sql
DELETE FROM orders WHERE amount < 100;
-- Remove 1 linha (o pedido de 89.99). Sem WHERE, esvazia a tabela inteira
-- Em dados de negócio, o comum é "exclusão suave" com uma marca deleted_at, em vez de DELETE de verdade
```
🏷️ `#escrita` `#perigo`

### Transações — tudo ou nada
```sql
BEGIN;
UPDATE employees SET dept_id = 2 WHERE name = 'Frank';
COMMIT;
-- Frank sai de HR para Sales. Se algo falhar no meio, ROLLBACK desfaz tudo como se nada tivesse acontecido
-- Transferências e baixa de estoque precisam rodar em transação. No MySQL, START TRANSACTION equivale a BEGIN
```
🏷️ `#transação` `#escrita`

### `UPSERT` — atualiza se existir, insere se não
```sql
INSERT INTO departments (id, name) VALUES (3, 'People')
ON CONFLICT(id) DO UPDATE SET name = excluded.name;
-- id=3 já existe → o departamento muda de HR para People; se não existisse, seria inserido
-- Esta é a sintaxe do SQLite / PostgreSQL; o MySQL usa INSERT ... ON DUPLICATE KEY UPDATE name = VALUES(name)
```
🏷️ `#escrita` `#dialeto`

## 🗂️ Índices e estrutura

### `CREATE TABLE` — define uma tabela
```sql
CREATE TABLE projects (
  id INTEGER PRIMARY KEY,
  title TEXT NOT NULL,
  lead_id INTEGER REFERENCES employees(id)
);
-- Chave primária, NOT NULL e chave estrangeira se definem no esquema, não no código da aplicação
-- Dialeto: INTEGER PRIMARY KEY é o id autoincremental do SQLite;
--           o MySQL escreve INT AUTO_INCREMENT PRIMARY KEY;
--           o PostgreSQL escreve INT GENERATED ALWAYS AS IDENTITY PRIMARY KEY (ou o clássico SERIAL)
```
🏷️ `#DDL` `#esquema`

### `ALTER TABLE` — adiciona uma coluna
```sql
ALTER TABLE employees ADD COLUMN email TEXT;
-- A coluna nova começa NULL. Preencha os dados primeiro e adicione NOT NULL depois: passos pequenos, menos travamento
```
🏷️ `#DDL`

### `CREATE INDEX` — acelera as buscas frequentes
```sql
CREATE INDEX idx_employees_dept ON employees(dept_id);
-- A partir daí, WHERE dept_id = ? usa o índice em vez de varrer a tabela inteira
-- Verifique: EXPLAIN QUERY PLAN no SQLite; EXPLAIN no MySQL / PostgreSQL
-- Cada índice deixa a escrita um pouco mais lenta: crie índices só nas colunas que você realmente usa no filtro
```
🏷️ `#DDL` `#índice` `#desempenho`

### `DROP` — remove um índice ou uma tabela
```sql
DROP INDEX idx_employees_dept;
-- MySQL spelling: DROP INDEX idx_employees_dept ON employees;
DROP TABLE projects;
-- ⚠️ Irreversível. Confira antes se alguma chave estrangeira aponta para a tabela e faça backup em produção
```
🏷️ `#DDL` `#perigo`

---

## Por que estas 30

- **Cobrem 90% do dia a dia**: consultar, calcular, juntar, alterar. O SQL de todo dia é esse, de novo e de novo; a sintaxe exótica fica de fora.
- **Cada uma traz o resultado esperado**: se o seu resultado não bater, você entendeu algo errado — e corrige na hora.
- **Dialetos sem disfarce**: LIMIT, ILIKE, UPSERT e chaves autoincrementais mudam de motor para motor; cada entrada deixa claro qual sintaxe é de quem.

## Contribuições são bem-vindas

Achou uma instrução errada, uma nota de dialeto equivocada ou uma tradução que dá para melhorar? *Pull requests* são bem-vindos. Mantenha o formato (descrição em uma linha / bloco sql / etiquetas) e **execute a instrução num banco de dados real antes de enviar**: o que não bater com o resultado esperado não entra.

## 📚 Mais folhas de referência

Outras folhas do mesmo autor, também em chinês, espanhol e português:

- [Folha de referência FFmpeg](https://github.com/Ezra-Zhao/ffmpeg-cheatsheet) — 28 comandos de áudio e vídeo
- [Folha de referência Git](https://github.com/Ezra-Zhao/git-cheatsheet) — 30 comandos do dia a dia e de resgate
- [Folha de referência Docker](https://github.com/Ezra-Zhao/docker-cheatsheet) — 28 comandos de contêineres
- [Folha de referência curl](https://github.com/Ezra-Zhao/curl-cheatsheet) — 29 comandos para depurar APIs
- [Folha de referência de expressões regulares](https://github.com/Ezra-Zhao/regex-cheatsheet) — 30 padrões do mundo real
- [Folha de referência jq](https://github.com/Ezra-Zhao/jq-cheatsheet) — 30 filtros para JSON
- [Folha de referência de ferramentas de terminal](https://github.com/Ezra-Zhao/terminal-superpowers-cheatsheet) — 14 ferramentas modernas de terminal
- [Alternativas gratuitas ao Adobe](https://github.com/Ezra-Zhao/adobe-free-alternatives-cheatsheet) — 16 alternativas gratuitas
- [Folha para cancelar assinaturas](https://github.com/Ezra-Zhao/desubscribe-cheatsheet) — 10 trocas que economizam ≈ $1,260 por ano
- [Folha de referência kubectl](https://github.com/Ezra-Zhao/kubectl-cheatsheet) — comandos frequentes de Kubernetes
- [Folha de referência SSH](https://github.com/Ezra-Zhao/ssh-cheatsheet) — acesso, túneis e transferência de arquivos

## Licença

MIT © 2026 Guangyi Zhao. Ver [LICENSE](LICENSE).

# Review: Clientes Premium Pipeline

*Merlin Language — Example Walkthrough*

---

## What this example does

In one pipeline expression, Merlin:

- Cross-references 3 vectors (CLIENTES, PEDIDOS, PRODUTOS)
- Calculates total spent, number of orders, and average ticket per client
- Finds each client's favourite product using a nested pipeline
- Filters only premium clients (above average spending)
- Sorts by highest spender
- Adds the favourite product name

**Zero SQL. Zero backend logic. Zero explicit loops.**

---

## The data

Three vectors in RAM — clients, products, and orders:

```merlin
#CLIENTES[] = #[ { "ID":1, "NOME":"Ana Silva", ... } , ... ]
#PRODUTOS[] = #[ { "ID":1, "NOME":"Francesinha", "PRECO":14.50, ... } , ... ]
#PEDIDOS[]  = #[ { "ID_CLIENTE":1, "ID_PRODUTO":1, "VALOR":14.50, ... } , ... ]
```

---

## Step 1 — Total spent per client

```merlin
ADD_FIELD(@TOTAL_GASTO = @PEDIDOS.VALOR.FILTER(@ID_CLIENTE @ID ==).SUM())
```

For each client, Merlin automatically expands `@ID` to `@CLIENTES[ix].ID` and `@ID_CLIENTE` to `@PEDIDOS[x].ID_CLIENTE`. The filter isolates that client's orders. `SUM()` is automatically promoted to `VET_SUM` because the context is a numeric vector.

```
Ana Silva:     [14.50, 4.00, 18.00, 5.00] → SUM = 41.50
Bruno Costa:   [16.00, 6.50]              → SUM = 22.50
Carla Mendes:  [14.50, 1.50, 1.50]        → SUM = 17.50
David Nunes:   [18.00, 4.00, 2.50]        → SUM = 24.50
Eva Rodrigues: [16.00, 18.00, 5.00]       → SUM = 39.00
```

---

## Step 2 — Number of orders

```merlin
ADD_FIELD(@NUM_PEDIDOS = PEDIDOS.FILTER(@ID_CLIENTE @ID ==).RETURN(#LEN(ITEMS)))
```

Filters orders for each client and returns the count of items. `#LEN(ITEMS)` gives the size of the filtered vector.

```
Ana Silva: 4  Bruno: 2  Carla: 3  David: 3  Eva: 3
```

---

## Step 3 — Average ticket

```merlin
ADD_FIELD(@TICKET_MEDIO = @TOTAL_GASTO @NUM_PEDIDOS /)
```

Uses fields created by the previous two `ADD_FIELD` operations. `PIPELINE::CLIENTES` scope makes them available immediately.

```
Ana Silva: 41.50 / 4 = 10.375
Eva Rodrigues: 39.00 / 3 = 13.000
```

---

## Step 4 — Favourite product (nested pipeline)

```merlin
ADD_FIELD(
    @PRODUTO_FAVORITO_ID = PEDIDOS |
        FILTER(@ID_CLIENTE @ID ==) |
        ADD_FIELD(@QTD = @PEDIDOS.ID_PRODUTO.FILTER(@ID_PRODUTO @ID_PRODUTO ==).RETURN(#LEN(ITEMS))) |
        SORT_DESC |
        RETURN(@PEDIDOS[0].ID_PRODUTO)
)
```

This is a **pipeline inside a pipeline inside an ADD_FIELD**. For each client:

1. Filter their orders from PEDIDOS
2. For each order, count how many times that product appears (`ADD_FIELD(@QTD)`)
3. Sort descending — most ordered product first
4. Return the ID of the top product

```
Ana Silva     → product ID 1 (Francesinha — ordered 3 times)
Eva Rodrigues → product ID 7 (Arroz de Pato)
```

---

## Step 5 — Filter premium clients

```merlin
FILTER(@TOTAL_GASTO @TOTAL_GASTO.AVG() >)
```

`@TOTAL_GASTO` expands to the current client's value.
`@TOTAL_GASTO.AVG()` — the dot before `AVG()` signals a pipeline operation, so Merlin uses the full vector `[41.5, 22.5, 17.5, 24.5, 39.0]` to calculate the average: **29.0**.

```
Ana Silva:     41.5 > 29.0 → ✅ premium
Bruno Costa:   22.5 > 29.0 → ❌
Carla Mendes:  17.5 > 29.0 → ❌
David Nunes:   24.5 > 29.0 → ❌
Eva Rodrigues: 39.0 > 29.0 → ✅ premium
```

---

## Step 6 — Sort by highest spender

```merlin
SORT_BY_DESC(@TOTAL_GASTO)
```

Ana Silva (41.5) comes first.

---

## Step 7 — Favourite product name

```merlin
ADD_FIELD(
    $NOME_PRODUTO_FAV = PRODUTOS |
        FILTER(@PRODUTOS[IDX].ID @PRODUTO_FAVORITO_ID ==) |
        RETURN(PRODUTOS[0].NOME)
)
```

Uses `@PRODUTOS[IDX].ID` to match the product by its `PRODUTO_FAVORITO_ID` field — the ID calculated in Step 4. Returns the product name.

```
Ana Silva     → "Francesinha"
Eva Rodrigues → "Arroz de Pato"
```

---

## Final result

```
[0] Ana Silva
    TOTAL_GASTO      : 41.5
    NUM_PEDIDOS      : 4
    TICKET_MEDIO     : 10.375
    PRODUTO_FAVORITO : 1
    NOME_PRODUTO_FAV : "Francesinha"

[1] Eva Rodrigues
    TOTAL_GASTO      : 39.0
    NUM_PEDIDOS      : 3
    TICKET_MEDIO     : 13.0
    PRODUTO_FAVORITO : 7
    NOME_PRODUTO_FAV : "Arroz de Pato"
```

---

## Why this matters

The equivalent in SQL would require:

```sql
SELECT c.nome,
       SUM(p.valor) as total_gasto,
       COUNT(p.id) as num_pedidos,
       SUM(p.valor) / COUNT(p.id) as ticket_medio,
       (SELECT pr.nome FROM produtos pr
        JOIN pedidos p2 ON p2.id_produto = pr.id
        WHERE p2.id_cliente = c.id
        GROUP BY p2.id_produto
        ORDER BY COUNT(*) DESC LIMIT 1) as produto_favorito
FROM clientes c
JOIN pedidos p ON p.id_cliente = c.id
WHERE SUM(p.valor) > (SELECT AVG(total) FROM (...))
GROUP BY c.id
ORDER BY total_gasto DESC
```

In Merlin — one pipeline expression. No database. No queries. Everything in RAM.

---

## Key concepts demonstrated

**Automatic field expansion** — `@ID` becomes `@CLIENTES[ix].ID` in context.

**Automatic operator promotion** — `SUM()` becomes `VET_SUM` when the context is a numeric vector.

**PIPELINE scope** — fields created by `ADD_FIELD` are immediately available to subsequent operations in the same pipeline.

**Nested pipelines** — a full pipeline expression inside an `ADD_FIELD`.

**Cross-reference** — three vectors interacting without explicit JOIN syntax.

**Context-aware AVG** — `@TOTAL_GASTO.AVG()` uses the full vector, not the current item.

---

## Side by side — Merlin vs SQL

### SQL

```sql
SELECT
    c.nome,
    SUM(p.valor)                    AS total_gasto,
    COUNT(p.id)                     AS num_pedidos,
    SUM(p.valor) / COUNT(p.id)      AS ticket_medio,
    (
        SELECT pr.nome
        FROM produtos pr
        WHERE pr.id = (
            SELECT p2.id_produto
            FROM pedidos p2
            WHERE p2.id_cliente = c.id
            GROUP BY p2.id_produto
            ORDER BY COUNT(*) DESC
            LIMIT 1
        )
    )                               AS nome_produto_fav
FROM clientes c
JOIN pedidos p ON p.id_cliente = c.id
WHERE (
    SELECT SUM(p3.valor)
    FROM pedidos p3
    WHERE p3.id_cliente = c.id
) > (
    SELECT AVG(total) FROM (
        SELECT SUM(p4.valor) AS total
        FROM pedidos p4
        GROUP BY p4.id_cliente
    ) AS sub
)
GROUP BY c.id, c.nome
ORDER BY total_gasto DESC
```

### Merlin

```merlin
#CLIENTES_PREMIUM[] = CLIENTES |
    ADD_FIELD(@TOTAL_GASTO = @PEDIDOS.VALOR.FILTER(@ID_CLIENTE @ID ==).SUM()) |
    ADD_FIELD(@NUM_PEDIDOS = PEDIDOS.FILTER(@ID_CLIENTE @ID ==).RETURN(#LEN(ITEMS))) |
    ADD_FIELD(@TICKET_MEDIO = @TOTAL_GASTO @NUM_PEDIDOS /) |
    ADD_FIELD(
        @PRODUTO_FAVORITO_ID = PEDIDOS |
            FILTER(@ID_CLIENTE @ID ==) |
            ADD_FIELD(@QTD = @PEDIDOS.ID_PRODUTO.FILTER(@ID_PRODUTO @ID_PRODUTO ==).RETURN(#LEN(ITEMS))) |
            SORT_DESC |
            RETURN(@PEDIDOS[0].ID_PRODUTO)
    ) |
    FILTER(@TOTAL_GASTO @TOTAL_GASTO.AVG() >) |
    SORT_BY_DESC(@TOTAL_GASTO) |
    ADD_FIELD(
        $NOME_PRODUTO_FAV = PRODUTOS |
            FILTER(@PRODUTOS[IDX].ID @PRODUTO_FAVORITO_ID ==) |
            RETURN(PRODUTOS[0].NOME)
    )
&END_PIPELINE
```

Same result. No database. No queries. Everything in RAM.

Merlin Language - RPN stack language

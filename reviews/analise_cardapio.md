# Code Analysis: Full Menu Analysis — Zero Backend

> **Source:** [examples/cardapio_analise.html](../examples/cardapio_analise.html)

This example performs a complete statistical analysis of a restaurant menu
directly inside an HTML template — with **zero backend logic**.

---

## What the backend does

```rust
// The entire Rust route:
let mut calculator = setup_merlin_from_globals(state).await;
processar_base_template("examples/cardapio_analise.html", &mut calculator, ..).await
```

That's it. The backend starts a Merlin instance and asks it to render the template.
All data — dishes, categories, photos — lives permanently in Merlin's RAM,
updated every morning at 11am, one hour before the restaurant opens its doors.

No queries. No serialization. No business logic in Rust.

---

## What the template returns

A single expression produces six values of different types:

```merlin
($PRATO_MAIS_CARO    ,   // string  — name of most expensive dish
 $PRATO_MAIS_BARATO  ,   // string  — name of cheapest dish
 @PRECO_MEDIO        ,   // number  — average price
 @TOTAL_CARDAPIO     ,   // number  — sum of all prices
 #PRATOS_ACIMA_MEDIA[],  // vector of dictionaries — dishes above average
 #RESUMO_CATEGORIAS[])   // vector of dictionaries — totals per category

= PRATOS.EXECUTA_EACH(...) | RETURN(...)
```

Merlin reads each value from the correct stack — strings from `pilha_strings`,
numbers from `pilha_numeros`, dictionary vectors from `pilha_vetores_dicionarios`.
The programmer declares the intent; Merlin handles the routing.

---

## How pipelines work

The `|` operator chains transformations. Each operator receives what the
previous one left on the stack:

```merlin
PRATOS | FILTER(@PRECO MEDIA >) | ADD_FIELD(@DIFERENCA = @PRECO MEDIA -) | SORT_BY_DESC("@PRECO")
//  ↓           ↓                      ↓                                        ↓
// source    keeps only            adds computed               sorts by PRECO
//           PRECO > MEDIA         field DIFERENCA             descending
```

Pipelines can be nested — a pipeline inside `EXECUTA_EACH` inside another pipeline.
Merlin resolves each level independently, with isolated scopes for loop variables.

---

## Part 1 — Statistics: most expensive, cheapest, average

```merlin
&BEGIN_EACH(
    #PRATO = ITEM        // ITEM is the current dictionary — available automatically

    @TOTAL    += @PRATO.PRECO
    @CONTAGEM += 1

    #IF @PRATO.PRECO MAIS_CARO >
        @MAIS_CARO      = @PRATO.PRECO
        $NOME_MAIS_CARO = PRATO.TITULO
    #END_IF

    #IF @PRATO.PRECO MAIS_BARATO <
        @MAIS_BARATO      = @PRATO.PRECO
        $NOME_MAIS_BARATO = PRATO.TITULO
    #END_IF
)

@MEDIA = TOTAL CONTAGEM /
```

`EXECUTA_EACH` has three invisible phases:

```
Before &BEGIN_EACH  →  INITIALIZE  (runs once — set up accumulators)
Inside &BEGIN_EACH  →  ITERATE     (runs for each item — ITEM and IDX available)
After  &BEGIN_EACH  →  FINALIZE    (runs once — compute final results)
```

Inside `&BEGIN_EACH`, `ITEM` holds the current dictionary automatically.
No need for `#PRATO = &ULTIMA_PILHA` — `ITEM` is always there.

Variables like `@TOTAL` and `@MAIS_CARO` are in the outer scope — they
accumulate across iterations naturally, without any explicit transfer.

---

## Part 2 — Dishes above average with computed field

```merlin
#ACIMA_MEDIA[] = PRATOS |
    FILTER(@PRECO MEDIA >) |
    ADD_FIELD(@DIFERENCA = @PRECO MEDIA -) |
    SORT_BY_DESC("@PRECO")
```

Reading this pipeline left to right:

**`FILTER(@PRECO MEDIA >)`**
Inside filter expressions, `@FIELD` refers to the current item's field.
`@PRECO` expands to `@PRATOS[current_index].PRECO` automatically.
`MEDIA` is a regular variable calculated in the previous step.
The filter keeps only items where `PRECO > MEDIA`.

**`ADD_FIELD(@DIFERENCA = @PRECO MEDIA -)`**

Breaking down the syntax:
```
@          →  the new field is a number
DIFERENCA  →  name of the new field
=          →  assignment
@PRECO     →  current item's PRECO field (number)
MEDIA      →  variable from outer scope
-          →  subtract (RPN: operands first, operator last)
```

Each item in the filtered vector now has a `DIFERENCA` field computed on the fly.
No intermediate variables. No loops. One line.

**`SORT_BY_DESC("@PRECO")`**
Sorts the filtered+enriched vector by `PRECO`, descending.
The `"@"` prefix tells Merlin to sort numerically, not alphabetically.

The result — `#ACIMA_MEDIA[]` — is a new vector. `PRATOS` is untouched.

---

## Part 3 — Category totals with triple nested pipelines

```merlin
#RESUMO[] = CATEGORIAS_PRODUTO |
    EXECUTA_EACH(
        #RESUMO_CAT[] = #[]

        &BEGIN_EACH(
            #CAT = ITEM

            PRATOS |
                FILTER(@CATEGORIA_ID @CAT.ID ==) |   // ← pipeline inside pipeline
                EXECUTA_EACH(
                    @TOTAL_CAT = 0
                    @QTD_CAT   = 0
                    &BEGIN_EACH(
                        @TOTAL_CAT += @ITEM.PRECO     // ← third level
                        @QTD_CAT   += 1
                    )
                )
            &END_PIPELINE

            #IF QTD_CAT
                #RESUMO_CAT[+] = {
                    "CATEGORIA" : CAT.TITULO ,
                    "TOTAL"     : TOTAL_CAT ,
                    "QUANTIDADE": QTD_CAT ,
                    "MEDIA_CAT" : TOTAL_CAT QTD_CAT /
                }
            #END_IF
        )

    ) |
    RETURN(#RESUMO_CAT)
```

Three levels of iteration — all inside the template, all in RAM:

```
Level 1: CATEGORIAS_PRODUTO | EXECUTA_EACH(...)
         iterates each category

Level 2: PRATOS | FILTER(...) | EXECUTA_EACH(...)
         for each category, filters matching dishes

Level 3: &BEGIN_EACH(...)
         for each matching dish, accumulates TOTAL_CAT and QTD_CAT
```

`@CAT.ID` in the inner `FILTER` refers to the outer loop's current category —
Merlin resolves it from the outer scope automatically. No parameter passing needed.

`&END_PIPELINE` is a documentation marker — it has no effect on execution.
It makes the boundaries of each pipeline visible when reading complex code.

`RETURN(#RESUMO_CAT)` exits the pipeline and places `RESUMO_CAT` on the
dictionary vector stack — ready to be assigned to `#RESUMO[]`.

---

## Part 4 — Collecting all results

```merlin
        &NO_STACK()

    ) |
    RETURN(NOME_MAIS_CARO , NOME_MAIS_BARATO , ACIMA_MEDIA , MEDIA , TOTAL , RESUMO)

&END_PIPELINE
```

`&NO_STACK()` tells `EXECUTA_EACH` not to leave the dish vector on the stack —
we're returning specific values via `RETURN`, not the original vector.

`RETURN` accepts any Merlin expression, not just field names:

```merlin
// Field names
RETURN($TITULO , @PRECO)

// Variables (any type — Merlin detects automatically)
RETURN(NOME_MAIS_CARO , ACIMA_MEDIA , RESUMO)

// Computed expressions
RETURN(@PRECO 1.1 * , $TITULO.UPPER() , TOTAL CONTAGEM /)

// String literals (commas inside quotes are not separators)
RETURN("Beef, Pasta, Fish" , TOTAL)
```

Each value goes to the correct stack based on its type.
The left-hand side destructures them in order.

---

## Part 5 — The HTML section

The template section uses the computed values directly:

```html
{{%% #FOR PRATO in PRATOS_ACIMA_MEDIA %%}}
    <div class="col-md-4 mb-3">
        <div class="card h-100">
            {{{{ HTML.PICTURE_RENDER_THUMB(PRATO) }}}}   ← module call inside #FOR
            <div class="card-body">
                <h6>{{{{ PRATO.TITULO }}}}</h6>
                <p class="text-danger fw-bold">
                    {{{{ $FORMATA_MOEDA(@PRATO.PRECO) }}}}      ← formatted number
                    <small>(+ {{{{ $FORMATA_MOEDA(@PRATO.DIFERENCA) }}}})</small>  ← computed field
                </p>
            </div>
        </div>
    </div>
{{%% #END_FOR %%}}
```

Even here, inside the HTML, Merlin's full power is available:
- `#FOR` iterates any vector
- Module calls like `HTML.PICTURE_RENDER_THUMB(PRATO)` render components
- `$FORMATA_MOEDA()` formats numbers as currency
- `@PRATO.DIFERENCA` accesses the field computed by `ADD_FIELD` in the pipeline

The HTML section is not a dumb renderer — it's part of the same language.

---

## Summary

| What | How |
|------|-----|
| Backend code | 2 lines of Rust |
| Data source | RAM — loaded at 11am daily |
| Nesting depth | 3 levels of pipelines |
| Values returned | 6 (mixed types) |
| Backend queries | 0 |
| Template engine limitations hit | 0 |

The key insight: **data lives in Merlin's RAM, not in the database**.
The template is not asking the backend for data — it already has everything,
and can cross-reference any of it, compute anything from it, and render
exactly what it needs. In one pass. In one file.

---

*[Back to examples](../examples/) · [Getting Started](quick_start.md) · [merlinlang.com](https://merlinlang.com)*

# Getting Started with Merlin

Merlin is a template engine built in Rust with a twist — instead of a limited
expression language bolted onto HTML, Merlin gives you a real programming language
inside your templates. One based on RPN (Reverse Polish Notation) and stacks,
the same philosophy behind HP calculators and PostScript.

If you've ever found yourself moving logic to the backend just because your
template engine couldn't handle it — Merlin was built for you.

---

## The philosophy in 60 seconds

Most template engines work like this:

```
Backend does all the work → Template just renders
```

Merlin works like this:

```
Backend sends data → Template transforms, filters, computes, renders
```

The key insight is **stacks**. Everything in Merlin pushes to a stack or
consumes from one. The left side of an assignment doesn't know what the right
side will produce — it just waits for the right type to appear on the stack.
This makes composition natural and unlimited.

```merlin
@TOTAL = @PRODUTOS.PRECO.SUM()   // reads PRECO from each product, sums all
@MEDIA = @PRODUTOS.PRECO.AVG()   // same data, different operation
```

No loops. No explicit iteration. The context (a vector of dictionaries)
tells Merlin to promote `SUM` to `VET_SUM` automatically.

---

## The three data types

Merlin has exactly three types — and their vector counterparts:

| Sigil | Type       | Vector     |
|-------|------------|------------|
| `@`   | Number     | `@VAR[]`   |
| `$`   | String     | `$VAR[]`   |
| `#`   | Dictionary | `#VAR[]`   |

Sigils appear **only on the left side** of assignments. On the right side,
you just use the variable name — Merlin knows the type from context.

```merlin
@PRECO   = 29.90
$TITULO  = "Beef Steak à Rosemary"
#PRODUTO = { "TITULO" : "Beef Steak" , "PRECO" : 29.90 , "ESTOQUE" : "15" }

// Dictionary fields are always strings internally
// Use @ prefix to read a field as number:
@VALOR = @PRODUTO.PRECO    // reads "29.90" and converts to f64
```

Variables don't need to be declared. Merlin fills missing values with
sensible defaults — `0.0`, `""`, `{}` — so templates never crash on
missing data.

---

## The six stacks

Behind the scenes, Merlin maintains six independent stacks:

```
pilha_numeros             (number stack)
pilha_strings             (string stack)
pilha_dicionarios         (dictionary stack)
pilha_vetores_numeros     (number vector stack)
pilha_vetores_strings     (string vector stack)
pilha_vetores_dicionarios (dictionary vector stack)
```

The right side of any expression pushes to the correct stack.
The left side consumes from it. When a stack value is consumed, it's gone —
no garbage collector needed, because there's no garbage.

```merlin
// This line:
@TOTAL = PRECO 1.1 *

// What happens:
// 1. PRECO → pushes its value to pilha_numeros
// 2. 1.1   → pushes 1.1 to pilha_numeros
// 3. *     → pops both, pushes result
// 4. @TOTAL = → pops from pilha_numeros, stores in TOTAL
// Stack is empty. No cleanup needed.
```

---

## Templates — three delimiters

Merlin templates are HTML files with three types of Merlin blocks:

### `%%%% ... %%%%` — Pure Merlin code block (no output)

```html
%%%%
    @PRECO_FINAL = PRECO 1.1 *
    $TITULO_UPPER = TITULO | UPPER
    #ITENS_FILTRADOS[] = ITENS | FILTER("@ESTOQUE 0 >")
%%%%
```

Use this for computation — setting up data before rendering.

### `{{%% ... %%}}` — Inline Merlin (no output)

```html
{{%% #FOR PRODUTO in PRODUTOS %%}}
{{%% #IF PRODUTO.ESTOQUE %%}}
{{%% @TOTAL += @PRODUTO.PRECO %%}}
{{%% #END_FOR %%}}
```

Use this for flow control and inline assignments.

### `{{{{ ... }}}}` — Merlin expression (outputs to HTML)

```html
<h2>{{{{ PRODUTO.TITULO }}}}</h2>
<span>{{{{ $FORMATA_MOEDA(@PRODUTO.PRECO) }}}}</span>
<div>{{{{ HTML.MENU_CARD(PRODUTO) }}}}</div>
```

The last string on the stack becomes the HTML output.

---

## Modules — reusable functions

Modules are Merlin's functions. They read parameters from the stack
(last pushed = first read) and leave results on the stack.

```merlin
&CREATE_MODULE("HTML.MENU_CARD") ::

%%%%
    // HTML.MENU_CARD( model pedido )
    @PEDIDO = &ULTIMA_PILHA    // reads last item from stack
    #MODEL  = &ULTIMA_PILHA    // reads next item
%%%%

    <div class="col-6 col-sm-4 col-lg-3 mb-4">
        <div class="card h-100">
            <a href="/{{{{ cliente.slug }}}}/produto/{{{{ model.id }}}}">
                {{{{ HTML.PICTURE_RENDER_THUMB(MODEL) }}}}
                <h5 class="text-center">{{{{ MODEL.TITULO }}}}</h5>
            </a>
            {{%% #IF @MODEL.PRECO 0.001 > %%}}
                <h5 class="text-center">
                    <strong>{{{{ $FORMATA_MOEDA(@MODEL.PRECO) }}}}</strong>
                </h5>
            {{%% #END_IF %%}}
        </div>
    </div>

&END_MODULE()
```

Calling a module:

```merlin
{{{{ HTML.MENU_CARD(PRODUTO 1) }}}}   // PRODUTO and 1 pushed to stack
{{{{ HTML.MENU_CARD(PRODUTO 0) }}}}   // module reads them in reverse order
```

Modules live in library files (`.html`). They're loaded at startup and
shared across all templates — zero file I/O per request.

---

## The pipeline — where Merlin shines

The pipeline operator `|` chains operations on data. Each operator receives
the result of the previous one. Any of the 300+ built-in operators and any
module from any library works in the pipeline automatically.

```merlin
// Filter, compute, sort and update in one line:
PRODUTOS | FILTER("@ESTOQUE 0 >") | ADD_FIELD(@TOTAL = @PRECO @ESTOQUE *) | SORT_BY("@TOTAL") | ATUALIZA

// or multiline — same result:
PRODUTOS |
    FILTER("@ESTOQUE 0 >") |
    ADD_FIELD(@TOTAL = @PRECO @ESTOQUE *) |
    SORT_BY("@TOTAL") |
    ATUALIZA

// Save to a new variable instead of updating in-place:
#RESUMO[] = PRODUTOS |
    FILTER("@ESTOQUE 0 >") |
    ADD_FIELD(@TOTAL = @PRECO @ESTOQUE *) |
    SORT_BY("@TOTAL")

// Equivalent with dot notation (parentheses required):
#RESUMO[] = PRODUTOS.FILTER("@ESTOQUE 0 >").ADD_FIELD(@TOTAL = @PRECO @ESTOQUE *).SORT_BY("@TOTAL")
```

Inside pipeline expressions, `@FIELD` and `$FIELD` refer to the current item:

```merlin
PRODUTOS | FILTER("@PRECO 15 >")                  // @PRECO = current product's price
PRODUTOS | FILTER("$CATEGORIA \"carnes\" $==")    // $CATEGORIA = current product's category
```

Modules work in the pipeline too — the current item is passed automatically:

```merlin
// These three are equivalent:
{{%% #FOR PRODUTO in PRODUTOS %%}}
    {{{{ HTML.MENU_CARD(PRODUTO) }}}}
{{%% #END_FOR %%}}

{{{{ PRODUTOS | HTML.MENU_CARD }}}}

// HTML.* and RESTAURANTE.* are priority namespaces — prefix is optional:
{{{{ PRODUTOS | MENU_CARD }}}}
```

### Comparison operators

```merlin
>    <    >=    <=   // boolean — returns 0 or 1
>?   <?   >=?   <=?  // value — returns the winning value, not boolean

@ESTOQUE 0 >         // returns 1 if ESTOQUE > 0, else 0
@ESTOQUE 0 >?        // returns ESTOQUE if ESTOQUE > 0, else 0
@PRECO 10 >=?        // returns PRECO if PRECO >= 10, else 10

$TITULO $RODAPE $>?  // returns the alphabetically greater string
$TITULO $RODAPE $<?  // returns the alphabetically lesser string
```

Available pipeline operators:

| Operator | Description |
|----------|-------------|
| `FILTER(expr)` | Keep items where expr is true |
| `WHERE(expr)` | Set condition for next operator |
| `ADD_FIELD(@F = expr)` | Compute and add field to each item |
| `SORT_BY("@FIELD")` | Sort ascending by field or expression |
| `SORT_BY_DESC("@FIELD")` | Sort descending |
| `EXECUTA(programa)` | Run full Merlin program on the vector |
| `EXECUTA_EACH(programa)` | Run Merlin program on each item |
| `ATUALIZA` | Write result back to original variable |

---

## Control flow

### #FOR with WHERE

```merlin
{{%% #FOR IDX, PRODUTO in PRODUTOS WHERE PRODUTO.CATEGORIA_ID CATEGORIA.ID $== %%}}
    {{{{ HTML.MENU_CARD(PRODUTO 1) }}}}
{{%% #END_FOR %%}}
```

### #FOR BY N — iterate in chunks

```merlin
{{%% #FOR LINHA in FOTOS BY 3 %%}}
    <div class="row">
        {{%% #FOR FOTO in LINHA %%}}
            <div class="col-4">
                <img src="{{{{ FOTO.IMAGEM_THUMB }}}}" alt="{{{{ FOTO.TITULO }}}}">
            </div>
        {{%% #END_FOR %%}}
    </div>
{{%% #END_FOR %%}}
```

### #IF — block and inline

```merlin
// Block
{{%% #IF @PRODUTO.PRECO 0.001 > %%}}
    <span>{{{{ $FORMATA_MOEDA(@PRODUTO.PRECO) }}}}</span>
{{%% #ELSE %%}}
    <span>Preço sob consulta</span>
{{%% #END_IF %%}}

// Inline — for short expressions
$THUMB = #IF FOTO.IMAGEM_THUMB :: "/_cache_/media/" FOTO.IMAGEM_THUMB $+ #ELSE "/_cache_/media/" FOTO.IMAGEM $+
$NOME  = #IF PRODUTO.TITULO :: PRODUTO.TITULO #ELSE "Produto sem nome"
```

### #MATCH

```merlin
{{%% #MATCH TIPO_MENU %%}}
    "card"  :: {{{{ HTML.MENU_CARD(PRODUTO 1) }}}}
    "flex"  :: {{{{ HTML.MENU_FLEX(PRODUTO 1) }}}}
    default :: {{{{ HTML.MENU_BOX(PRODUTO 1) }}}}
{{%% #END_MATCH %%}}
```

---

## Scope control

Merlin has explicit, named scope control. Every scope is isolated —
variables created inside die when the scope closes.

### &LOCAL_SCOPE — everything is local automatically

```merlin
@TOTAL = 0

&LOCAL_SCOPE
    @SUBTOTAL = PRECO 1.1 *   // local — dies here
    @TEMP     = SUBTOTAL 2 *  // also local
    @OUTER::TOTAL += SUBTOTAL // OUTER:: writes to the layer behind
&END_LOCAL_SCOPE

// SUBTOTAL and TEMP don't exist here ✓
// TOTAL was updated via OUTER:: ✓
```

### &LOCAL_NAMED_SCOPE — scope with a name

The name lets child scopes write directly to this scope by name:

```merlin
&LOCAL_NAMED_SCOPE("CALCULOS")
    @BASE = 100

    &LOCAL_SCOPE
        @RESULTADO = BASE 1.1 *
        @CALCULOS::BASE = RESULTADO   // writes directly to CALCULOS scope by name
    &END_LOCAL_SCOPE

    // BASE is now updated ✓
&END_LOCAL_NAMED_SCOPE
```

### &LOCAL_EXPLICIT_SCOPE — only explicit writes are local

Name is required. Without a prefix, variables go to the outer scope:

```merlin
&LOCAL_EXPLICIT_SCOPE("LOOP")
    @LOOP::CONTADOR = 0    // explicitly local — uses scope name
    @TOTAL += CONTADOR     // no prefix → goes to outer scope naturally
    @OUTER::TOTAL += 1     // OUTER:: is also explicit — same result
&END_LOCAL_EXPLICIT_SCOPE
```

### Scope prefixes — complete system

```
NOME::    →  write to named scope (from any child)
OUTER::   →  write to the layer behind current scope
MERLIN::  →  write directly to Merlin local (skip all scopes)
GLOBAL::* →  immutable (loaded at 10am)
(no prefix, LOCAL_SCOPE)          →  local automatically
(no prefix, LOCAL_EXPLICIT_SCOPE) →  goes to outer scope naturally
```

### #FOR — automatic isolated scope

Every `#FOR` loop opens an automatic `LOCAL_EXPLICIT_SCOPE` named `FOR_SCOPE_N`
(where N = current scope depth). Loop variables are local, everything else
writes to the outer scope naturally:

```merlin
@TOTAL = 0

#FOR IDX, PRODUTO in PRODUTOS
    @TEMP = PRODUTO.PRECO 1.1 *   // local to this iteration
    @TOTAL += TEMP                 // goes to outer scope — correct
#END_FOR

// TEMP doesn't exist here ✓
// TOTAL accumulated correctly ✓
```

---

## Context hierarchy

Merlin organises variables and modules in three global layers:

```
CLIENTE  (restaurant-specific — highest priority)
    ↓
SITE     (platform-wide — all restaurants)
    ↓
CORE     (generic — HTML modules, utilities)
    ↓
LOCAL    (request-specific — per template)
```

A restaurant can override any module from CORE or SITE just by defining
the same module name in its CLIENTE layer. No configuration needed.

```merlin
// In CORE — default menu card
&CREATE_MODULE("HTML.MENU_BOX") :: ...

// In CLIENTE (Pecado Capital) — custom version
&CREATE_MODULE("HTML.MENU_BOX") ::
    GLOBAL::CORE::HTML.MENU_BOX()    // calls the CORE version first
    // then adds restaurant-specific logic
&END_MODULE()
```

---

## A real example — restaurant menu

This is the actual template powering a restaurant menu page.
The Rust route sends only `CAT_SELECT`, `PEDIDO`, and `TIPO_MENU`.
Everything else — products, categories, client data — is already in RAM.

```html
%%%%
    // Data is in RAM, loaded at 10am daily
    // Route only injects request parameters
%%%%

<<<< BEGIN :: RESTAURANTE.MENU_SELECT_TIPO(CLIENTE.TITULO TIPOS_MENU CAT_SELECT) >>>>

    {{%% #FOR CATEGORIA in CATEGORIAS_PRODUTO
             WHERE CAT_SELECT 0 == .OR. CAT_SELECT @CATEGORIA.ID == %%}}

        {{%% @TEM_CATEGORIA = 0 %%}}

        {{%% #FOR P, PRODUTO in PRODUTOS
                 WHERE PRODUTO.CATEGORIA_ID CATEGORIA.ID $== %%}}

            {{%% #IF .NOT. TEM_CATEGORIA %%}}
                {{%% @TEM_CATEGORIA = 1 %%}}
                {{{{ RESTAURANTE.MENU_SELECT_CATEGORIA(TIPO_MENU) }}}}
                <div class="row">
            {{%% #END_IF %%}}

            {{%% #MATCH TIPO_MENU %%}}
                "card"  :: {{{{ HTML.MENU_CARD(PRODUTO 1) }}}}
                "flex"  :: {{{{ HTML.MENU_FLEX(PRODUTO 1) }}}}
                default :: {{{{ HTML.MENU_BOX(PRODUTO 1) }}}}
            {{%% #END_MATCH %%}}

        {{%% #END_FOR %%}}

        {{%% #IF TEM_CATEGORIA %%}}
            </div>
        {{%% #END_IF %%}}

    {{%% #END_FOR %%}}

<<<< END :: RESTAURANTE.MENU_SELECT_TIPO >>>>
```

The `<<<< BEGIN :: ... >>>>` delimiter is for modules with phases
(opening container, middle items, closing container) — `RESTAURANTE.MENU_SELECT_TIPO`
handles the `<section>` wrapper, the type selector dropdown, and closing tags.

---

## What's next

- [Syntax Reference](docs/syntax.md) — complete language reference
- [Library: FINANCE](libs/finance/) — Tabela Price, TIR, VPL
- [Library: HTML](libs/html/) — ready-made UI components
- [Library: RESTAURANTE](libs/restaurante/) — restaurant platform modules
- [Philosophy](docs/philosophy.md) — why RPN and stacks

---

## Status

Merlin is in production use powering a restaurant management platform
in Viana do Castelo, Portugal. The codebase handles:

- Full restaurant menu with 4 layout types
- Photo gallery with lightbox
- Financial calculations (Tabela Price, TIR, VPL)
- Declarative page generator via database
- Zero backend logic for most templates

Built by Márcio Small do Vale — started in 2023, written entirely in Rust.

# Merlin

**A Rust-powered template engine with a real programming language inside.**

Merlin is built on RPN (Reverse Polish Notation) and stacks — the same philosophy
behind HP calculators. Instead of fighting with a limited expression language,
you get a full programming language directly in your templates.

In production since 2024, powering a restaurant management platform
in Viana do Castelo, Portugal.

---

## The problem with existing template engines

Every template engine eventually forces you to move logic to the backend —
not because the logic belongs there, but because the template can't handle it.

```python
# Django/Jinja2 — you end up doing this in the backend:
for p in produtos:
    p['total'] = p['preco'] * p['estoque']   # create computed field
produtos_filtrados = [p for p in produtos if p['estoque'] > 0]
produtos_ordenados = sorted(produtos_filtrados, key=lambda p: p['total'])
context['produtos'] = produtos_ordenados
```

With Merlin, the template handles it — and updates PRODUTOS in-place:

```merlin
PRODUTOS | FILTER("@ESTOQUE 0 >") | ADD_FIELD(@TOTAL = @PRECO @ESTOQUE *) | SORT_BY("@TOTAL") | ATUALIZA
```

And rendering is just as simple:

```merlin
// Merlin — explicit loop in template:
{{%% #FOR PRODUTO in PRODUTOS %%}}
    {{{{ HTML.MENU_CARD(PRODUTO) }}}}
{{%% #END_FOR %%}}

// Merlin using pipeline — same result:
{{{{ PRODUTOS | HTML.MENU_CARD }}}}

// or without the prefix — Merlin resolves HTML.MENU_CARD automatically:
{{{{ PRODUTOS | MENU_CARD }}}}
// HTML.* and RESTAURANTE.* are priority namespaces in templates.
// MENU_CARD resolves to HTML.MENU_CARD with no registration needed.
```

---

## What makes Merlin different

| Feature | Merlin | Jinja2 | Tera | Handlebars |
|---------|:------:|:------:|:----:|:----------:|
| Pipeline — any of 300+ operators and all lib modules | ✓ | ✗ | ✗ | ✗ |
| Reusable modules (real functions) | ✓ | partial | partial | partial |
| In-template computation | ✓ | limited | ✗ | ✗ |
| Context hierarchy (CORE/SITE/CLIENT) | ✓ | ✗ | ✗ | ✗ |
| Automatic operator promotion (SUM → VET_SUM) | ✓ | ✗ | ✗ | ✗ |
| FOR BY N — iterate in chunks | ✓ | ✗ | ✗ | ✗ |
| Libraries (FINANCE, STATS, HTML...) | ✓ | ✗ | ✗ | ✗ |
| Zero backend for complex logic | ✓ | ✗ | ✗ | ✗ |

> The pipeline column deserves emphasis: any of the 300+ built-in operators,
> any module from any library (HTML, FINANCE, RESTAURANTE...) and any module
> you create — all work in the pipeline automatically. No registration needed.

---

## Three template delimiters

```html
%%%%
    // Pure Merlin code — no output
    #ITENS[] = PRODUTOS | FILTER("@ESTOQUE 0 >") | SORT_BY("@PRECO")
    @TOTAL   = @PRODUTOS.PRECO.SUM()
%%%%

{{%% #FOR ITEM in ITENS %%}}        <!-- Merlin inline — no output (flow control) -->
    <div class="card">
        <h5>{{{{ ITEM.TITULO }}}}</h5>   <!-- Merlin inline — outputs to HTML -->
        <p>{{{{ $FORMATA_MOEDA(@ITEM.PRECO) }}}}</p>
    </div>
{{%% #END_FOR %%}}
```

---

## Reusable modules — defined once, used everywhere

```merlin
&CREATE_MODULE("HTML.MENU_CARD") ::

%%%%
    // HTML.MENU_CARD( model pedido )
    @PEDIDO = &ULTIMA_PILHA
    #MODEL  = &ULTIMA_PILHA
%%%%

    <div class="col-6 col-sm-4 col-lg-3 mb-4">
        <div class="card h-100">
            <a href="/{{{{ cliente.slug }}}}/produto/{{{{ model.id }}}}">
                <h5>{{{{ MODEL.TITULO }}}}</h5>
            </a>
            {{%% #IF @MODEL.PRECO 0.001 > %%}}
                <strong>{{{{ $FORMATA_MOEDA(@MODEL.PRECO) }}}}</strong>
            {{%% #END_IF %%}}
        </div>
    </div>

&END_MODULE()
```

```merlin
// Use anywhere — in templates, in other modules, in pipelines:
{{{{ HTML.MENU_CARD(PRODUTO 1) }}}}

// In templates, HTML. and RESTAURANTE. prefixes are optional —
// Merlin searches HTML.MENU_CARD first, then RESTAURANTE.MENU_CARD, then MENU_CARD:
{{{{ MENU_CARD(PRODUTO 1) }}}}

// Modules work in pipelines too — PRODUTO passed as current item:
{{{{ PRODUTOS | MENU_CARD(1) }}}}

// Automatic operator promotion — Merlin knows PRODUTOS is a vector:
@TOTAL = @PRODUTOS.PRECO.SUM()   // SUM → VET_SUM automatically
@MEDIA = @PRODUTOS.PRECO.AVG()
@MAX   = @PRODUTOS.PRECO.MAX()
```

---

## Context hierarchy — built-in multi-tenancy

Merlin resolves variables and modules in order:

```
CLIENTE  (restaurant-specific — highest priority)
    ↓
SITE     (platform-wide — shared across restaurants)
    ↓
CORE     (generic — HTML modules, utilities)
    ↓
LOCAL    (request-specific — per template)
```

A restaurant can override any module just by defining it in its own layer:

```merlin
// CORE — generic menu card for all restaurants
&CREATE_MODULE("HTML.MENU_BOX") :: ...

// CLIENTE (Pecado Capital) — custom version, no config needed
&CREATE_MODULE("HTML.MENU_BOX") ::
    GLOBAL::CORE::HTML.MENU_BOX()   // calls CORE version first
    // then adds restaurant-specific logic
&END_MODULE()
```

---

## Libraries — extensible by design

Merlin modules live in `.html` files or in a database table.
Load them at startup, use them everywhere:

```rust
// Rust route — minimal, no business logic
let mut calculator = setup_merlin_from_globals(state).await;
load_modules(&mut calculator, db, lib_finance).await;
processar_base_template("finance/tabela_price.html", &mut calculator, ..).await
```

```merlin
%%%%
    // Template — all logic here, zero backend
    FINANCE.TABELA_PRICE(300000.0 0.75 360)   // 360 iterations, 4 vectors
    FINANCE.PRICE_RESUMO
    FINANCE.PRICE_COMPARATIVO(0.65)
    FINANCE.PRICE_SIMULACAO_ANTECIPACAO(50000)
%%%%

{{{{ RESUMO_HTML }}}}
{{{{ TABELA_HTML }}}}
```

Available libraries (in this repository):

- **[HTML](libs/html/)** — ready-made UI components (gallery, lightbox, menu cards)
- **[FINANCE](libs/finance/)** — Tabela Price, TIR, VPL
- **[RESTAURANTE](libs/restaurante/)** — restaurant platform modules

---

## Scope control — explicit by design

```merlin
&LOCAL_SCOPE
    @SUBTOTAL = PRECO 1.1 *    // local — dies here
    @TOTAL += SUBTOTAL          // TOTAL is from outer scope — updated correctly
&END_LOCAL_SCOPE
// SUBTOTAL doesn't exist here ✓
// TOTAL was updated ✓

// Every #FOR loop automatically opens an isolated scope:
#FOR IDX, PRODUTO in PRODUTOS
    @TEMP = PRODUTO.PRECO 2 *   // local to this iteration
    @TOTAL += TEMP               // TOTAL from outer scope — correct
#END_FOR
// TEMP doesn't exist here ✓
```

---

## RPN — why stacks make sense

RPN eliminates ambiguity. Every operation is explicit:

```merlin
// PMT formula — readable step by step
@TAXA_DECIMAL  = TAXA_MENSAL 100 /
@FATOR         = 1 TAXA_DECIMAL +
@POTENCIA_N    = FATOR NUM_PARCELAS ^
@NUMERADOR     = TAXA_DECIMAL POTENCIA_N *
@DENOMINADOR   = POTENCIA_N 1 -
@PRESTACAO     = VALOR NUMERADOR DENOMINADOR / *

// Stacks are always empty after each statement — no garbage collector needed
// If something remains on the stack, it's a programming error
```

---

## Status

- **Language:** Rust
- **In production:** Restaurant management platform, Viana do Castelo, Portugal
- **Template files:** `.html` (VS Code friendly, syntax highlighting works)
- **Operators:** ~400 built-in (numbers, strings, dictionaries, dates, vectors)
- **Architecture:** RPN + 6 stacks + module system + library loader

---

## Documentation

- [Getting Started](docs/quick_start.md) — concepts, syntax, first template
- [Syntax Reference](docs/syntax.md) — complete language reference
- [Libraries](libs/) — ready-made modules

---

## Author

**Márcio Small do Vale**
[marciosmall@merlinlang.com](mailto:marciosmall@merlinlang.com)
[merlinlang.com](https://merlinlang.com)

> *Started in 2023. Zero CS background. Zero compiler theory studied.  
> Just 25 years of C/C++ and a HP 41CV calculator from 1983.*

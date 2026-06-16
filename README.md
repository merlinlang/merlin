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
    p['total'] = p['preco'] * p['estoque']
produtos_filtrados = [p for p in produtos if p['estoque'] > 0]
produtos_ordenados = sorted(produtos_filtrados, key=lambda p: p['total'])
context['produtos'] = produtos_ordenados
```

With Merlin, the template handles it — and updates PRODUTOS in-place:

```merlin
PRODUTOS | FILTER(@ESTOQUE 0 >) | ADD_FIELD(@TOTAL = @PRECO @ESTOQUE *) | SORT_BY("@TOTAL") | ATUALIZA
```

Rendering is equally direct:

```merlin
// Explicit loop:
{{%% #FOR PRODUTO in PRODUTOS %%}}
    {{{{ HTML.MENU_CARD(PRODUTO) }}}}
{{%% #END_FOR %%}}

// Pipeline — same result:
{{{{ PRODUTOS | HTML.MENU_CARD }}}}

// Prefix optional — Merlin resolves HTML.MENU_CARD automatically:
{{{{ PRODUTOS | MENU_CARD }}}}
```

---

## What makes Merlin different

| Feature | Merlin | Jinja2 | Tera | Handlebars |
|---------|:------:|:------:|:----:|:----------:|
| Pipeline — 400+ operators and all lib modules | ✓ | ✗ | ✗ | ✗ |
| Reusable modules (real functions) | ✓ | partial | partial | partial |
| In-template computation | ✓ | limited | ✗ | ✗ |
| Context hierarchy (CORE / SITE / CLIENT) | ✓ | ✗ | ✗ | ✗ |
| Automatic operator promotion (SUM → VET_SUM) | ✓ | ✗ | ✗ | ✗ |
| FOR BY N — iterate in chunks | ✓ | ✗ | ✗ | ✗ |
| Built-in libraries (FINANCE, STATS, HTML) | ✓ | ✗ | ✗ | ✗ |
| Zero backend for complex logic | ✓ | ✗ | ✗ | ✗ |
| Named optional parameters | ✓ | ✗ | ✗ | ✗ |
| Dynamic string evaluation with $$(expr) | ✓ | ✗ | ✗ | ✗ |

> Any of the 400+ built-in operators, any module from any library (HTML, FINANCE,
> RESTAURANTE) and any module you create — all work in pipelines automatically.
> No registration needed.

---

## Three template delimiters

```html
%%%%
    // Pure Merlin code — no output
    #ITENS[] = PRODUTOS | FILTER(@ESTOQUE 0 >) | SORT_BY("@PRECO")
    @TOTAL   = @PRODUTOS.PRECO.SUM()
%%%%

{{%% #FOR ITEM in ITENS %%}}             <!-- flow control — no output -->
    <div class="card">
        <h5>{{{{ ITEM.TITULO }}}}</h5>   <!-- outputs to HTML -->
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
// Use anywhere — templates, modules, pipelines:
{{{{ HTML.MENU_CARD(PRODUTO 1) }}}}

// HTML. and RESTAURANTE. prefixes are optional in templates:
{{{{ MENU_CARD(PRODUTO 1) }}}}

// Works in pipelines — PRODUTO passed as current item:
{{{{ PRODUTOS | MENU_CARD(1) }}}}
```

---

## Operator promotion

Merlin knows when to promote operators automatically:

```merlin
@TOTAL = @PRODUTOS.PRECO.SUM()   // SUM → VET_SUM automatically
@MEDIA = @PRODUTOS.PRECO.AVG()
@MAX   = @PRODUTOS.PRECO.MAX()

// Compact — all at once:
(@TOTAL, @MEDIA, @MAX) = @PRODUTOS.PRECO.SUM().AVG().MAX()
```

---

## Pipelines

See [examples/clientes_premium.html](examples/clientes_premium.html)
and [reviews/clientes_premium_review.md](reviews/clientes_premium_review.md)
for a complete walkthrough — 3 vectors, nested pipelines, zero SQL.

See [examples/cardapio_analise.html](examples/cardapio_analise.html)
and [reviews/cardapio_analise_review.md](reviews/cardapio_analise_review.md)
for a menu analysis that returns 6 values of different types from a single expression.

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
    GLOBAL::CORE::HTML.MENU_BOX()   // calls CORE version first, then extends
&END_MODULE()
```

---

## Libraries — extensible by design

```rust
// Rust route — 2 lines, no business logic
let mut calculator = setup_merlin_from_globals(state).await;
processar_base_template("finance/tabela_price.html", &mut calculator, ..).await
```

```merlin
%%%%
    // All logic here — zero backend
    FINANCE.TABELA_PRICE(300000.0 0.75 360)
    FINANCE.PRICE_RESUMO
    FINANCE.PRICE_COMPARATIVO(0.65)
    FINANCE.PRICE_SIMULACAO_ANTECIPACAO(50000)
%%%%

{{{{ RESUMO_HTML }}}}
{{{{ TABELA_HTML }}}}
```

Available libraries:

- **[HTML](libs/html/)** — UI components (gallery, lightbox, menu cards)
- **[FINANCE](libs/finance/)** — Tabela Price, TIR, VPL
- **[RESTAURANTE](libs/restaurante/)** — restaurant platform modules

---

## Named optional parameters

Modules accept both positional and named parameters:

```merlin
// Positional — classic RPN style
{{{{ RESTAURANTE.CARD_DESTAQUE_OPCOES(PRODUTO "md" 1 0) }}}}

// Named — optional, any order, with defaults
{{{{ RESTAURANTE.CARD_DESTAQUE_OPCOES(PRODUTO, "md", @USA_PRECO=1, $COR_FUNDO="#1a1a2e") }}}}

// Mixed — positional first, named after
{{{{ RESTAURANTE.MENU_PREMIUM(PRODUTO, "lg", $COR_FUNDO="#8B0000", $COR_TEXTO="#fff") }}}}
```

Inside the module, named parameters are read defensively:

```merlin
$COR_FUNDO = #IF $DEFINED("_PARAMETROS_::COR_FUNDO") :: _PARAMETROS_::COR_FUNDO #ELSE ""
@USA_PRECO = #IF @DEFINED("_PARAMETROS_::USA_PRECO") :: _PARAMETROS_::USA_PRECO #ELSE 1
&END_LOCAL_ONLY_SCOPE(_PARAMETROS_)
```

---

## Dynamic string evaluation

```merlin
// $$(expression) evaluates any Merlin expression that leaves a string on the stack
$STYLE = $$("background: ${COR_FUNDO}; color: ${COR_TEXTO}; opacity: ${@@OPACIDADE}%;")

// ${EXPR} inside $$() evaluates any Merlin expression inline
$URL   = $$("/${cliente.slug}/produto/${@@MODEL.ID}/")
$LABEL = $$("${MODEL.TITULO} — ${$FORMATA_MOEDA(@@MODEL.PRECO)}")

// Indirection — $$NOME reads the variable whose name is stored in NOME
$CAMPO  = "TITULO"
$VALOR  = $$("item: $$CAMPO")   // → "item: Bacalhau"
```

---

## Scope control — explicit by design

```merlin
&LOCAL_SCOPE
    @SUBTOTAL = PRECO 1.1 *    // local — dies here
    @TOTAL += SUBTOTAL          // TOTAL from outer scope — updated correctly
&END_LOCAL_SCOPE
// SUBTOTAL doesn't exist here ✓
// TOTAL was updated ✓

// Every #FOR opens an isolated scope automatically:
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
@TAXA_DECIMAL  = TAXA_MENSAL 100 /
@FATOR         = 1 TAXA_DECIMAL +
@POTENCIA_N    = FATOR NUM_PARCELAS ^
@NUMERADOR     = TAXA_DECIMAL POTENCIA_N *
@DENOMINADOR   = POTENCIA_N 1 -
@PRESTACAO     = VALOR NUMERADOR DENOMINADOR / *

// Stacks are always empty after each statement.
// If something remains, it is a programming error — caught immediately.
```

---

## Status

- **Language:** Rust
- **In production:** Restaurant management platform, Viana do Castelo, Portugal
- **Template files:** `.html` (VS Code syntax highlighting works out of the box)
- **Operators:** ~400 built-in (numbers, strings, dictionaries, dates, vectors)
- **Architecture:** RPN + 6 stacks + module system + library loader

---

## Documentation

- [Getting Started](docs/quick_start.md)
- [Syntax Reference](docs/merlin_reference.md)
- [Named Parameters](docs/merlin_parametros_nomeados.md)
- [Libraries](libs/)
- [Examples](examples/)

---

## Author

**Márcio Small do Vale**
[marciosmall@merlinlang.com](mailto:marciosmall@merlinlang.com)
[merlinlang.com](https://merlinlang.com)

> *Started in 2023. Zero CS background. Zero compiler theory studied.*
> *Just 25 years of C/C++ and a HP 41CV calculator from 1983.*

# Merlin — Referência da Linguagem

Linguagem criada em Rust, puramente RPN (Reverse Polish Notation) e baseada em pilhas.
Usada principalmente com templates HTML para sites de Restaurantes.

---

## Tipos e Sigils

| Sigil | Tipo       | Vetor       |
|-------|------------|-------------|
| `@`   | Número     | `@VAR[]`    |
| `$`   | String     | `$VAR[]`    |
| `#`   | Dicionário | `#VAR[]`    |

- Sigils **só no LHS** (lado esquerdo da atribuição)
- No RHS acede-se sem sigil — excepto `@CAMPO` para ler field de dicionário como número
- Variáveis não precisam de ser declaradas
- Merlin é **case-insensitive** — `PRODUTO.TITULO`, `produto.titulo`, `Produto.Titulo` são iguais

---

## As 6 Pilhas

```
pilha_numeros
pilha_strings
pilha_dicionarios
pilha_vetores_numeros
pilha_vetores_strings
pilha_vetores_dicionarios
```

- RHS empurra para a pilha → LHS consome
- Pilha consumida desaparece automaticamente — **sem garbage collector**
- LHS não sabe o que vem do RHS; RHS só precisa deixar o resultado na pilha certa
- Se quiser usar um valor mais do que uma vez, guarde numa variável

## Segurança da Pilha e Valores Padrão

Os operadores da Merlin não fazem validação prévia da pilha.

Eles sempre executam, consumindo os valores que estiverem disponíveis.  
Se algum valor necessário estiver ausente, a própria pilha resolve isso usando o valor padrão do tipo correspondente.

Valores padrão:

- Número ausente → `0`
- String ausente → `""`
- Vetor ausente → `[]`
- Dicionário ausente → `{}`

Isso significa que valores ausentes não são tratados como erro por padrão; eles são interpretados como valores neutros.

Exemplo:

```merlin
#IF .NOT. FORBIDDEN_CREATE
    // executa quando FORBIDDEN_CREATE não existe,
    // porque valores numéricos ausentes resolvem para 0,
    // e NOT 0 retorna 1
#END_IF
---

## Atribuição

```merlin
@VALOR = 100.0
$NOME  = "Marcio Small do Vale"
#ITEM  = { "TITULO" : "Lapiseira" , "ESTOQUE" : "200" , "PRECO" : 3.85 }
```

Todos os fields de dicionário são guardados como string.
Para ler um field como número: `@ITEM.PRECO`

### Atribuição múltipla

```merlin
($VAR , #DIC[] , @VET[] , $STR) = expr1 expr2 expr3 expr4
```

---

## Vectores — Indexação

```merlin
VAR[0]       // acesso directo
VAR[+]       // push no fim
VAR[-]       // pop do fim
VAR[N+]      // insert na posição N
VAR[N-]      // delete na posição N
VAR[x]       // aplica operação a todos os elementos
VAR[1..3]    // slice
VAR[..3]     // slice do início até 3
VAR[2..]     // slice do índice 2 até ao fim
```

### Vetor de dicionários — acumular

```merlin
#JSON_ITEMS[+] = { "src" : SRC , "cap" : CAP , "sub" : SUB }
```

Acesso posterior:
```merlin
JSON_ITEMS[IDX].CAP
JSON_ITEMS[IDX].SRC
```

---

## Operadores

```merlin
+   -   *   /           // aritméticos
+=  -=  *=  /=          // compostos
%                       // percentual  ( IX NUM_COLS %  →  IX=1, NUM_COLS=5  →  0.05 )
$+                      // concatenação de strings
$*                      // repetição de string  ( MARIDO *= 3 " " )
$==                     // comparação de strings (case-insensitive)
==                      // comparação de números
>?  <?  >=?  <=?        // comparações numéricas
.AND.   .OR.   .NOT.    // lógicos
%+                      // aplica percentual e soma  ( PRECO.EXECUTA(20 %+) )
```

---

## Encadeamento — dois estilos equivalentes

```merlin
$CATALOGO.NOME.UPPER().SORT().ATUALIZA()
$CATALOGO.NOME | UPPER | SORT | ATUALIZA
```

`.ATUALIZA()` — actualiza a variável original in-place  
`.PILHA()` — coloca o resultado na pilha sem actualizar  
`.EXECUTA(expr)` — executa expressão no contexto do valor

---

## #IF

### Bloco completo

```merlin
#IF CONDICAO
    // ...
#END_IF

#IF CONDICAO
    // ...
#ELSE
    // ...
#END_IF
```

### Inline — linhas curtas, SEM #ELSE_IF, SEM #END_IF

```merlin
$NOME     = #IF ITEM          :: ITEM.NOME          #ELSE "SEM NOME"
@RES      = #IF CATALOGO[2]   :: 1                  #ELSE 0
$THUMB    = #IF FOTO.IMAGEM_THUMB :: "/_cache_/media/" FOTO.IMAGEM_THUMB $+ #ELSE "/_cache_/media/" FOTO.IMAGEM $+
```

- Se campo não existe, Merlin preenche com `0.0`, `""` ou `{}` — elimina muitos #ELSE
- `#IF` inline **não tem `#ELSE_IF`** — para lógica com HTML misturado usar sempre o bloco

### Existência de variável/campo

```merlin
@R = #IF CATALOGO[2]          :: 1 #ELSE 0    // dicionário existe no vetor?
@R = #IF ITEM.NOME            :: 1 #ELSE 0    // field existe?
@R = #IF CATALOGO[1] .AND. ITEM.NOME :: 1 #ELSE 0
@R = #IF CATALOGO[1] .OR.  CATALOGO[20] :: 1 #ELSE 0
```

---

## #FOR

```merlin
#FOR ITEM in VETOR
    // ...
#END_FOR

#FOR IDX, ITEM in VETOR         // IDX automático, sem inicializar nem incrementar
    // ...
#END_FOR

// Com WHERE
#FOR P, PRODUTO in PRODUTOS WHERE PRODUTO.CATEGORIA_ID CATEGORIA.ID $==
    // ...
#END_FOR
```

---

## #MATCH

```merlin
$RESULTADO = #MATCH VARIAVEL
    "valor1" :: expressao1
    "valor2" :: expressao2
    default  :: expressao_default
#END_MATCH

// Atribuição múltipla com #MATCH
($COLS , $STYLE) = #MATCH SIZE
    "sm"  :: "col-6 col-sm-4 col-lg-3 mb-4"  "4px"
    "lg"  :: "col-12 col-sm-12 col-lg-6 mb-4" "5px"
    default :: "col-12 col-sm-6 col-md-4 col-lg-3 mb-4" "4px"
#END_MATCH
```

---

## Módulos

```merlin
&CREATE_MODULE("NOME") :: [OPCOES]
    @PARAM3 = &ULTIMA_PILHA    // lê parâmetros da pilha — último push = primeiro a ler
    @PARAM2 = &ULTIMA_PILHA
    @PARAM1 = &ULTIMA_PILHA
    // lógica
&END_MODULE()
```

### Opções de módulo

| Opção               | Significado                                      |
|---------------------|--------------------------------------------------|
| `NOT_LOCAL_VARIABLES` | módulo acede às variáveis do scope pai         |
| `PARAM_IS_VETOR`    | parâmetro é um vetor                             |
| `MERLIN_WRITE`      | pode escrever variáveis no scope pai             |

Escrever para o scope pai:
```merlin
@MERLIN::X = 152
$MERLIN::BRASIL[] = $["DEZEMBRO" ; "JANEIRO" ; "FEVEREIRO"]
```

### Chamadas

```merlin
NOME(param1 param2 param3)
VAR.NOME()
VAR.NOME(param_extra)
```

### Nomenclatura com ponto (namespace)

```merlin
HTML.LIGHTBOX
HTML.GALLERY_HOVER_FOTOS
RESTAURANTE.MENU_SELECT_TIPO
```

---

## SANDBOX

```merlin
SANDBOX.HTML.MENU_CARD(PRODUTO PEDIDO)
```

Abre uma nova instância completa da Merlin isolada — ideal para módulos que usam
um contexto totalmente diferente (ex: passeios de outra base de dados).
Os módulos normais já são isolados sem necessidade de SANDBOX.

---

## DEFAULT — ideal para templates

```merlin
$DEFAULT1 = ESPOSA | DEFAULT("valor padrão")
$DEFAULT2 = "" | DEFAULT("string vazia usa default")
$DEFAULT3 = CATALOGO[1].CHAVE_INEXISTENTE | DEFAULT("chave não existe")
$DEFAULT4 = CATALOGO[1].NOME | REPLACE("X" "") | DEFAULT("fallback após pipe")
```

---

## Strings — operações úteis

```merlin
TEXTO | SANITIZE                        // sanitiza HTML
TEXTO | LOWER                           // minúsculas
TEXTO | UPPER                           // maiúsculas
TEXTO | REPLACE("antigo" "novo")        // substituição
TEXTO | STRING_ESCAPE                   // escapa para HTML
TEXTO.LEN()                             // comprimento
$FORMATA_MOEDA(@MODEL.PRECO)            // formata moeda
$NUM(cat_select)                        // string → número
$FORMATA_VALOR(VALOR "######.##")       // formata número
```

---

## Datas

```merlin
~DT<TODAY>                          // número (dias)
~DT<TODAY_STR>                      // string data de hoje
~DT<NUM_STR>(HOJE)                  // número → string
~DT<SOMA_MESES_STR>(MESES)          // soma meses → string
~DT<SYSTEM_TO_EXTENSO>(data)        // data por extenso
~DT<SISTEMA>                        // data/hora do sistema
```

---

## Comandos de debug / print

```merlin
&DEBUG_MERLIN
&CLEAR
&NL
&PRINT_NL("texto" ; VARIAVEL)
&MOSTRA_PILHA
&MOSTRA_PILHA_STRING
&MOSTRA_PILHA_VETOR_NUMERO
&MOSTRA_PILHA_VETOR_STRING
&MOSTRA_PILHA_DICIONARIO
&MOSTRA_PILHA_VETOR_DICIONARIO
&MOSTRA_VARIAVEIS_DICIONARIO
&MOSTRA_VARIAVEIS_STRING
&MOSTRA_VARIAVEIS
&MOSTRA_MODULOS
```

---

## Templates — Delimitadores

| Delimitador          | Uso                                                      |
|----------------------|----------------------------------------------------------|
| `%%%% ... %%%%`      | Bloco multi-linha de Merlin puro — sem output HTML       |
| `{{%% ... %%}}`      | Merlin numa linha — fluxo: `#IF`, `#FOR`, `#MATCH`, atribuições |
| `{{{{ ... }}}}`      | Merlin numa linha — última pilha string → output HTML    |

### Regras dos delimitadores

- `{{%% %%}}` — Merlin puro, sem output. Ideal para fluxo e atribuições inline.
- `{{{{ }}}}` — produz output. Tudo o que ficar na pilha string é renderizado.
- Para número no output: `{{{{ @$(VARIAVEL) }}}}`
- Variáveis Merlin dentro de `<script>`: `{{{{ JSON_ITEMS[IDX].src }}}}`

### Quarto delimitador — módulos com fases

```html
<<<< BEGIN  :: RESTAURANTE.MENU_SELECT_TIPO(TITULO TIPOS_MENU CAT_SELECT) >>>>
<<<< MIDDLE :: SANDBOX.HTML.MENU_TABLE(PRODUTO PEDIDO) >>>>
<<<< END    :: HTML.MENU_TABLE(PEDIDO) >>>>
```

O módulo recebe `$CONTROLE` e faz `#MATCH` internamente — ideal para containers
que precisam de abertura, itens e fecho separados (ex: `<table>`, `<section>`).

---

## Contexto global — sempre disponível em RAM

Carregado às 10h (2h antes da abertura), actualizado diariamente:

| Variável              | Conteúdo                        |
|-----------------------|---------------------------------|
| `cliente`             | dados do restaurante            |
| `produtos`            | todo o cardápio                 |
| `categorias_produto`  | categorias dos produtos         |
| `tipos_menu`          | tipos de menu disponíveis       |
| `config_global`       | configurações globais           |
| `messages`            | mensagens flash pendentes       |

Da rota chegam apenas parâmetros da request — ex: `CAT_SELECT`, `PEDIDO`, `TIPO_MENU`.

---

## Estrutura de Foto (#DIC)

```
foto.id
foto.titulo              // usado como alt text
foto.texto_antes
foto.rodape
foto.descricao
foto.categoria_principal
foto.categoria_sec1
foto.categoria_sec2
foto.imagem              // path — 800px
foto.imagem_thumb        // path — 400px
foto.video               // webm (Android)
foto.video_mp4           // mp4 (iPhone)
foto.width_original / height_original
foto.width_webp / height_webp
```

Paths de imagem montados assim:
```merlin
$SRC = "/_cache_/media/" FOTO.IMAGEM $+
```

---

## Vetores aninhados em fields de dicionário

A Merlin suporta vetores (números, strings, dicionários) dentro de fields de dicionário,
construídos de dentro para fora com funções dedicadas:

```merlin
#DIC_FIELD_VETOR_NEW("PRODUTO" "FOTOS")
#DIC_FIELD_VETOR_PUSH_DIC("PRODUTO" "FOTOS" { "IMAGEM" : "lasanha1.webp" , "DESCRICAO" : "Detalhe" })
#DIC_FIELD_VETOR_PUSH_DIC("PRODUTO" "FOTOS" { "IMAGEM" : "lasanha2.webp" , "DESCRICAO" : "Porção" })
#DIC_FIELD_VETOR_NEW("PRODUTO" "VENDAS")
#DIC_FIELD_VETOR_PUSH_NUM("PRODUTO" "VENDAS" 150.50)

// Acesso
@PRODUTO.FOTOS[0].IMAGEM
#FOTO     = #DIC_FIELD_VETOR_GET_DIC("PRODUTO" "FOTOS" 1)
@SOMA     = #DIC_FIELD_VETOR_SUM_NUM("PRODUTO" "VENDAS")
#CAMPOS[] = #DIC_FIELD_VETOR_DIC("PRODUTO" "FOTOS")

// Operações em massa
@PRODUTO.FOTOS[x].PRECO  = @PRODUTO.FOTOS[x].PRECO 1.1 *
@PRODUTO.VENDAS[x]       = @PRODUTO.VENDAS[x] 2 *
```

**Passagem via contexto JSON** — ainda não implementado.
Quando vier do JSON externo com array dentro de field, a solução aguarda
uma abordagem simples e escalável — filosofia da Merlin: só implementar
quando a solução for clara. A Merlin já tem todas as ferramentas para
construir e manipular estas estruturas internamente.

---

## ITENS_COM_FOTOS — estrutura padrão

Todos os ficheiros com fotos usam esta estrutura:
```
ITENS[]
  ITENS[N].ITEM   →  #DIC  (o modelo — produto, passeio, etc.)
  ITENS[N].FOTO   →  #DIC  (a foto associada)
```

---

## Módulos criados — Biblioteca

### `HTML.LIGHTBOX`
Lightbox independente. Incluir **uma vez** na template.
- Navegação: setas ← → e Esc
- Array de itens em `window._mlbItems`
- Abrir: `mlbOpen(idx)`

### `HTML.COLS_ANALISA_SIZE(size)`
Retorna `($COLS, $STYLE_PADDING)` — classes Bootstrap responsivas.

| Size    | Colunas Bootstrap                                          |
|---------|------------------------------------------------------------|
| `xxsm`  | col-3 col-sm-3 col-md-3 col-lg-2 col-xl-1                 |
| `xsm`   | col-4 col-sm-4 col-md-3 col-lg-3 col-xl-2                 |
| `sm`    | col-6 col-sm-6 col-md-4 col-lg-3 col-xl-3 col-xxl-2       |
| `md`    | col-12 col-sm-6 col-md-6 col-lg-4                          |
| `lg`    | col-12 col-sm-12 col-md-12 col-lg-6 col-xl-6              |
| `xl`    | col-12 col-sm-12 col-md-12 col-lg-12 col-xl-6             |
| `xxl`   | col-12 col-sm-12 col-md-12 col-lg-12 col-xl-12            |

### `HTML.GALLERY_GRID_FOTOS(itens[] route_url size)`
Galeria em grid Bootstrap com cards — título e rodapé em `card-body`.

### `HTML.GALLERY_HOVER_FOTOS(itens[] route_url size)`
Galeria em grid Bootstrap — foto ocupa o card inteiro, legenda como overlay no hover.
Abre lightbox ao clicar. **Requer `HTML.LIGHTBOX`**.

```merlin
// Monta vetor para o lightbox (no bloco %%%%)
#FOR REG in ITENS
    // ... calcula SRC, CAP, SUB ...
    #JSON_ITEMS[+] = { "src" : SRC , "cap" : CAP , "sub" : SUB }
#END_FOR

// No <script> — usa variáveis Merlin directamente
window._mlbItems = [
    {{%% #FOR IDX, REG in ITENS %%}}
        { src: "{{{{ JSON_ITEMS[IDX].src }}}}", cap: "{{{{ JSON_ITEMS[IDX].cap }}}}", sub: "{{{{ JSON_ITEMS[IDX].sub }}}}" },
    {{%% #END_FOR %%}}
];

// No onclick
onclick="mlbOpen({{{{ @$(IDX) }}}})"
```

### `HTML.PICTURE_RENDER_THUMB(foto)`
Renderiza `<picture>` com `imagem_thumb` (fallback para `imagem`).

### `HTML.PICTURE_RENDER_IMAGEM(foto)`
Renderiza `<picture>` com `imagem` full (fallback para `imagem_thumb`).

### `HTML.VIDEO_RENDER(foto)`
Renderiza `<video>` com source webm + mp4.

### `HTML.SETA_UP(size)` / `HTML.SETA_DOWN(size)`
SVG de seta para cima/baixo. `size` default = `"36"`.

### `HTML.MESSAGES_VIEW`
Mostra mensagens flash pendentes. Include automático no início de templates.

### `HTML.DIVS_TEMA_SILVER_E_ROUNDED` / `HTML.FECHA_TEMA_SILVER_E_ROUNDED`
Abre/fecha divs condicionais de tema baseado em `CONFIG_GLOBAL`.

### `HTML.FRASE(frase formatada)`
Mostra frase com ou sem formatação `<h2>`.

### `HTML.GOOGLE_MAP(model)`
Renderiza iframe do Google Maps. Usa `model.google_map` ou fallback `cliente.google_map`.

### `HTML.FOTOS_CATEGORIAS(foto)`
Mostra categorias da foto com links.

### `HTML.HORARIO_DADOS_CLIENTE()`
Card com horário de funcionamento e dados de contacto do restaurante.

### `HTML.SHOW_HTML_TEXT(texto)`
Mostra texto HTML escapado dentro de `<pre>`.

### `HTML.MENU_BOX(model pedido)`
Card de cardápio tipo box — foto à esquerda, texto à direita.

### `HTML.MENU_CARD(model pedido)`
Card de cardápio tipo card — foto em cima, título e preço em baixo.

### `HTML.MENU_CARD_ID(id_produto pedido)`
Igual ao `MENU_CARD` mas busca o produto por ID no vetor `PRODUTOS`.

### `HTML.MENU_FLEX(model pedido)`
Card de cardápio tipo flex — layout media com foto lateral.

### `HTML.MENU_TABLE(controle)` — módulo com fases BEGIN / MIDDLE / END
Tabela de cardápio. Usar com `<<<<  >>>>`.

### `HTML.CARD_TITULO(model)`
Card simples com foto e título — sem preço.

### `RESTAURANTE.SCRIPT_DUPLO_SELECT`
Script JS para os selects de tipo de menu e categoria.

### `RESTAURANTE.MENU_SELECT_CATEGORIA(tipo_menu)`
Select dropdown de categorias de produto.

### `RESTAURANTE.MENU_SELECT_TIPO(controle)` — BEGIN / END
Container principal do cardápio com select de tipo de menu.

### `RESTAURANTE.SLUG_PASSEIO_SELECT(passeio slug_selecao)`
Select dropdown para navegar entre slugs/passeios.

### `RESTAURANTE.INFORMA_PASSEIO(passeio)`
Mostra informações do passeio — cidade, país, criador, data, descrição.

### `HTML.FOTOS_EXTRAS(fotos_extras)`
Grid de fotos extras de um passeio.

---

## Padrão de uso — template de cardápio

```html
%%%%
    // contexto já em RAM — só parâmetros da rota chegam aqui
    // CAT_SELECT e PEDIDO injectados pela rota Rust
%%%%

<<<< BEGIN :: RESTAURANTE.MENU_SELECT_TIPO(CLIENTE.TITULO TIPOS_MENU CAT_SELECT) >>>>

    {{%% #FOR CATEGORIA in CATEGORIAS_PRODUTO WHERE CAT_SELECT 0 == .OR. CAT_SELECT @CATEGORIA.ID == %%}}
        {{%% @TEM_CATEGORIA = 0 %%}}

        {{%% #FOR P, PRODUTO in PRODUTOS WHERE PRODUTO.CATEGORIA_ID CATEGORIA.ID $== %%}}

            {{%% #IF .NOT. TEM_CATEGORIA %%}}
                {{%% @TEM_CATEGORIA = 1 %%}}
                {{{{ RESTAURANTE.MENU_SELECT_CATEGORIA(TIPO_MENU) }}}}
                {{%% #IF TIPO_MENU "table" $== %%}}
                    <div class="card mb-6">
                    <<<< BEGIN :: HTML.MENU_TABLE(PEDIDO) >>>>
                {{%% #ELSE %%}}
                    <br><div class="row">
                {{%% #END_IF %%}}
            {{%% #END_IF %%}}

            {{%% #MATCH TIPO_MENU %%}}
                "table" :: <<<< MIDDLE :: SANDBOX.HTML.MENU_TABLE(PRODUTO PEDIDO) >>>>
                "card"  :: {{{{ SANDBOX.HTML.MENU_CARD(PRODUTO PEDIDO) }}}}
                "flex"  :: {{{{ SANDBOX.HTML.MENU_FLEX(PRODUTO PEDIDO) }}}}
                default :: {{{{ SANDBOX.HTML.MENU_BOX(PRODUTO PEDIDO) }}}}
            {{%% #END_MATCH %%}}

        {{%% #END_FOR %%}}

        {{%% #IF TEM_CATEGORIA %%}}
            {{%% #IF TIPO_MENU "table" $== %%}}
                <<<< END :: HTML.MENU_TABLE(PEDIDO) >>>>
            {{%% #END_IF %%}}
            </div>
        {{%% #END_IF %%}}

    {{%% #END_FOR %%}}

<<<< END :: RESTAURANTE.MENU_SELECT_TIPO >>>>
```

---

## Arquitectura Global — CORE / SITE / CLIENTE

### Hierarquia de resolução

```
CLIENTE  →  SITE  →  CORE  →  LOCAL
```

Merlin procura **do mais específico para o mais geral** — vale para variáveis, módulos e expressões.

| Contexto  | Ficheiro de módulos                          | Conteúdo                                        |
|-----------|----------------------------------------------|-------------------------------------------------|
| `CORE`    | `core_html.html` + `pagina_html.html`        | Módulos genéricos — `HTML.*`                    |
| `SITE`    | `site_html.html`                             | Módulos de Restaurantes — `RESTAURANTE.*`       |
| `CLIENTE` | `cliente_html.html`                          | Módulos específicos do restaurante (ex: Pecado Capital) |

### Overriding de módulos

Um módulo pode existir nos três contextos — CLIENTE sobrepõe SITE que sobrepõe CORE.
Cada restaurante pode ter o seu próprio `HTML.MENU_BOX` completamente diferente.

Dentro de uma template também é possível redefinir um módulo localmente:

```merlin
%%%%
    &CREATE_MODULE("HTML.MENU_BOX") ::
        // versão local — só existe durante esta template
        // sobrepõe qualquer versão de CORE / SITE / CLIENTE
    &END_MODULE()
%%%%
```

O módulo local morre no final da template.

### Ciclo do dia — inicialização às 10h

```
1. Merlin::new()           →  instância temporária limpa
2. load_modules()          →  carrega site_html.html
3. load_cliente_context()  →  dados frescos da BD
4. criar_variaveis_merlin  →  injeta PRODUTOS, CLIENTE, CONFIG_GLOBAL, etc.
5. executa_programa()      →  processa dados em Merlin (ver abaixo)
6. write lock Arc          →  move tudo para GlobalMerlin
7. Repete para CLIENTE e CORE  →  só módulos, sem dados
```

Programa Merlin que corre na inicialização:

```merlin
#PRODUTOS[] = #PRODUTOS.ITEM.FIELDS("ID" ; "CATEGORIA_ID" ; "TITULO" ; "SLUG" ; "RODAPE" ; "DESCRICAO" ; "PRECO")
#FOTOS[] = PRODUTOS.FOTO
#DIC_VET_NOVA_CHAVE("PRODUTOS" "IMAGEM_ID" "")
#PRODUTOS[x].IMAGEM_ID    = FOTOS[{x}].ID              // [x] = todos ; {x} = índice paralelo
#PRODUTOS[x].IMAGEM_THUMB = FOTOS[{x}].IMAGEM_THUMB
#PRODUTOS[x].IMAGEM       = FOTOS[{x}].IMAGEM
$TIPOS_MENU[] = [ "card" ; "box" ; "flex" ; "table" ]
$TIPO_MENU = "box"
```

### Por request — setup_merlin_from_globals

Cada request clona os `Arc` (barato — só incrementa contador de referência).
Variáveis locais da request (`CAT_SELECT`, `PEDIDO`, `TIPO_MENU`) ficam separadas dos globais.

### Lógica de actualização

```rust
restaurante_atual != cliente_context.cliente.id          // mudou de restaurante
|| hoje - data_atualizou > 1.0                           // passou mais de um dia
|| (horas_agora >= 10.0 && hoje - data_atualizou == 1.0) // são 10h e ainda é do dia anterior
```

### processar_base_template

```
BASE_MERLIN.HTML  +  template_da_rota
        ↓
substitui  {{{{ TEMPLATE_CONTENT }}}}
        ↓
calculator.processar_template_merlin(template_final)
```

Cache de templates (`precompiled_templates`) evita I/O de disco — lê de `HashMap` em memória.

---

## Pipeline de Transformação — EXECUTA_EACH

O `EXECUTA_EACH` é um pipeline de transformação com três fases invisíveis,
separadas pelo `&BEGIN_EACH`. Pode ser aninhado dentro de outros pipelines.

### Estrutura — 3 fases

```merlin
PRATOS.EXECUTA_EACH(

    // ── INICIALIZA — executa uma vez antes de iterar
    @TOTAL     = 0
    @MAIS_CARO = 0

    &BEGIN_EACH(
        // ── ITERA — executa para cada item
        // ITEM = dicionário actual (disponível automaticamente)
        // IDX  = índice actual
        // ITEMS = vetor completo (para agregações)
        #PRATO = ITEM
        @TOTAL += @PRATO.PRECO
        #IF @PRATO.PRECO MAIS_CARO >
            @MAIS_CARO = @PRATO.PRECO
        #END_IF
    )

    // ── FINALIZA — executa uma vez depois de iterar
    @MEDIA = TOTAL CONTAGEM /
)
```

Variáveis criadas no `INICIALIZA` e `ITERA` ficam disponíveis no `FINALIZA`.
`ITEM`, `IDX` e `ITEMS` são locais a cada iteração — tudo o resto é do scope exterior.

### &BEGIN_EACH — controlo do vetor

O que fica na pilha dentro do `&BEGIN_EACH` define o que continua no vetor:

```merlin
&BEGIN_EACH(
    #PRATO = ITEM
    #IF @PRATO.PRECO 10 >
        PRATO    // devolve à pilha → fica no vetor
    #END_IF
    // não devolveu → item removido do vetor
)
```

### RETURN — extrai valores específicos do pipeline

`RETURN` termina o pipeline e coloca valores escolhidos na pilha.
Aceita nomes de chaves, variáveis de qualquer tipo, ou qualquer expressão Merlin:

```merlin
// Chaves do dicionário
$TITULOS[] = PRODUTOS | SORT_BY("$TITULO") | RETURN($TITULO)
@PRECOS[]  = PRODUTOS | FILTER(@ESTOQUE 0 >) | RETURN(@PRECO)
#FOTOS[]   = PRODUTOS | FILTER(@PRECO MEDIA >) | RETURN(#FOTO)

// Multi-retorno — tipos diferentes numa expressão só
($MAIS_CARO , @TOTAL , #FILTRADOS[]) = PRODUTOS |
    FILTER(@ESTOQUE 0 >) |
    RETURN($TITULO , @PRECO , #ITEM)

// Variáveis (não chaves) — tipo detectado automaticamente
PRODUTOS | EXECUTA_EACH(...) | RETURN(NOME_MAIS_CARO , TOTAL , RESUMO)

// Expressões calculadas — Merlin não é babá de programador
PRODUTOS | RETURN(@PRECO 1.1 * , $TITULO.UPPER() , TOTAL CONTAGEM /)

// Strings literais — vírgulas dentro das aspas não são separadores
PRODUTOS | RETURN("Beef, Pasta, Fish" , TOTAL)
```

### Pipelines aninhados

Pipelines podem ser aninhados — um pipeline dentro do `EXECUTA_EACH` dentro de outro pipeline.
A Merlin resolve cada nível independentemente, com scopes isolados para `ITEM` e `IDX`:

```merlin
// Três níveis de pipeline — tudo na template, tudo em RAM
#RESUMO[] = CATEGORIAS |
    EXECUTA_EACH(
        #RESUMO_CAT[] = #[]

        &BEGIN_EACH(
            #CAT = ITEM

            PRATOS |
                FILTER(@CATEGORIA_ID @CAT.ID ==) |   // pipeline dentro de pipeline
                EXECUTA_EACH(
                    @TOTAL_CAT = 0
                    @QTD_CAT   = 0
                    &BEGIN_EACH(
                        @TOTAL_CAT += @ITEM.PRECO     // terceiro nível
                        @QTD_CAT   += 1
                    )
                )
            &END_PIPELINE   // marcador documental — sem efeito na execução

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
&END_PIPELINE
```

`@CAT.ID` no `FILTER` interno refere-se à categoria do loop exterior —
a Merlin resolve do scope exterior automaticamente. Sem passagem de parâmetros.

### Marcadores de fluxo

```merlin
&END_PIPELINE     // documental — marca o fim de um bloco de pipeline, sem efeito
&NO_STACK()       // diz ao EXECUTA_EACH para não deixar o vetor na pilha
```

### WHERE no pipeline

Filtra os itens para as operações seguintes. Um novo `WHERE` substitui o anterior:

```merlin
CATALOGO
    | WHERE("ESTOQUE 50 >?")   // filtra estoque > 50
    | EXECUTA_EACH(...)        // processa só esses
    | WHERE("PRECO 10 >?")    // novo filtro — substitui o anterior
    | EXECUTA_EACH(...)        // processa só os de preço > 10
    | ATUALIZA
```

### Tabela de operadores do pipeline

| Operador | Descrição |
|----------|-----------|
| `FILTER(expr)` | Mantém itens onde expr é verdadeiro |
| `WHERE(expr)` | Define condição para operadores seguintes |
| `ADD_FIELD(@F = expr)` | Cria field calculado em cada item |
| `SORT_BY("@FIELD")` | Ordena ascendente por field ou expressão |
| `SORT_BY_DESC("@FIELD")` | Ordena descendente |
| `SORT_DESC("@FIELD")` | Descendente pela chave actual |
| `EXECUTA(prog)` | Executa programa no vetor inteiro |
| `EXECUTA_EACH(...)` | Inicializa / itera / finaliza |
| `RETURN(a , b , c)` | Extrai valores — chaves, variáveis ou expressões |
| `ATUALIZA` | Grava resultado na variável original |

### #MATCH com expressões abertas

Nada no `#MATCH` precisa de ser fixo — nem o valor a comparar, nem as chaves, nem o resultado:

```merlin
#MATCH { expressao_qualquer }
    { expressao_chave_1 } :: { bloco de várias linhas }
    { expressao_chave_2 } :: { outro bloco }
    default               :: { fallback }
#END_MATCH
```

A comparação pode ser de **múltiplas pilhas** em simultâneo.
O resultado de cada braço pode ser qualquer tipo — número, string, dicionário, HTML.

### Índice como expressão

Qualquer índice de vetor pode ser uma expressão, módulo ou programa completo:

```merlin
VAR[MODULO_QUE_CALCULA_INDICE()]
VAR[{ expressao complexa }]
```

---

## Sistema de Cache — arquitectura completa

### 3 tipos de cache em RAM

| Cache | Rota | TTL browser | Fallback |
|---|---|---|---|
| `PrecompiledPhotos` | `/_cache_/media/<path>` | 86400s (24h) | `/media/<path>` disco, max-age=300 |
| `PrecompiledCss` | `/_cache_/css/<path>` | 86400s (24h) | `static/css/<path>` disco |
| `PrecompiledJs` | `/_cache_/js/<path>` | 86400s (24h) | `static/js/<path>` disco |

### Fluxo por request — nunca panic, nunca erro

```
RAM (HashMap)  →  hit    →  serve bytes + ETag + Cache-Control
      ↓ miss
disco          →  serve  →  max-age=300 (fotos novas aparecem imediatamente)
      ↓ miss
SVG 1x1 vazio  →  serve  →  nunca retorna 404
```

### ETag — fingerprint eficiente

```rust
logical_path + hash_bd + bytes.len() + bytes[..16]
```

Usa o `hash_original`/`hash_webp` já calculado na BD — sem re-hash dos bytes em cada request.

### Header de debug

```
X-Memory-Image: hit | disk-fallback | empty
X-Memory-Css:   hit | disk-fallback | empty
X-Memory-Js:    hit | disk-fallback | empty
```

Visível nas DevTools do browser — diagnóstico imediato.

### `get_merlin_thumb_map()` — ponte para a Merlin

Exporta só os thumbs (`/thumbs/` no path) como `HashMap<photo_id, {url, width, height}>`.
É assim que a Merlin conhece os paths das imagens sem tocar na BD por request.

### Ciclo de actualização

```
10h (ou 11h)  →  reload automático de tudo
                  (alguns segundos para qualquer restaurante)
12h           →  restaurante abre — dados já frescos em RAM
```

- Foto nova → aparece imediatamente via `disk-fallback`, entra no RAM no próximo reload
- CSS/JS novo → idem
- Dono pode pedir `force_reload()` manualmente a qualquer hora
- Timestamp de controlo em ficheiro no servidor (`should_update_cache / mark_cache_updated`)

### HTML final — tudo via `/_cache_/`

```html
<!-- CSS em RAM -->
<link href="/_cache_/css/bootstrap_merlin.css" rel="stylesheet">
<link href="/_cache_/css/styles.css" rel="stylesheet">
<link href="/_cache_/css/temas/tema_light.css" rel="stylesheet">

<!-- Background gerado pela Merlin com dados do cliente -->
<style>
  body { background-image: url("/_cache_/media/background/marble_jeans.webp"); }
</style>

<!-- Imagens — path resolvido pela Merlin -->
<img src="/_cache_/media/photos/thumbs/beef_steak_400px.webp">

<!-- JS no fim do body -->
<script src="/_cache_/js/bootstrap.bundle.min.js"></script>
```

Zero ficheiros estáticos expostos directamente. Zero I/O por request em condições normais.

### Tags CSS semânticas customizadas

Sistema de tipografia próprio — evita classes Bootstrap aninhadas:

```html
<minimo>    <minimo10>   <minimo20>    <!-- tamanho mínimo de fonte -->
<maximo>                               <!-- tamanho máximo -->
<mais10>    <mais20>                   <!-- aumenta N px relativo -->
<menos10>                              <!-- reduz N px relativo -->
<cursivo>                              <!-- fonte cursiva -->
<small_minimo>                         <!-- small + mínimo -->
```

---

## Funções especiais de template

### `#EXECUTA_TEMPLATE(path)`
Carrega e executa uma sub-template dinamicamente pelo path:
```merlin
{{{{ #EXECUTA_TEMPLATE("views_paginas/pagina_menu_card.html") }}}}

// Path construído dinamicamente a partir da BD:
$pagina_executar = $TEXTO("views_paginas/" ; pagina.tipo ; ".html")
{{{{ #EXECUTA_TEMPLATE(pagina_executar) }}}}
```

### `#EXECUTA_HTML_MERLIN(texto)`
Executa o conteúdo de um field como se fosse uma template Merlin.
Adiciona automaticamente `%%%%\n   %%%%\n` no início para identificar como bloco Merlin.
Permite que conteúdo da BD contenha código Merlin:
```merlin
{{{{ #EXECUTA_HTML_MERLIN(pagina.html_antes | sanitize) }}}}
```

### `$TEXTO("a" ; "b" ; "c")`
Concatenação multi-argumento com `;` — alternativa mais limpa a vários `$+` encadeados.
(`$FORMAT` não foi usado — palavra reservada do Rust, causava aviso mesmo em maiúsculas)
```merlin
$PATH = $TEXTO("views_paginas/" ; pagina.tipo ; ".html")
// equivalente a:
$PATH = "views_paginas/" pagina.tipo $+ ".html" $+
```

### `#IF VAR :: OUTRA_VAR` — output condicional silencioso
Se `OUTRA_VAR` não existe, não deixa nada na pilha — não renderiza nada:
```merlin
{{{{ #IF HREF_FOTO :: FECHA_HREF }}}}    // se FECHA_HREF não existe → zero output
{{{{ HREF_FOTO }}}}                       // abre o <a href="...">
    {{{{ HTML.PICTURE_RENDER_THUMB(foto) }}}}
{{{{ #IF HREF_FOTO :: FECHA_HREF }}}}    // fecha o </a> só se havia link
```

---

## Gerador de páginas — arquitectura declarativa

Páginas completas geradas sem escrever HTML — só configurando registos na BD.
A LIB PAGINA cresce em Merlin puro — sem tocar no Rust.

### Tabelas

**`cliente_pagina`** — define a página: `slug`, `titulo_principal`, `nome_config`, `nome_background_card`, `photo_link_id`

**`cliente_pagina_fotos`** — define os blocos por `ordem`:
```
tipo              →  módulo a chamar ou ficheiro .html
referencia_foto   →  SELECIONA_PRODUTOS, SELECIONA_EVENTO=RANDOM...
frase             →  conteúdo textual ou título de categoria
class_colunas     →  size para COLS_ANALISA_SIZE (xsm, md, lg...)
href_foto         →  URL ao clicar (/SLUG_CLIENTE/cardapio/card/0)
primeiro_item_sequencia / ultimo_item_sequencia  →  abre/fecha <div class="row">
card_silver, card_clean, card_dark, card_light   →  flags de estilo
cartao_texto, cartao_dark, cartao_light          →  flags de cartão
mostra_imagem, mostra_video, formatar_frases     →  flags de conteúdo
```

### Motor — `pagina_especial_padrao.html`

```merlin
#FOR item_pagina in paginas_itens
    #pagina = item_pagina.item
    #foto   = item_pagina.foto

    #MATCH pagina.tipo
        "GOOGLE_MAP"            ::  HTML.GOOGLE_MAP(pagina)
        "HORARIO_DADOS_CLIENTE" ::  HTML.HORARIO_DADOS_CLIENTE
        "FRASE"                 ::  HTML.FRASE(pagina.frase @pagina.formatar_frases)
        default :: {
            tem_passeios?        →  PAGINA.VIEW_PASSEIO
            caminho_fotos_extra? →  grid inline
            senão               →  #EXECUTA_TEMPLATE("views_paginas/" + pagina.tipo + ".html")
        }
    #END_MATCH
#END_FOR
```

### LIB PAGINA — `pagina_html.html`

#### `PAGINA.VIEW_PASSEIO(pag_passeio)`
Dispatcher principal — decide como renderizar cada bloco:

```merlin
item.tipo.starts_with("menu_")  →  PAGINA.MENU_TIPO(item item.tipo)
item.tipo existe                →  #EXECUTA_TEMPLATE("views_paginas/" + tipo + ".html")
senão                           →  pagina_menu_card.html (default)
```

#### `PAGINA.MENU_TIPO(produto tipo)`
Consolida `menu_box.html`, `menu_card.html` e `menu_flex.html` num único módulo.
Prepara os campos e despacha para o módulo HTML correcto:

```merlin
// Adapta campos BD → campos que HTML.MENU_* espera
$produto.preco        = produto.preco_produto
$produto.imagem       = foto.imagem
$produto.imagem_thumb = foto.imagem_thumb
$produto.id           = produto.id_item_selecao

// Despacha pelo tipo
#MATCH TIPO
    "menu_card" ::  HTML.MENU_CARD(produto 1)
    "menu_flex" ::  HTML.MENU_FLEX(produto 1)
    default     ::  HTML.MENU_BOX(produto 1)
#END_MATCH
```

Gere também `primeiro_item_sequencia` / `ultimo_item_sequencia` para abrir/fechar `<div class="row">`.

### Padrão de adaptador — views_paginas/

Ficheiros `.html` que adaptam o registo genérico `item` para os módulos `HTML.*`:

```
views_paginas/pagina_menu_card.html  →  adaptador completo com todos os flags
views_paginas/card_titulo.html       →  adapta item → HTML.CARD_TITULO
```

Novos tipos de bloco = novo módulo `PAGINA.*` em Merlin ou novo ficheiro em `views_paginas/`.
A rota não muda, o motor não muda — só a biblioteca cresce.

### Exemplo — Home Pecado Capital

```
ordem 00   FRASE            "Venha saborear o que tem de melhor..."
ordem 01   card_titulo      produto RANDOM   →  /slug/cardapio/card/0
ordem 01.1 card_titulo      evento RANDOM    →  /slug/evento/menu
ordem 01.3 card_titulo      galeria RANDOM   →  /slug/galeria/menu
ordem 01.7 card_titulo      videos RANDOM    →  /slug/galeria/videos
ordem 01.9 card_titulo      passeio RANDOM   →  /slug/passeio/menu
ordem 02   pagina_menu_card produtos com selecção por categoria
ordem 09   menu_box         todos os produtos
ordem 95   HORARIO_DADOS_CLIENTE
ordem 99   GOOGLE_MAP
```

Zero HTML escrito — a página é uma lista de registos ordenados na BD.

---

## Regras de Sintaxe — as poucas regras rígidas

### Pipeline de templates — espaço obrigatório em torno do `|`

```merlin
// CORRECTO
produtos.titulo | UPPER | REPLACE("LAPIS" "LAPISEIRA") | ATUALIZA

// ERRADO — Merlin não aceita
produtos.titulo|UPPER|REPLACE("LAPIS" "LAPISEIRA")|ATUALIZA
```

`produtos.titulo` num pipeline — Merlin detecta automaticamente que é vetor de dicionários
e aplica `[x]` implicitamente em cada item. Não é preciso escrever `produtos[x].titulo`.

`REPLACE("LAPIS" "LAPISEIRA")` — `produtos[x].titulo` é o primeiro parâmetro implícito,
`x` sendo cada item do vetor. Os parâmetros dentro de `()` podem ser separados por `;`
mas é opcional — como é RPN, o que fica nas pilhas são os parâmetros.

### RPN — espaço como separador universal

Sem parênteses, o espaço é o único separador. Cada operando e cada operador precisam
de estar separados por espaço:

```merlin
// CORRECTO
@TOTAL = PRECO 2 * 5 +

// ERRADO
@TOTAL = PRECO 2*5+
```

### Pilhas sempre vazias no final — filosofia central

Sobrou algo na pilha = erro de programação.
Não há garbage collector porque não há lixo — tudo o que entra na pilha tem destino.
Esta disciplina força código limpo por design.

```
pilha_numeros             → vazia
pilha_strings             → vazia
pilha_dicionarios         → vazia
pilha_vetores_numeros     → vazia
pilha_vetores_strings     → vazia
pilha_vetores_dicionarios → vazia
```

Existe opção de limpar qualquer pilha manualmente, mas raramente é necessária.
**Regra:** se quer usar um valor mais de uma vez, guarde numa variável — ao guardar, sai da pilha.

### Defaults silenciosos — filosofia de erro

Quando um campo não existe, Merlin preenche com o zero do tipo:

```
número      →  0.0
string      →  ""
dicionário  →  {}
```

Isto absorve a maioria dos erros silenciosamente — o frontend recebe um default útil
sem precisar de tratamento explícito na maioria dos casos.

### pilha_logicos — em evolução

Pilha criada para retorno de código de sucesso/erro de operadores.
Direcção prevista: `""` = sucesso, string com descrição = erro
(mais expressivo que `0.0` = sucesso, especialmente para debug e mensagens no frontend).

---

## Filosofia — Explícito é melhor que implícito

A Merlin prefere nomes longos e declarativos a sintaxe concisa e ambígua.

```merlin
// Outras linguagens — implícito, ambíguo
{
    let x = 0;    // será local? depende do contexto...
}

// Merlin — declarativo, sem ambiguidade
&LOCAL_SCOPE
    @X = 0        // claramente local
&END_LOCAL_SCOPE
```

O nome **diz exactamente o que faz**. Quem lê uma template Merlin nunca adivinha.
É a mesma filosofia do `" | "` com espaços obrigatórios — verboso e claro
é sempre melhor que conciso e ambíguo.

---

## Gestão de Escopo — LocalScope

### `&LOCAL_SCOPE` / `&END_LOCAL_SCOPE`
Tudo criado dentro é local automaticamente — sem prefixo necessário:

```merlin
@TOTAL = 0

&LOCAL_SCOPE
    @SUBTOTAL = PRECO 1.1 *    // local — morre no END
    @TEMP = SUBTOTAL 2 *       // local
    @OUTER::TOTAL += SUBTOTAL  // OUTER:: escreve no layer anterior
&END_LOCAL_SCOPE

// SUBTOTAL não existe ✓  TEMP não existe ✓  TOTAL actualizado ✓
```

### `&LOCAL_NAMED_SCOPE("NOME")` / `&END_LOCAL_NAMED_SCOPE`
Igual ao `LOCAL_SCOPE` mas com nome — permite que scopes filhos escrevam
directamente neste scope pelo nome:

```merlin
&LOCAL_NAMED_SCOPE("CALCULOS")
    @BASE = 100

    &LOCAL_SCOPE
        @RESULTADO = BASE 1.1 *
        @CALCULOS::BASE = RESULTADO   // escreve no scope CALCULOS pelo nome
    &END_LOCAL_SCOPE

    // BASE foi actualizado ✓
&END_LOCAL_NAMED_SCOPE
```

O nome pode ser qualquer expressão Merlin:
```merlin
&LOCAL_NAMED_SCOPE("CALCULOS")        // string literal
&LOCAL_NAMED_SCOPE(NOME_VARIAVEL)     // variável string
&LOCAL_NAMED_SCOPE(PRODUTO.SCOPE)     // field de dicionário
&LOCAL_NAMED_SCOPE("SCOPE_" IX $+)    // expressão calculada
```

### `&LOCAL_EXPLICIT_SCOPE("NOME")` / `&END_LOCAL_EXPLICIT_SCOPE`
Nome obrigatório. Só fica local o que usar `NOME::`.
Sem prefixo, vai para o layer anterior naturalmente:

```merlin
&LOCAL_EXPLICIT_SCOPE("LOOP")
    @LOOP::CONTADOR = 0    // local explícito — usa o nome do scope
    @TOTAL += CONTADOR     // sem prefixo → vai para layer anterior
    @OUTER::TOTAL += 1     // explícito — mesmo resultado
&END_LOCAL_EXPLICIT_SCOPE
```

### Implementação — operadores

```rust
"<LOCAL_SCOPE>" => {
    self.push_local_scope();                          // explicit = false, sem nome
}
"<LOCAL_NAMED_SCOPE>" => {
    self.calculadora(params);                         // resolve o nome
    let nome = self.pilha_strings.apaga_ultima_pilha();
    self.push_local_named_scope(&nome);               // explicit = false, com nome
}
"<LOCAL_EXPLICIT_SCOPE>" => {
    self.calculadora(params);
    let nome = self.pilha_strings.apaga_ultima_pilha();
    self.push_local_explicit_scope(&nome);            // explicit = true, nome obrigatório
}
"<END_LOCAL_SCOPE>" | "<END_LOCAL_NAMED_SCOPE>" | "<END_LOCAL_EXPLICIT_SCOPE>" => {
    self.pop_local_scope();
}
```

### `#FOR` — LocalScope automático

Todo `#FOR` abre automaticamente um `LOCAL_EXPLICIT_SCOPE` com nome `FOR_SCOPE_N`
(onde N = `self.locais.len()` antes do push):

```rust
let nome_scope = format!("FOR_SCOPE_{}", self.locais.len());
self.push_local_explicit_scope(&nome_scope);
// runtime cria FOR_SCOPE_N::IDX e FOR_SCOPE_N::ITEM automaticamente
```

O programador não vê nada — é invisível:

```merlin
@TOTAL = 0
#FOR IDX, PRODUTO in PRODUTOS
    @TEMP = @PRODUTO.PRECO 1.1 *   // local ao FOR — morre na próxima iteração
    @TOTAL += TEMP                  // vai para layer anterior — correcto
#END_FOR
// TEMP não existe ✓  TOTAL actualizado ✓
```

### Sistema completo de prefixos de escopo

```merlin
// Escrita:
@NOME::VAR   = 0    // escreve no scope com esse nome (qualquer nível acima)
@OUTER::VAR  = 0    // escreve no layer imediatamente anterior
@MERLIN::VAR = 0    // escreve directamente em Merlin local (ignora todos os layers)
@GLOBAL::VAR = 0    // imutável — erro silencioso

// Mesmo padrão para strings e dicionários:
$CALCULOS::NOME = "teste"
#OUTER::ITEM    = { "CAMPO" : "valor" }

// Pipeline também suporta:
CATALOGO.NOME | UPPER | CALCULOS::ATUALIZA
CATALOGO.NOME | UPPER | MERLIN::ATUALIZA
CATALOGO.NOME | UPPER | OUTER::ATUALIZA
```

### Lógica de resolução — função `resolve_scope_index`

```rust
// Retorna (nome_limpo, Option<ix_do_scope>)
fn resolve_scope_index(&self, nome_var: &str) -> (&str, Option<usize>)

// Ordem de resolução:
OUTER::      →  começa no penúltimo scope, procura para trás
NOME::       →  procura scope com esse nome em qualquer nível
sem prefixo  →  procura onde existe; se explicit_only salta; cria no primeiro normal
None         →  escreve em Merlin local
```

### Leitura de variáveis — sempre percorre toda a cadeia

```
FOR_SCOPE_N (explicit) → LOCAL_SCOPE → NAMED_SCOPE → MERLIN → SITE → CORE
```

A leitura não é afectada pelos prefixos — percorre sempre do mais recente para o mais antigo.

---

## Novos Operadores de Vetor

### `#DIC_VET_ZIP(VET_A VET_B)`
Combina dois vetores de dicionários em paralelo — campo a campo.
O primeiro parâmetro tem prioridade em campos com o mesmo nome:

```merlin
#PRODUTOS_COM_FOTOS[] = #DIC_VET_ZIP(PRODUTOS.ITEM PRODUTOS.FOTO)

// Acesso normal depois:
PRODUTOS_COM_FOTOS[0].TITULO      // vem de ITEM
PRODUTOS_COM_FOTOS[0].IMAGEM      // vem de FOTO
```

Equivalente manual (mais verboso):
```merlin
#PRODUTOS_COM_FOTOS[] = PRODUTOS.ITEM
#DIC_VET_JOIN("PRODUTOS_COM_FOTOS" "PRODUTOS.FOTO")
```

## Novos Operadores de Vetor

### `#DIC_VET_ZIP(VET_A VET_B)`
Combina dois vetores de dicionários em paralelo — campo a campo.
O primeiro parâmetro tem prioridade em campos com o mesmo nome:

```merlin
#PRODUTOS_COM_FOTOS[] = #DIC_VET_ZIP(PRODUTOS.ITEM PRODUTOS.FOTO)

// Acesso normal depois:
PRODUTOS_COM_FOTOS[0].TITULO      // vem de ITEM
PRODUTOS_COM_FOTOS[0].IMAGEM      // vem de FOTO
```

Equivalente manual (mais verboso):
```merlin
#PRODUTOS_COM_FOTOS[] = PRODUTOS.ITEM
#DIC_VET_JOIN("PRODUTOS_COM_FOTOS" "PRODUTOS.FOTO")
```

### `#FOR LINHA in VETOR BY N` — chunks de N itens
Itera o vetor em grupos de N. `LINHA` é um vetor temporário com até N itens.
`BY` aceita qualquer expressão Merlin — número literal, variável ou expressão:

```merlin
#FOR LINHA in FOTOS BY 3
    <div class="row">
        #FOR REG in LINHA
            %%%%
                #FOTO = REG.FOTO
                $THUMB = #IF FOTO.IMAGEM_THUMB :: "/_cache_/media/" FOTO.IMAGEM_THUMB $+ #ELSE "/_cache_/media/" FOTO.IMAGEM $+
            %%%%
            <div class="col-4">
                <img src="{{{{ THUMB }}}}" alt="{{{{ FOTO.TITULO }}}}">
            </div>
        #END_FOR
    </div>
#END_FOR

// BY aceita qualquer expressão Merlin:
#FOR LINHA in FOTOS BY N_COLUNAS
#FOR LINHA in FOTOS BY @CONFIG_GLOBAL.COLUNAS 1 +
```

Usa `LOCAL_EXPLICITY_SCOPE` automaticamente — `LINHA` é local ao loop.

---

## Pipeline de Dicionários — Operadores avançados

Todos os operadores abaixo funcionam em pipeline com `|` ou com `.`:

```merlin
// Com | (pipe template)
PRODUTOS | FILTER(ITEM.ESTOQUE 0 >?) | ADD_FIELD(@TOTAL = @PRECO @ESTOQUE *) | SORT_BY("@TOTAL") | ATUALIZA

// Com . (encadeamento)
PRODUTOS.FILTER(ITEM.ESTOQUE 0 >?).ADD_FIELD(@TOTAL = @PRECO @ESTOQUE *).SORT_BY("@TOTAL").ATUALIZA()
```

### `@FIELD` e `$FIELD` no contexto do pipeline

Dentro de expressões dos operadores do pipeline, `@FIELD` e `$FIELD` referem-se
aos campos do item actual — sem precisar de escrever o nome do vetor:

```merlin
// Equivalentes:
PRODUTOS | FILTER("@PRECO 10 >?")
PRODUTOS | FILTER(ITEM.PRECO 10 >?)    // ITEM disponível automaticamente

// Expansão automática:
@PRECO  →  @PRODUTOS[ix].PRECO
$TITULO →  PRODUTOS[ix].TITULO
```

Quando a expressão começa com `"` (string literal) — expande `@FIELD`/`$FIELD` automaticamente.
Quando é expressão directa — `ITEM` e `IX` estão disponíveis como variáveis locais.

### `FILTER` / `FILTRA`
Filtra o vetor imediatamente — descarta itens que não passam na condição.
Diferente de `WHERE` que guarda a condição para os operadores seguintes.

```merlin
// String literal — expande @FIELD automaticamente
PRODUTOS | FILTER("@PRECO 10 >?")
PRODUTOS | FILTER("@CATEGORIA_ID 2 ==")
PRODUTOS | FILTER("$TITULO \"Bacalhau\" $!=")

// Expressão directa — ITEM e IX disponíveis
PRODUTOS | FILTER(ITEM.PRECO 10 >?)
PRODUTOS | FILTER(ITEM.CATEGORIA_ID 2 ==)
PRODUTOS | FILTER(IX 5 <?)             // só os primeiros 5

// Expressão complexa — qualquer Merlin
PRODUTOS | FILTER(
    @PRECO 10 >?
    @ESTOQUE 0 >?
    .AND.
)

// MERLIN:: persiste fora do FILTER
PRODUTOS | FILTER(
    @MERLIN::CONTADOR += 1
    @PRECO 10 >?
)
```

Cada iteração abre `LOCAL_SCOPE` automático — variáveis criadas dentro morrem no fim da iteração.

### `WHERE`
Guarda a condição para os operadores seguintes — não filtra o vetor imediatamente.

```merlin
// WHERE aplica-se ao EXECUTA seguinte
PRODUTOS | WHERE("@CATEGORIA_ID 2 ==") | EXECUTA(@DIC_VET_SORT("PRECO"))

// FILTER vs WHERE
PRODUTOS | FILTER("@ESTOQUE 0 >?") | SORT_BY("@PRECO")   // filtra primeiro, ordena os filtrados
PRODUTOS | WHERE("@ESTOQUE 0 >?") | EXECUTA(...)          // condição para o EXECUTA
```

### `ADD_FIELD` / `NOVO_FIELD`
Cria ou actualiza um field calculado em cada item do vetor.
O LHS determina o nome e tipo do field (`@` = numérico, `$` = string):

```merlin
// Cria field TOTAL = PRECO × ESTOQUE
PRODUTOS | ADD_FIELD(@TOTAL = @PRECO @ESTOQUE *)

// Cria field string
PRODUTOS | ADD_FIELD($LABEL = $TITULO " - " $+ $RODAPE $+)

// Field com valor fixo
PRODUTOS | ADD_FIELD(@ESTOQUE = 10)

// Pipeline completo
#RESUMO[] = PRODUTOS
    | FILTER("@ESTOQUE 0 >?")
    | ADD_FIELD(@TOTAL = @PRECO @ESTOQUE *)
    | SORT_BY("@TOTAL")
```

Quando usado sem chave (`PRODUTOS | ADD_FIELD(...)`) — a Merlin descobre
a chave automaticamente a partir do LHS da expressão.

### `SORT_BY` / `ORDENA_POR`
Ordena o vetor por field ou expressão. Aceita variável ou literal:

```merlin
// Por field numérico
PRODUTOS | SORT_BY("@PRECO")
PRODUTOS | SORT_BY("@TOTAL")

// Por field string
PRODUTOS | SORT_BY("$TITULO")

// Por expressão calculada
PRODUTOS | SORT_BY("@PRECO @ESTOQUE *")

// Com variável
$ORDEM = "@PRECO"
PRODUTOS | SORT_BY(ORDEM)
```

### `SORT_DESC` / `SORT_BY_DESC` — ordem descendente

```merlin
PRODUTOS | SORT_DESC("@PRECO")        // descendente por PRECO
PRODUTOS | SORT_BY_DESC("@TOTAL")     // descendente por TOTAL calculado
PRODUTOS | SORT_BY_DESC("$TITULO")    // descendente alfabético
```

Implementado com flag `desc: bool` na função `sort_vetor_dicionario_por_chave` —
melhoria futura (locale, case_insensitive, null_last) aplica-se automaticamente
às versões ascendente e descendente.

### `EXECUTA`
Passa o vetor inteiro para a pilha e executa o programa.
Aplica `WHERE` se definido antes:

```merlin
// Ordena usando operador existente
#PROD[] = $PRODUTOS.TITULO | EXECUTA(@DIC_VET_SORT("PRECO"))

// Com WHERE
$PRODUTOS.TITULO | WHERE("@CATEGORIA_ID 2 ==") | EXECUTA(@DIC_VET_SORT("PRECO"))
```

### `ATUALIZA` com GLOBAL
Quando a variável é GLOBAL (carregada às 10h), o `ATUALIZA` cria
automaticamente uma cópia em local — sem precisar de criar manualmente:

```merlin
// PRODUTOS é GLOBAL — ATUALIZA cria cópia local automaticamente
PRODUTOS | ADD_FIELD(@ESTOQUE = 10) | ATUALIZA
// Agora PRODUTOS local existe com ESTOQUE, GLOBAL intacta

// Sem ATUALIZA — atribuição explícita
#PRODUTOS[] = PRODUTOS | ADD_FIELD(@ESTOQUE = 10)  // mesmo efeito
```

### Funções auxiliares de expansão

```rust
// Expande @FIELD/$FIELD para acesso indexado no vetor
expandir_fields_vetor_dicionario("@PRECO 10 >?", "PRODUTOS", 3)
// → "@PRODUTOS[3].PRECO 10 >?"

// Expande @FIELD/$FIELD para acesso no dicionário escalar
expandir_fields_dicionario("@PRECO @ESTOQUE *", "ITEM")
// → "@ITEM.PRECO @ITEM.ESTOQUE *"
```

Ambas sem Regex — percorrem char a char, consistente com o parser da Merlin.

---

## Promoção automática de operadores

Quando o contexto é um vetor de dicionários, a Merlin promove automaticamente
operadores escalares para vectoriais — sem código especial, sem configuração:

```merlin
// Escalar (não faz sentido num vetor)
@TOTAL = @VALOR.SUM()

// Vetor de dicionários — promove automaticamente
@TOTAL  = @PRODUTOS.PRECO.SUM()    // SUM    → VET_SUM
@MEDIA  = @PRODUTOS.PRECO.AVG()    // AVG    → VET_AVG
@MAX    = @PRODUTOS.PRECO.MAX()    // MAX    → VET_MAX
@MIN    = @PRODUTOS.PRECO.MIN()    // MIN    → VET_MIN
@QTD   = @PRODUTOS.PRECO.LEN()    // LEN    → VET_LEN
```

### Cadeia de lookup para promoção

```
operador = "SUM", is_vetor = true

Escalar tenta (em ordem):
1. <SUM>         →  operador de dicionário?  não
2. <DIC_SUM>     →  operador de dicionário prefixado?  não
3. <SUM>         →  operador numérico?  não
4. <SUM>         →  operador de data?  não
5. busca_modulo  →  módulo Merlin?  não

Vetor tenta adicionalmente:
1. <DIC_VET_SUM> →  operador de dicionário vectorial?  não
2. <VET_SUM>     →  operador numérico vectorial?  SIM! ✓
    ↓
executa VET_SUM sobre o vetor de PRECOs
retorna total em pilha_numeros
```

O programador escreve a intenção (`SUM`), a Merlin encontra
a implementação correcta (`VET_SUM`) pelo contexto. Invisível e natural.

---

## Sistema de Libraries — Comunidade

### Arquitectura de libs

Módulos organizados em libraries — ficheiros `.html` editáveis no VS Code
ou guardados na BD para partilha:

```sql
-- Tabela libs
lib: "FINANCE"      description: "Cálculos financeiros"
lib: "STATS"        description: "Estatística"
lib: "HTML"         description: "Componentes HTML"
lib: "RESTAURANTE"  description: "Módulos para restaurantes"

-- Tabela modules
name_exp: "FINANCE.TABELA_PRICE"   lib_id: FINANCE
name_exp: "FINANCE.TIR"            lib_id: FINANCE
name_exp: "FINANCE.VPL"            lib_id: FINANCE
name_exp: "STATS.MEDIA"            lib_id: STATS
name_exp: "HTML.GALLERY_HOVER"     lib_id: HTML
```

### Carregamento de libs na rota

```rust
// Rota minimalista — zero lógica de negócio
let mut calculator = setup_merlin_from_globals(state).await;
load_modules(&mut calculator, db, lib_id_finance).await;  // carrega FINANCE da BD
load_modules(&mut calculator, db, lib_id_stats).await;    // carrega STATS da BD
processar_base_template("finance/tabela_price.html", &mut calculator, ...).await
```

### Exemplo real — Tabela Price sem backend

```merlin
%%%%
    // Toda a lógica na template — zero Rust
    @VALOR_CASA   = 300000.0
    @TAXA_MENSAL  = 0.75
    @PRAZO_MESES  = 360

    FINANCE.TABELA_PRICE(VALOR_CASA TAXA_MENSAL PRAZO_MESES)
    FINANCE.PRICE_RESUMO
    FINANCE.PRICE_ANALISE_PARCELA(60)
    FINANCE.PRICE_COMPARATIVO(0.65)
    FINANCE.PRICE_SIMULACAO_ANTECIPACAO(50000)
%%%%

{{{{ RESUMO_HTML }}}}
{{{{ TABELA_HTML }}}}
```

360 iterações, 4 vectores paralelos, HTML gerado — tudo na Merlin.
Editável sem recompilar. Disponível para qualquer restaurante com `load_modules`.

### Flags de módulo

```rust
pub struct Module {
    pub name_exp: String,       // nome do módulo
    pub expression: String,     // código Merlin
    pub is_private: bool,       // privado vs comunidade
    pub is_published: bool,     // publicado vs em desenvolvimento
    pub exp_evaluate: bool,     // avalia na carga ou guarda como texto
    pub save_with_lib_name: bool, // "FINANCE.TABELA_PRICE" vs "TABELA_PRICE"
}
```

### Herança de módulos entre contextos

```merlin
// CORE — versão base genérica
&CREATE_MODULE("HTML.MENU_BOX") ::
    // implementação padrão para todos os restaurantes
&END_MODULE()

// CLIENTE Pecado Capital — sobrepõe o CORE
&CREATE_MODULE("HTML.MENU_BOX") ::
    // chama a versão CORE primeiro
    GLOBAL::CORE::HTML.MENU_BOX()
    // adiciona badge de prato especial do chef
    #IF PRODUTO.ESPECIAL_CHEF :: {{{{ HTML.BADGE_CHEF }}}}
&END_MODULE()
```

Herança sem classes, sem `super()` — só `GLOBAL::CORE::MODULO()`.
A hierarquia `CLIENTE → SITE → CORE` já é a cadeia de herança.

---

## Operadores Vectoriais — [x] e {x}

`[x]` — aplica a operação a **todos os elementos** do vetor:

```merlin
@VENDAS_ANO[x] = VENDAS_ANO[x] 1.2 *        // aumenta 20% em cada elemento
@VENDAS_ANO[x] = VENDAS_ANO[x] 20 %+        // aplica 20% e soma
@VENDAS_ANO[x] = VENDAS_ANO[x] 10 %-        // desconto de 10%
```

`{x}` — acede ao **índice corrente** para indexar um vetor paralelo:

```merlin
#PRODUTOS[x].IMAGEM_THUMB = FOTOS[{x}].IMAGEM_THUMB   // zip entre dois vetores
```

### Formas equivalentes

```merlin
// As três formas aumentam 20% em todos os elementos:
@VENDAS_ANO[x] = VENDAS_ANO[x] 1.2 *
VENDAS_ANO.EXECUTA(1.2 *).ATUALIZA()

// Só quer o total sem actualizar o vetor:
@TOTAL = @VET_SUM(VENDAS_ANO 1.2 *)
// Merlin sabe que VENDAS_ANO é vetor → multiplica cada elemento → deixa vetor na pilha
// VET_SUM soma tudo → resultado em pilha_numeros → atribuído a @TOTAL
```

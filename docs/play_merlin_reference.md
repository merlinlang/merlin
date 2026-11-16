# Play Merlin — Referência da Plataforma

**Versão inicial de documentação · 16 de novembro de 2026**  
**Estado:** baseada nos exemplos, explicações e trechos de código compartilhados pelo criador. Algumas regras de integração ainda exigem confirmação no código completo.

> **Princípio arquitetural:** a linguagem Merlin não conhece a Play Merlin. A Play Merlin é uma aplicação web, construída com Rust/Rocket, que utiliza Merlin nas próprias templates. A infraestrutura da aplicação fornece contexto, rotas, carregamento de recursos e persistência; a linguagem interpreta os programas.

## 1. Escopo deste documento

Este guia documenta **a aplicação Play Merlin**, não a sintaxe fundamental da linguagem. Para operadores RPN, pilhas tipadas, estruturas de controlo, módulos e escopos, consultar `merlin_reference.md`.

| `merlin_reference.md` | `play_reference.md` |
|---|---|
| Pilhas, operadores, pipelines, `#FOR`, `#IF`, módulos, escopos, delimitadores | Rotas, parâmetros HTTP, carregamento de templates, contexto de cliente, formulários, ADMIN, editor, APIs e gravação |
| Comportamento do interpretador Merlin | Comportamento da aplicação Play Merlin, que utiliza o interpretador |

**Importante:** a aplicação Play Merlin pode disponibilizar variáveis e funções aos programas Merlin, mas isso **não torna essas funcionalidades parte intrínseca da linguagem**.

## 2. Visão da arquitetura

```text
Browser / HTTP
       |
       v
Play Merlin (Rust + Rocket)
  |- identifica cliente pelo slug
  |- resolve rota e recurso
  |- prepara contexto da requisição
  |- fornece ficheiros, dados, utilizador e módulos
  |- cria uma instância do interpretador Merlin
  |                |
  |                v
  |         Linguagem Merlin
  |         interpreta a template
  |         executa módulos e pipelines
  |         produz conteúdo
  |                |
  +<---------------+
       |
       v
Resposta HTML ou JSON
```

A Play Merlin é **um site que utiliza a linguagem Merlin nas suas templates**. Não há uma dependência conceitual inversa: Merlin não precisa conhecer o nome, as rotas nem o ADMIN da Play Merlin para funcionar.

### Por que isso importa

- As aplicações podem acrescentar comportamento combinando recursos já existentes do interpretador.
- Um projeto pode definir a sua interface e lógica de negócio nas templates Merlin, sem uma rota Rocket exclusiva para cada página.
- A Play Merlin pode evoluir como produto sem exigir que o núcleo da linguagem passe a conhecer conceitos específicos de administração web.
- Recursos como editor visual, API HTTP e gestão de formulários são **serviços da plataforma**, não requisitos da linguagem.

## 3. Rota genérica GET

A implementação apresentada inclui:

```rust
#[get("/<slug_cliente>/<path..>", rank = 15)]
pub async fn cliente_template_generica(...) -> RawHtml<String>
```

A rota cria uma instância Merlin para o cliente e extrai os segmentos do caminho, disponibilizando até seis parâmetros, `PARAM0` a `PARAM5`.

Exemplos de URLs:

```text
/merlin-dinner/background/list
/merlin-dinner/background/update/2
/merlin-dinner/xiaomi
/finance/finance_home
```

### 3.1. Tentativas de resolução de template

Para uma URL com segmentos, a rota tenta variantes de nome em sequência. Entre os candidatos observados:

```text
{slug}_{param0}.html
{param0}_{param1}_{param2}.html
{param1}_{param0}.html
{param0}_{param1}.html
{param0}.html
{param1}.html
```

Há também variantes com hífen convertido em `_` e modelos genéricos de fallback:

```text
view_model_list.html
view_model_update.html
```

A ordem efetiva importa: o primeiro candidato resolvido é usado. Existe tratamento especial dos parâmetros quando a primeira alternativa é encontrada.

### 3.2. Hierarquia de ficheiros e aliases

A função `resolver_template` tenta o nome diretamente e depois consulta aliases em:

```text
data/{slug}/lista_templates.json
data/lista_templates.json
```

A procura física considera estas camadas, por ordem:

```text
templates/merlin/{slug}/templates/{nome}
templates/merlin/_{categoria}/templates/{nome}
templates/merlin/{nome}
templates/{nome}
```

Isso permite personalização por cliente, partilha por categoria e recursos globais. Os detalhes de permissão pertencem à aplicação e devem ser verificados no servidor.

### 3.3. Templates virtuais

**Comportamento confirmado pelo criador da linguagem:** se a template física não existir e houver uma variável string com o nome que a rota procura (por exemplo `fabricantes_list`), o conteúdo dessa variável pode servir de código-fonte da template.

A string pode ter sido carregada a partir de um campo de registo JSON. Assim, uma template é **texto executável**, independentemente de ter vindo de um `.html` ou de dados.

**Nota de implementação:** o trecho de `resolver_template` apresentado mostra a resolução de ficheiros e aliases. A etapa concreta de resolução de strings como templates não está explícita nesse trecho; documenta-se aqui o comportamento descrito pelo criador, sem inventar onde a implementação está localizada.

## 4. Contexto disponível nas templates

As rotas disponibilizam uma instância Merlin com variáveis provenientes da aplicação. Entre as observadas:

| Variável | Uso |
|---|---|
| `CLIENTE` | Dados do cliente/site, incluindo `CLIENTE.SLUG` |
| `CURRENT_USER` | Informação do utilizador atual |
| `USER_IS_OWNER` | Indica se o utilizador corresponde ao criador do site |
| `WAS_LOGGED_IN`, `IS_LOGGED_IN` | Estado de autenticação |
| `PARAM0` … `PARAM5` | Segmentos da URL |
| `ROTA_RENDER` | Caminho para enviar o formulário de renderização |
| `TEMPLATE_CONTEUDO` | Código-fonte disponibilizado ao editor, escapado para HTML |
| `PATH_TEMPLATE` | Caminho/nome da template resolvida |
| `VISITOR_ID` | Identificador de visitante, obtido/criado com cookie |
| `MESSAGES` / `HTML.MESSAGES_VIEW` | Mensagens da interface, conforme o contexto e módulos carregados |

**Atenção:** a lista exata de variáveis depende da rota. `ID`, `CREATION`, `NOME_MODEL` e outras variáveis de conveniência foram demonstradas em templates da aplicação, mas a origem e as regras precisas de cada uma devem ser verificadas no fluxo correspondente.

## 5. GET e POST na mesma template

Padrão recomendado pelo criador da Play Merlin: **concentrar apresentação e processamento numa mesma template**, quando isso torna a aplicação mais simples.

No exemplo real `destaques_update.html`:

```merlin
%%%%
    $PATH       = $$("${cliente.slug}/${nome_model}")
    $PATH_MODEL = $$("data/${cliente.slug}/${nome_model}.json")

    #IF FORM .AND. PARAM2 "json" $$==
        #DESTAQUE_POST = JSON.STRING_TO_DIC(FORM.JSON)
        $ACAO = FORM.ACAO

        #MATCH ACAO
            "delete" :: JSON.DELETE_BY_ID(PATH_MODEL, ID)
            default :: {
                #IF ID 0 >
                    JSON.UPDATE_BY_ID(PATH_MODEL, ID, DESTAQUE_POST)
                #ELSE
                    JSON.CREATE(PATH_MODEL, DESTAQUE_POST)
                #END_IF
            }
        #END_MATCH

        $REDIRECT_TO = $$("/${PATH}/list")
    #ELSE
        #DESTAQUE = #IF ID 0 > :: JSON.FIND_BY_ID(PATH_MODEL, ID) #ELSE {}
    #END_IF
%%%%

{{{{ #IF REDIRECT_TO :: HTML.REDIRECT(REDIRECT_TO) }}}}
```

O mesmo recurso:

- Em GET, carrega um registo existente ou prepara um formulário novo.
- Em POST, lê `FORM`, converte o JSON em dicionário e seleciona `CREATE`, `UPDATE` ou `DELETE`.
- Após a operação, prepara o redirecionamento para a listagem.

O código Merlin controla o CRUD; a rota genérica fornece a infraestrutura HTTP.

## 6. Formulários sem `struct Form` específica para cada modelo

Um formulário pode ser escrito em HTML comum dentro de uma template Merlin. O desenvolvedor determina os campos, os nomes, os elementos visuais e o processamento.

### 6.1. Exemplo: enviar um registo JSON

O exemplo de Destaques usa campos ocultos `acao` e `json`. JavaScript lê os elementos da página, constrói um objeto com os valores e envia:

```javascript
const dados = montarDestaque();
document.getElementById('acao').value = 'save';
document.getElementById('json').value = JSON.stringify(dados);
document.getElementById('form-json-update').submit();
```

Na template Merlin:

```merlin
#DADOS_POST = JSON.STRING_TO_DIC(FORM.JSON)
$ACAO = FORM.ACAO
```

Desse modo, a definição do modelo pode evoluir sem criar uma nova `struct` Rust a cada campo.

### 6.2. Outro formato da rota de renderização

A rota de renderização inline mostrada recebe uma estrutura genérica:

```rust
pub struct RenderForm {
    pub conteudo: Option<String>,
    pub modulos: Vec<ModuloEditado>,
    pub campos: Vec<CampoForm>,
}

pub struct CampoForm {
    pub nome: String,
    pub valor: String,
}
```

Ela coloca os pares de `campos` num dicionário `FORM`. Para um campo começando com `[` ou `{`, também tenta criar um vetor de dicionários ou dicionário Merlin, respetivamente.

**Ponto para confirmar na implementação completa:** o exemplo `destaques_update.html` usa `name="acao"` e `name="json"`, enquanto a estrutura `RenderForm` recebe `campos[i][nome]` e `campos[i][valor]`. Existe possivelmente outra rota ou etapa de adaptação. Não presumir que essas duas codificações sejam intercambiáveis sem teste.

## 7. O ADMIN: catálogo de recursos administráveis

O ADMIN é um gerenciador de recursos que utiliza um JSON por site (o criador referiu `admin_cliente.json`) como catálogo. Não é necessário gerar uma rota Rust dedicada para cada recurso.

Exemplo reduzido de uma entrada:

```json
{
  "ID": "1",
  "IS_ACTIVE": "1",
  "NOME_ARQUIVO": "produto",
  "VIEW_LIST": "/SLUG_CLIENTE/produto/list",
  "VIEW_UPDATE": "/SLUG_CLIENTE/produto/update",
  "DESCRICAO": "Arquivo de Produtos / Pratos",
  "TIPO_ARQUIVO": "admin",
  "TEM_ID": "1"
}
```

Fluxo:

```text
admin_cliente.json
       |
       v
ADMIN /admin/list  — cadastra e mantém os recursos
       |
       v
ADMIN /admin/gerenciar  — seleciona o recurso
       |
       v
VIEW_LIST  ->  {modelo}_list.html  ->  {modelo}_update.html
       |
       v
GET/POST da rota genérica + operações Merlin/JSON
```

O ADMIN também administra seu próprio catálogo, porque o recurso `admin_cliente` pode constar dele. Cada cliente/site mantém suas próprias definições.

As templates de listagem e edição são **convenções e escolhas do desenvolvedor**, não uma exigência de que todas as interfaces sejam visualmente idênticas.

## 8. Templates como scripts

Uma template Merlin pode executar processamento sem ser usada como uma página convencional.

Exemplo de migração de JSON Django para um formato próprio:

```merlin
%%%%
    #REGS[] = JSON.FIND_ALL("data/SLUG/background_django.json")
    #NOVOS[] = #[]

    #FOR ITEM in REGS
        #NOVOS[+] = {
            "NOME_BACKGROUND" : ITEM.PK,
            "DESCRICAO"       : ITEM.PK,
            "CSS_ADICIONAL"   : ITEM.FIELDS.CSS_ADICIONAL,
            "SHADOW_COLOR"    : ITEM.FIELDS.SHADOW_COLOR,
            "IMAGEM"          : ITEM.FIELDS.IMAGEM
        }
    #END_FOR

    JSON.WRITE("data/SLUG/background.json", NOVOS)
%%%%

Migração concluída — {{{{ NOVOS.LEN() }}}} registos
```

A execução pode ser iniciada por URL, desde que o recurso exista e a aplicação permita o acesso. A mesma infraestrutura é utilizada para scripts de manutenção, páginas e transformações de dados.

### 8.1. Scripts parametrizados pela URL

No exemplo compartilhado, `PARAM1` escolhe um modelo e `PARAM2` o campo a renomear:

```merlin
%%%%
    $MODEL    = PARAM1 | LOWER
    $KEY_NAME = PARAM2 | LOWER
    #REGS[]   = JSON.FIND_ALL(JSON_PATH.$$MODEL)

    REGS |
        ADD_FIELD(@ID = IDX 1 +) |
        ADD_FIELD($TITLE = ITEM.$$KEY_NAME) |
        DELETE_FIELD(KEY_NAME) |
        ATUALIZE

    JSON.WRITE(JSON_PATH.$$MODEL, REGS)
%%%%

Migração concluída — {{{{ REGS.LEN() }}}} registos
```

Esse padrão reutiliza uma template para vários ficheiros. A validação de nomes de ficheiros, permissões e backups deve ocorrer antes de permitir operações destrutivas em produção.

## 9. Execução de sub-templates

Merlin dispõe de `#EXECUTA_TEMPLATE(path)`, que permite executar uma sub-template.

```merlin
{{%% #EXECUTA_TEMPLATE("scripts/corrige_background.html") %%}}
```

`{{%% ... %%}}` não imprime a expressão como HTML. A execução, entretanto, pode ter efeitos sobre dados.

```merlin
{{{{ #EXECUTA_TEMPLATE("componentes/card.html") }}}}
```

`{{{{ ... }}}}` participa da produção de conteúdo HTML. **A renderização final da página acontece após a template ser produzida**, não imediatamente na chamada.

**Escopos:** conforme esclarecido pelo criador, variáveis comuns criadas em sub-templates executadas via `#EXECUTA_TEMPLATE` podem permanecer acessíveis à template chamadora; os módulos/funções usam escopos próprios e só escrevem nos escopos anteriores quando fazem referência explícita ao destino.

## 10. Editor embutido: `HTML.RENDER_FORM`

`HTML.RENDER_FORM` é um **módulo Merlin**, e não uma nova função Rust. Ele monta a interface do editor com HTML e JavaScript.

O módulo apresentado inclui:

- Barra de ferramentas de desenvolvimento.
- Editor de código da template.
- Editor de vários módulos referidos pela página.
- Estado temporário no navegador (`modulosEditados`).
- Botão **Renderizar** que envia template e módulos alterados à rota genérica POST.
- Botões de gravação que chamam APIs distintas.

### 10.1. Renderização em memória

A rota mostrada é:

```rust
#[post("/<slug_cliente>/<path..>", data = "<form>", rank = 20)]
pub async fn cliente_render_inline(...) -> RawHtml<String>
```

Recebe `conteudo`, `modulos` e `campos`; cria uma instância Merlin; injeta módulos temporários com `entra_modulo_owned`; executa o conteúdo enviado; devolve HTML. **Não grava ficheiros na operação de renderização em si.**

Isto permite testar alterações conjuntas na template e nos módulos antes de decidir salvá-las.

### 10.2. Edição sem perder alterações da sessão

Ao aplicar uma alteração a um módulo, o JavaScript guarda-a no objeto `modulosEditados`. Ao reabrir o módulo, usa primeiro a versão local; se não existir, consulta a API de leitura.

### 10.3. Gravação separada

Endpoints apresentados:

```text
GET  /{slug}/api/template?name={nome}
GET  /{slug}/api/modulo?name={nome}
POST /{slug}/api/template/salvar
POST /{slug}/api/modulo/salvar
```

Os endpoints POST de gravação apresentados verificam no Rust se o utilizador é o criador do cliente. A gravação do módulo também atualiza o contexto Merlin do cliente na memória da aplicação.

O nome `HTML.RENDER_FORM` pertence ao **módulo do editor** e não deve ser confundido com o dicionário `FORM` utilizado para processar formulários comuns.

## 11. Bibliotecas e personalizações por cliente

Segundo o criador, a hierarquia de recursos inclui `GLOBAL::CORE`, `GLOBAL::SITE` e `GLOBAL::CLIENTE`. **As variáveis globais são imutáveis.**

Um módulo compartilhado pode ter uma versão própria do cliente: guardar a personalização numa biblioteca do cliente não altera o original nem outros sites.

O editor apresentado usa inicialmente:

```text
nome_lib=cliente.html
```

O Rust grava sob:

```text
templates/merlin/{slug}/libs/{nome_lib}
```

A evolução proposta é permitir escolher a biblioteca de destino, mantendo `cliente.html` como valor padrão. Essa escolha é **uma proposta de melhoria**, não uma funcionalidade já demonstrada da interface.

O servidor deve assegurar que `nome_lib` representa apenas um ficheiro permitido dentro do diretório de bibliotecas do cliente.

## 12. API genérica JSON

A Play Merlin também define:

```rust
#[get("/<slug_cliente>/api/<path..>", rank = 20)]
pub async fn api_json(...) -> (ContentType, String)
```

Essa rota tenta resolver templates e, se necessário, módulos com nomes iniciados por `API.`. Executa o recurso escolhido e devolve a resposta com `ContentType::JSON`.

**Observação:** a declaração do tipo de conteúdo não valida automaticamente se o texto produzido é JSON válido; a template ou módulo precisa produzir dados no formato adequado.

## 13. Papel do desacoplamento

O desacoplamento é um dos principais motivos pelos quais recursos inicialmente simples podem ser reutilizados em aplicações inesperadas:

1. **A linguagem Merlin** oferece avaliação de expressões, pilhas, dicionários, pipelines, módulos, escopos e processamento de templates.
2. **A Play Merlin** escolhe URLs, contexto HTTP, identidade do utilizador, localização dos recursos, políticas de autorização e interfaces administrativas.
3. **As aplicações dos desenvolvedores** combinam os recursos da linguagem para resolver problemas de negócio: restaurantes, finanças, páginas de demonstração, migração JSON, formulários e outros.

A Play Merlin **não é a linguagem Merlin**. É uma aplicação dela — embora também ofereça as ferramentas para criar outras aplicações.

## 14. Cuidados para produção

Estas recomendações não alteram a arquitetura descrita; tornam explícitas as responsabilidades da plataforma:

- Autorizar no **servidor** qualquer operação de gravação e qualquer execução de código com efeitos persistentes.
- Validar `slug`, caminhos de ficheiros, nomes de bibliotecas e alvos de escrita, respeitando o isolamento entre clientes.
- Distinguir **renderizar uma template sem salvar o código** de **executar operações de escrita dentro dessa template**. Uma renderização pode ter efeitos persistentes se o código chamar `JSON.WRITE` ou funções semelhantes.
- Para migrações, oferecer pré-visualização e backup quando necessário.
- Evitar operações destrutivas por GET e usar proteção CSRF nos POST administrativos.
- Tratar com cuidado HTML/JavaScript e conteúdo editável de templates; o grau de confiança do código importa.

## 15. Notas de validação da documentação

**Confirmado nas explicações do criador e/ou código compartilhado:**

- Merlin é independente da Play Merlin.
- O padrão de GET/POST na mesma template é adotado pelo criador.
- Rotas genéricas GET, POST inline e API JSON; resolução de templates por candidatos e aliases.
- ADMIN por cliente com registos JSON e views configuráveis.
- Editor de templates e módulos com renderização temporária e APIs próprias de gravação.
- Execução de templates como scripts.
- Globais imutáveis e possibilidade de personalizar módulos no cliente.
- Execução de templates cujo código está em strings, conforme relato do criador.

**A verificar com trechos complementares:**

- Ponto concreto em que uma string é selecionada como template virtual no caminho da rota.
- Diferenças entre os formatos POST de formulários comuns e de renderização inline.
- Origem exata de `ID`, `CREATION`, `NOME_MODEL` e outras variáveis de conveniência em todas as rotas.
- Limites de execução, autorização e persistência do contexto nos caminhos genéricos.

## 16. Relação com a Merlin Academy e Merlin AI

A Merlin Academy pode ensinar duas disciplinas complementares:

**Programação Merlin:** sintaxe, avaliação em pilhas, operadores, pipelines, módulos, escopos, templates e persistência disponível ao interpretador.

**Desenvolvimento com Play Merlin:** rotas genéricas, contexto de cliente, CRUD com GET/POST, ADMIN, editor, testes em memória, APIs e scripts de migração.

O futuro Merlin AI Agent poderá utilizar o editor e as APIs já existentes para consultar código, propor alterações, renderizá-las e salvar versões aprovadas. Isso é uma **proposta de integração futura**, não significa que o agente já esteja implementado.

---

### Fontes para esta versão

- Explicações e exemplos do criador nesta conversa, incluindo as migrações JSON, o ADMIN e `HTML.RENDER_FORM`.
- Código compartilhado da rota genérica, renderização inline, API JSON e endpoints de gravação.
- Exemplo `destaques_update.html` de processamento GET/POST em Merlin.
- Referência da linguagem Merlin anteriormente fornecida, utilizada apenas para identificar a fronteira entre linguagem e plataforma.

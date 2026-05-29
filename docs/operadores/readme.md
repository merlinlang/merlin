# Operadores Merlin

Esta pasta contém os arquivos JSON usados como base para a documentação dos operadores da linguagem Merlin.

Esses arquivos podem ser usados para:

- gerar o manual oficial da linguagem;
- alimentar o help do `play.merlinlang.com`;
- servir de base para autocomplete e hover/help em uma futura extensão/editor Merlin;
- manter uma fonte simples e editável da documentação dos operadores.

## Arquivos

```text
docs/
  operadores/
    README.md
    tipos.json
    numeros.json
    strings.json
    dicionarios.json
    datas_horas.json
    merlin.json
```

## Famílias de operadores

| Arquivo | Chamada | Descrição |
|---|---:|---|
| `numeros.json` | `@<OPERADOR>` ou `@OPERADOR(...)` | Operadores numéricos e vetores numéricos |
| `strings.json` | `$<OPERADOR>` ou `$OPERADOR(...)` | Operadores de strings e vetores de strings |
| `dicionarios.json` | `#<OPERADOR>` ou `#OPERADOR(...)` | Operadores de dicionários e vetores de dicionários |
| `datas_horas.json` | `~dt<OPERADOR>` ou `~dt<OPERADOR>(...)` | Operadores de datas e horas |
| `merlin.json` | `&OPERADOR` ou `&OPERADOR(...)` | Operadores internos da Merlin |

## Tipos de pilha

Os campos `pilhas_entrada` e `pilhas_saida` usam a simbologia da própria Merlin:

| Símbolo | Significado |
|---:|---|
| `@` | número |
| `@[]` | vetor de números |
| `$` | string |
| `$[]` | vetor de strings |
| `#` | dicionário |
| `#[]` | vetor de dicionários |

Operadores de datas e horas usam o namespace `~dt`, mas continuam usando `@` e `$` como entrada/saída, pois datas e horas são representadas internamente como números ou strings.

Operadores internos da Merlin usam `&`.

## Estrutura dos JSONs

Cada operador segue uma estrutura simples baseada em chaves e valores de texto:

```json
{
  "categoria": "numeros",
  "operadores": "EXP",
  "flags": "TYPE_IS_NUMERO | RETURN_IS_NUMERO | PIPE_RETURN",
  "parametros": "base, expoente",
  "pilhas_entrada": "@, @",
  "pilhas_saida": "@",
  "descricao": "Calcula uma potência.",
  "funcao": "@EXP(2, 3) = 8",
  "rpn": "2 3 @<EXP> = 8"
}
```

## Campos

| Campo | Descrição |
|---|---|
| `categoria` | Família do operador: `numeros`, `strings`, `dicionarios`, `datas_horas` ou `operadores_merlin` |
| `operadores` | Nome principal e aliases, separados por vírgula, sem `<` e `>` |
| `flags` | Flags retornadas pelo modo de verificação do operador |
| `parametros` | Nome dos parâmetros, na ordem de uso em função |
| `pilhas_entrada` | Tipos consumidos pelo operador |
| `pilhas_saida` | Tipos deixados pelo operador |
| `descricao` | Explicação curta do operador |
| `funcao` | Exemplo no formato mais usado pelo usuário |
| `rpn` | Exemplo em forma RPN, usada internamente pela Merlin |

## Observações

A Merlin é baseada em pilhas e RPN internamente, mas a documentação prioriza a forma funcional quando ela for a forma mais comum de uso.

Exemplo:

```text
$FORMATA_MOEDA(1234.5)
```

Ainda assim, o campo `rpn` fica disponível para documentação técnica, debug, testes e futura IDE.

## Datas e horas

Datas e horas são documentadas em arquivo separado por domínio semântico, mas não possuem pilha própria.

Exemplo:

```text
~dt<TODAY>
~dt<FMT_BR>(46100)
~dt<SYSTEM_TO_EXTENSO>(passeio.created_at)
```

Entradas e saídas continuam usando:

```text
@  número/data/tempo numérico
$  string/data/tempo formatado
```

## Operadores Merlin

Operadores Merlin são chamados com `&`.

Exemplos:

```text
&DEBUG_MERLIN
&PRINT("Teste")
&MOSTRA_PILHA_STRING
&EXECUTA_TEMPLATE("components/modal_upload.html")
```

Eles são usados principalmente para debug, escopos, execução de módulos/templates, include, input/output e inspeção do runtime.

## Futuro

A documentação das libs Merlin, como `HTML`, `RESTAURANTE`, `FINANCE`, `ESTATISTICA`, `CORE`, `SITE` e `CLIENTE`, deve ficar em uma etapa posterior, pois essas bibliotecas ainda podem mudar bastante.

A documentação inicial deve focar nos operadores estáveis da linguagem.

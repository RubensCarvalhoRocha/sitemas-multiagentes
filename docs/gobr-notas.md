# Notas sobre a linguagem GOBR (Fase 0)

Fontes: `govendas-rules.pdf` (Guia do Usuário GoBR v0.89.0, 58 páginas), os 3 pares em `exemplos/` e as 246 regras em `referencias/`. As referências a capítulos (cap. N) apontam para o manual.

---

## 1. Achado principal: o cap. 10 não traz a gramática completa

O kickoff assume que o cap. 10 contém a gramática formal. Ele traz apenas:

- a produção de alto nível (abaixo);
- a tabela de tipos básicos (String, Inteiro, Real, Hora, Data DMY/YMD, Tag `@nome`, Variável);
- regras léxicas (`#` comenta até o fim da linha, acentos permitidos, strings sem `"` interno).

```
gorules:
    version
    ( regrapadrao
    | times segmentacoes
      ( carteiras | parametros | carteiras parametros
      | alertas | carteiras alertas | tags | carteiras tags ) )
    end
```

A sintaxe de cada filtro, ação e seletor só existe espalhada pelas tabelas dos caps. 3 a 9 e pelos exemplos. Consequência para a Fase 1: o `gobr.lark` terá que ser montado a partir dessas tabelas, e o corpus de referências serve para conferir as variações que o processador real aceita (ver seção 9).

Da produção acima também se tira que:
- a ordem das categorias é fixa: `time` → `segmentacao` → `carteira` → `parametros`/`alerta`/`tag`;
- **parametros, alertas e tags não se misturam** no mesmo arquivo (o cap. 8 confirma isso para parametros + alertas);
- `regrapadrao` e `end` aparecem na gramática, mas não são explicados em nenhum lugar (ver dúvidas).

## 2. Estrutura de um arquivo

```
version 1.0                       # obrigatório (cap. 3.1)

time "<nome>":                    # 1+ times
    vendedores <seleção>          # várias linhas são somadas

segmentacao "<nome>":             # opcional, auxiliar (não gera agenda)
    clientes <seleção>
    quando <filtros>

carteira ["livre"] "<nome>":      # opcional
    clientes <seleção>
    quando <filtros>

parametros "<nome>":              # ou: alerta "<nome>": / tag ...
    <fonte>                       # do time(...) | da carteira(...) | da segmentacao(...)
    <filtros temporais>           # opcional
    quando <filtro> [e <filtro>]* # opcional
    <ações>
```

- Indentação no estilo Python: o cabeçalho do bloco termina em `:` e o corpo vem indentado. Continuações (`e ...`, modificadores de `vendedores`, linhas de `preferencia`) usam indentação maior ainda.
- Os nomes dos blocos ficam entre aspas duplas e precisam ser únicos dentro da sua categoria. No caso de alertas, a unicidade vale entre todos os conjuntos de regras da mesma empresa.
- Os filtros são combinados com `e`. O manual **não tem `ou` entre filtros**. O `ou` só aparece dentro de construções específicas (`ou prospects`, `ou indefinido`, cascata de frequência e de preferência).

## 3. Blocos

### 3.1 `time` (cap. 3)
- `vendedores todos | todostelevendas | todosdecampo | porid("id", ...)`
- Os modificadores valem só para `todos*`, nesta ordem: `dos supervisores("id",...)` / `do supervisor("id")`, `ativados` / `inativados`, `menos("id",...)`.
- Um mesmo vendedor pode estar em vários times. O manual fala de nomes únicos "no mesmo arquivo".

### 3.2 `segmentacao`, `carteira`, `carteira livre` (cap. 4)
- Seleção: `clientes carteirafixa(T) | carteiraacessivel(T) | carteiraruptura(T) | portfolio(T) | inativos(T, tempo) | segmentados(S) | todos(T) | ruptura(T1 para T2) <opções> | ruptura da segmentacao(S para T) <opções>`
- Opções de ruptura: `de positivacao(tempo)`, `de visita(tempo)`, `de inativos(T, tempo)`, cada uma com `ou prospects` opcional e combináveis com `e`. Exemplo de tempo nesse contexto: `(90 dias)`, `(1 mes)`.
- **Carteira exclusiva**: clientes já alocados por uma carteira anterior saem do pool das seguintes. **Carteira livre**: não retira ninguém do pool. A ordem importa.
- Uma carteira aceita várias fontes de time, mas **no máximo uma segmentação**. Uma segmentação aceita várias.
- Padrão: um time usado em parametros/alertas sem carteira correspondente recebe automaticamente a `carteirafixa` do time.

### 3.3 `parametros` (cap. 7). É o bloco central do trabalho.
Fluxo: fonte → filtros temporais → `quando` → ações → `sem choque` (opcional).

| Ação | Formas |
|---|---|
| Frequência fixa | `frequencia diaria() / semanal() / quinzenal(['dd-mm-yyyy']) / mensal() / bimestral(['dd-mm-yyyy']) / trimestral() / semestral() / anual() / de N dias` |
| Limites | `no máximo semanal()`, `no mínimo mensal()` (o manual só mostra a forma com função) |
| Frequência dinâmica | `do govendas`, `do vendedor [mais/menos N dias]`, `do historico`, `de qualquer vendedor`, `de ativos/inativos da empresa`, `de ativos/inativos do vendedor`, `remover` |
| Cascata | `frequencia do vendedor ou de qualquer vendedor ou de 35 dias` |
| Preferência de dias | `preferencia na segunda e na terça`, intensidades `maior`/`normal`/`menor` (3/2/1), dias não citados ficam com 0; `nos dias uteis`; `do historico [ou ...]` |
| Restrição de semana | `restrito a partir da segunda semana`, `restrito ate a terceira semana` |
| Outras | `roteirizavel`, `fixo` (o padrão é nenhum dos dois) |
| Remoção | `nao agendar`, **exclusiva**: não pode vir junto de frequência nem de preferência |
| Conflito | `sem choque na carteira("X")`, `sem choque na rota e na pasta do time("X")` (`na rota`/`fora da rota`/`na pasta`) |
| Limpeza | bloco `limpar parametros "nome":` + fonte |

Quando a regra não define preferência, o padrão é 1 de segunda a sexta e 0 no sábado e no domingo (cap. 7.3.1).

### 3.4 `alerta` (cap. 8) e `tag` (cap. 9)
- Alerta: fonte → temporal (pode ter horário) → `quando` → `onde <gatilho>` → `informe ...` → `descreva com o texto("...")` → `entregue para vendedor|supervisor|gestor [e redirecione("email")]` → `pelos canais govendas() [e whatsapp("tpl")]` → `e template com os valores(...)`.
- Tag: o manual lista os gatilhos e os filtros, mas **não mostra a sintaxe do bloco** (nenhum exemplo). Nenhuma referência usa tags.
- **Nenhum dos 246 arquivos usa alerta ou tag.** O assunto do trabalho (frequência de agendamento) também é todo de parametrização. Sugiro deixar alertas e tags fora do escopo do gerador (ver dúvidas).

## 4. Filtros temporais (cap. 5)

A ordem é obrigatória: dia do mês → dia da semana → semana do mês → mês do ano → horário (só em alertas). Os filtros são combinados com `e`.
- Dia do mês: `no dia N`, `no ultimo dia do mês`, `antes do dia N`, `depois do dia N`, `a partir do dia N`, `entre os dias N e M`
- Dia da semana: `na segunda` … `no domingo`, `todos os dias`, `nos dias uteis`, `no fim de semana`
- Semana: `na primeira|segunda|terceira|quarta|ultima semana`, `na primeira|segunda quinzena`
- Mês: `em janeiro` … `em dezembro`, `no <ordinal> bimestre|trimestre|semestre`
- Horário: `às 'hh:mm' [e às ...]`, `de hora em hora [entre 'h' e 'h' | a partir de 'h']`

## 5. Filtros de cliente (cap. 6), por família

| Família | Exemplos |
|---|---|
| Atividade | `ativos [para o vendedor/a empresa]`, `inativos [...]`, `inatividade [global] de N dias`, `sem venda [ou prospects]`, `unica compra com a empresa/o vendedor`, `prospects do vendedor/da empresa`, `[não] tem venda [do vendedor/da empresa] <cond>`, `atividade [global/do vendedor] de <tempo>` |
| Recência | `[fora da] recencia de visita/positivacao [global/do vendedor] <cond>` |
| Financeiro | `adimplentes`, `inadimplentes`, `[não] tem credito`, `com N% de credito`, `com menos de R$ V de credito disponivel`, ticket médio, `total faturado/media faturada no <periodo> com a empresa maior/menor que R$ V [ou indefinido]` |
| Agenda | `na agenda [do vendedor/global] de hoje`, `na agenda na semana`, `fora da agenda ...`, `[não] agendavel`, `reagendado ...` |
| Localização/cadastro | `nome da cidade/bairro`, `uf do estado`, `nome/codigo da rota`, `ramo de atividade`, `cnae ...` + `igual/diferente/contém/inicia com(...)`; `status na receita federal definido como ativa/suspensa/inapta/baixada`; `pessoa fisica/juridica`; `cliente principal/filial`; `[não] excluidos/bloqueados`; `[não] tem coordenada confiavel` |
| Curva / ciclo de vida | `curva A/B/C com a empresa/o vendedor`; `no ciclo de vida ativo/inativo/prospect/ativado/reativado com a empresa/o vendedor` |
| Frequência | `não tem parametro [manual]`, `fora da frequencia de venda ...`, `frequencia historica maior/menor que N dias [ou indefinida]`, `frequencia historica entre N dias e M dias` |
| Segmentação | `[não] contido na segmentacao("...")` |
| Ruptura/contato | `em ruptura global do bimestre/trimestre/semestre/ano`, `sem/com contato no ultimo agendamento ...` |
| Outros | `cadastrado faz N dias`, `cadastrado a partir do dia 'data'`, `elegivel`, `data da rota ...`, `data da próxima/última visita do representante ...`, `total de compras ...`, `queda de faturamento ...` |

Condicionais de tempo (6.12/6.13): `em N <unidade>`, `há N <unidade> atras`, `entre <tempo> e <tempo>`, `entre 'data' e 'data'`, tempos fixos (`na semana`, `no mês`, `no ano anterior`...), `nos proximos N <unidade>`, `em N <unidade> a frente`.

## 6. Semântica de ordem e avaliação (o mais importante para analisar lacunas)

1. **Os parametros são avaliados na ordem e quem casa primeiro fica com o cliente.** O cap. 7.6 diz: "clientes parametrizados em uma regra anterior não são reprocessados por regras subsequentes. A ordem importa."
2. Por isso `nao agendar` também "consome" o cliente. As exclusões (inadimplente, baixado na receita, rota da semana, CPF...) **têm que vir antes** das regras de frequência.
3. Uma regra **sem `quando`** casa com todos os clientes que sobraram, ou seja, é o **fallback**. Ela fica por último e pode agendar com uma frequência padrão ou usar `nao agendar`. 126 das 246 referências têm algum `parametros` sem `quando`, e em 119 delas o último bloco é esse fallback.
4. Como o primeiro casamento vence, as faixas de recência podem ser escritas de forma **cumulativa**: `em 60 dias` → `em 90 dias` → `em 120 dias`. A segunda regra só recebe quem não caiu na primeira, então `em 90 dias` passa a significar, na prática, "entre 60 e 90". Os exemplos usam esse padrão (Maxlub, Infinitto). Também existe o estilo explícito, com `entre N dias e M dias` ou `em 45 dias e fora da recencia ... em 30 dias` (V1Vidros).
5. O mesmo vale para as carteiras exclusivas (cap. 4.7). As carteiras livres e os alertas não dependem da ordem.
6. A parametrização do GoBR **não sobrescreve parametrização manual** feita pelo usuário no Govendas (cap. 7).
7. Lacunas que o agente consegue detectar só com essa semântica:
   - **buraco** entre faixas: em `regra_230`, `entre 16 e 45` é seguido de `entre 60 e 89`, e quem está entre 46 e 59 dias fica sem regra;
   - **regra inalcançável**: uma regra ampla antes de uma mais específica. Exemplo: em V1Vidros a regra de prospects vem **depois** de `fora da recencia de positivacao em 45 dias`. Se um prospect, que nunca comprou, conta como "fora da recência", a regra de prospects nunca é usada (ver dúvidas);
   - **ausência de fallback**: clientes que não casam com nenhuma regra ficam sem parametrização do GoBR;
   - **exclusão depois de agendamento**: a exclusão não tem efeito sobre quem já casou antes;
   - **nome que diverge do conteúdo**: em `regra_128` o nome diz "A partir do dia 20" e a regra usa `a partir do dia 16`.

## 7. Padrões de tradução mapa mental → GOBR (os 3 pares)

| Mapa mental | GOBR usado |
|---|---|
| Galhos = segmentos; texto do galho = condição; folha "N dias" = frequência | um `parametros "<nome do galho>"` por segmento, `frequencia de N dias` |
| "compra nos últimos 60 dias" / "pedidos nos últimos 60" | `recencia de positivacao [global] em 60 dias` |
| "compraram nos últimos 90 mas estão a 60 dias sem comprar" | ou `recencia de positivacao em 90 dias` contando com a ordem (Maxlub), ou `em 45 dias` + `e fora da recencia de positivacao em 30 dias` (V1Vidros) |
| "sem pedidos há mais de 90/45 dias" | `fora da recencia de positivacao em N dias`, ou fallback sem `quando` (Infinitto) |
| "ciclo de vida prospect" | `no ciclo de vida prospect com a empresa` |
| "Não agendar inadimplentes" | `quando inadimplentes` + `nao agendar`, no topo |
| "Somente clientes que não estão na rota da semana" (repetido em todo galho) | **um único** `parametros` de exclusão `quando data da rota na semana` + `nao agendar` no topo, em vez de repetir em cada regra |
| Público implícito "televendas" | `time "televendas": vendedores todostelevendas ativados` e `do time("televendas")` em cada regra |
| "Agendar inadimplentes" (V1Vidros) | **nada**: é o comportamento padrão, porque não há regra que exclua inadimplentes |
| "Capacidade de atendimento: 35 clientes/dia" e "não conflitar com o externo do Sales Force" (Maxlub) | **não aparecem no GOBR**. A capacidade não tem construção na linguagem e o conflito virou a exclusão por rota da semana |

O que os especialistas acrescentam por conta própria, sem estar no mapa:
- `nao agendar` para `status na receita federal definido como baixada` (Infinitto, V1Vidros);
- um fallback final (Maxlub com `nao agendar`, Infinitto como "Inativo");
- prospects posicionados antes das regras de recência (Infinitto, Maxlub).

O agente deve **sugerir** esses acréscimos como boas práticas, e não aplicá-los em silêncio.

Inconsistências dentro dos próprios exemplos (cuidado na avaliação da Fase 4):
- `exemplos/V1Vidros-RegraComercial.gobr` **não tem `version 1.0`**, que o manual diz ser obrigatório;
- em Infinitto, "Ativo" usa `global` e "Risco" não usa;
- os exemplos aceitam `do time ("x")`, com espaço antes do parêntese.

## 8. Estatísticas do corpus `referencias/` (246 arquivos)

- **Blocos**: 1.699 `parametros`, 350 `time`, 160 `carteira`, 45 `carteira livre`, 12 `segmentacao`. Nenhum `alerta`, `tag` ou `limpar parametros`.
- **Combinações**: 117 com time+carteira+parametros; 113 com time+parametros; 12 com carteira livre; 4 com segmentação.
- **Duplicação**: 123 arquivos usam o template "Regra Padrao do goVendas Customizavel" (regra_001 é um exemplo). Ao normalizar nomes e números, restam **190 estruturas distintas**: há um cluster de 31 e outro de 21 arquivos idênticos.
- **Fonte**: `do time(...)` (~960) e `da carteira(...)` (~730); `da segmentacao(...)` é raro; vários times na mesma fonte aparecem em `do time("a","b")`.
- **Ação**: `frequencia de N dias` (1.069), `de inativos da empresa` (110), `semanal()` (95), `de ativos da empresa` (55), `do historico ou no minimo de N dias e no maximo de N dias` (~25); `nao agendar` (273); `preferencia nos dias uteis` (125), `... e no sabado` (43); `fixo` (59); `sem choque` (13); `roteirizavel` (5).
- **Filtros mais usados**: `ativos para o vendedor` (137), `sem venda` (114), `recencia de positivacao [global] em N dias` (206), `inativos para o vendedor` (87), `ramo de atividade igual(...)` (~150 somando variações), `inadimplentes` (59 isolado, mais combinações), `recencia de positivacao [global] entre N e M dias` (100), `no ciclo de vida prospect com a empresa` (51), `media faturada no ano ... maior que R$` (48), `fora da recencia de positivacao em N dias` (46), `status ... baixada` (35). São 193 formas distintas de filtro depois da normalização.
- **Estilo**: 103 arquivos começam com comentário `# Solicitação: <link>` (147 não têm `version` como primeira linha não vazia, mas todos têm `version 1.0`); seções `## ...`; 9 arquivos indentados com tab; nomes no formato `Freq. de 14d para 'Ativo: ...'` ou `Não agendar <grupo>`.

## 9. Divergências manual × corpus (decidir na Fase 1)

Construções aceitas em produção (estão nas referências), mas **ausentes do manual ou diferentes dele**:

| No corpus | No manual |
|---|---|
| `sem venda`, `sem vendas`, `prospect`, `sem venda do vendedor` | só `sem venda ou prospects`, `prospects do vendedor/da empresa` |
| `preferencia do vendedor` | só `preferência do historico` |
| `sem choque no time("X")`, `sem choque na rota com time("X")` | `sem choque na carteira(...)`, `sem choque na rota e na pasta do time(...)` |
| `frequencia do historico ou no minimo de N dias e no maximo de N dias` | limites só na forma com função (`no máximo semanal()`) |
| `frequencia historica de N dias` | `maior que` / `menor que` / `entre` |
| `o rotulo contem(...)`, `cnae é igual(...)`, `é o cliente filial`, `uf do estado igual(...)` | `o rótulo diferente`, `cnae igual`, `cliente filial`, `uf do estado contém` |
| `curva c` (minúscula) | `curva C` |
| `recencia de positivacao entre 'data' e 'data'` | condicional `entre <<data>> e <<data>>` (coerente com o manual) |
| `data da rota na semana`, `data da rota nos proximos N dia`, `data da rota ha N dias atras` | só exemplos pontuais (`no mês anterior`, `futura em 5 dias a frente`). O padrão "data da rota + condicional de tempo" é generalizável |
| `parametros` com `da segmentacao(...)` **e** `e da carteira(...)` (regra_169) | não documentado |
| Variantes com acento: `frequência`, `não agendar`, `histórico`, `mínimo`, `positivação`, `terça` | o manual mistura as duas grafias |
| `frequencia de 1 dia` (singular), espaços duplos, `do time ("x")` | — |

Ruído que provavelmente não passaria no parser real: um bloco `parametros` indentado com 1 espaço (regra_047) e uma aspa sobrando no fim de `preferencia ... no sabado"` (regra_221).

Proposta para a Fase 1: o validador segue o manual, aceita as variantes de acento e de espaçamento, e as construções que só existem no corpus entram como **extensões marcadas**. Cada falha é reportada no formato "corpus × manual" para você decidir caso a caso.

## 10. Notas para as próximas fases

- **Extração do manual**: `pdftotext -layout` embaralha as tabelas de duas colunas (célula de uma linha cola na descrição da outra). `pdftotext -raw` mantém "construção → descrição" na mesma linha e é o modo certo para o chunking da Fase 2. `pdfplumber` também está instalado localmente como alternativa.
- **Descrições (Fase 2)**: gerar uma descrição por estrutura distinta (~190), não por arquivo. Isso evita que os 52 clones do template dominem o top-k e reduz o custo de LLM.
- O nome do `parametros` costuma ser o próprio rótulo do galho no mapa mental. É um bom sinal para alinhar as descrições ao estilo dos exemplos.

## 11. Dúvidas que o manual não responde

1. ~~**Gramática completa**: existe o arquivo ANTLR real?~~ **Respondida:** não há outra fonte; o `govendas-rules.pdf` é a referência. O PDF tem 62 páginas, sem anexos nem imagens com conteúdo, e termina no cap. 10 (pág. 58 impressa). O `gobr.lark` será derivado das tabelas dos caps. 3 a 9 mais a estrutura de alto nível do cap. 10, e conferido com o corpus.
2. **Prospect × recência**: um cliente que nunca comprou casa com `fora da recencia de positivacao em N dias`? Isso define se a ordem de V1Vidros (prospects por último) é um bug.
3. **`recencia ... em N dias` vs `ativos para o vendedor`**: qual é o critério padrão de "ativo" ("critérios padrão do sistema", sem número)? Um mapa que diga "ativo = comprou em 60 dias" deve virar `ativos` ou `recencia ... em 60 dias`?
4. **`global` vs sem qualificador**: na recência, "sem qualificador" é "algum vendedor do mesmo grupo". Quando o especialista escolhe `global`? (Infinitto mistura as duas formas.)
5. **Capacidade diária** ("35 clientes/dia" no Maxlub) e outros requisitos sem construção na linguagem: o agente deve só avisar que ficam fora do GOBR e precisam ser configurados em outro lugar?
6. **`version 1.0` ausente** (V1Vidros): o processador aceita? O validador deve tratar isso como erro ou como aviso?
7. **`regrapadrao` e `end`** na gramática: o que são? O `end` parece ser só o EOF do ANTLR.
8. **Sintaxe de `tag`**: o manual não traz exemplo. Confirmar se alertas e tags podem ficar fora do escopo do agente.
9. **Clientes sem regra e sem fallback**: ficam sem parametrização, com o padrão do Govendas, ou não entram na agenda?
10. **`nao agendar` com fonte em times diferentes**: em arquivos com vários times, a ordem "first-match" vale por time ou no arquivo inteiro?

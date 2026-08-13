# Aura Agent — Investigador de Fraude

Sugestão de configuração para cadastrar um [Aura
Agent](https://neo4j.com/docs/aura/agents/) sobre o grafo construído nos
notebooks. Com ele, a análise deixa de ser código e passa a ser conversa:
*"quais grupos de fraude existem?"*, *"mostre o anel desse CPF"*, *"quem são as
contas-laranja mais influentes?"*.

Os campos abaixo podem ser preenchidos direto no **Aura Console → Agents → Create
agent**, ou enviados de uma vez via API com o [`agent-config.json`](agent-config.json)
deste diretório.

> **Pré-requisitos:** ter rodado o notebook **02** (carga) e o **04** (análise).
>
> O notebook 04 é o que torna este agente interessante: ele grava os resultados dos
> algoritmos de volta no grafo (`:Suspeito`, `grupo_fraude`, `tamanho_grupo`,
> `:ContaLaranja`, `score_laranja`). As tools 9 e 10 consultam exatamente esses
> campos — sem o notebook 04, elas retornam vazio.
>
> Todas as queries foram **testadas** contra uma instância AuraDB Free com a base
> padrão do material (10.000 clientes, 140.531 transações), tanto com parâmetro
> preenchido quanto vazio.

---

## Name

```
Investigador de Fraude Pix
```

## Description

```
Investiga indícios de fraude de identidade e lavagem de dinheiro em um grafo de
transações bancárias brasileiras. Encontra clientes que compartilham RG, e-mail ou
telefone (identidades forjadas), consulta os anéis de fraude e as contas-laranja
detectados por algoritmos de grafo, e rastreia o fluxo de Pix entre contas.
```

## Prompt instruction (system prompt)

```
Você é um assistente especialista em investigação de fraude bancária sobre um
grafo de clientes, identificadores e transações Pix/boleto/cartão.

Você pode:
- Localizar clientes por nome ou CPF e detalhar o perfil de um cliente
- Encontrar clientes que compartilham RG, e-mail ou telefone entre si
- Consultar os anéis de fraude e as contas-laranja já detectados por algoritmos
- Rastrear Pix enviados e recebidos, com valores e contrapartes

Você NÃO pode alterar dados, criar cadastros, acessar informação fora do grafo,
nem afirmar que alguém cometeu fraude. Você reporta **indícios**; a conclusão é
sempre do investigador humano.

Preferência de ferramentas: para buscas por valor conhecido (CPF, nome), use as
CypherTemplate específicas antes da ferramenta de consulta aberta. Se o usuário
citar um nome, resolva primeiro para CPF com "Buscar Cliente por Nome" e confirme
qual cliente é, caso haja mais de um.

Algumas ferramentas aceitam parâmetro vazio: numa pergunta geral, sem cliente
específico, omita o parâmetro (ou envie null) para obter a visão global em vez de
pedir um CPF que o usuário não tem. Cada ferramenta devolve no máximo 10 linhas —
avise quando a lista for truncada.

Baseie todas as respostas nos dados retornados pelas ferramentas — nunca no seu
conhecimento prévio. Sempre inclua o CPF nas respostas, para o usuário poder
verificar. Use tabelas quando comparar vários clientes. Se as ferramentas não
retornarem resultado, diga isso explicitamente e não especule.

Se o usuário pedir para você explicar sua resposta:
1. Diga qual ferramenta usou, quais nós e relacionamentos foram percorridos e por quê.
2. Sugira como ele poderia perguntar de forma mais precisa.
3. Se uma nova CypherTemplate responderia melhor, proponha a query e os parâmetros.
4. Se uma mudança no modelo tornaria a consulta mais rápida ou precisa, descreva qual.

## Sinais de Fraude

**Identidade compartilhada** — `(Cliente)-[:TEM_RG|TEM_EMAIL|TEM_TELEFONE]->(RG|Email|Telefone)`
- O CPF é validado e único por cliente; RG, e-mail e telefone não são validados.
- Dois CPFs distintos apontando para o MESMO nó de identificador é o principal
  indício de identidade forjada. Quanto mais identificadores em comum, mais forte.

**Anel de fraude (resultado de algoritmo)** — `(Cliente:Suspeito)`
- Propriedades: `grupo_fraude` (id do anel), `tamanho_grupo` (nº de membros)
- Gravadas por componentes conectados (WCC). Clientes do mesmo `grupo_fraude`
  formam uma identidade coletiva suspeita.

**Conta-laranja (resultado de algoritmo)** — `(Cliente:ContaLaranja)`
- Propriedade: `score_laranja` (PageRank ponderado por valor; maior = mais central)
- Marca quem está acima do 95º percentil de influência na rede de Pix suspeitos.
- Atenção: `:Suspeito` e `:ContaLaranja` são padrões DIFERENTES. Um cliente pode ser
  um sem ser o outro — anel é identidade forjada, laranja é movimentação de dinheiro.

**Fluxo de Pix** — `(Cliente)-[:REALIZOU]->(Pix)-[:PARA]->(Cliente)`
- Propriedade `valor`. O relacionamento `TRANSFERIU_PIX_PARA` agrega o total
  enviado de um `:Suspeito` para outro cliente.

Se `grupo_fraude` ou `score_laranja` vierem nulos, não conclua que o cliente é
inocente: pode ser que a análise não tenha rodado nesta base.
```

---

## Tools

Ordem importa: a ferramenta de consulta aberta (Text2Cypher) fica **por último**,
porque o agente tenta as ferramentas na ordem em que aparecem e ela é o fallback.

Nas tabelas, "opcional" significa que a query trata `null`/vazio como "sem filtro"
— o agente pode omitir o valor para obter a visão global.

### 1. Buscar Cliente por Nome

| Campo | Valor |
|---|---|
| **Type** | `cypherTemplate` |
| **Name** | `Buscar Cliente por Nome` |
| **Description** | Use quando o usuário mencionar o **nome** de uma pessoa em vez do CPF. Retorna CPF, nome, data de cadastro e total de transações dos clientes cujo nome contém o texto buscado (máx. 10). Sempre use esta ferramenta primeiro para converter um nome em CPF antes de chamar as demais. NÃO use se o usuário já informou o CPF. |

**Parameters**

| name | data_type | obrigatório | description |
|---|---|---|---|
| `nome` | `string` | sim | Parte do nome do cliente a buscar, sem diferenciar maiúsculas/minúsculas. Ex.: um primeiro nome ou nome completo parcial. |

**Cypher**

```cypher
MATCH (c:Cliente)
WHERE toLower(c.nome) CONTAINS toLower($nome)
OPTIONAL MATCH (c)-[:REALIZOU]->(t:Transacao)
RETURN c.cpf AS cpf, c.nome AS nome, c.data_cadastro AS data_cadastro,
       count(t) AS total_transacoes
ORDER BY nome
LIMIT 10
```

---

### 2. Detalhar Perfil do Cliente

Reúne cadastro, identificadores e **os dois veredictos dos algoritmos** numa linha.

| Campo | Valor |
|---|---|
| **Type** | `cypherTemplate` |
| **Name** | `Detalhar Perfil do Cliente` |
| **Description** | Use quando o usuário pedir os dados de um cliente específico pelo CPF. Retorna cadastro, RG, e-mail, telefone, volume de transações e o resultado das duas análises: se está num anel de fraude (com id e tamanho do grupo) e se foi marcado como conta-laranja (com o score de influência). NÃO use para descobrir com quem ele compartilha identificadores — use "Listar Identidades Compartilhadas". |

**Parameters**

| name | data_type | obrigatório | description |
|---|---|---|---|
| `cpf` | `string` | sim | CPF do cliente, formatado com pontos e traço (ex.: `'123.456.789-00'`). |

**Cypher**

```cypher
MATCH (c:Cliente {cpf: $cpf})
OPTIONAL MATCH (c)-[:TEM_RG]->(rg:RG)
OPTIONAL MATCH (c)-[:TEM_EMAIL]->(em:Email)
OPTIONAL MATCH (c)-[:TEM_TELEFONE]->(tel:Telefone)
WITH c, rg, em, tel
OPTIONAL MATCH (c)-[:REALIZOU]->(t:Transacao)
WITH c, rg, em, tel, count(t) AS total_transacoes, sum(t.valor) AS valor_total
RETURN c.cpf AS cpf, c.nome AS nome, c.data_cadastro AS data_cadastro,
       rg.valor AS rg, em.valor AS email, tel.valor AS telefone,
       total_transacoes, round(valor_total, 2) AS valor_total_movimentado,
       'Suspeito' IN labels(c) AS em_anel_de_fraude,
       c.grupo_fraude AS grupo_fraude, c.tamanho_grupo AS tamanho_do_grupo,
       'ContaLaranja' IN labels(c) AS marcado_conta_laranja,
       round(c.score_laranja, 4) AS score_influencia
```

---

### 3. Listar Identidades Compartilhadas — CPF opcional

| Campo | Valor |
|---|---|
| **Type** | `cypherTemplate` |
| **Name** | `Listar Identidades Compartilhadas` |
| **Description** | Use para encontrar pares de clientes que compartilham RG, e-mail ou telefone — o principal indício de identidade forjada. Informe o CPF para ver só os pares que envolvem aquele cliente; omita o CPF (ou envie null) para ver os pares mais fortes de toda a base, ideal quando o usuário ainda não tem um suspeito. Retorna os dois CPFs do par, quantos identificadores coincidem e de quais tipos (máx. 10). NÃO use para reconstruir o grupo inteiro em cadeia — use "Reconstruir Anel de Fraude". |

**Parameters**

| name | data_type | obrigatório | description |
|---|---|---|---|
| `cpf` | `string` | **não** | CPF do cliente investigado, formatado com pontos e traço. **Opcional**: deixe vazio ou envie null para listar os pares com mais identificadores em comum de toda a base, sem filtrar por cliente. |

**Cypher**

```cypher
MATCH (c1:Cliente)-[:TEM_RG|TEM_EMAIL|TEM_TELEFONE]->(id)<-[:TEM_RG|TEM_EMAIL|TEM_TELEFONE]-(c2:Cliente)
WHERE c1.cpf < c2.cpf
  AND ($cpf IS NULL OR $cpf = '' OR $cpf IN [c1.cpf, c2.cpf])
WITH c1, c2, collect(DISTINCT head(labels(id))) AS tipos, count(DISTINCT id) AS qtd
RETURN c1.cpf AS cpf_a, c1.nome AS nome_a,
       c2.cpf AS cpf_b, c2.nome AS nome_b,
       qtd AS identificadores_em_comum, tipos AS tipos_compartilhados
ORDER BY identificadores_em_comum DESC, cpf_a
LIMIT 10
```

> O par é retornado uma única vez (`c1.cpf < c2.cpf`), então ao filtrar por um CPF
> ele pode aparecer na coluna `cpf_a` **ou** `cpf_b` — o que interessa é o outro lado.

---

### 4. Reconstruir Anel de Fraude

Funciona mesmo sem o notebook 04: percorre o grafo em vez de ler o resultado gravado.

| Campo | Valor |
|---|---|
| **Type** | `cypherTemplate` |
| **Name** | `Reconstruir Anel de Fraude` |
| **Description** | Use quando o usuário pedir o **grupo** ou **anel** completo ligado a um cliente, e não apenas as ligações diretas. Percorre até 3 saltos de identificadores compartilhados e retorna todos os membros do anel — incluindo o próprio cliente — com RG e e-mail de cada um. NÃO use para ver só os vizinhos imediatos ("Listar Identidades Compartilhadas") nem para listar todos os anéis da base ("Listar Grupos de Fraude Detectados"). |

**Parameters**

| name | data_type | obrigatório | description |
|---|---|---|---|
| `cpf` | `string` | sim | CPF de qualquer membro do anel, formatado com pontos e traço. |

**Cypher**

```cypher
MATCH caminho = (inicio:Cliente {cpf: $cpf})
                ((:Cliente)-[:TEM_RG|TEM_EMAIL|TEM_TELEFONE]->()
                 <-[:TEM_RG|TEM_EMAIL|TEM_TELEFONE]-(:Cliente)){1,3}
                (membro:Cliente)
WITH inicio, collect(DISTINCT membro) AS membros
UNWIND ([inicio] + membros) AS m
WITH DISTINCT m
OPTIONAL MATCH (m)-[:TEM_RG]->(rg:RG)
OPTIONAL MATCH (m)-[:TEM_EMAIL]->(em:Email)
RETURN m.cpf AS cpf, m.nome AS nome, rg.valor AS rg, em.valor AS email
ORDER BY cpf
LIMIT 10
```

---

### 5. Rastrear Pix Enviados — CPF opcional

| Campo | Valor |
|---|---|
| **Type** | `cypherTemplate` |
| **Name** | `Rastrear Pix Enviados` |
| **Description** | Use para ver fluxos de Pix pelo lado de quem **envia**. Informe o CPF para ver para quem aquele cliente enviou Pix; omita o CPF (ou envie null) para ver os maiores fluxos de Pix de toda a base, o que expõe os pares remetente→destinatário mais volumosos sem precisar de um suspeito prévio. Retorna origem, destino, quantidade de Pix e valor total (máx. 10). NÃO use para o histórico de quem recebeu — use "Rastrear Pix Recebidos". |

**Parameters**

| name | data_type | obrigatório | description |
|---|---|---|---|
| `cpf` | `string` | **não** | CPF do cliente **remetente**, formatado com pontos e traço. **Opcional**: deixe vazio ou envie null para listar os maiores fluxos de Pix da base inteira. |

**Cypher**

```cypher
MATCH (origem:Cliente)-[:REALIZOU]->(p:Pix)-[:PARA]->(destino:Cliente)
WHERE $cpf IS NULL OR $cpf = '' OR origem.cpf = $cpf
WITH origem, destino, count(p) AS qtd_pix, sum(p.valor) AS valor_total
RETURN origem.cpf AS cpf_origem, origem.nome AS nome_origem,
       destino.cpf AS cpf_destino, destino.nome AS nome_destino,
       qtd_pix, round(valor_total, 2) AS valor_total
ORDER BY valor_total DESC
LIMIT 10
```

---

### 6. Rastrear Pix Recebidos

| Campo | Valor |
|---|---|
| **Type** | `cypherTemplate` |
| **Name** | `Rastrear Pix Recebidos` |
| **Description** | Use quando o usuário quiser saber **de quem** um cliente específico recebeu Pix. Retorna cada remetente com quantidade, valor total e maior Pix (máx. 10). Muitos Pix de valor alto vindos de poucas origens é o padrão de conta-laranja. NÃO use sem um CPF: para o ranking geral de quem mais recebe, use "Ranquear Contas que Mais Recebem Pix". |

**Parameters**

| name | data_type | obrigatório | description |
|---|---|---|---|
| `cpf` | `string` | sim | CPF do cliente **destinatário**, formatado com pontos e traço. |

**Cypher**

```cypher
MATCH (origem:Cliente)-[:REALIZOU]->(p:Pix)-[:PARA]->(destino:Cliente {cpf: $cpf})
RETURN origem.cpf AS cpf_origem, origem.nome AS nome_origem,
       count(p) AS qtd_pix, round(sum(p.valor), 2) AS valor_total,
       round(max(p.valor), 2) AS maior_pix
ORDER BY valor_total DESC
LIMIT 10
```

---

### 7. Listar Identificadores Mais Reaproveitados — limiar opcional

| Campo | Valor |
|---|---|
| **Type** | `cypherTemplate` |
| **Name** | `Listar Identificadores Mais Reaproveitados` |
| **Description** | Use quando o usuário quiser começar uma investigação sem ter um CPF em mãos, perguntando quais RGs, e-mails ou telefones são usados por mais de um cliente. Retorna o tipo, o valor do identificador, quantos clientes o usam e exemplos de CPF (máx. 10). Funciona mesmo sem os algoritmos terem rodado. NÃO use quando o usuário já tem um cliente específico — use "Listar Identidades Compartilhadas". |

**Parameters**

| name | data_type | obrigatório | description |
|---|---|---|---|
| `minimo_clientes` | `integer` | **não** | Número mínimo de clientes distintos usando o mesmo identificador. **Opcional**: se omitido, usa `2`, que já significa "compartilhado". Use `3` ou mais para focar nos anéis maiores. |

**Cypher**

```cypher
MATCH (c:Cliente)-[:TEM_RG|TEM_EMAIL|TEM_TELEFONE]->(id)
WITH id, count(DISTINCT c) AS qtd_clientes, collect(DISTINCT c.cpf) AS cpfs
WHERE qtd_clientes >= coalesce($minimo_clientes, 2)
RETURN head(labels(id)) AS tipo_identificador, id.valor AS valor,
       qtd_clientes, cpfs[0..6] AS exemplos_cpf
ORDER BY qtd_clientes DESC
LIMIT 10
```

---

### 8. Ranquear Contas que Mais Recebem Pix — limiar opcional

| Campo | Valor |
|---|---|
| **Type** | `cypherTemplate` |
| **Name** | `Ranquear Contas que Mais Recebem Pix` |
| **Description** | Use para identificar possíveis contas-laranja **por volume**, sem depender de algoritmo. Retorna os clientes que mais receberam Pix, com quantidade, número de origens distintas, valor total e ticket médio (máx. 10). Um volume alto concentrado em poucas origens é o sinal a investigar. Para o ranking por influência na rede (PageRank), use "Ranquear Contas Laranja por Influência". |

**Parameters**

| name | data_type | obrigatório | description |
|---|---|---|---|
| `minimo_pix` | `integer` | **não** | Quantidade mínima de Pix recebidos para entrar no ranking. **Opcional**: se omitido, usa `10`, que é o corte que separa conta-laranja de cliente comum nesta base. |

**Cypher**

```cypher
MATCH (origem:Cliente)-[:REALIZOU]->(p:Pix)-[:PARA]->(destino:Cliente)
WITH destino, count(p) AS qtd_pix, sum(p.valor) AS valor_total,
     count(DISTINCT origem) AS qtd_origens
WHERE qtd_pix >= coalesce($minimo_pix, 10)
RETURN destino.cpf AS cpf, destino.nome AS nome,
       qtd_pix, qtd_origens,
       round(valor_total, 2) AS valor_total_recebido,
       round(valor_total / qtd_pix, 2) AS ticket_medio
ORDER BY valor_total_recebido DESC
LIMIT 10
```

---

### 9. Listar Grupos de Fraude Detectados — requer notebook 04

Lê o resultado do algoritmo de componentes conectados gravado no grafo.

| Campo | Valor |
|---|---|
| **Type** | `cypherTemplate` |
| **Name** | `Listar Grupos de Fraude Detectados` |
| **Description** | Use quando o usuário perguntar quais anéis ou grupos de fraude existem na base, sem citar um cliente. Lê os grupos identificados pelo algoritmo de componentes conectados e retorna o id do grupo, quantos membros tem e exemplos de CPF e nome (máx. 10). É a melhor primeira pergunta de uma investigação. Retorna vazio se a análise de grafo ainda não foi executada. NÃO use para o anel de um cliente específico — use "Reconstruir Anel de Fraude". |

**Parameters**

| name | data_type | obrigatório | description |
|---|---|---|---|
| `minimo_membros` | `integer` | **não** | Tamanho mínimo do grupo para aparecer. **Opcional**: se omitido, usa `3`. Use valores maiores para ver só os anéis mais numerosos. |

**Cypher**

```cypher
MATCH (c:Cliente:Suspeito)
WHERE c.tamanho_grupo >= coalesce($minimo_membros, 3)
WITH c.grupo_fraude AS grupo, c.tamanho_grupo AS tamanho,
     collect(c.cpf) AS cpfs, collect(c.nome) AS nomes
RETURN grupo AS grupo_fraude, tamanho AS membros,
       cpfs[0..6] AS cpfs_do_grupo, nomes[0..6] AS nomes_do_grupo
ORDER BY membros DESC, grupo_fraude
LIMIT 10
```

---

### 10. Ranquear Contas Laranja por Influência — requer notebook 04

Lê o resultado do PageRank gravado no grafo.

| Campo | Valor |
|---|---|
| **Type** | `cypherTemplate` |
| **Name** | `Ranquear Contas Laranja por Influência` |
| **Description** | Use quando o usuário quiser as contas-laranja mais relevantes segundo o algoritmo, e não apenas por volume recebido. Retorna os clientes marcados como conta-laranja com o score de influência (PageRank ponderado por valor), quantos suspeitos distintos transferiram para eles e o total recebido (máx. 10). Retorna vazio se a análise de grafo ainda não foi executada. Para o ranking simples por volume, use "Ranquear Contas que Mais Recebem Pix". |

**Parameters**

| name | data_type | obrigatório | description |
|---|---|---|---|
| `minimo_score` | `number` | **não** | Score mínimo de influência para entrar no ranking. **Opcional**: se omitido, usa `0` e traz os 10 mais influentes. Scores típicos ficam entre 0,15 e 0,6. |

**Cypher**

```cypher
MATCH (laranja:Cliente:ContaLaranja)
OPTIONAL MATCH (fraudador:Cliente:Suspeito)-[t:TRANSFERIU_PIX_PARA]->(laranja)
WITH laranja, count(DISTINCT fraudador) AS fraudadores_ligados, sum(t.valor) AS valor
WHERE laranja.score_laranja >= coalesce($minimo_score, 0.0)
RETURN laranja.cpf AS cpf, laranja.nome AS nome,
       round(laranja.score_laranja, 4) AS score_influencia,
       fraudadores_ligados,
       round(valor, 2) AS valor_recebido_de_suspeitos
ORDER BY score_influencia DESC
LIMIT 10
```

---

### 11. Consultar Grafo Livremente (fallback)

| Campo | Valor |
|---|---|
| **Type** | `text2cypher` |
| **Name** | `Consultar Grafo Livremente` |
| **Description** | Use para contagens, agregações e perguntas exploratórias não cobertas pelas outras ferramentas. Exemplos: "quantas transações existem de cada tipo?", "qual o valor médio de um boleto?", "quantos clientes estão marcados como suspeitos?". NÃO use para buscar um cliente por CPF ou nome — use "Detalhar Perfil do Cliente" ou "Buscar Cliente por Nome". NÃO use para listar anéis ou contas-laranja — use as ferramentas específicas. |

Sem parâmetros: o agente gera o Cypher a partir do schema.

---

## Perguntas para testar o agente

Os dados são gerados aleatoriamente a cada execução do notebook 01, então os nomes
e CPFs da **sua** base serão diferentes. Substitua os marcadores por valores reais
— use as perguntas 1 e 2 para descobri-los.

| # | Pergunta | Exercita |
|---|---|---|
| 1 | *"Quais grupos de fraude existem na base?"* | tool 9 (resultado do WCC) |
| 2 | *"Quem são as contas-laranja mais influentes?"* | tool 10 (resultado do PageRank) |
| 3 | *"Quais e-mails ou RGs estão sendo usados por 3 ou mais clientes?"* | tool 7 |
| 4 | *"Existem clientes compartilhando documentos entre si?"* | tool 3 **sem CPF** |
| 5 | *"Me mostre o anel de fraude do CPF `<CPF DE UM MEMBRO DE ANEL>`"* | tool 4 |
| 6 | *"Quem compartilha identificador com `<NOME DE UM CLIENTE>`?"* | tool 1, depois tool 3 |
| 7 | *"Quais são os maiores fluxos de Pix da base?"* | tool 5 **sem CPF** |
| 8 | *"De quem `<NOME DE UMA CONTA-LARANJA>` recebeu Pix?"* | tool 1, depois tool 6 |
| 9 | *"O cliente `<CPF>` está em algum anel? É conta-laranja?"* | tool 2 |
| 10 | *"Quantas transações existem de cada tipo?"* | tool 11 (fallback) |
| 11 | *"Explique como você chegou nessa resposta"* | modo de explicação |

Três demonstrações que valem o palco:

- **Perguntas 1 e 2** são a prova de que o notebook 04 fez diferença: elas só
  funcionam porque os algoritmos gravaram o resultado no grafo.
- **Perguntas 4 e 6** são a *mesma ferramenta* respondendo à visão global e à
  investigação de um cliente, só pela presença ou ausência do CPF.
- **Pergunta 9** mostra que os dois padrões são independentes: um cliente pode
  estar num anel de identidades sem ser conta-laranja, e vice-versa.

# OLV DISTRIBUIDORA
## Painel de Vendas, Estoque e Financeiro

**Documentação do funcionamento e plano de evolução**

Versão 3.4 · 31/07/2026
Preparado para Rogério

---

## INSTRUÇÕES PARA CLAUDE (leia antes de editar)

Este documento é a fonte única de verdade do projeto. Quando você o mencionar em nova conversa, carregue-o como contexto. Regras de atualização:

1. **Seção com asterisco (*)**: Rogério atualiza via conversa.
2. **Sem asterisco**: já foi decidido e finalizado. Não altere sem permissão explícita.
3. **Status e Andamento**: atualize conforme Rogério relata progresso, com a data do dia.
4. **Falta/Pendência**: quando uma tarefa ficar pronta, mude "Falta" para "Concluído" e inclua a data.
5. **Versão**: sempre que editar, atualize o topo (formato V.X · DD/MM/AAAA).
6. **Modelo indicado**: ao iniciar qualquer tarefa, informe na primeira linha o modelo indicado conforme a seção 12, e avise quando o modelo em uso for mais pesado que o necessário.
7. **Conferência de versão (obrigatória)**: ao carregar este documento, informe na primeira linha a versão lida. Se for menor que a última registrada no log ou que a informada por Rogério, **pare e avise**, porque provavelmente veio de cache. Vale mesmo quando ler o documento não fazia parte do plano da tarefa. Motivo no log da v2.9.
8. **Auditoria de permissão (criada em 29/07/2026)**: qualquer tarefa que altere papel, permissão ou trava de acesso deve auditar **todos os workflows**, não só os que parecem relacionados. Motivo na seção 8.2.

---

## 1. VISÃO GERAL

O projeto é um painel operacional da OLV Distribuidora que roda no navegador, celular ou computador. Centraliza vendas, estoque, clientes e, na evolução planejada, o controle financeiro de contas a pagar e a receber.

O objetivo é dar autonomia para lançar e consultar o dia a dia do negócio em um só lugar, com cálculos automáticos de lucro e de saldo de estoque, sem depender de aplicativo instalado.

**Estado atual**: existe uma versão do painel, acessível pela web, ainda operando sobre a base anterior. Desde 25/07/2026 o acesso é por login individual com papéis. O painel novo, sobre o Supabase, foi construído e publicado em 30/07/2026 em https://painel.olvdistribuidora.com.br (Cloudflare Workers), rodando em paralelo ao painel antigo até a virada.

**Decisão de 25/07/2026**: o sistema é construído do zero sobre o Supabase (PostgreSQL). Não há migração de dados antigos: o sistema começa vazio e vai sendo preenchido pelo uso, com apenas clientes, produtos e usuários inseridos na abertura.

---

## 2. ARQUITETURA

```
Navegador (celular/PC) → n8n (servidor) → Supabase
```

O navegador envia as ações para o n8n. O n8n valida o login e o token de sessão, grava no banco e devolve a resposta. A leitura vem de um endpoint do n8n que consulta o banco e devolve JSON compacto.

O painel funciona 24/7 mesmo com o computador desligado. As dependências são o n8n e o Supabase estarem no ar. Os anexos do financeiro ficarão no Supabase Storage.

**Regra de arquitetura (29/07/2026)**: **o navegador nunca fala direto com o Supabase.** Toda leitura e gravação passa pelo n8n, que é onde a permissão é validada. Ver seções 3.10 e 8.

**Status em 30/07/2026**: conexão n8n para Supabase ativa e testada. Esquema do banco concluído. Endpoints do painel novo construídos e ativados. Painel novo publicado e testado ponta a ponta (login, leitura e telas). Religação dos fluxos antigos e aposentadoria do painel atual seguem como pendência aberta da seção 9.1.

### 2.1 Camada de login e usuários

**Tabela de usuários ("OLV Usuarios")**: Data Table interna do n8n com usuário, nome, senha embaralhada, papel e se está ativo. Manter os acessos apartados dos dados do negócio é mais seguro, como ter a portaria num cômodo separado do cofre.

**Fluxo de Login ("OLV Login")**: recebe usuário e senha, confere na tabela e devolve um token de sessão, um crachá temporário assinado que carrega o papel e o nome e expira em 12 horas.

**Fluxo de Contas ("OLV Contas")**: cria usuário, troca senha, lista, muda papel e ativa/desativa acessos. Ações de administração exigem papel de administrador.

**HTML do painel**: hoje fica codificado em base64 numa Data Table chamada "OLV Painel HTML", para o painel antigo. **O painel novo já nasceu diferente** (ver 9.2): é um arquivo HTML único, versionado e editável, publicado direto no Cloudflare Workers, sem passar pela Data Table.

**Os endpoints do painel novo usam o mesmo token e o mesmo segredo do OLV Login.** Quem entra no painel antigo entra no novo com o mesmo crachá, o que permite testar em paralelo sem criar segunda camada de acesso.

---

## 3. ESTRUTURA DE DADOS (SUPABASE)

Cada assunto vive em uma tabela própria, ligada às outras por um código (ID). Oito tabelas e duas views. Tranca de segurança (RLS) ativa em todas.

**Reestruturação de 30/07/2026**: a venda deixou de ser uma linha por produto e passou a ser um **pedido com vários itens**. Motivo na seção 3.4.

### Como ler as tabelas

- **ID**: número que o banco gera sozinho, sem repetir. Identidade permanente do registro.
- **Obrigatório**: o banco recusa a gravação se vier vazio.
- **Calculada**: o banco calcula sozinho. Não é digitada e não sai errada.
- **Validado**: lista fechada de valores. O banco recusa qualquer coisa fora dela.

### 3.1 clientes

| Campo | Tipo | Regra |
|-------|------|-------|
| id | ID | Chave do registro |
| nome | Texto | Obrigatório |
| endereco, bairro, telefone | Texto | Opcionais |
| telefone_norm | Texto | **Gerado pelo banco**: o telefone só com dígitos. **Único**, impede cliente duplicado |
| observacao | Texto | Opcional |
| criado_em | Data e hora | Preenchido sozinho |

**Trava de duplicidade**: o banco cria sozinho uma versão do telefone só com números e não permite dois clientes com o mesmo. Assim "(27) 99999-8888" e "27999998888" são o mesmo contato.

Dois limites aceitos: cliente sem telefone não é protegido, porque o banco não tem como saber se é a mesma pessoa; e telefone compartilhado bloqueia o segundo cadastro, o que na prática costuma ser o mesmo cliente de entrega.

**Resolvido em 31/07/2026**: o endpoint `olv2-dados` (ação `tudo`) não incluía `telefone_norm` na consulta de clientes, só `telefone`. O nó `Consultar Clientes T` do workflow OLV2 Dados foi ajustado para incluir `telefone_norm` no SELECT, e o workflow foi publicado. O painel novo ainda extrai os dígitos do campo `telefone` no navegador como caminho alternativo; simplificar essa lógica do lado do cliente para usar o campo pronto é uma limpeza opcional, sem urgência.

### 3.2 produtos

| Campo | Tipo | Regra |
|-------|------|-------|
| id | ID | Chave |
| nome | Texto | Obrigatório e **único** |
| unidade | Texto | Opcional |
| custo_atual | Número | **Custo médio ponderado**, atualizado sozinho pelas entradas de estoque |
| custo_ultima_compra | Número | Preço da última entrada, para decisão de preço de venda |
| custo_atualizado_em | Data | Data da última entrada que mexeu no custo |
| estoque_minimo | Número | Começa em 0 |
| ativo | Sim/Não | Permite aposentar um produto sem apagar o histórico |

**Cadastro inicial (30/07/2026)**, com custo e mínimo da planilha do Rogério:

| Produto | Custo inicial | Estoque mínimo |
|---|---|---|
| Gás 13kg | 87,00 | 30 |
| Água 20L | 7,50 | 50 |
| Garrafão Vazio | 17,50 | 80 |
| Botija Vazia | 115,00 | 50 |
| Regulador Médio | 33,09 | 15 |
| Mangueira 1,2m | 9,98 | 15 |

### 3.3 vendas (o pedido)

| Campo | Tipo | Regra |
|-------|------|-------|
| id | ID | Chave |
| cliente_id | Ligação | Aponta para clientes |
| data | Data | Começa com hoje |
| status | Texto | Obrigatório, **validado**: Aguardando, Em rota, Concluído |
| tipo_entrega | Texto | Obrigatório, **validado**: Entrega ou Retirada |
| pre_pago | Sim/Não | Marca o pedido pago antes do resgate |
| data_conclusao | Data | **Preenchida pelo banco** quando vira Concluído. Volta a ficar vazia se o status recuar |
| desconto | Número | Desconto do pedido inteiro. Começa em 0 |
| valor_total | Número | **Mantido pelo banco**: soma dos itens menos o desconto |
| idempotency_key | Texto | **Único**. Impede pedido duplicado por toque duplo ou reenvio de rede |
| observacao, responsavel | Texto | Opcionais |

**Dois campos, duas perguntas.** O pedido tem eixos independentes; misturá-los num campo só impede representar situações reais.

| Campo | Valores | Responde |
|---|---|---|
| tipo_entrega | Entrega, Retirada | Como o pedido sai |
| status | Aguardando, Em rota, Concluído | Em que etapa está |

Fluxo por tipo: Entrega segue Aguardando, Em rota, Concluído. Retirada segue Aguardando, Concluído. O banco recusa Retirada em rota, porque pedido de balcão não sai para rota.

**Pedido pré-pago**: o cliente paga hoje e resgata depois, podendo levar meses. É um terceiro eixo, independente dos outros dois.

| Momento | Movimenta | Não movimenta |
|---|---|---|
| Pedido criado e pago | Financeiro | Estoque |
| Pedido concluído | Estoque, na data da conclusão | Financeiro |

É por isso que existe a `data_conclusao`. Sem ela, o estoque cairia no dia do pagamento com o produto ainda no depósito.

**Sobre o campo `responsavel`**: ele guarda **quem lançou** o pedido, não quem entrega. Não existe hoje campo de entregador. Isso tem efeito na pendência 11, seção 8.3.

### 3.4 venda_itens (criada em 30/07/2026)

| Campo | Tipo | Regra |
|-------|------|-------|
| id | ID | Chave |
| venda_id | Ligação | Aponta para vendas. Obrigatório, com cascata |
| produto_id | Ligação | Aponta para produtos. Obrigatório |
| quantidade | Número | Obrigatório, maior que zero |
| valor_unitario | Número | Obrigatório |
| custo_unitario | Número | Copiado do cadastro do produto no momento da venda |
| valor_total | **Calculada** | quantidade × valor_unitario |
| custo_total | **Calculada** | quantidade × custo_unitario |
| lucro | **Calculada** | valor_total − custo_total, **bruto, sem o desconto do pedido** |

**Por que essa tabela nasceu.** Até 30/07 a venda tinha um produto só, herança da planilha, onde cada linha era um produto. Um cliente que comprava gás e água gerava dois registros de venda.

A prova estava nos próprios números de julho: as 835 vendas do mês eram exatamente a soma das vendas por produto (400 + 414 + 16 + 3 + 1 + 1). Cada "venda" era uma linha de produto, não um pedido.

**O que melhora**: o ticket médio passa a ser o valor de um pedido, não de uma linha; o número de vendas passa a ser número de pedidos; e o pagamento dividido finalmente faz sentido, porque a divisão é do pedido inteiro.

**O custo é copiado, não referenciado.** Cada item guarda o custo do produto no dia da venda. Se o custo mudar amanhã, o histórico não muda. Isso também encerra a pendência antiga de vendas lançadas sem custo: o campo saiu do formulário e o valor vem sozinho do cadastro.

**Atenção ao lucro por item.** A coluna `lucro` de `venda_itens` não conhece o desconto do pedido. Para o lucro correto, use a view `vw_pedidos` (seção 3.9).

### 3.5 pagamentos

Uma venda pode ter várias linhas aqui, o que permite pagamento dividido.

| Campo | Tipo | Regra |
|-------|------|-------|
| id | ID | Chave |
| venda_id | Ligação | Obrigatório, com cascata |
| forma | Texto | Obrigatório, **validado** com as 8 formas abaixo |
| valor | Número | Obrigatório, não negativo |

**As 8 formas de pagamento (definidas em 30/07/2026):**

| Forma | Gera conta a receber | Vencimento |
|---|---|---|
| Crediário | **Sim** | informado no formulário, obrigatório |
| Crédito | Não, ver Fase 2 | recebível de terceiro |
| Débito | Não, ver Fase 2 | recebível de terceiro |
| Dinheiro | Não | entra no caixa |
| Em aberto | **Sim** | data da venda + 3 dias |
| Gás do Povo | Não, ver Fase 2 | repasse do programa em 3 dias úteis |
| Gratuidade | Não | zera o pedido, ver abaixo |
| Pix | Não | entra no caixa |

**Crédito, Débito e Gás do Povo formam uma família**: dinheiro que já é da empresa mas ainda não chegou, com um terceiro no meio. Ficam para a Fase 2 e serão tratados juntos. Na Fase 1 gravam em `pagamentos` normalmente, só não geram recebível.

**Gratuidade (decidido em 30/07/2026)**: o pedido entra com `valor_total` zero. Baixa estoque normalmente e o custo aparece como prejuízo daquele lançamento. Não entra no faturamento nem no caixa. A função de gravação zera os valores sozinha quando essa forma é escolhida, inclusive o valor do recebível gerado por uma linha de Crediário no mesmo pedido (corrigido em 31/07/2026, ver pendência 14 na seção 3.11).

**Pré-pago não gera recebível**, porque o dinheiro já entrou.

### 3.6 estoque_movimentacoes

| Campo | Tipo | Regra |
|-------|------|-------|
| id | ID | Chave |
| produto_id | Ligação | Obrigatório |
| tipo | Texto | **Validado**: Entrada, Ajuste ou Estoque Inicial |
| quantidade | Número | Obrigatório |
| data | Data | Começa com hoje |
| custo_unitario | Número | Informado nas Entradas. **Alimenta o custo do produto** |
| fornecedor | Texto | Opcional. Serve depois para ligar com contas a pagar |
| responsavel, observacao | Texto | Opcionais |

### 3.7 contas_pagar

| Campo | Tipo | Regra |
|-------|------|-------|
| id | ID | Chave |
| fornecedor, descricao, categoria | Texto | Opcionais |
| valor | Número | Obrigatório |
| vencimento, pago_em | Data | Opcionais |
| status | Texto | **Validado**: Em aberto ou Pago |
| anexo_url | Texto | Endereço do boleto no Supabase Storage |

### 3.8 contas_receber

| Campo | Tipo | Regra |
|-------|------|-------|
| id | ID | Chave |
| cliente_id | Ligação | Aponta para clientes |
| venda_id | Ligação | Aponta para vendas, **com cascata** |
| pagamento_id | Ligação | **Aponta para a linha de pagamento que gerou o recebível**, com cascata |
| descricao | Texto | Preenchida pela função |
| valor | Número | Obrigatório |
| vencimento, recebido_em | Data | |
| status | Texto | **Validado**: Em aberto ou Recebido |

**A regra estrutural: quem gera recebível é a linha de pagamento, não a venda.** Um pedido de R$ 220 pago com R$ 150 em dinheiro e R$ 70 em crediário gera **um** recebível de R$ 70. Sem o `pagamento_id`, um pedido com duas linhas de crediário teria dois recebíveis indistinguíveis, e a baixa ficaria ambígua.

### 3.9 Como as tabelas se ligam, e as duas views

```
clientes ──→ vendas (pedido) ──┬──→ venda_itens ──→ produtos
                               └──→ pagamentos ──→ contas_receber

produtos ──→ estoque_movimentacoes
contas_pagar (independente)
```

Três níveis: um pedido, N itens, N pagamentos. É o desenho padrão de sistema de vendas.

O banco impede apagar cliente ou produto que tenha lançamento ligado, o que protege o histórico contra exclusão acidental. Já apagar um pedido leva junto seus itens, pagamentos e recebíveis, por cascata.

**`vw_pedidos`**: o pedido pronto para leitura, com cliente, itens, formas de pagamento e, principalmente, **lucro e margem já descontados**. O painel lê daqui, não das tabelas cruas.

**A `vw_pedidos` não traz os itens do pedido.** Ela diz "3 itens, 5 unidades", mas não diz quais produtos. Por isso o endpoint de leitura faz **uma segunda consulta** em `venda_itens` com a lista de pedidos do período. Decisão de 30/07/2026, tomada por auditabilidade: numa lista plana, o filtro de custo e lucro remove campos nomeados e se confere a olho nu. Num bloco JSON aninhado por pedido, repassar o bloco inteiro por descuido é uma linha de código que ninguém percebe olhando a tela. É a assinatura do incidente da seção 8.2.

**`vw_estoque_atual`**: saldo por produto. Reconstruída em 30/07/2026, porque abatia estoque na data do pedido em vez da conclusão, contrariando a regra da seção 6. Agora só conta pedido concluído, pela `data_conclusao`.

### 3.10 Travas do banco

Regras que o banco garante sozinho, independente de quem grave:

| Trava | O que impede |
|---|---|
| `valor_total` mantido por gatilho | Total do pedido divergir da soma dos itens menos desconto |
| Trava adiada de consistência | Pedido sem nenhum item, ou soma dos pagamentos diferente do total |
| `idempotency_key` única | Pedido duplicado por toque duplo ou reenvio |
| Lista fechada em `forma` | Forma de pagamento inventada por erro de digitação |
| Lista fechada em `status` e `tipo_entrega` | Estado inválido de pedido |
| Retirada não pode estar Em rota | Pedido de balcão sair para rota |
| `telefone_norm` único | Cliente duplicado |
| Cascatas | Pagamento e recebível órfãos |

**Sobre a trava adiada**: ela confere no fim da transação, não a cada linha. Sem isso, dispararia ao gravar a primeira das duas linhas de pagamento, quando a soma ainda não bate.

**Função `registrar_venda`**: grava pedido, itens, pagamentos e recebíveis **numa transação só**. O n8n a chama uma única vez. Como o n8n não tem transação entre nós, gravar em etapas deixaria uma janela em que o pedido existe sem pagamento. Se qualquer parte falhar, nada é gravado.

**Por que não existe edição de valores de pedido (30/07/2026).** Cada nó do n8n é uma transação própria, que fecha sozinha. Alterar a quantidade de um item dispara o recálculo do total e, no fim daquela transação, a trava adiada compara com a soma dos pagamentos e recusa. O nó seguinte, que corrigiria o pagamento, nunca chega a rodar. Não é arriscado: **não roda**. Editar valores exige uma função `atualizar_venda` no banco, no mesmo molde da `registrar_venda`, e antes disso exige uma decisão de negócio: se a função preserva o custo original de cada item ou reestampa o custo do dia da edição. As duas escolhas mudam o lucro de meses já fechados.

**RLS**: tranca ativa nas 8 tabelas, sem nenhuma política liberando acesso. As chaves públicas do Supabase ficam totalmente bloqueadas nas tabelas; o n8n conecta com usuário que ignora a tranca. **Atenção**: essa proteção vale para as tabelas, mas não para as views e funções SECURITY DEFINER — ver pendência 16 na seção 8.3, descoberta em 31/07/2026. **A proteção por papel depende inteiramente da validação dentro do n8n.** Se um dia o painel acessar o banco direto, será obrigatório escrever políticas antes.

### 3.11 Pendências e decisões em aberto *

| # | Ponto | Situação |
|---|---|---|
| 1 | ~~Forma de pagamento saiu de vendas~~ | **Resolvido em 30/07/2026**: função `registrar_venda` grava tudo numa transação |
| 2 | ~~clientes.nome não é único~~ | **Resolvido em 26/07/2026**: trava pelo telefone normalizado |
| 3 | ~~vendas.status aceita qualquer texto~~ | **Resolvido em 26/07/2026**: três valores validados |
| 4 | ~~Soma dos pagamentos não é garantida~~ | **Resolvido em 30/07/2026**: trava adiada no banco |
| 5 | **Não existe tabela de caixa nem de usuários no Supabase** | Caixa é a etapa 9.5. O login segue na Data Table do n8n; falta decidir se migra |
| 6 | ~~Gatilho da baixa de estoque~~ | **Resolvido em 26/07/2026**: baixa na conclusão |
| 7 | ~~pagamentos.forma não é validada~~ | **Resolvido em 30/07/2026**: 8 formas em lista fechada |
| 8 | ~~contas_receber não aponta para o pagamento~~ | **Resolvido em 30/07/2026**: campo `pagamento_id` |
| 9 | **Recebíveis de terceiros** | Crédito, Débito e Gás do Povo serão tratados juntos na Fase 2 |
| 10 | **Lucro por item ignora o desconto** | Contornado pela view `vw_pedidos`. Se um dia o desconto precisar ser rateado item a item, é aqui |
| 13 | **Excluir ou editar movimentação de estoque não desfaz o custo médio** | Descoberto em 30/07/2026. O gatilho `trg_custo_produto` dispara **só no INSERT**. Apagar uma Entrada devolve a quantidade ao saldo, mas deixa o `custo_atual` num valor que nenhuma compra real justifica, e o erro é invisível na tela. Por isso o endpoint de estoque **não tem exclusão**: correção se faz por Ajuste, que deixa rastro. Resolver exige recálculo no banco — decisão de regra financeira, fica para sessão em Opus 5 |
| 14 | ~~Gratuidade combinada com Crediário gera recebível com valor cheio~~ | **Resolvido em 31/07/2026**: função `registrar_venda` corrigida para zerar também `contas_receber.valor` nesse caso. Testado com pedido real (Gratuidade + Crediário), recebível confirmado em R$ 0,00, dado de teste removido |
| 15 | ~~Endpoint `olv2-dados` não devolve `telefone_norm`~~ | **Resolvido em 31/07/2026**: nó `Consultar Clientes T` do OLV2 Dados passou a incluir `telefone_norm` no SELECT; workflow publicado |

A numeração 11, 12 e 16 fica na seção 8.3, porque são pendências de segurança. A série é única.

---

## 4. OS WORKFLOWS N8N

### 4.1 Em produção, sobre a base anterior

| Workflow | O que faz | Endpoint |
|---|---|---|
| OLV Painel Mobile (web) | Serve a página e o endpoint de leitura | /webhook/olv-painel e /webhook/olv-dados |
| OLV Vendas – Lançamento | Cria, edita e exclui vendas | POST /webhook/olv-venda |
| OLV Estoque – Lançamento | Movimentações de estoque | POST /webhook/olv-estoque |
| OLV Login | Autentica e devolve o token de 12h | POST /webhook/olv-login |
| OLV Contas | Gestão de usuários, só administrador | POST /webhook/olv-contas |

**Confirmado ativo em 31/07/2026** (validação técnica da virada, seção 9.1): os cinco continuam ativos e no ar, ponto de retorno intacto.

### 4.2 Do painel novo, sobre o Supabase (construídos em 30/07/2026)

Prefixo OLV2 para não haver confusão com o conjunto em produção. **Ativados em 30/07/2026**, junto com a correção de CORS descrita na seção 9.2. O painel antigo continua intocado e no ar; os dois conjuntos rodam em paralelo até a virada.

| Workflow | Endpoint | Ações |
|---|---|---|
| OLV2 Dados (painel novo) | POST /webhook/olv2-dados | `tudo` (carga inicial) e `pedidos` (recarga leve) |
| OLV2 Pedido (painel novo) | POST /webhook/olv2-pedido | `criar`, `status`, `editar_leve`, `excluir` |
| OLV2 Estoque (painel novo) | POST /webhook/olv2-estoque | `lancar`, `historico` |
| OLV2 Clientes (painel novo) | POST /webhook/olv2-clientes | `criar`, `editar`, `buscar` |

**Confirmado ativo em 31/07/2026** (validação técnica da virada, seção 9.1): os quatro continuam ativos e no ar.

**Bloco de entrada idêntico nos quatro**: Normalizar, Autenticar, Assinar, Autorizar, Autorizado?, Perfil, Rotear. O nó `Perfil` calcula uma coisa só, `isAdmin = papel === 'admin'`, e nenhum ponto dos quatro compara papel contra rótulo. O nó Crypto tem `type: SHA256` e `encoding: hex` fixados explicitamente desde o rascunho, que é a armadilha descrita na seção 8.2.

**O que cada ação faz.**

`criar` monta o pacote com venda, itens e pagamentos e chama a `registrar_venda` **uma vez só**. `status` altera apenas a situação e é liberado a qualquer usuário ativo, porque é operacional e reversível. `editar_leve` cobre `status`, `tipo_entrega`, `observacao` e `data`, restrito ao dono do lançamento ou ao administrador. `excluir` tem a mesma restrição. Valores de item, desconto, cliente e pré-pago ficam de fora, pelo motivo da seção 3.10.

**A permissão de dono é conferida dentro da própria instrução de gravação**, não num nó separado que lê antes e grava depois. Assim não existe janela entre conferir e gravar. Quando o pedido não é encontrado ou o usuário não é o responsável, a resposta é a mesma mensagem, e nada é alterado.

**Testado contra o banco real em 30/07/2026**, com limpeza conferida depois: pedido completo com recebível, reenvio com a mesma chave devolvendo `repetido` sem duplicar, edição bloqueada para quem não é dono e liberada para o dono, e cliente duplicado barrado com o nome do cliente já cadastrado na mensagem.

**Correção de CORS em 30/07/2026**: nenhum dos cinco webhooks usados pelo painel novo (OLV Login e os quatro OLV2) tinha a opção `allowedOrigins` configurada no nó de webhook. Sem isso, o navegador bloqueava toda chamada feita a partir de `https://painel.olvdistribuidora.com.br`, mesmo com o restante do fluxo correto. Corrigido nó a nó, liberando essa origem específica (não `*`), e publicado em cada um dos cinco workflows. Detalhe completo na seção 9.2.

**Pendente**: os fluxos em produção ainda leem e gravam na base anterior, e continuam intocados. A virada só acontece na validação da Fase 1, e o conjunto antigo permanece como ponto de retorno.

**Arquivado**: `TEMP - Leitura OLV Painel HTML`, usado na renomeação de 28/07. Não roda mais, mas continua guardado na seção de arquivados, porque as ferramentas usadas não têm exclusão definitiva.

---

## 5. O PAINEL (FUNCIONALIDADES ATUAIS)

### 5.1 Vendas

Filtros de período, indicadores do período, quantidade por produto, valores por forma de pagamento, gráfico de evolução, formulário com autocompletar de cliente, editar e excluir com confirmação, filtros de consulta.

### 5.2 Estoque

Tabela de estoque atual por produto, lançamento de Entrada, Ajuste e Estoque Inicial, histórico com editar e excluir.

### 5.3 Login, usuários e papéis

Login com usuário e senha, crachá com nome e papel, troca de senha obrigatória no primeiro acesso, área de Usuários só para administrador, responsável preenchido automaticamente, trava de dono conferida também no servidor.

**Não existe hoje**: redefinição de senha de outro usuário pelo administrador. Se alguém esquecer a senha, o caminho é criar usuário novo ou alterar na Data Table. Registrado como melhoria desejada.

---

## 6. REGRAS DE CÁLCULO

### Estoque atual

Estoque atual = último Estoque Inicial + entradas ± ajustes − itens de pedidos concluídos, considerando apenas datas iguais ou posteriores à contagem inicial.

**Gatilho da baixa**: a venda abate estoque quando o pedido é **concluído**, pela data da conclusão. Pedido lançado não é produto que saiu. Num pré-pago a diferença entre pagar e resgatar pode ser de meses.

**O que o número significa**: produto disponível no depósito hoje. Pedidos Aguardando e Pré-pagos não abateram, porque o produto continua lá.

**Ponto de atenção**: pedidos Em rota também não abatem, embora o produto já esteja no caminhão. É proposital, para que uma entrega não realizada não precise de estorno. Durante a rota o estoque do sistema fica um pouco acima do físico da loja.

### Custo do produto (definido em 30/07/2026)

**O custo não é digitado no cadastro. Ele nasce da compra.**

Ao lançar uma entrada de estoque, você informa quanto pagou. O banco calcula e atualiza o produto sozinho. O número vem da nota fiscal em vez da memória, e cada entrada guarda o preço daquele dia, formando o histórico de custo.

**Método: custo médio ponderado.**

```
custo médio novo = (valor do estoque atual + valor da compra)
                   ─────────────────────────────────────────
                   (quantidade atual        + quantidade comprada)
```

Exemplo real testado: 20 botijões a R$ 85 mais 60 a R$ 87 resulta em R$ 86,50. Vender não altera o custo médio; só comprar altera.

Quando o preço cai por bater meta, o custo médio desce aos poucos, conforme o estoque antigo e mais caro vai sendo vendido. Isso é proposital: aqueles botijões custaram mais mesmo.

**Dois números, duas perguntas.** O `custo_atual` responde "quanto custou o que estou vendendo" e é a base do lucro. O `custo_ultima_compra` responde "quanto custa repor" e serve para decidir preço de venda. A tela de produtos mostra os dois.

**Estoque Inicial zera a conta**: numa contagem física, o custo informado substitui a média, sem misturar com o histórico anterior.

**O custo se forma na entrada e não se desfaz na exclusão (30/07/2026).** O gatilho que atualiza o custo dispara **só quando a movimentação é criada**. Apagar ou editar uma Entrada depois devolve a quantidade ao saldo, mas **não desfaz o efeito no custo médio**. O produto fica com um custo que nenhuma compra real justifica, e nada na tela denuncia isso. É como rasgar a nota fiscal e esperar que o preço volte sozinho: o papel some, o número não. Por esse motivo o endpoint de estoque não oferece exclusão de movimentação. **Correção de saldo se faz por Ajuste**, que deixa rastro e não mexe em custo. Ver pendência 13.

### Lucro

Lucro do pedido = valor_total (já com desconto) − soma dos custos dos itens. Calculado pela view `vw_pedidos`.

Lucro do item = valor do item − custo do item, **bruto, sem o desconto do pedido**. Serve para saber qual produto rende mais.

---

## 7. ACESSO E SEGURANÇA ATUAL

Acesso por https://n8n-wmtt.srv1830312.hstgr.cloud/webhook/olv-painel, com login individual. A chave única OLV2026 foi removida.

### Papéis

**Administrador**: vê tudo, edita e exclui qualquer lançamento, gerencia usuários e acessa o Dashboard.

**Colaborador**: vê o operacional, sem faturamento, lucro nem margem; só edita e exclui os próprios lançamentos; **não acessa o Dashboard** (definido em 29/07/2026).

**Contas iniciais**: Rogério e Gabriele administradores, e a conta Vendedor como colaborador.

### Permissões por tipo de ação (definidas em 30/07/2026)

A proteção fica onde está o risco, não onde está a tela.

| Ação | Quem pode | Motivo |
|---|---|---|
| Mudar situação do pedido | Qualquer usuário ativo, em qualquer pedido | Operacional e reversível |
| Editar campos leves ou excluir pedido | Dono do lançamento ou administrador | Irreversível na prática |
| Cadastrar e editar cliente, buscar cliente | Qualquer usuário ativo | Operacional, sem dado financeiro |
| **Entrada e Estoque Inicial** | **Só administrador** | **Alimentam o custo médio, que é a base do lucro** |
| **Ajuste de estoque** | **Qualquer usuário ativo** | **Corrige saldo e não toca em custo** |

**Por que Entrada é exclusiva do administrador.** O custo unitário é um dos sete pontos financeiros protegidos da seção 8.1. Se o colaborador não pode informar custo, uma Entrada lançada por ele entraria sem custo e não atualizaria a média, deixando o lucro errado sem aviso. Fechar a Entrada e abrir o Ajuste mantém o número do custo confiável sem travar a operação do dia a dia.

### Como as senhas e a sessão são protegidas

**Senha embaralhada (hash)**: a senha nunca é guardada como texto. Guardamos um resumo irreversível (SHA-256) somado a um sal, um tempero aleatório único por usuário que impede que senhas iguais gerem o mesmo código.

**Token assinado**: o crachá é validado por uma assinatura que só o servidor gera. Crachá forjado não passa.

**Sessão com validade**: expira em 12 horas.

**Nota técnica**: este servidor n8n bloqueia criptografia dentro do nó de código, então o hash da senha e a assinatura do token usam o nó Crypto nativo, com SHA-256 e segredo interno.

---

## 8. CONVENÇÕES E CUIDADOS

- Cada registro é identificado por um ID próprio do banco, estável e independente da ordem na tela.
- O endpoint de leitura usa cabeçalho no-store e parâmetro anti-cache, para o painel sempre puxar dados frescos.
- **Planilha corrompe telefone (30/07/2026)**: ao editar telefone em Excel ou Sheets, formate a coluna como texto antes. Caso contrário o programa converte para número, come o zero da frente ou acrescenta um no fim. Aconteceu com 15 registros na importação de clientes.
- **Toda gravação viaja em base64 (30/07/2026)**: ver 8.4.

### 8.1 Como o painel esconde os dados financeiros

Mecanismo indireto, em duas etapas, que já causou incidente. Quem for mexer em papéis precisa ler antes.

**No navegador**: se o papel for exatamente `'colaborador'`, o HTML aplica a classe CSS `vend` no `<body>`, e a regra `body.vend .hideVend{display:none}` esconde os elementos marcados.

**Os 7 pontos protegidos**: card Faturamento, card Lucro, bloco Valores por forma de pagamento, botão de alternar o gráfico para R$, campo Custo unitário, dica de lucro e colunas de faturamento nas tabelas.

**No servidor**: o nó `Montar Payload` decide se envia lucro e custo. Desde 29/07/2026 a decisão é `papel != 'admin'`.

**Três avisos:**

1. Os nomes internos `EH_VEND`, `.vend` e `.hideVend` **não foram renomeados**. Não confunda o nome interno com o rótulo.
2. **Esconder no navegador não é controle de acesso.** O CSS só deixa de exibir; o dado, se enviado, continua no navegador e pode ser lido com as ferramentas de desenvolvedor.
3. O `Valor Total` de cada venda **é enviado para qualquer papel**. Ver pendência 11.

**Nos endpoints do painel novo o mecanismo é outro.** Não existe decisão espalhada por nó: cada endpoint tem um nó chamado `Filtrar Saída` que roda depois da consulta e antes de responder, e remove campos nomeados de listas planas.

| Origem | Removido quando o papel não é admin |
|---|---|
| `vw_pedidos` | `custo_total`, `lucro`, `margem_pct` |
| `venda_itens` | `custo_unitario`, `custo_total`, `lucro` |
| `produtos` | `custo_atual`, `custo_ultima_compra`, `custo_atualizado_em` |
| `estoque_movimentacoes` | `custo_unitario` |
| Totais agregados | **Não são sequer calculados** |

### 8.2 Incidente de 28/07/2026 e correção

A renomeação de 28/07 auditou OLV Vendas e OLV Estoque procurando comparações negativas de papel, e concluiu corretamente que ali as travas eram positivas sobre `admin`. Mas **não auditou o OLV Painel Mobile**, onde vive o endpoint de leitura.

Naquele workflow havia a única comparação negativa do lado servidor: `papel === 'vendedor'`. Com o papel renomeado, ela deixou de ser verdadeira, e o servidor **passou a enviar lucro e custo de todas as vendas para o colaborador** por cerca de 16 horas. A tela continuava correta, porque o CSS escondia, mas o dado trafegava.

**Corrigido em 29/07/2026**: a comparação virou `papel != 'admin'`, publicado e verificado no navegador com login de colaborador.

**Ajuste adicional na mesma publicação**: o nó `Assinar D` tinha `type: SHA256` e `encoding: hex` só na versão publicada, ausentes no rascunho. Publicar sem tratar quebraria a validação de todos os tokens de sessão. Os valores foram fixados explicitamente.

**Regra que fica**: toda verificação de permissão no servidor deve ser **positiva sobre `admin`**. Assim qualquer papel novo nasce restrito por padrão. Comparação negativa contra um rótulo específico é proibida.

**Conferido em 30/07/2026** nos quatro endpoints novos: nenhuma comparação negativa de papel, e o nó Crypto com `type` e `encoding` gravados desde o rascunho.

### 8.3 Pendências de segurança

| # | Ponto | Situação |
|---|---|---|
| 11 | **Valor Total enviado a todos os papéis** | Ver detalhamento abaixo |
| 12 | ~~Ticket médio exposto ao colaborador~~ | **Resolvido em 30/07/2026**: o indicador saiu do painel novo, e o Dashboard passou a ser exclusivo do administrador |
| 16 | **Views e funções acessíveis direto pela API do Supabase, sem passar pelo n8n** | Ver detalhamento abaixo |

**Pendência 11, detalhada.**

*Reduzido em 30/07/2026*: nos endpoints do painel novo, os totais agregados (faturamento, lucro, margem, itens vendidos) **não são calculados** para quem não é admin. Não é que existam escondidos: eles não existem na resposta. O `valor_total` do pedido individual continua saindo, por necessidade operacional, porque o entregador precisa saber quanto cobrar na porta do cliente.

*Por que segue aberta*: como o valor individual sai, o faturamento do período continua derivável por soma nas ferramentas de desenvolvedor. A exposição diminuiu, não acabou.

**Direção decidida em 30/07/2026**: o colaborador passará a receber o valor apenas dos pedidos **atribuídos a ele em rota**, mais os que ele mesmo lançou.

**Execução na etapa 9.7**, junto com setores, porque muda o modelo de permissão e a 9.7 já determina que isso vem depois da religação. Fazer agora significaria escrever regra de permissão duas vezes, que é a forma exata de repetir o incidente de 28/07.

*Depende de resolver quatro coisas*:

| Ponto | O que falta |
|---|---|
| Campo de entregador | Não existe. O `responsavel` guarda quem **lançou**, não quem entrega. São pessoas diferentes: uma lança no balcão, outra leva |
| Momento da atribuição | Definir se o administrador atribui ou se o entregador se atribui ao pegar a rota |
| Retirada | Nunca fica Em rota. Quem atende o balcão precisa do valor para cobrar, e não há a quem atribuir |
| Aguardando | Ainda não tem entregador. Sem valor nenhum, o colaborador não confere o que digitou |

**Limite aceito**: quem lança continua vendo o valor do que lançou, e isso não tem como tirar sem quebrar a conferência. Se uma pessoa concentra os lançamentos, o faturamento segue parcialmente derivável.

**Pendência 16, detalhada.**

Descoberta em 31/07/2026, durante a validação técnica da virada (seção 9.1). O auditor de segurança do Supabase apontou que duas views (`vw_pedidos`, `vw_estoque_atual`) e cinco funções (`registrar_venda`, `atualiza_custo_produto`, `checa_venda_consistente`, `recalc_total_desconto`, `recalc_total_venda`) foram criadas como **SECURITY DEFINER**, o que faz elas ignorarem a RLS das tabelas de origem. Ao mesmo tempo, os papéis públicos do Supabase — `anon` (sem login) e `authenticated` — têm `SELECT` nas views e `EXECUTE` nas funções liberados.

Na prática, isso significa que hoje, sem passar pelo painel nem pelo n8n:

- Qualquer pessoa com a URL do projeto e a chave pública do Supabase (que é pública por natureza, não um segredo) consegue ler `vw_pedidos` direto pela API REST e ver lucro, custo e margem de todo pedido — o mesmo dado que o mecanismo da seção 8.1 se esforça para esconder do colaborador.
- Qualquer pessoa consegue chamar `registrar_venda` direto pela API REST e criar pedidos, sem token, sem autenticação e sem as validações do n8n.

A RLS das 8 tabelas está correta (bloqueia acesso direto, sem política = acesso negado por padrão), mas as views e funções SECURITY DEFINER contornam essa trava. Isso contradiz a regra da seção 2 ("o navegador nunca fala direto com o Supabase"): a brecha independe do painel, existe hoje, e fica mais grave depois da virada, quando o Supabase vira a fonte única de verdade.

**Nada foi alterado no banco.** A correção (revogar `SELECT`/`EXECUTE` de `anon` e `authenticated` nessas views e funções, e por precaução em todas as tabelas do schema, já que o n8n nunca usa a API REST) é rápida, mas por alterar trava de acesso, a decisão fica para sessão em Opus 5 (regra 8, seção 12). **A aprovação final da virada (seção 9.1) fica condicionada a essa decisão.**

### 8.4 Toda gravação viaja em base64 (30/07/2026)

O nó Postgres do n8n separa os parâmetros da consulta **por vírgula**. Não há como escapar uma vírgula dentro de um valor. Qualquer observação de pedido com vírgula quebraria a gravação, e texto livre quase sempre tem vírgula.

**Solução adotada**: cada gravação envia **um único parâmetro**, que é o pacote JSON inteiro codificado em base64. O SQL decodifica na primeira linha da consulta e lê os campos de dentro. O alfabeto do base64 não tem vírgula, então o problema desaparece na origem.

É como despachar uma encomenda lacrada em vez de espalhar os itens soltos na esteira: o que vai dentro não interfere no transporte.

**Efeito colateral bom**: nada do que o usuário digita entra na consulta como texto. Fecha a porta para injeção de SQL sem esforço adicional.

**Testado em 30/07/2026** com o texto "teste tecnico, com virgula", gravado e lido corretamente.

---

## 9. PLANO DE EVOLUÇÃO *

Ciclo de cada etapa: definir o escopo, implementar sem afetar o que funciona, testar com dados reais incluindo erro, você validar, publicar. Antes de cada etapa que mexe no painel, guardamos cópia da versão anterior.

### 9.1 Fundação: Base de dados no Supabase *

**Status**: em andamento desde 25/07/2026.

#### Concluído

- 25/07: banco criado (projeto olv-distribuidora_sistema, região São Paulo), 7 tabelas, RLS ativa, n8n conectado pelo Session Pooler e testado ponta a ponta.
- 26/07: esquema especificado, índices e RLS verificados.
- 30/07: **modelo de pedido com itens**, 8 formas de pagamento, recebíveis automáticos, custo médio ponderado, função `registrar_venda`, travas de consistência e as duas views. Tudo testado com casos de borda e o banco limpo depois.
- 30/07: **6 produtos cadastrados** com custo e estoque mínimo.
- 30/07: **910 clientes importados**, conferidos sem campo vazio, sem telefone inválido e com 910 telefones únicos.
- 30/07: **quatro endpoints do painel novo construídos**, testados contra o banco real com limpeza conferida. Ver seção 4.2.
- 30/07: **painel novo ligado aos quatro endpoints e ao login**, testado com login real, leitura de dados e as telas de Vendas, Estoque, Clientes e Dashboard. Publicado em https://painel.olvdistribuidora.com.br.
- 30/07: **DNS de painel.olvdistribuidora.com.br configurado** no Cloudflare (nameservers apontados, domínio anexado ao Worker como Custom Domain), confirmado no ar respondendo por esse endereço.
- 31/07: **telefone_norm incluído no endpoint `olv2-dados`** (pendência 15) e **bug da Gratuidade + Crediário corrigido** na função `registrar_venda` (pendência 14). Detalhes na seção 3.11.

#### Falta

- Fazer a virada e aposentar os fluxos antigos, com a sua aprovação.

**Validação técnica realizada em 31/07/2026**: conferidos o status real dos workflows n8n (os 5 antigos e os 4 OLV2 seguem ativos, ponto de retorno intacto) e o estado do Supabase (dados batendo com o documentado: 910 clientes, 6 produtos, demais tabelas vazias como esperado). Encontrada a pendência 16 (seção 8.3), uma brecha de segurança não documentada até então. **A aprovação final da virada fica condicionada a essa decisão, que é de Opus 5.**

#### A importação de clientes (30/07/2026)

Partida: 1.598 contatos exportados do Google Contatos, no formato "nome - endereço, número e bairro".

| Resultado | Linhas |
|---|---|
| **Importados** | **910** |
| Incompletos, separados para revisão | 677 |
| Duplicados removidos | 19 |

Todos os 910 têm nome, endereço, bairro e telefone. Os incompletos são em maioria contatos que nunca tiveram endereço salvo, e muitos nem são clientes, como bancos e fornecedores.

**24 bairros** identificados, com apelidos unificados: BS1 e Bomss 1 viraram Bomssucesso 1, MDL virou Morada Do Lago, NVSM virou Nova São Mateus, BV virou Boa Vista. Distribuição dos 910 importados: Vitória 309, São Pedro 113, Bomssucesso 1 76, Novo Horizonte 63, Sto Antônio 60, Bomssucesso 2 53, Nova São Mateus 51, Morada Do Lago 33, Ayrton Senna 29, Vila Nova 26, Caiçaras 24, Parque Das Brisas 22, e mais 12 bairros com menos de 10 cada.

Textos que estavam no campo de bairro mas não são bairro, como "Supergasbras" e "Atrás do Posto", foram movidos para observação, e esses clientes foram para a lista de revisão.

**Sobre os DDDs**: 860 dos 910 são DDD 27. Os demais foram conferidos pelo Rogério e estão corretos, de clientes que mudaram de estado e mantiveram o número.

### 9.2 Fase 1: Painel operacional novo *

**Status**: painel construído e publicado em 30/07/2026 em https://painel.olvdistribuidora.com.br (Cloudflare Workers), rodando em paralelo ao painel antigo. Login e as quatro telas (Dashboard, Vendas, Estoque, Clientes) testadas contra os endpoints reais. **Falta a virada final e a aposentadoria do painel antigo**, que depende da sua aprovação.

**Decisão de 30/07/2026**: o painel não foi adaptado, foi **reescrito**. Adaptar o HTML atual, com 878 linhas em base64 dentro de uma Data Table, ao formato novo de dados custaria mais que escrever do zero, e sem ganho.

**O HTML novo nasceu como arquivo único**, editável e comparável, publicado direto no Cloudflare Workers (não passa pela Data Table do n8n).

**Construção em paralelo**: o painel atual continua no ar e intocado, operando o dia a dia. O novo roda em endpoints separados. A virada só acontece na sua aprovação, e o antigo permanece como ponto de retorno.

#### Endpoints: as seis decisões aprovadas em 30/07/2026

1. **Itens do pedido por segunda consulta**, não por bloco JSON aninhado. Motivo na seção 3.9: auditabilidade do filtro de custo.
2. **Pendência 11 tratada agora no que dava**: totais agregados não são calculados para quem não é admin. Detalhe na seção 8.3.
3. **Edição de pedido dividida**: `criar`, `status`, `excluir` e um `editar_leve` de quatro campos. Edição de valores vira tarefa própria, porque depende de uma função no banco e de uma decisão sobre custo histórico. Motivo na seção 3.10.
4. **Permissão positiva sobre admin** nos quatro, com um único nó `Perfil`.
5. **Entrada e Estoque Inicial exclusivos do administrador**, Ajuste liberado. Motivo na seção 7.
6. **Sem exclusão de movimentação de estoque** na Fase 1. Motivo na pendência 13.

**Ajuste feito na construção**: o endpoint de leitura foi desenhado com cinco ações e nasceu com **duas**, `tudo` e `pedidos`. Puxar os 910 clientes a cada mudança de situação de pedido é peso desnecessário no celular, e estoque, produtos e clientes já vêm na carga inicial. Separar as ações depois é trabalho de meia hora, se o uso pedir.

#### Estrutura aprovada

Menu lateral em três grupos, com botão de recolher: Operação (Dashboard, Vendas, Estoque, Clientes), Financeiro (Contas a Pagar, a Receber, Fluxo de Caixa) e Administração (Usuários).

#### Dashboard (aprovado em 30/07/2026)

Filtros de período no topo, ao lado do título. Quatro indicadores principais: **Itens vendidos, Faturamento, Lucro e Nº de vendas**, com margem do período e **comparação com o período anterior** em cada um. Essa comparação é o ganho sobre o painel antigo, que mostrava o número sem dizer se estava melhor ou pior.

**Construído em 30/07/2026**: os quatro indicadores comparados, a margem do período e a lista de quantidade por produto.

**Falta, não estava no escopo fechado**: gráfico de linha com quantidade e valor por dia comparado ao mês anterior, lista "Atenção hoje", valores por forma de pagamento e movimentos prioritários. Fica como pendência em aberto, sem data definida.

**Exclusivo do administrador.** Indicadores de pré-pagos, a receber e saldo vivem nas suas próprias seções, não no Dashboard.

#### Vendas: quadro de operação

**Redesenhado em 30/07/2026**: as quatro situações (Aguardando, Pré-pagos, Em rota, Concluídos) aparecem **ao mesmo tempo, em colunas lado a lado**, no lugar das abas originalmente aprovadas. Mudar a situação de um pedido move o cartão de coluna, sem esconder da tela. Motivo: com abas, o pedido "sumia" da vista ao mudar de situação, e só reaparecia entrando na aba de destino, o que não correspondia ao uso real do dia a dia.

Roteamento de cada pedido para uma única coluna: Aguardando (não concluído, não pré-pago), Pré-pagos (pago e não resgatado), Em rota, Concluídos (dia atual por padrão, com filtro de período próprio no topo da página). Tipo Entrega ou Retirada aparece como marca visual, não como coluna. Botão de mudança rápida de situação com um toque.

Indicadores da seção: pedidos hoje, aguardando, em rota e pré-pagos a entregar.

**Permissões por tipo de ação, não por tela**: ver a tabela da seção 7.

#### Formulário de pedido (aprovado em 30/07/2026)

Quatro blocos mais "Mais opções" recolhido:

1. **Cliente**: busca por nome ou telefone; ao escolher, mostra endereço, bairro e telefone para o entregador conferir.
2. **Produtos**: vários itens, cada um com quantidade e valor unitário, subtotal por linha, campo de desconto e total do pedido. O custo aparece por item, puxado do cadastro.
3. **Entrega**: tipo, situação inicial e interruptor de pré-pago. Escolher Retirada some com a opção "Em rota".
4. **Pagamento**: começa com uma linha preenchida com o total. "Dividir pagamento" adiciona linha com o restante. Faixa mostra **"Falta alocar"** em âmbar e vira verde quando zera; o botão Salvar fica desabilitado até lá. Crediário abre campo de vencimento obrigatório; Em aberto vem preenchido com 3 dias, editável.

**O painel envia uma `idempotency_key` própria em cada pedido novo.** É o que faz o toque duplo devolver o mesmo pedido em vez de criar dois. Testado em 30/07/2026.

**Ajuste de 30/07/2026**: ordem de navegação por Tab corrigida. Os botões de ação dentro do formulário (remover item, adicionar item, dividir pagamento, trocar cliente) foram tirados da sequência de Tab, para o teclado avançar direto de campo em campo.

#### Estoque

Indicadores, tabela de produtos com situação, e **histórico de movimentações** com filtro por tipo, mostrando a saída automática por venda concluída como lançamento do sistema. Sem botão de excluir movimentação, pelo motivo da pendência 13.

#### Clientes

Busca por nome ou telefone, cadastro e edição. Sem histórico de pedidos por cliente nem indicadores, porque dependem de consultas que ainda não existem, previstas na seção 9.4.

**Ajuste de 30/07/2026**: a busca por telefone usava um campo (`telefone_norm`) que o endpoint não devolvia. Corrigido para extrair os dígitos do campo `telefone`, que existe na resposta. **Atualização de 31/07/2026**: o endpoint passou a devolver `telefone_norm` pronto (pendência 15, seção 3.11); simplificar a busca do painel para usar o campo pronto é limpeza opcional, sem urgência.

#### Correção de CORS (30/07/2026)

Ao publicar o painel no domínio final, o navegador bloqueava toda chamada aos cinco webhooks (OLV Login e os quatro OLV2), porque nenhum tinha a opção `allowedOrigins` configurada. O sintoma era a chamada travando antes de chegar ao servidor. Corrigido liberando `https://painel.olvdistribuidora.com.br` especificamente (não `*`) em cada um dos cinco nós de webhook, com publicação de cada workflow. Verificado com chamada real ao `olv2-dados` retornando 200 sem bloqueio.

#### Outros ajustes visuais (30/07/2026)

Ícones próprios para cada item do menu lateral, no lugar de um marcador genérico. Tela inicial após o login passa a ser o Dashboard para administrador, e Vendas para os demais papéis (antes era sempre Vendas).

#### Cores da marca: testadas e mantidas as originais (decisão fechada em 31/07/2026)

As cores foram extraídas por amostragem de pixel direto do arquivo da logo (azul-marinho #013090, azul médio #0041BC, verde #82D602, mais um ciano #01C9FE na chama interna, fora da lista original) e aplicadas no CSS do painel novo para avaliação. **Rogério testou e decidiu manter as cores de trabalho originais**, já em uso desde a construção do painel: azul-marinho `#0b2545`, azul-médio `#1d6fd6`, verde `#1c9c5b`, amarelo `#e3a92b`, vermelho `#c0392b`. As cores extraídas da logo foram revertidas. Amarelo e vermelho nunca vieram da logo (são cores de alerta do painel) e seguem sem mudança.

#### Bugs do menu lateral corrigidos (31/07/2026)

Dois problemas encontrados e corrigidos no botão de recolher/expandir o menu:

1. **Sobreposição em telas estreitas**: no CSS mobile (`max-width:720px`), o conteúdo tinha margem esquerda fixa de 64px independente do menu estar recolhido ou expandido. Ao expandir (210px), o menu cobria ~146px do conteúdo. Corrigido para a margem acompanhar o estado real do menu.
2. **Logo não escondia ao recolher**: não existia regra para esconder o texto "OLV Distribuidora" quando o menu recolhe para 64px; o texto só ficava cortado pela largura menor, sobrando o "O". Corrigido: a logo some por completo ao recolher, como os demais rótulos do menu.

#### Filtro de período redesenhado (31/07/2026)

O filtro de "De/Até" com botão Buscar virou pills de atalho (Hoje, Ontem, 7 dias, Este mês, Mês ant., Tudo) que preenchem o intervalo e já disparam a busca, mantendo os campos de data editáveis manualmente com o rótulo "X dias" ao lado. Aplicado nos dois usos existentes (Dashboard e Vendas/Concluídos), reaproveitando a mesma função e a mesma validação de intervalo máximo já existente (90 dias em Concluídos). Bordas dos campos de data arredondadas (8px, mesmo padrão dos outros campos do painel).

#### Favicon e ícone de app (31/07/2026)

A partir da imagem da logo (formato de ícone quadrado) enviada por Rogério: favicon (16px e 32px, aba do navegador), apple-touch-icon (180px, tela inicial do iPhone) e ícone de app via manifest (192px, "Adicionar à tela inicial" no Android), com a cor de fundo do app (`theme-color`) combinando com o painel. Os cantos, que na imagem original eram um quadrado preto sólido com a forma arredondada desenhada por dentro, foram recortados com transparência real (raio medido nos próprios pixels da arte, 186px) para o ícone ficar arredondado em qualquer fundo de aba, não só em tema escuro. Tudo embutido como dado direto no HTML, sem precisar de arquivos extras no deploy.

#### Cuidados

Performance com o volume crescendo; não quebrar o que já funciona; a aba de concluídos é a que mais cresce e sem filtro de período padrão vira o ponto de lentidão.

### 9.3 Login para múltiplos usuários (concluído)

**Status**: concluído e no ar em 25/07/2026, com a Opção A, contas individuais com papéis. Rótulo renomeado para colaborador em 28/07/2026.

A Opção B, vários logins simples sem distinção de papel, foi descartada por não ter controle de permissão nem rastreio de quem fez o quê.

Pontos de segurança resolvidos: usuários fora do código, senha com hash, sessão com validade e usuário registrado em cada lançamento.

### 9.4 Fase 2: Contas a pagar e a receber *

Duas seções próprias no painel, não sub-abas de um Financeiro único.

**A receber**: recebíveis já são criados automaticamente pela função de gravação, para Crediário e Em aberto. Falta a tela, a baixa e os alertas de vencimento no Telegram. Entram também os recebíveis de terceiros: Crédito, Débito e Gás do Povo.

**A pagar**: cadastro com fornecedor, descrição, valor, vencimento, status e categoria; ligação com as entradas de estoque, aproveitando o custo e o fornecedor já registrados; anexo de boletos no Storage; alertas no Telegram.

**Clientes**: histórico de pedidos por cliente e indicadores. O endpoint de manutenção de clientes já existe desde 30/07/2026; falta a consulta de histórico.

### 9.5 Fase 2: Controle de caixa *

Abertura com saldo inicial por operador, entradas de vendas em dinheiro e reforços, saídas de sangrias e pagamentos, fechamento com saldo esperado contra contado e a diferença destacada, histórico por dia e operador.

Cuidados: vendas em dinheiro alimentam as entradas automaticamente; depende do login para saber o operador; regra de um caixa aberto por vez e bloqueio de lançamento em caixa fechado.

### 9.6 Módulo fiscal via API (visão de futuro) *

Emitir o documento fiscal a partir do painel, integrando com a API de um provedor.

**Contexto já levantado**: Espírito Santo, emissão pela SEFAZ-ES; Lucro Presumido, então a nota traz PIS e COFINS; gás com Substituição Tributária, ICMS recolhido na origem; água com ICMS normal; certificado e-CNPJ já existe; provedor avaliado, NFe.io.

**Documentos**: NFC-e modelo 65 para a maioria das vendas a consumidor final, com QR Code; NF-e modelo 55 para vendas a CNPJ.

**Como funcionaria**: ao confirmar a venda, o n8n chama a API, que gera o XML assinado, transmite para a SEFAZ e devolve chave, QR Code e PDF. O painel guarda junto da venda e permite enviar ao cliente por WhatsApp.

**Pendências**: gerar o CSC e o Identificador do Token na SEFAZ-ES; concluir o cadastro no provedor, onde houve erro na inscrição estadual 083592253, testar com zeros à esquerda ou validar com o contador; cadastro fiscal dos produtos com NCM, CFOP e tributação.

**Cuidados**: vasilhame retornável tem tratamento fiscal específico e normalmente não entra como venda, confirmar com o contador; ordem correta é venda confirmada, estoque, depois nota; o certificado vence e precisa renovação; a NFC-e é obrigatória mesmo quando o cliente não pede; o pico de domingo justifica a emissão automática.

Confirmar tudo com o contador antes da execução.

### 9.7 Fase 2: Setores e permissões por área *

**Status**: definido em princípio, execução **depois da religação do Supabase**, para não construir controle de acesso sobre estrutura que será substituída.

Separar duas coisas hoje misturadas:

| Dimensão | Valores | Função |
|---|---|---|
| papel | admin ou comum | Nível. Admin vê tudo, ignora setores |
| setores | vendas, financeiro, operacional | Escopo. Só para quem é comum |

**Por que separar em vez de trocar**: todas as travas hoje são comparações positivas sobre `admin`. Mantendo `papel` como está, elas continuam válidas sem alteração, e os setores entram como camada adicional. Evita repetir o incidente de 28/07.

**Vários setores por pessoa**, decidido em 29/07/2026. Em empresa pequena a mesma pessoa acumula funções.

| Setor | Vê |
|---|---|
| vendas | Pedidos, clientes e valores de venda. Sem lucro, custo nem margem |
| financeiro | Contas, fluxo de caixa, faturamento, lucro e margem |
| operacional | Estoque, movimentações e alertas |
| comum sem setor | Nada além da própria troca de senha |

**Entra nesta etapa a execução da pendência 11**: campo de entregador em `vendas`, momento da atribuição, tratamento de Retirada e de Aguardando. Detalhe na seção 8.3.

**A regra inegociável: a filtragem acontece no servidor, antes de o dado sair.** Esconder no navegador não conta. O incidente de 28/07 demonstra: a proteção visual continuou intacta enquanto o dado sensível trafegava. Prometer separação de áreas e entregar ocultação visual seria pior que não prometer nada.

**Decisões em aberto**: lista final de setores; se um setor tem níveis internos, por exemplo financeiro que consulta contra financeiro que dá baixa; se quem lançou continua vendo o próprio lançamento ao sair do setor.

---

## 10. ORDEM E DECISÕES EM ABERTO

| Ordem | Etapa | Situação |
|---|---|---|
| 1º | Login multiusuário | Concluído e no ar |
| 2º | Base de dados no Supabase | Esquema, produtos, clientes e endpoints do painel novo concluídos e ativados. Validação técnica feita em 31/07/2026 |
| 3º | **Fase 1: painel operacional novo** | Painel construído e publicado em painel.olvdistribuidora.com.br, em teste real com dados reais. Falta a virada final e aposentar o painel antigo |
| 4º | Setores e permissões | Depois da virada, antes do financeiro, para os endpoints já nascerem com validação |
| 5º | Contas a pagar e receber | Fase 2 |
| 6º | Controle de caixa | Fase 2 |
| 7º | Módulo fiscal | Visão de futuro |

**Decisões a fechar:**

- **Pendência 16 (nova, seção 8.3)**: revogar o acesso de `anon`/`authenticated` às views e funções do Supabase que hoje ignoram a RLS. Decisão de Opus 5, bloqueia a aprovação final da virada.
- Setores: lista final, níveis internos e tratamento do histórico.
- Pendência 11: quem atribui o entregador e como tratar Retirada e Aguardando.
- Edição de valores de pedido: se a função preserva o custo original de cada item ou reestampa o custo do dia da edição.
- Recebíveis de terceiros: crédito, débito e Gás do Povo, realizado na venda ou previsto até o crédito bancário.
- Financeiro: categorias de contas a pagar.
- Caixa: único ou por operador; o que entra como sangria e reforço; como tratar diferenças no fechamento.
- Fiscal: ST do gás e vasilhame com o contador; CSC na SEFAZ-ES; inscrição estadual no provedor.

---

## 11. REFERÊNCIAS RÁPIDAS

| Item | Valor |
|---|---|
| URL do painel (antigo, em produção) | https://n8n-wmtt.srv1830312.hstgr.cloud/webhook/olv-painel |
| URL do painel novo | https://painel.olvdistribuidora.com.br, publicado no Cloudflare Workers desde 30/07/2026 |
| Autenticação | Login com papéis (administrador e colaborador), token de 12h |
| Instância n8n | n8n-wmtt.srv1830312.hstgr.cloud |
| Workflows em produção | OLV Painel Mobile (web), OLV Vendas, OLV Estoque, OLV Login, OLV Contas |
| Workflows do painel novo | OLV2 Dados, OLV2 Pedido, OLV2 Estoque, OLV2 Clientes. Todos ativos desde 30/07/2026 |
| Tabelas internas do n8n | OLV Usuarios e OLV Painel HTML |
| Pontos de restauração (v1.4) | Painel 63bf15bb; Vendas 348ed17a; Estoque 2adfcba0 |
| Identificadores dos workflows novos (v3.1) | Dados fi2DPaA6qL7MxwIV; Pedido aPr6vx4oesVfkLis; Estoque 3l15lOfGCeYqLEu7; Clientes ClsDIM8jVRB5fijC |
| Banco de dados | Supabase PostgreSQL 17, projeto olv-distribuidora_sistema, região São Paulo. 8 tabelas, 2 views, RLS ativa |
| Conexão n8n para Supabase | Session Pooler IPv4, host aws-0-sa-east-1.pooler.supabase.com, porta 5432, base postgres, usuário postgres.ggvfrnympdrqyqxgcyex, SSL ativo. Credencial "Supabase OLV" |
| Domínio | olvdistribuidora.com.br no Registro.br. painel.olvdistribuidora.com.br configurado no Cloudflare e no ar desde 30/07/2026 |
| Fonte de clientes | Google Contatos, 910 importados |
| Cores do painel | Mantidos os valores de trabalho originais: azul-marinho #0b2545, azul-médio #1d6fd6, verde #1c9c5b, amarelo #e3a92b, vermelho #c0392b. Testada e descartada a troca pelas cores da logo (31/07/2026) |
| Favicon / ícone de app | Embutido no HTML como data URI: favicon 16/32px, apple-touch-icon 180px, ícone de manifest 192px, cantos com transparência real |
| Documento oficial | GitHub rogeriosandre/olv-distribuidora, Painel_OLV_Documentacao_e_Evolucao.md |
| Documento visual | Painel_OLV_Visual.md, no mesmo repositório |

### Endpoints do painel novo

| Endpoint | Corpo mínimo |
|---|---|
| POST /webhook/olv2-dados | `{acao, token, de, ate}` |
| POST /webhook/olv2-pedido | `{acao, token, venda, itens, pagamentos, idempotency_key}` ou `{acao, token, venda_id, ...}` |
| POST /webhook/olv2-estoque | `{acao, token, tipo, produto_id, quantidade, custo_unitario, fornecedor}` |
| POST /webhook/olv2-clientes | `{acao, token, nome, endereco, bairro, telefone}` ou `{acao, token, termo}` |

Toda resposta traz `ok` verdadeiro ou falso. Quando falso, traz `erro` com mensagem pronta para exibir ao usuário.

### Cuidado com cache ao carregar o documento

O `raw.githubusercontent.com` entrega por rede de distribuição que guarda cópias. Os caminhos `refs/heads/main/...` e `main/...` são endereços diferentes e podem ter cópias de idades diferentes. Em 29/07/2026 o primeiro entregou a v1.6 enquanto o segundo entregava a v1.7, três dias de diferença.

**Use o caminho curto** e sempre confira a versão lida, conforme a regra 7.

---

## 12. GUIA DE MODELOS POR ETAPA *

Define qual modelo de IA usar em cada tipo de tarefa, para reduzir consumo sem aumentar risco.

Um modelo funciona como um profissional contratado por hora. O mais experiente resolve problema difícil com menos erro, mas custa mais. O mais rápido resolve tarefa repetitiva por uma fração do preço. Contratar o sênior para arquivar papel é desperdício; contratar o júnior para desenhar a fundação é risco.

| Modelo | Perfil | Uso |
|---|---|---|
| Haiku 4.5 | Rápido e econômico | Tarefa repetitiva e automação rodando dentro do n8n |
| Sonnet 5 | Equilíbrio | Execução do que já foi especificado |
| Opus 5 | Raciocínio profundo | Decisão estrutural, regra de negócio, segurança |
| Fable 5 | Topo de linha | Reserva, se o Opus travar |

### Regra de acionamento

- **Opus 5**: decisão difícil de desfazer (esquema de banco, regra financeira, segurança, autenticação), cruzamento de várias seções deste documento, ou planejamento de etapa.
- **Sonnet 5**: o "o quê" está definido e falta o "como" (escrever SQL já especificado, ajustar nó, montar tela, revisar texto, depurar erro pontual).
- **Haiku 4.5**: tarefa repetitiva e qualquer chamada de IA que rode dentro de fluxo em produção.

### Modelo por etapa

| Etapa | Tarefa | Modelo |
|---|---|---|
| 9.1 Supabase | Desenho de esquema e regras | Opus 5 |
| 9.1 Supabase | SQL já especificado, importação, limpeza | Sonnet 5 |
| 9.1 Supabase | Plano de virada e ponto de retorno | Opus 5 |
| 9.1 Supabase | Religar fluxos: desenho | Opus 5 |
| 9.1 Supabase | Religar fluxos: aplicação nó a nó | Sonnet 5 |
| 9.2 Painel | Arquitetura de navegação e performance | Opus 5 |
| 9.2 Painel | Telas, estilo, componentes | Sonnet 5 |
| 9.4 Financeiro | Regras e casos de borda | Opus 5 |
| 9.4 Financeiro | Telas e cadastros | Sonnet 5 |
| 9.5 Caixa | Regras de abertura, sangria e fechamento | Opus 5 |
| 9.7 Setores | Modelo de permissões e filtragem no servidor | Opus 5 |
| 9.7 Setores | Aplicação nos nós e telas | Sonnet 5 |
| 9.6 Fiscal | Mapeamento e integração | Opus 5 |
| Qualquer | Alteração de papel, permissão ou trava | Opus 5 |
| Documento | Merge, revisão cruzada, mudança de versão | Opus 5 |
| Documento | Ajuste de texto e tabela | Sonnet 5 |

**Regras fixas**: nada do módulo fiscal vai para produção sem validação do contador. Qualquer tarefa que altere permissão audita todos os workflows.

### Como o Claude avisa

Primeira linha da resposta: **Modelo indicado: [X]. Motivo: [tipo de tarefa].** Quando o modelo em uso for mais pesado que o necessário, o aviso é explícito.

### Limites desta prática

- O Claude **não troca de modelo sozinho**. A troca é manual.
- Trocar no meio da conversa **carrega todo o histórico**. A economia é parcial.
- A economia maior vem de **abrir conversa nova e curta** no modelo leve.
- **Cuidado aprendido em 29/07/2026**: economizar contexto instruindo uma sessão a **não carregar o documento** removeu a única defesa contra o cache e causou a perda descrita no log da v2.9. Se a sessão puder acabar lendo ou editando o documento, a regra 7 precisa ir junto no prompt.
- **Pendência**: dividir este documento em um núcleo enxuto mais anexos por etapa. É a economia estrutural, maior que a troca de modelo.

---

## LOG DE MUDANÇAS

| Data | Versão | Mudança |
|---|---|---|
| 25/07/2026 | 1.6 | Documento criado; login multiusuário implementado; banco Supabase criado; decisão de construção do zero sem migração. |
| 26/07/2026 | 1.7 a 2.8 | Seção 12 criada; referências à planilha removidas; seção 3 reescrita com o esquema real; índices e RLS verificados; status validado e reduzido a três valores com `tipo_entrega` separado; quadro de vendas por status definido; papel renomeado no documento; pedido pré-pago criado; baixa de estoque movida para a conclusão; trava de cliente duplicado por telefone. |
| 29/07/2026 | 2.9 | Restaurada a v2.8, sobrescrita em 28/07 por um commit que partiu da v1.6 recebida em cache. Incorporada a renomeação aplicada no sistema. Documentado o mecanismo de ocultação financeira. Registrado o incidente do nó Montar Payload, que expôs lucro e custo ao colaborador por 16 horas, e sua correção. Criada a etapa de setores e a regra de conferência de versão. |
| 30/07/2026 | 3.0 | **Reestruturação do pedido e fundação de dados concluída.** (a) Criada a tabela `venda_itens`: a venda deixou de ser uma linha por produto e virou um pedido com vários itens, com custo e lucro por item. `produto_id`, `quantidade` e `custo_unitario` saíram de `vendas`, que ganhou `desconto` e `idempotency_key`. (b) Definidas as 8 formas de pagamento em lista fechada, com a regra de recebível por linha de pagamento e o tratamento da Gratuidade, que zera o pedido. (c) Criada a função `registrar_venda`, que grava pedido, itens, pagamentos e recebíveis numa transação só, encerrando as pendências 1 e 4. (d) Criadas as travas de consistência do pedido, adiadas para o fim da transação. (e) `contas_receber` ganhou `pagamento_id` e cascata, encerrando a pendência 8. (f) Custo do produto passou a nascer da entrada de estoque, por média ponderada, com o custo da última compra visível ao lado; `estoque_movimentacoes` ganhou custo e fornecedor. (g) `vw_estoque_atual` reconstruída: abatia estoque na data do pedido em vez da conclusão, contrariando a regra da seção 6. (h) Criada `vw_pedidos`, que calcula lucro e margem já com o desconto, corrigindo distorção descoberta nos testes. (i) 6 produtos cadastrados e 910 clientes preparados a partir de 1.598 contatos, com 24 bairros padronizados. (j) Simulação visual do painel novo aprovada, com Dashboard exclusivo do administrador e o indicador de ticket médio removido, encerrando a pendência 12. (k) Decidido reescrever o HTML do painel como arquivo versionado, em construção paralela ao painel atual. |
| 30/07/2026 | 3.1 | **Endpoints do painel novo construídos e desativados (tarefa 4).** (a) Criados os quatro workflows OLV2, com bloco de autenticação idêntico, permissão positiva sobre `admin` e nó Crypto com `type` e `encoding` fixados desde o rascunho. Seção 4.2. (b) Registradas as seis decisões de desenho aprovadas, incluindo itens do pedido por segunda consulta e a divisão da edição de pedido em `criar`, `status`, `editar_leve` e `excluir`. Seção 9.2. (c) Nova convenção 8.4: toda gravação viaja como um único parâmetro em base64, porque o nó Postgres separa parâmetros por vírgula e texto livre tem vírgula. Fecha também a porta para injeção de SQL. (d) Nova regra de permissão na seção 7: Entrada e Estoque Inicial exclusivos do administrador, porque alimentam o custo médio; Ajuste liberado a qualquer usuário ativo. (e) Pendência 13 criada: o gatilho de custo dispara só no INSERT, então excluir movimentação não desfaz o custo médio; por isso não existe exclusão no endpoint de estoque. (f) Pendência 14 criada: Gratuidade combinada com Crediário grava recebível com o valor cheio. (g) Pendência 11 reduzida e detalhada: totais agregados deixam de ser calculados para quem não é admin, e ficou decidida a direção de entregar valor apenas dos pedidos em rota atribuídos ao colaborador, para executar na etapa 9.7. (h) Endpoint de leitura nasceu com duas ações em vez de cinco, por peso no celular. (i) Registrado que `responsavel` guarda quem lançou, não quem entrega, e que não existe campo de entregador. |
| 30/07/2026 | 3.2 | **Painel novo construído, publicado e ligado aos endpoints; workflows OLV2 ativados.** (a) `index.html` do painel novo escrito e publicado em https://painel.olvdistribuidora.com.br via Cloudflare Workers (Cloudflare Drop, domínio próprio anexado). (b) Corrigido bloqueio de CORS nos cinco webhooks usados pelo painel (OLV Login e os quatro OLV2): nenhum tinha `allowedOrigins` configurado, o que impedia qualquer chamada do navegador a partir do domínio novo. Corrigido nó a nó e publicado; os quatro workflows OLV2 saíram de desativados para ativos. (c) Quadro de Vendas redesenhado: as quatro situações passam a aparecer em colunas simultâneas, no lugar de abas que escondiam as demais ao trocar de situação. (d) Filtro de período (Dashboard e Concluídos) movido para o topo da página, ao lado do título. (e) Corrigida busca de cliente por telefone: usava o campo `telefone_norm`, que o endpoint `olv2-dados` não devolve; passou a extrair os dígitos do campo `telefone`. Nova pendência 15 registrada na seção 3.11. (f) Ícones próprios no menu lateral, no lugar de um marcador genérico. (g) Ordem de navegação por Tab corrigida no formulário de novo pedido. (h) Tela inicial pós-login passa a ser o Dashboard para administrador, e Vendas para os demais papéis. (i) Nova pendência registrada: cores da marca no painel novo ainda são valores de trabalho, não confirmados como oficiais. (j) DNS de painel.olvdistribuidora.com.br confirmado configurado e no ar. |
| 31/07/2026 | 3.4 | **Validação técnica da virada, correções de bug e ajustes visuais no painel novo.** (a) Conferido o status real dos workflows n8n (5 antigos + 4 OLV2, todos ativos, ponto de retorno intacto) e o estado do Supabase (dados batendo com o documentado). (b) Nova pendência 16 registrada na seção 8.3: `vw_pedidos`, `vw_estoque_atual` e cinco funções (incluindo `registrar_venda`) são SECURITY DEFINER com privilégio liberado para `anon`/`authenticated`, permitindo acesso direto à API do Supabase sem passar pelo n8n; correção fica para sessão em Opus 5, por alterar trava de acesso. A aprovação final da virada depende dessa decisão. (c) Pendência 15 resolvida: `telefone_norm` incluído no SELECT do nó `Consultar Clientes T` (OLV2 Dados), workflow publicado. (d) Pendência 14 resolvida: função `registrar_venda` corrigida para zerar `contas_receber.valor` quando o pedido é Gratuidade, mesmo combinado com Crediário; testado com pedido real e dado de teste removido. (e) Testada a troca das cores do painel pelas cores extraídas da logo (azul-marinho #013090, azul médio #0041BC, verde #82D602); Rogério avaliou e decidiu manter as cores de trabalho originais (#0b2545/#1d6fd6/#1c9c5b). Decisão fechada. (f) Corrigidos dois bugs do menu lateral: sobreposição do conteúdo em telas ≤720px ao expandir o menu, e a logo "OLV Distribuidora" não sumindo por completo ao recolher. (g) Filtro de período do Dashboard e de Vendas/Concluídos redesenhado com pills de atalho (Hoje, Ontem, 7 dias, Este mês, Mês ant., Tudo), mantendo os campos de data editáveis; bordas dos campos arredondadas. (h) Favicon, apple-touch-icon e ícone de manifest (PWA) adicionados a partir da logo, com cantos recortados com transparência real; tudo embutido como data URI no próprio HTML. |

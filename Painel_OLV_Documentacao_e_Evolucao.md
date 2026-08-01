# OLV DISTRIBUIDORA
## Painel de Vendas, Estoque e Financeiro

**Documentação do funcionamento e plano de evolução**

Versão 3.9 · 01/08/2026
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
7. **Conferência de versão (obrigatória)**: ao carregar este documento, informe na primeira linha a versão lida. Se for menor que a última registrada no log ou que a informada por Rogério, **pare e avise**, porque provavelmente veio de cache. Vale mesmo quando ler o documento não fazia parte do plano da tarefa. Motivo no log da v2.9. **Caminho que resolve o cache (31/07/2026)**: quando os dois caminhos de branch entregarem versão velha, carregue pelo código do commit, no formato `raw.githubusercontent.com/rogeriosandre/olv-distribuidora/<commit>/Painel_OLV_Documentacao_e_Evolucao.md`. Endereço de commit é único e não tem cópia guardada. Ver seção 11.
8. **Auditoria de permissão (criada em 29/07/2026)**: qualquer tarefa que altere papel, permissão ou trava de acesso deve auditar **todos os workflows**, não só os que parecem relacionados. Motivo na seção 8.2.
9. **Fechamento de acesso após migração (criada em 01/08/2026)**: toda migração termina com `select fechar_acesso_publico();` e a conferência `select * from auditar_acesso_publico() where anon or autenticado or publico;`, que precisa devolver **zero linhas**. Objeto novo **não** nasce fechado: função nova nasce aberta. Motivo na seção 8.6.
10. **O sistema está em uso real desde 01/08/2026**, com pedidos, clientes e estoque de verdade. Antes de testar contra o banco, tire uma impressão digital dos dados existentes e confira no fim que nada mudou. Limpe todo dado de teste.

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

Cada assunto vive em uma tabela própria, ligada às outras por um código (ID). **Oito tabelas de movimento, seis de cadastro e duas views.** Tranca de segurança (RLS) ativa em todas.

As oito de movimento guardam o que acontece: clientes, produtos, vendas, venda_itens, pagamentos, estoque_movimentacoes, contas_pagar e contas_receber. As seis de cadastro, criadas em 31/07/2026, guardam as regras e os parâmetros do negócio, e estão na seção 3.12.

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
| preco_venda | Número | **Criado em 31/07/2026.** Preço sugerido, preenche o formulário de pedido. Editável pelo administrador. Não trava a venda, que segue negociável item a item |
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
| acrescimo | Número | **Criado em 31/07/2026.** Acréscimo do pedido inteiro, como taxa de entrega. Começa em 0 |
| valor_total | Número | **Mantido pelo banco**: soma dos itens, menos o desconto, mais o acréscimo |
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
| valor | Número | Obrigatório, não negativo. **Sempre o valor cheio**, o que o cliente pagou |
| maquininha_id | Ligação | **Criado em 01/08/2026.** Aponta para maquininhas, com RESTRICT. Só no cartão |
| taxa_id | Ligação | Qual linha de `taxas_cartao` foi aplicada. Serve para auditar de onde veio o número |
| grupo | Texto | **Validado**: Padrão ou Outras. É o interruptor da tela de venda |
| modalidade | Texto | **Validado**: Débito ou Crédito |
| parcelas | Número | Nulo no débito, 1 ou mais no crédito |
| taxa_pct | Número | **Copiada** da taxa vigente no momento da venda, não referenciada |
| taxa_valor | **Calculada** | valor × taxa_pct ÷ 100, em 2 casas |
| valor_liquido | **Calculada** | valor − taxa_valor. O que a operadora credita |

**Por que `valor` continua cheio.** Gravar o líquido é impossível: a trava adiada da seção 3.10 confere se a soma dos pagamentos bate com o total do pedido, e o líquido quebraria todo pedido no cartão. Os dois números são verdade e os dois ficam guardados.

**Travas novas de 01/08/2026**: `ck_pag_grupo`, `ck_pag_modalidade`, `ck_pag_parcelas`, `ck_pag_taxa_pct` e `ck_pag_cartao_coerente`, esta última impedindo pagamento meio cartão e meio não.

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
| custo_unitario | Número | Informado na Entrada e no Estoque Inicial. **O banco recusa se vier num Ajuste** (31/07/2026) |
| quantidade | Número | Obrigatório |
| data | Data | Começa com hoje |
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

**Refeita em 01/08/2026, com as duas margens lado a lado:**

| Coluna | O que responde |
|---|---|
| `lucro`, `margem_pct` | Mantidos com o significado anterior, **brutos**, sem a taxa do cartão |
| `taxa_total` | Quanto a operadora ficou no pedido |
| `lucro_liquido`, `margem_liquida_pct` | Quanto o pedido rendeu de verdade |
| `recebido_liquido` | Quanto entra na conta |
| `acrescimo` | **Faltava na view desde a v3.6.** Entrou junto |

Manter `lucro` e `margem_pct` com o nome e o sentido antigos é o que permitiu acrescentar a margem líquida sem quebrar o painel. Qual das duas aparece no card vira decisão de tela, reversível, em vez de decisão de banco.

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
| Releitura do custo médio (31/07/2026) | Custo do produto ficar preso a um lançamento que foi apagado, editado ou datado para trás. Ver pendência 13 |
| `ck_ajuste_sem_custo` (31/07/2026) | Ajuste de estoque carregar custo. Só compra altera o custo médio |

**Sobre a trava adiada**: ela confere no fim da transação, não a cada linha. Sem isso, dispararia ao gravar a primeira das duas linhas de pagamento, quando a soma ainda não bate.

**Função `registrar_venda`**: grava pedido, itens, pagamentos e recebíveis **numa transação só**. O n8n a chama uma única vez. Como o n8n não tem transação entre nós, gravar em etapas deixaria uma janela em que o pedido existe sem pagamento. Se qualquer parte falhar, nada é gravado.

**Religada em 01/08/2026.** O comportamento das oito formas saiu de dentro do código e passou a ser lido da tabela `formas_pagamento`. Antes de escrever, os campos da tabela foram comparados um a um com o comportamento que estava no código: batem exatamente. Por isso a religação **não muda comportamento nenhum**, ela troca a origem do comportamento.

Prova de que é real: com `prazo_dias` de Em aberto alterado de 3 para 5, um pedido de 01/08 gerou recebível vencendo em 06/08 em vez de 04/08. O valor foi devolvido para 3 em seguida.

A mesma função passou a **capturar a taxa do cartão** e copiá-la para o pagamento. Três decisões de projeto:

1. **Nunca bloqueia a venda por causa de taxa.** Quando a taxa não é encontrada, grava sem taxa e devolve o motivo em `avisos`. Bloquear travaria o balcão no domingo por um cadastro incompleto.
2. **A maquininha se resolve sozinha quando só existe uma ativa.** Isso fez a captura ligar sem depender de campo novo na tela, e reduziu a mudança de tela ao interruptor de bandeira e às parcelas. Com uma segunda maquininha, o painel passa a ser obrigado a informar qual.
3. **A modalidade é deduzida do nome da forma.** Acoplamento à lista fechada, aceito porque a lista é fechada de propósito.

Consulta para achar venda no cartão que ficou sem taxa capturada:

```sql
select p.venda_id, v.data, p.forma, p.grupo, p.parcelas, p.valor
from pagamentos p join vendas v on v.id = p.venda_id
where p.maquininha_id is not null and p.taxa_pct is null
order by v.data desc;
```

**Função `atualizar_venda` (criada em 01/08/2026)**: troca itens e pagamentos de um pedido existente numa transação só, no mesmo molde da `registrar_venda`. Detalhe abaixo, no lugar do bloco que dizia que a edição de valores não existia.

**Edição completa de pedido, liberada em 01/08/2026.**

*Por que não existia antes.* Cada nó do n8n é uma transação própria, que fecha sozinha. Alterar a quantidade de um item dispara o recálculo do total e, no fim daquela transação, a trava adiada compara com a soma dos pagamentos e recusa. O nó seguinte, que corrigiria o pagamento, nunca chega a rodar. Não era arriscado: **não rodava**.

*Por que virou necessário.* Motivo de negócio dado pelo Rogério: quase todo pedido é pago na entrega, e a forma informada no lançamento é palpite. Produto e quantidade também mudam na porta do cliente.

*A proposta que foi descartada.* Renomear Concluídos para Entregue e criar uma situação Finalizado, com o estoque baixando só no Finalizado. Descartada porque cria uma janela em que o produto já saiu e o sistema ainda o conta no depósito, e esquecer de finalizar vira erro silencioso de estoque.

*O que se descobriu.* O problema que tornava a edição perigosa **já tinha sido resolvido em 31/07**: os gatilhos de releitura da pendência 13 refazem custo e estoque sozinhos quando um item de pedido muda. Não era preciso situação nova nem travar edição de pedido concluído. Faltava só a função.

*Como funciona.* A `atualizar_venda` troca itens e pagamentos do pedido inteiro numa transação só. Vale para qualquer pedido, **inclusive já concluído**. Recebível e taxa de cartão são refeitos junto. Permissão: dono do lançamento ou administrador, mesma regra do `editar_leve`, porque é a mesma categoria de risco.

*Custo dos itens, decisão do Rogério em 01/08/2026*: item que já estava no pedido **preserva** o custo do dia da venda; produto novo entra com o custo de hoje. Protege o lucro do que já foi vendido, e como a edição costuma acontecer no mesmo dia, na prática dá no mesmo quase sempre.

*Aviso que virou a pendência 19.* A função **apaga e recria os pagamentos**. Quando a tabela de baixas existir, a baixa cairia junto por cascata, então a função vai precisar recusar pedido que já tenha recebimento lançado. Está escrito no comentário da função no banco.

**RLS**: tranca ativa nas 8 tabelas, sem nenhuma política liberando acesso. As chaves públicas do Supabase ficam totalmente bloqueadas nas tabelas; o n8n conecta com usuário que ignora a tranca. **A proteção por papel depende inteiramente da validação dentro do n8n.** Se um dia o painel acessar o banco direto, será obrigatório escrever políticas antes.

**Fechamento de 31/07/2026 (pendência 16 resolvida).** A RLS protegia as tabelas, mas as views e as funções SECURITY DEFINER passavam por cima dela. Fechado em duas frentes, detalhe na seção 8.3:

| Trava nova | O que garante |
|---|---|
| `anon` e `authenticated` sem `USAGE` no schema `public` | Os papéis públicos não enxergam nem o schema. Nenhuma tabela, view ou função é alcançável pela API REST |
| Privilégio padrão fechado no schema `public` | **Tabela, view ou função criada daqui para frente nasce sem acesso para os papéis públicos.** Antes nascia liberada, que era a causa raiz |
| `vw_pedidos` e `vw_estoque_atual` com `security_invoker` | As views passam a rodar com os direitos de quem consulta. Deixam de contornar a RLS por construção, não só por falta de privilégio |

**Regra corrigida em 01/08/2026.** A regra escrita aqui até a v3.8 dizia que "objeto novo no schema `public` nasce fechado". **Está errada para funções.** Tabela nova nasce fechada; **função nova nasce aberta**, com execução para `PUBLIC`. Testado três vezes. O detalhe e a rotina permanente estão na seção 8.6.

O que fica: se algum dia for preciso abrir algo para a API REST, a abertura é explícita, escrita e acompanhada da política de RLS correspondente. Nunca por herança. E toda migração termina com o fechamento e a conferência da seção 8.6.

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
| 9 | **Recebíveis de terceiros** | **Redesenhado em 01/08/2026**: cartão **não** vira recebível. A taxa já é capturada, o líquido calculado e o prazo cadastrado, então o Fluxo de Caixa projeta direto de `pagamentos`. **Gás do Povo continua virando recebível de verdade**, porque o repasse do programa pode atrasar ou vir diferente |
| 10 | **Lucro por item ignora o desconto** | Contornado pela view `vw_pedidos`. Se um dia o desconto precisar ser rateado item a item, é aqui |
| 13 | ~~Excluir ou editar movimentação de estoque não desfaz o custo médio~~ | **Resolvido em 31/07/2026**: custo passou a ser recalculado por releitura do histórico. Ver detalhamento abaixo |
| 14 | ~~Gratuidade combinada com Crediário gera recebível com valor cheio~~ | **Resolvido em 31/07/2026**: função `registrar_venda` corrigida para zerar também `contas_receber.valor` nesse caso. Testado com pedido real (Gratuidade + Crediário), recebível confirmado em R$ 0,00, dado de teste removido |
| 15 | ~~Endpoint `olv2-dados` não devolve `telefone_norm`~~ | **Resolvido em 31/07/2026**: nó `Consultar Clientes T` do OLV2 Dados passou a incluir `telefone_norm` no SELECT; workflow publicado |
| 18 | **Tabela de baixas não existe** | É a decisão 1 da seção 9.4 e nunca foi criada. Sem ela, `contas_receber.status` só aceita Em aberto ou Recebido, o que não representa o crediário: o cliente deve R$ 200 e paga R$ 50. **Bloqueia a tela de Contas a Receber** |
| 19 | **`atualizar_venda` versus baixa parcial** | A função apaga e recria os pagamentos, e a baixa cairia por cascata. Quando a 18 for feita, a função precisa recusar pedido com recebimento lançado |
| 20 | **Conta de destino não preenchida** | Nenhuma das 8 formas nem a maquininha Rede têm `conta_destino_id`. Sem isso o Fluxo de Caixa não sabe onde o dinheiro cai. **Rogério resolve na tela de Cadastros, sem depender de sessão** |
| 21 | **Saldo inicial das contas zerado** | As 4 contas estão com `saldo_inicial` em 0,00 e `data_inicial` vazia. O Fluxo de Caixa começaria do zero em vez do saldo real. **Também é da tela de Cadastros** |
| 22 | **Anexo de boleto sem caminho** | Zero buckets no Storage, e a regra da seção 2 diz que o navegador nunca fala direto com o Supabase. Falta decidir por onde o arquivo sobe: n8n recebe e grava, ou exceção só para Storage. Trava a tela de Contas a Pagar |
| 23 | **Código morto no painel** | O formulário de edição leve deixou de ser usado quando a edição completa entrou em 01/08/2026. Remover numa próxima passada |

A numeração 11, 12, 16 e 17 fica na seção 8.3, porque são pendências de segurança e de acesso. A série é única.

**Pendência 13, detalhada e encerrada em 31/07/2026.**

Ao abrir o código para corrigir, apareceram **quatro falhas, não uma**. Todas nasciam da mesma escolha: o gatilho era `BEFORE INSERT` e dividia pelo saldo lido da `vw_estoque_atual`, que é o saldo de hoje.

| # | Falha | Estava documentada? |
|---|---|---|
| 1 | Excluir ou editar uma Entrada não desfazia o custo médio | Sim |
| 2 | **Lançamento retroativo já calculava errado na inserção**, porque usava o saldo de hoje e não o saldo da data do lançamento | Não |
| 3 | **Mexer numa venda antiga corrompia o custo**: concluir, desconcluir ou excluir um pedido muda o saldo histórico, e o custo nunca recalculava | Não |
| 4 | **Erro de digitação no custo era incorrigível.** Ajuste conserta quantidade, não custo; não havia exclusão; e o cadastro não aceita custo digitado | Não |

**Causa raiz**: o `custo_atual` era atualizado de forma incremental, mas é derivado de um histórico que pode mudar. Manter de cabeça um valor que depende da ordem dos eventos, sobre um histórico editável, é frágil por construção. É a diferença entre anotar o saldo do banco a cada movimento e somar o extrato: no primeiro jeito, um lançamento errado contamina tudo daí para frente e não há como descobrir onde começou.

**Por que não bastava tratar o DELETE.** Subtrair aquela compra da média não funciona: depois que outras compras entraram, a média não é reversível. Não se tira um ingrediente do bolo pronto.

**Solução aplicada**: função `recalcular_custo_produto`, que refaz a conta do zero lendo o histórico em ordem cronológica, com a mecânica descrita na seção 6. O gatilho `trg_custo_produto` e a função `atualiza_custo_produto` foram removidos. Três gatilhos novos disparam a releitura: movimentação de estoque em INSERT, UPDATE e DELETE; item de pedido; e mudança de status ou de data de conclusão do pedido.

**Testado contra o banco real em 31/07/2026**, com o cenário da falha 4 e limpeza conferida depois: custo em 7,50; Entrada de 20 unidades com custo digitado como 90,00 levou a média a 32,50; **exclusão da Entrada devolveu o custo a 7,50 sozinho**.

**Concluído em 31/07/2026**: o endpoint OLV2 Estoque ganhou a ação `excluir`, restrita ao administrador, e o painel ganhou o botão correspondente na linha da movimentação, com confirmação em duas etapas (clique no botão mais confirmação do navegador). A saída automática gerada por uma venda concluída nunca mostra o botão, porque ela não é um lançamento gravado, é calculada.

### 3.12 Cadastros do financeiro (criados em 31/07/2026)

Criados a pedido do Rogério, para que os parâmetros do negócio sejam **visíveis e editáveis por ele**, sem depender de sessão com IA para mudar um dado simples.

**Construídas e publicadas em 31/07/2026**: o workflow OLV2 Cadastros e a tela de Cadastros foram construídos e publicados nesta mesma data, antecipando parte da etapa 9.7 a pedido de Rogério, porque o ganho de não depender de sessão com IA para ajustar um dado simples superava o risco. A permissão aqui é simples (administrador ou nada), sem depender do desenho de setores ainda em aberto na 9.7. Detalhe do workflow na seção 4.2 e da tela na seção 5.4.

| Tabela | Guarda | Preenchida em 31/07 |
|---|---|---|
| `contas` | Onde o dinheiro para. Nome, tipo, banco, agência, número, saldo inicial | Banco Itaú, Banco Sicredi, Banco Digital, Caixa da Loja |
| `maquininhas` | Operadora de cartão e a conta onde credita | Rede |
| `bandeiras` | Mapeia a bandeira para o grupo de taxa | Mastercard e Visa em Padrão, Elo em Outras |
| `taxas_cartao` | Taxa por maquininha, grupo, modalidade e parcelas, com vigência | 8 linhas da Rede |
| `formas_pagamento` | As formas e o **comportamento** de cada uma | 8 formas espelhando o comportamento atual |
| `categorias_conta_pagar` | Categorias de despesa | Mercadoria, Combustível, Veículo, Salários, Impostos, Aluguel, Utilidades, Outros |

#### Por que a taxa depende do grupo e não da bandeira

A tabela real da Rede mostrou um padrão: **Mastercard e Visa cobram idêntico em todas as modalidades, e a Elo cobra exatamente 0,80 ponto a mais em todas elas.**

| Modalidade | Padrão | Outras |
|---|---|---|
| Débito | 0,69% | 1,49% |
| Crédito à vista | 2,71% | 3,51% |
| Crédito 2x | 3,78% | 4,58% |
| Crédito 3x | 4,35% | 5,15% |

Tudo credita em **1 dia útil**, inclusive o parcelado. Isso simplifica bastante o Fluxo de Caixa: cartão não vira agenda de recebíveis futuros, vira dinheiro em um dia.

Então não existem três bandeiras para efeito de taxa, existem **dois grupos**. Na venda isso vira **um interruptor "Outras"**, não um seletor de bandeiras: um toque a mais só nas vendas em cartão.

**Decisão do Rogério, e o motivo dela**: chamar o grupo de "Outras" em vez de "Elo". Se amanhã entrar Amex ou Hipercard com taxa alta, basta cadastrar a bandeira no grupo Outras. Nenhuma linha de código muda. Se o rótulo fosse "Elo", cada bandeira nova exigiria mexer no sistema.

**Sem o interruptor, o erro é grande no débito**: 1,49% contra 0,69% é mais que o dobro. Num botijão de R$ 120, são R$ 0,96 por venda que o sistema diria estar ganhando e não ganhou.

#### Taxa não se sobrescreve, se sucede

`taxas_cartao` tem `vigencia_inicio` e `vigencia_fim`, e um índice único que só permite **uma linha vigente** por combinação de maquininha, grupo, modalidade e parcelas.

Quando a taxa mudar, não se edita a linha: encerra-se a vigente e cria-se a nova. Motivo idêntico ao da pendência 13: **a taxa aplicada é copiada para o pagamento no momento da venda**, e sobrescrever o cadastro reescreveria a margem de meses já fechados. É a mesma regra do custo, na seção 3.4.

#### A tabela de formas de pagamento está ligada desde 01/08/2026

A religação foi feita junto com a captura da taxa do cartão, numa operação só, para não mexer duas vezes na função de maior risco do sistema. O comentário da tabela no banco, que avisava o contrário, foi corrigido na mesma sessão.

O que passou a valer:

- Editar `gera_recebivel`, `prazo_dias`, `zera_pedido`, `exige_vencimento` e `usa_maquininha` **muda o comportamento na hora**.
- **Criar forma nova continua bloqueado**: a lista fechada em `pagamentos.forma` foi mantida de propósito, como segunda trava.
- **Excluir forma passou a ser impossível**, por gatilho. Antes era um buraco, ver o bloco de exclusão adiante.
- **Forma desativada é recusada na gravação.** Desativar Pix hoje faz o sistema recusar venda em Pix, que é o comportamento correto de "desativado" e não existia antes.

**O risco aceito continua o mesmo**: comportamento em campo dá flexibilidade e dá poder de errar. As travas cobrem as combinações contraditórias, mas não cobrem o erro plausível, como marcar "exige vencimento" no Dinheiro. A tela de cadastro ainda não avisa antes de salvar.

#### Travas de coerência

Com o comportamento virando dado, a coerência dele passa a ser responsabilidade do banco:

| Trava | O que impede |
|---|---|
| `ck_forma_zera_nao_gera` | Forma que zera o pedido e gera recebível ao mesmo tempo |
| `ck_forma_venc_exige_receb` | Forma que exige vencimento sem gerar recebível |
| `ck_forma_maquina_sem_taxa` | Forma com taxa própria e taxa de maquininha ao mesmo tempo, ou seja, duas verdades |
| `ck_taxa_grupo`, `ck_taxa_modalidade`, `ck_taxa_parcelas` | Grupo, modalidade ou parcela inválidos. Débito não tem parcela; crédito começa em 1 |
| `ux_taxa_vigente` | Duas taxas vigentes para a mesma combinação |

**O risco aceito**: comportamento em campo dá flexibilidade e dá poder de errar. As travas cobrem as combinações contraditórias, mas não cobrem o erro plausível, como marcar "exige vencimento" no Dinheiro. Quando a religação acontecer, a tela de cadastro precisa avisar antes de salvar.

#### Regra de exclusão, comum a todos os cadastros

Enquanto o registro não tiver sido usado em nenhum lançamento, pode excluir. **Depois de usado, desativa em vez de excluir.** É o mesmo princípio que já protege cliente e produto com lançamento ligado.

**A regra não estava funcionando (descoberto e corrigido em 01/08/2026).** Ela não é decidida por consulta: o `cadastro_operar` **tenta o DELETE e só desativa quando o banco recusa com violação de chave**. A regra inteira depende de existir uma chave que recuse. Onde não havia, a regra simplesmente não acontecia.

| Cadastro | O que acontecia | Correção |
|---|---|---|
| `maquininhas` | `taxas_cartao` tinha **ON DELETE CASCADE**. Excluir a Rede apagaria as 8 taxas em silêncio, e a desativação nunca era acionada | Chave trocada para **ON DELETE RESTRICT** |
| `formas_pagamento` | **Nenhuma chave aponta para ela**, porque `pagamentos.forma` é texto. O DELETE sempre passava. Depois da religação, apagar uma linha quebraria toda venda naquela forma | Gatilho que recusa DELETE sempre |
| `categorias_conta_pagar` | Mesmo caso, `contas_pagar.categoria` é texto | Gatilho que recusa DELETE quando já usada |

`bandeiras` fica de fora de propósito: o grupo é copiado para o pagamento como texto, então excluir bandeira não corrompe histórico nenhum.

**Ponto fraco que sobra**: a mensagem devolvida é sempre "Já usado em algum lançamento: desativado em vez de excluído". Para forma de pagamento é impreciso, porque o motivo real é a lista ser fechada. Melhoria de tela, sem urgência.

---

## 4. OS WORKFLOWS N8N

### 4.1 O conjunto antigo, e a separação descoberta em 31/07/2026

Até a v3.4 estes cinco eram tratados como um bloco só, "os fluxos antigos a aposentar". **A auditoria da virada mostrou que não são um bloco.** Dois deles sustentam também o painel novo e não podem ser desligados.

| Workflow | ID | Endpoint | Situação em 31/07/2026 |
|---|---|---|---|
| OLV Painel Mobile (web) | `WAiagumIwB8viELn` | /webhook/olv-painel e /webhook/olv-dados | **Desativado em 31/07/2026.** Guardado como ponto de retorno |
| OLV Vendas – Lançamento | `9Fx0Y4zvPq7PcHKK` | POST /webhook/olv-venda | **Desativado em 31/07/2026.** Guardado como ponto de retorno |
| OLV Estoque – Lançamento | `fgmMA5a6hiEZr9z2` | POST /webhook/olv-estoque | **Desativado em 31/07/2026.** Guardado como ponto de retorno |
| **OLV Login** | `W4qgfRIna8BIRUVz` | POST /webhook/olv-login | **Ativo, e permanece ativo.** Camada compartilhada |
| **OLV Contas** | `UBMRIgBy2jcBESBw` | POST /webhook/olv-contas | **Ativo, e permanece ativo.** Camada compartilhada |

**Por que OLV Login e OLV Contas não são "sistema antigo".** Eles não falam com a base anterior nem com o Supabase: leem e gravam na Data Table `OLV Usuarios`, interna do n8n. São a **camada de acesso**, compartilhada pelos dois painéis, conforme a seção 2.1. Desligar o OLV Login derrubaria o login do painel novo por inteiro. Desligar o OLV Contas eliminaria a única forma de criar usuário e trocar senha.

**Regra que fica**: a virada aposenta os fluxos de **dados**, não os de **acesso**. A camada de acesso só sai quando for substituída por outra, não quando o painel antigo sair.

**Nenhum workflow foi excluído.** Os três desativados continuam guardados, com o histórico de versões intacto, e voltam ao ar religando o botão de ativo.

**Efeito colateral registrado**: com o OLV Painel Mobile desativado, some a única tela que alcança o OLV Contas. Ver pendência 17, seção 8.3.

### 4.2 Do painel novo, sobre o Supabase (construídos em 30/07/2026)

Prefixo OLV2 para não haver confusão com o conjunto em produção. **Ativados em 30/07/2026**, junto com a correção de CORS descrita na seção 9.2. O painel antigo continua intocado e no ar; os dois conjuntos rodam em paralelo até a virada.

| Workflow | Endpoint | Ações |
|---|---|---|
| OLV2 Dados (painel novo) | POST /webhook/olv2-dados | `tudo` (carga inicial) e `pedidos` (recarga leve) |
| OLV2 Pedido (painel novo) | POST /webhook/olv2-pedido | `criar`, `status`, `editar_leve`, `excluir` |
| OLV2 Estoque (painel novo) | POST /webhook/olv2-estoque | `lancar`, `historico` |
| OLV2 Clientes (painel novo) | POST /webhook/olv2-clientes | `criar`, `editar`, `buscar` |
| OLV2 Cadastros (painel novo, 31/07/2026) | POST /webhook/olv2-cadastros | `listar`, `criar`, `editar`, `excluir` (ou `desativar` quando já usado), por tabela |

**Alterações de 01/08/2026, todas publicadas:**

| Workflow | O que mudou |
|---|---|
| **OLV Contas** | `allowedOrigins` liberado para a origem do painel, fechando a pendência 17. Ação nova `resetar_senha`, exclusiva do administrador |
| **OLV2 Dados** | Consultas novas `Consultar Pagamentos T` e `Consultar Pagamentos P`, trazendo o valor por linha de pagamento e o vencimento do recebível. Campos novos da view incluídos e filtrados por papel. Totais passaram a contar só pedido concluído e ganharam `taxa_cartao`, `lucro_liquido`, `margem_liquida_pct`, `por_forma` e `pedidos_em_aberto` |
| **OLV2 Pedido** | Ação nova `editar`, que chama a `atualizar_venda`. O nó `OK Criar` passou a devolver `avisos` |

**Achado que evitou trabalho**: o nó `Prep Criar` do OLV2 Pedido repassa o array de pagamentos inteiro, sem mapear campo a campo. Por isso os campos de cartão passam direto para o banco, e o interruptor de bandeira não exigiu nenhuma mudança de workflow.

**Confirmado ativo em 31/07/2026** (validação técnica da virada, seção 9.1): os quatro primeiros continuam ativos e no ar. O quinto, OLV2 Cadastros, foi construído e publicado na sessão seguinte, mesmo dia.

### 4.3 OLV2 Cadastros (construído em 31/07/2026)

Workflow ID `fV9fzAM81cT2Pvma`. Cobre as sete tabelas de cadastro do financeiro (seção 3.12): `produtos` (edição do `preco_venda`), `contas`, `maquininhas`, `bandeiras`, `taxas_cartao`, `formas_pagamento` e `categorias_conta_pagar`.

Mesmo bloco de entrada dos outros quatro (Normalizar, Autenticar, Assinar, Autorizar, Autorizado?, Perfil, Rotear), mais um gate de administrador logo em seguida, porque a tela de Cadastros inteira é exclusiva dele, sem exceção por tabela.

Regra geral de exclusão, comum às sete tabelas: registro sem uso em nenhum lançamento é excluído de verdade; registro já usado tem o botão trocado para Desativar, sem apagar o histórico. `custo_atual` e `custo_ultima_compra` de produtos nunca são editáveis pela tela, só o `preco_venda`. Taxa de cartão nunca é sobrescrita: editar fecha a vigência da linha atual e cria uma linha nova. Criar forma de pagamento nova é bloqueado, porque a lista é fechada no banco (ver 3.12).

**Testado contra o banco real em 31/07/2026**, com limpeza conferida depois, e publicado.

**Auditoria da instância atualizada**: com o OLV2 Cadastros, a instância passa a ter **15 workflows**, não 14. Usa a mesma e única credencial "Supabase OLV", do tipo Postgres, como todos os outros. Não muda a conclusão da pendência 16.

**Bloco de entrada idêntico nos quatro**: Normalizar, Autenticar, Assinar, Autorizar, Autorizado?, Perfil, Rotear. O nó `Perfil` calcula uma coisa só, `isAdmin = papel === 'admin'`, e nenhum ponto dos quatro compara papel contra rótulo. O nó Crypto tem `type: SHA256` e `encoding: hex` fixados explicitamente desde o rascunho, que é a armadilha descrita na seção 8.2.

**O que cada ação faz.**

`criar` monta o pacote com venda, itens e pagamentos e chama a `registrar_venda` **uma vez só**. `status` altera apenas a situação e é liberado a qualquer usuário ativo, porque é operacional e reversível. `editar_leve` cobre `status`, `tipo_entrega`, `observacao` e `data`, restrito ao dono do lançamento ou ao administrador. `excluir` tem a mesma restrição. Valores de item, desconto, cliente e pré-pago ficam de fora, pelo motivo da seção 3.10.

**A permissão de dono é conferida dentro da própria instrução de gravação**, não num nó separado que lê antes e grava depois. Assim não existe janela entre conferir e gravar. Quando o pedido não é encontrado ou o usuário não é o responsável, a resposta é a mesma mensagem, e nada é alterado.

**Testado contra o banco real em 30/07/2026**, com limpeza conferida depois: pedido completo com recebível, reenvio com a mesma chave devolvendo `repetido` sem duplicar, edição bloqueada para quem não é dono e liberada para o dono, e cliente duplicado barrado com o nome do cliente já cadastrado na mensagem.

**Correção de CORS em 30/07/2026**: nenhum dos cinco webhooks usados pelo painel novo (OLV Login e os quatro OLV2) tinha a opção `allowedOrigins` configurada no nó de webhook. Sem isso, o navegador bloqueava toda chamada feita a partir de `https://painel.olvdistribuidora.com.br`, mesmo com o restante do fluxo correto. Corrigido nó a nó, liberando essa origem específica (não `*`), e publicado em cada um dos cinco workflows. Detalhe completo na seção 9.2.

**Atualização de 31/07/2026**: a virada foi feita. Os três fluxos de dados da base anterior foram desativados e o painel novo passou a ser o único caminho de operação. Ver seção 9.1.

**Auditoria completa da instância (31/07/2026, regra 8).** A instância tem **14 workflows**, não 9. Além dos cinco antigos e dos quatro OLV2, existem cinco de outros assuntos: OLV Atendimento (Agente WhatsApp), Automação Estoque Crítico (aula) e três do Casa em Ordem. **Nenhum deles usa o Supabase**: a instância inteira tem uma única credencial apontando para o banco, a "Supabase OLV", do tipo Postgres, e não existe credencial de API REST em lugar nenhum. Por isso a correção da pendência 16 não os afeta.

**Arquivado**: `TEMP - Leitura OLV Painel HTML`, usado na renomeação de 28/07. Não roda mais, mas continua guardado na seção de arquivados, porque as ferramentas usadas não têm exclusão definitiva.

---

## 5. O PAINEL (FUNCIONALIDADES ATUAIS)

### 5.1 Vendas

Filtros de período, indicadores do período, quantidade por produto, valores por forma de pagamento, gráfico de evolução, formulário com autocompletar de cliente, editar e excluir com confirmação, filtros de consulta.

### 5.2 Estoque

Tabela de estoque atual por produto, lançamento de Entrada, Ajuste e Estoque Inicial, histórico com editar e excluir.

### 5.3 Login, usuários e papéis

Login com usuário e senha, crachá com nome e papel, troca de senha obrigatória no primeiro acesso, área de Usuários só para administrador, responsável preenchido automaticamente, trava de dono conferida também no servidor.

**Resolvido em 01/08/2026**: o administrador redefine a senha de outro usuário pela tela de Usuários. A ação `resetar_senha` gera sal e senha novos e **sempre marca a troca como pendente**, então quem redefine nunca fica sabendo a senha definitiva de ninguém.

**A troca obrigatória do primeiro acesso acontece dentro do painel novo desde 01/08/2026.** Até então o login **bloqueava** quem tinha a marca de troca pendente, com a mensagem "troque a senha pelo painel atual". O painel atual saiu do ar em 31/07, então a saída indicada não existia mais e a pessoa ficava **trancada fora do sistema**. Foi o que aconteceu com a usuária Gabriele.

Agora quem tem troca pendente entra numa tela isolada, sem menu e sem dados carregados, e só sai dela trocando a senha. A sessão só é guardada depois da troca, então recarregar a página não pula a etapa. O OLV Login não precisou de mudança: ele já devolvia o token junto com a marca.

### 5.4 Cadastros (construído em 31/07/2026)

Tela exclusiva do administrador. Cobre as sete tabelas de cadastro do financeiro: produtos (edição do preço de venda sugerido), contas, maquininhas, bandeiras, taxas de cartão, formas de pagamento e categorias de conta a pagar.

Regra geral: registro sem uso é excluído de verdade; registro já usado em algum lançamento tem o botão trocado para Desativar. Custo do produto (`custo_atual`, `custo_ultima_compra`) nunca editável pela tela, só o preço de venda sugerido. Taxa de cartão nunca é sobrescrita, sempre sucede a vigência anterior. Criação de forma de pagamento nova bloqueada, porque a lista é fechada no banco (ver 3.12).

### 5.5 Usuários (construído em 01/08/2026)

Tela exclusiva do administrador, ligada ao OLV Contas. Faz listar, criar, mudar papel, ativar, desativar, redefinir a senha de outra pessoa e trocar a própria.

Duas travas de coerência: ninguém desativa a si mesmo, e a coluna "Primeiro acesso" mostra quem está com a troca de senha pendente.

**Nota de contrato**: o OLV Contas usa `action`, não `acao`, porque é anterior aos endpoints OLV2. Mantido como está de propósito. Mexer no contrato do fluxo de acesso em produção é risco sem ganho.

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

**O custo é recalculado a partir do histórico (31/07/2026).** Até esta data o custo era um número atualizado a cada compra, guardado no cadastro. Isso quebrava de quatro formas, descritas na pendência 13. Agora **o custo não tem memória**: a cada mudança no histórico, o banco refaz a conta do zero, lendo os lançamentos em ordem de data.

A releitura funciona assim, do último Estoque Inicial em diante:

| Lançamento | Efeito na quantidade | Efeito no custo |
|---|---|---|
| Estoque Inicial | Passa a ser a quantidade contada | Passa a ser o custo informado. Zera o histórico anterior |
| Entrada com custo | Soma | **Média ponderada.** Atualiza também o `custo_ultima_compra` |
| Entrada sem custo | Soma | Nenhum |
| Ajuste | Soma ou subtrai | **Nenhum.** O banco recusa Ajuste com custo |
| Pedido concluído | Subtrai | Nenhum. Vender nunca altera o custo médio |

**Só a compra altera o custo.** Estoque Inicial e Entrada são os únicos lançamentos que mexem no custo médio, e os dois são exclusivos do administrador. Ajuste corrige saldo (quebra, perda, conferência) e nunca toca no número que é base do lucro.

**Avaliado e descartado em 31/07/2026: Ajuste com custo.** Chegou a ser implementado, com o custo do ajuste substituindo a média, e foi revertido no mesmo dia. Motivo: criava um **segundo caminho para alterar a base do lucro**, concorrente com a Entrada e sem nota fiscal por trás. Também obrigaria a fechar o Ajuste para o administrador ou a inventar uma trava por campo, quando o Ajuste precisa ficar livre para quem opera o balcão.

**Como corrigir custo errado, então**: apagar a Entrada errada e lançar de novo, agora que a exclusão está liberada e a releitura desfaz o efeito sozinha. O histórico fica honesto, porque a compra passa a ter o valor que realmente teve, em vez de carregar um remendo por cima.

**O lucro histórico não muda.** O `venda_itens.custo_unitario` é copiado no momento da venda e fica congelado. A releitura mexe apenas no `produtos.custo_atual`, que é retrato do presente. Meses fechados ficam intactos.

**Ordem dentro do mesmo dia**: as datas não guardam hora. Quando compra e venda caem no mesmo dia, a releitura assume compra primeiro. Só importa nesse caso. Se um dia incomodar, a saída é gravar hora nas movimentações.

**Consequência prática, testada em 31/07/2026: não lance Estoque Inicial num produto que já teve Entrada no mesmo dia.** A contagem zera o histórico anterior, mas a Entrada do mesmo dia é considerada posterior a ela e entra de novo por cima. No teste, uma contagem de 45 unidades num produto com Entrada de 46 no mesmo dia resultou em saldo 90. Nesse caso, use **Ajuste**, que corrige o saldo sem duplicar e sem tocar no custo.

**Limite aceito**: produto sem nenhum Estoque Inicial e sem nenhuma Entrada **mantém o custo que já tinha**, em vez de zerar. É a trava que protege os custos semeados no cadastro de 30/07. Enquanto a contagem física não for lançada, esses números seguem sem origem no histórico.

### Lucro

Lucro do pedido = valor_total (já com desconto e acréscimo) − soma dos custos dos itens. Calculado pela view `vw_pedidos`.

**Desconto e acréscimo são colunas separadas (31/07/2026)**, e não um único campo que aceita negativo. Motivo: assim o relatório de descontos concedidos no mês não se anula com os acréscimos cobrados. Num campo só, R$ 500 de desconto e R$ 500 de acréscimo apareceriam como zero, e os dois números se perderiam.

**A taxa da maquininha entra no cálculo desde 01/08/2026**, sem substituir a margem bruta. A `vw_pedidos` passou a trazer as duas, lado a lado. Decisão do Rogério, detalhada na seção 9.4.

Prova numérica, botijão de R$ 120 com custo de R$ 87:

| Forma | Taxa | Margem bruta | Margem líquida |
|---|---|---|---|
| Dinheiro | 0,00 | 27,50% | 27,50% |
| Débito Padrão | 0,83 | 27,50% | 26,81% |
| Débito Outras | 1,79 | 27,50% | 26,01% |
| Crédito 3x Outras | 6,18 | 27,50% | 22,35% |

A diferença de R$ 0,96 entre débito Padrão e Outras é exatamente a que a seção 3.12 previu.

### Faturamento conta só pedido concluído (decidido em 01/08/2026)

Até esta data o estoque baixava na conclusão mas o faturamento contava desde o lançamento: **dois critérios diferentes para o mesmo pedido**. Agora é um só, o da conclusão.

Vale para os quatro indicadores do Dashboard, para a margem do período, para a lista de quantidade por produto e para os valores por forma de pagamento. O Dashboard mostra uma linha discreta dizendo quantos pedidos do período seguem em aberto, para o número não parecer que sumiu quando se lança uma venda.

### Correção de custo do primeiro dia de uso (01/08/2026)

O Estoque Inicial do Gás 13kg foi lançado com **quantidade 37 e custo 37,00**. O custo real é 87,00. Os outros cinco produtos bateram com a planilha; só o gás repetiu a quantidade no campo de custo.

Efeito: cada botijão aparecia rendendo R$ 88,00 em vez de R$ 38,00, e a margem do dia aparecia em 63,17% em vez de 34,35%.

Corrigido em **duas frentes**, porque o custo vive em dois lugares por desenho:

1. `produtos.custo_atual`: o Estoque Inicial errado foi excluído e relançado com custo 87,00, pelo caminho que esta seção já define, com observação registrada na própria movimentação.
2. `venda_itens.custo_unitario`: corrigido nas 4 vendas de gás já lançadas. **Nem relançar a movimentação, nem editar o pedido resolveria isso**, porque o custo do item é congelado no momento da venda e a `atualizar_venda` preserva o original de propósito.

Conferido depois: saldo de estoque inalterado em 34 botijões, faturamento inalterado, lucro do dia caindo de R$ 438,28 para R$ 238,28, exatamente os R$ 200,00 previstos pela conta.

**Regra que fica**: erro de digitação no custo do Estoque Inicial exige as duas correções. Corrigir só a movimentação deixa o lucro das vendas já feitas errado para sempre.

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
| **Ajuste de estoque** | **Qualquer usuário ativo** | **Corrige saldo e não toca em custo.** O banco recusa Ajuste com custo |
| **Excluir movimentação** | **Só administrador** | **Liberado em 31/07/2026**, depois que a releitura passou a desfazer o efeito no custo |
| **Cadastros do financeiro** (criar, editar, excluir ou desativar) | **Só administrador** | **Construído em 31/07/2026.** Tela inteira é financeira e de configuração, sem uso operacional para o colaborador |
| **Editar pedido por completo** (itens, valores, forma de pagamento) | **Dono do lançamento ou administrador** | **Liberado em 01/08/2026.** Mesma regra do `editar_leve`, porque é a mesma categoria de risco: irreversível na prática |
| **Redefinir senha de outro usuário** | **Só administrador** | **Criado em 01/08/2026.** Conferido no servidor, com trava positiva sobre `admin` |

**Por que Entrada é exclusiva do administrador.** O custo unitário é um dos sete pontos financeiros protegidos da seção 8.1. Se o colaborador não pode informar custo, uma Entrada lançada por ele entraria sem custo e não atualizaria a média, deixando o lucro errado sem aviso. Fechar a Entrada e abrir o Ajuste mantém o número do custo confiável sem travar a operação do dia a dia.

**Por que a exclusão de movimentação é exclusiva do administrador (31/07/2026).** Apagar uma Entrada refaz o custo médio do produto, então ela mexe na base do lucro pelo mesmo motivo que a Entrada mexe. Já o Ajuste, que não toca em custo, segue liberado a qualquer usuário ativo, porque quebra e perda são lançadas no balcão, na hora, por quem está lá.

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
| `pagamentos` (01/08/2026) | `taxa_pct`, `taxa_valor`, `valor_liquido`. **Forma, valor, vencimento, grupo, modalidade e parcelas continuam saindo**, porque o colaborador precisa deles para corrigir o pedido na entrega |
| Totais agregados | **Não são sequer calculados** |

**Ampliado em 01/08/2026**, junto com os campos novos da view. A lista de campos a remover virou constante nomeada no topo do código dos nós `Filtrar Saída`, `FIN_PEDIDO` e `FIN_PAGAMENTO`, para o próximo campo novo ter um lugar óbvio para entrar. Os campos financeiros da `vw_pedidos` que entraram são `taxa_total`, `lucro_liquido`, `margem_liquida_pct` e `recebido_liquido`.

**Achado tranquilizador da auditoria**: os nós que consultam a `vw_pedidos` listam colunas nomeadas, não usam `select *`. Por isso os campos novos **não vazaram em nenhum momento**, nem entre a criação da view e o ajuste do filtro. É o motivo registrado na seção 3.9 para preferir lista plana com campos nomeados, funcionando na prática.

`acrescimo` sai para todos os papéis, como `desconto` já saía. É operacional, não financeiro.

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
| 16 | ~~Views e funções acessíveis direto pela API do Supabase, sem passar pelo n8n~~ | **Resolvido em 31/07/2026**: acesso dos papéis públicos revogado e privilégio padrão fechado. Ver detalhamento abaixo |
| 17 | ~~Gestão de usuários sem caminho no painel novo~~ | **Resolvida em 01/08/2026**, em três frentes. Ver detalhamento abaixo |

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

Descoberta em 31/07/2026, durante a validação técnica da virada (seção 9.1). O auditor de segurança do Supabase apontou que duas views (`vw_pedidos`, `vw_estoque_atual`) e cinco funções (`registrar_venda`, `atualiza_custo_produto`, `checa_venda_consistente`, `recalc_total_desconto`, `recalc_total_venda`) foram criadas como **SECURITY DEFINER**, o que faz elas ignorarem a RLS das tabelas de origem. Ao mesmo tempo, os papéis públicos do Supabase, `anon` (sem login) e `authenticated`, tinham `SELECT` nas views e `EXECUTE` nas funções liberados.

Na prática, isso significa que hoje, sem passar pelo painel nem pelo n8n:

- Qualquer pessoa com a URL do projeto e a chave pública do Supabase (que é pública por natureza, não um segredo) conseguia ler `vw_pedidos` direto pela API REST e ver lucro, custo e margem de todo pedido, o mesmo dado que o mecanismo da seção 8.1 se esforça para esconder do colaborador.
- Qualquer pessoa consegue chamar `registrar_venda` direto pela API REST e criar pedidos, sem token, sem autenticação e sem as validações do n8n.

A RLS das 8 tabelas está correta (bloqueia acesso direto, sem política = acesso negado por padrão), mas as views e funções SECURITY DEFINER contornam essa trava. Isso contradiz a regra da seção 2 ("o navegador nunca fala direto com o Supabase"): a brecha independe do painel, existe hoje, e fica mais grave depois da virada, quando o Supabase vira a fonte única de verdade.

**Resolvida em 31/07/2026.** A avaliação de impacto veio antes da correção, com cinco conferências:

| Conferência | Resultado |
|---|---|
| Papel usado pelo n8n | `postgres`, com `bypassrls` ligado e dono de todas as tabelas. Ignora RLS e privilégio, logo não é afetado por revogação |
| Credenciais de Supabase no n8n | Uma só, do tipo Postgres. **Não existe credencial de API REST em nenhum dos 14 workflows da instância** |
| Log da API do Supabase nas últimas 24h | **Vazio.** Nada consumia a API REST. Confirmação por evidência, não por dedução |
| Usuários no Supabase Auth | Zero. O papel `authenticated` nunca foi usado |
| Buckets de Storage | Zero. Nada a quebrar do lado dos anexos |

**A primeira tentativa de correção falhou, e a conferência pegou.** Revogar apenas de `anon` e `authenticated` não fechou nada: `registrar_venda` continuou chamável sem login. O motivo é uma armadilha do PostgreSQL que vale registrar, porque vai reaparecer.

**Toda função nasce com permissão de execução para `PUBLIC`**, um papel coringa que significa "qualquer um", e o schema nasce com permissão de uso para `PUBLIC` também. Como `anon` herda de `PUBLIC`, tirar o nome dele da lista não adianta: ele continua entrando pela herança. É trancar a porta da frente e deixar a de serviço aberta.

**O que foi aplicado, em duas migrações:**

1. Revogado todo privilégio de `anon` e `authenticated` nas 8 tabelas, nas 2 views e nas 6 funções.
2. Revogado o privilégio herdado de `PUBLIC` nas funções e no schema. **Este era o buraco real.**
3. Revogado o `USAGE` no schema `public`, de forma que os papéis públicos não enxergam sequer o schema.
4. Fechados os **privilégios padrão**, para que objeto novo nasça sem acesso. Sem isso, a próxima tabela criada reabriria a brecha sozinha.
5. As duas views passaram a `security_invoker`, deixando de contornar a RLS por construção.
6. Incluída a função `set_data_conclusao`, que também estava exposta e **não constava no diagnóstico original**.

**Conferido depois, item a item**: `anon` e `authenticated` sem acesso a schema, tabelas, views e funções; `postgres` e `service_role` com tudo preservado; consulta real pelo papel do n8n devolvendo os 910 clientes e os 6 produtos.

**Auditor de segurança do Supabase rodado depois**: restaram 8 avisos, todos de nível informativo e todos do mesmo tipo, `rls_enabled_no_policy`. Esses 8 **não são falha, são o desenho**: RLS ligada sem política significa acesso negado por padrão, exatamente o que a seção 3.10 determina. O aviso existe para quem esqueceu de escrever a política, não para quem decidiu não ter nenhuma.

**Pendência 17, detalhada.**

Aberta em 31/07/2026, durante a auditoria da virada. **Não existe caminho para gerenciar usuários a partir do painel novo.**

O workflow OLV Contas continua ativo e funcional, mas o nó de webhook dele está **sem `allowedOrigins`**, ao contrário dos cinco que receberam a correção de CORS em 30/07. Na prática, o navegador bloqueia qualquer chamada vinda de `painel.olvdistribuidora.com.br`. E não existe um OLV2 Contas.

O que fica sem caminho pela tela nova: criar usuário, mudar papel, ativar e desativar acesso, e **a troca de senha obrigatória do primeiro acesso**. Enquanto isso não for resolvido, essas ações se fazem editando a Data Table `OLV Usuarios` direto no n8n.

**Era pior do que este diagnóstico.** O painel novo não só deixava de alcançar a gestão de usuários: ele **bloqueava o login** de quem tinha `precisa_trocar_senha`, com a mensagem "troque a senha pelo painel atual". O painel atual foi desativado em 31/07, então a saída indicada não existia mais e a pessoa ficava **trancada fora do sistema**, sem caminho nenhum. Aconteceu com a usuária Gabriele.

**Resolvida em 01/08/2026, em três frentes indissociáveis:**

| Frente | O que mudou |
|---|---|
| **CORS** | O nó `Contas Webhook` estava com `options` vazio. Recebeu `allowedOrigins` com a origem específica do painel, no padrão dos outros cinco |
| **Redefinição por administrador** | Ação nova `resetar_senha` no OLV Contas. Gera sal e senha novos e marca a troca como pendente, então quem redefine nunca fica sabendo a senha definitiva de ninguém |
| **Troca obrigatória dentro do painel** | O login deixou de bloquear. Quem tem troca pendente entra numa tela isolada, sem menu e sem dados, e só sai dela trocando a senha. A sessão só é guardada depois, então recarregar não pula a etapa |

**Por que as três juntas**: sem a terceira, a redefinição recriaria a mesma trava que ela resolve.

**O OLV Login não precisou de mudança**: ele já devolvia o token junto com a marca de troca pendente, então o fluxo mais crítico em produção ficou intocado.

**Auditoria do OLV Contas, feita junto (regra 8)**: todas as travas de papel são positivas sobre `admin`, nenhuma comparação negativa contra rótulo, nó Crypto com `type` e `encoding` gravados. De acordo com a regra da 8.2.

**Testado**: fluxo completo de redefinição, e a recusa quando quem chama é colaborador, com o nó de gravação nem chegando a rodar.

### 8.4 Toda gravação viaja em base64 (30/07/2026)

O nó Postgres do n8n separa os parâmetros da consulta **por vírgula**. Não há como escapar uma vírgula dentro de um valor. Qualquer observação de pedido com vírgula quebraria a gravação, e texto livre quase sempre tem vírgula.

**Solução adotada**: cada gravação envia **um único parâmetro**, que é o pacote JSON inteiro codificado em base64. O SQL decodifica na primeira linha da consulta e lê os campos de dentro. O alfabeto do base64 não tem vírgula, então o problema desaparece na origem.

É como despachar uma encomenda lacrada em vez de espalhar os itens soltos na esteira: o que vai dentro não interfere no transporte.

**Efeito colateral bom**: nada do que o usuário digita entra na consulta como texto. Fecha a porta para injeção de SQL sem esforço adicional.

**Testado em 30/07/2026** com o texto "teste tecnico, com virgula", gravado e lido corretamente.

### 8.5 queryReplacement do nó Postgres separa por vírgula, e descarta parâmetro vazio (achado em 31/07/2026)

Achado nos testes reais desta sessão, em dois workflows que já estavam publicados antes dela, sem relação com as tarefas do dia.

**O problema**: o campo `queryReplacement` do nó Postgres do n8n separa os valores por vírgula, criando um parâmetro posicional (`$1`, `$2`...) para cada pedaço. Isso quebra de duas formas. Primeiro, um valor único que contém vírgula, como uma lista de IDs no formato `{19,18,12}`, é cortado em pedaços e vira parâmetros errados. Segundo, um valor que resolve para string vazia é **descartado** da lista de parâmetros, deslocando a numeração e quebrando qualquer `$N` que vinha depois dele na consulta.

**Onde apareceu**:

| Workflow, nó | Sintoma |
|---|---|
| OLV2 Dados, `Consultar Itens T` e `Consultar Itens P` | Quebrava sempre que havia mais de um pedido no mesmo dia (parâmetro cortado pela metade), ou quando não havia nenhum pedido no período (parâmetro vazio descartado) |
| OLV2 Estoque, `Consultar Histórico` | O filtro "Tipo = Todos" mandava parâmetro vazio, descartado pelo n8n, quebrando a referência a `$3` na consulta |

**Correção aplicada nos três nós**: trocar o separador de vírgula por outro caractere (`|`) e usar `string_to_array($1,'|')::bigint[]` no SQL para listas de IDs; e sempre garantir um valor de segurança (`'0'` para lista vazia, `'TODOS'` para filtro vazio) quando o campo pudesse resolver para string vazia, para o parâmetro nunca ser descartado. Testado com os dois cenários (com dado e vazio) e publicado.

**Regra que fica**: todo `queryReplacement` novo que monte lista ou possa resolver para vazio precisa desse cuidado. Vale para qualquer nó Postgres novo no projeto, não só os três corrigidos.

### 8.6 Função nova nasce aberta (achado em 01/08/2026)

**A regra escrita na seção 3.10 até a v3.8 estava errada.** Ela afirmava que "objeto novo no schema `public` nasce fechado", como resultado dos privilégios padrão aplicados em 31/07. Testado três vezes nesta sessão, com o mesmo resultado:

| Objeto novo criado por `postgres` no schema `public` | Nasce |
|---|---|
| Tabela | **Fechada**, só `postgres` e `service_role` |
| Função | **Aberta**, com execução para `PUBLIC` |

Tentar corrigir pelo caminho óbvio não resolve: `ALTER DEFAULT PRIVILEGES IN SCHEMA public REVOKE EXECUTE ON FUNCTIONS FROM PUBLIC` foi aplicado e a função criada em seguida continuou nascendo aberta.

**O que ainda protege hoje**: `anon` e `authenticated` não têm `USAGE` no schema `public`, então não alcançam nem o schema. É uma tranca só, e ela é a única. Se alguém devolver o `USAGE` por engano, tudo abre de uma vez.

#### A rotina criada

| Função | O que faz |
|---|---|
| `fechar_acesso_publico()` | Revoga `anon`, `authenticated` e `PUBLIC` em schema, tabelas, sequências e funções, e fecha os privilégios padrão do papel corrente. Idempotente. Não toca em `postgres` nem em `service_role` |
| `auditar_acesso_publico()` | Lista todo objeto do schema e diz quais são alcançáveis por `anon`, `authenticated` ou `PUBLIC`. Já trata a armadilha da ACL nula em função, que significa aberta e não fechada |

**Regra que fica, e que substitui a da 3.10**: toda migração termina com `select fechar_acesso_publico();` e a conferência `select * from auditar_acesso_publico() where anon or autenticado or publico;`, que precisa devolver **zero linhas**.

**Testado de ponta a ponta**: com uma função e uma tabela abertas de propósito, mais `USAGE` devolvido para `anon`, a auditoria apontou os 3 pontos, o fechamento zerou os 3, e os objetos de teste foram removidos depois.

#### Risco residual registrado

Existe um privilégio padrão do papel `supabase_admin` no schema `public` que **libera tudo para `anon` e `authenticated`**. Ele não afeta o que as migrações criam, porque elas rodam como `postgres`, cujo privilégio padrão está fechado. Mas objeto criado por outro caminho pode nascer liberado. É mais um motivo para a rotina acima ser obrigatória e não opcional.

#### Aviso do auditor resolvido no mesmo dia

`cadastro_operar` era a única função com `search_path` livre, único aviso de nível WARN do auditor do Supabase. Fixado em `public, pg_temp` sem tocar no corpo, porque a função usa nomes sem schema. Restam **14 avisos** informativos do tipo `rls_enabled_no_policy`, que são o desenho e não falha. Eram 8 na v3.5 e passaram a 14 porque as 6 tabelas de cadastro entraram.

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

- 31/07: **pendência 16 resolvida** (seção 8.3), destravando a virada.
- 31/07: **virada feita.** Os três fluxos de dados da base anterior desativados, o painel novo passou a ser o único caminho de operação.
- 31/07: **pendência 13 encerrada por completo**: endpoint OLV2 Estoque ganhou a ação `excluir`, restrita ao administrador, e o botão correspondente foi publicado no painel. Seção 3.11.
- 31/07: **workflow e tela de Cadastros construídos e publicados**, antecipando parte da etapa 9.7. Seções 3.12, 4.3 e 5.4.

- 01/08/2026: **religação da `registrar_venda`** à tabela de formas e captura da taxa do cartão, fechando a decisão da 9.4.
- 01/08/2026: **função `atualizar_venda`**, edição completa de pedido. Seção 3.10.
- 01/08/2026: **pendência 17 resolvida** em três frentes. Seção 8.3.
- 01/08/2026: **contagem física lançada** como Estoque Inicial nos 6 produtos, pelo próprio Rogério.

#### Falta

- Publicar o painel v5 no Cloudflare, e depois redefinir a senha da Gabriele e conferir o CORS da tela de Usuários no navegador.
- Preencher conta de destino nas formas e na maquininha (pendência 20) e saldo inicial das contas (pendência 21). São da tela de Cadastros.

#### A virada, feita em 31/07/2026

**Estado do banco no momento da virada**: 910 clientes, 6 produtos, 1 movimentação de estoque, **0 pedidos**. O painel novo não havia registrado nenhum pedido real até aqui.

**Recomendação apresentada**: adiar, operar em paralelo e virar depois de um domingo real, que é o dia de pico. **Decisão do Rogério**: seguir com a virada, porque **o painel antigo já não estava em uso**. Sem operação viva do lado antigo, o argumento de esperar perdia a base, e manter dois sistemas de pé só adiaria a estreia sem reduzir risco.

**O que foi desativado**: OLV Painel Mobile, OLV Vendas e OLV Estoque. **O que continua ativo**: OLV Login, OLV Contas e os quatro OLV2. Motivo da separação na seção 4.1.

**Ponto de retorno**: os três estão **desativados, não excluídos**, com histórico de versões intacto. Voltar é religar o botão de ativo em cada um.

**O que observar nos primeiros dias**: o primeiro pedido real de cada tipo (pré-pago, dividido, Crediário gerando recebível, Gratuidade e Retirada), a contagem de estoque lançada como Estoque Inicial, e o comportamento no domingo, que é o pico.

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

**Ajustes de 31/07/2026**: campo "Data do pedido" adicionado em Mais opções, com o dia de hoje por padrão e editável, para lançar pedido retroativo. Corrigido junto um bug pré-existente: o campo de data sempre mandava a data de hoje ao servidor, mesmo já havendo suporte no backend para outra data. O valor do pagamento passou a ser preenchido automaticamente com o total do pedido quando há uma única linha, continuando editável. O bloco de Produtos ganhou um seletor de Desconto ou Acréscimo, ligado à coluna `vendas.acrescimo` (criada na v3.6, seção 3.3): um único campo de valor, nunca os dois ao mesmo tempo, validado também no servidor. O campo de valor unitário passou a ser preenchido com `produtos.preco_venda` quando o produto é escolhido, continuando editável e negociável item a item; se a coluna estiver vazia para aquele produto, o campo fica em branco como antes. Os cards de pedido passaram a mostrar só a data, sem horário.

#### Carregamento automático (31/07/2026)

Todas as telas com filtro de período (Dashboard, Histórico de estoque; Vendas e Cadastros já carregavam assim) passaram a buscar os dados sozinhas ao abrir a tela, sem precisar clicar em Buscar.

#### Estoque

Indicadores, tabela de produtos com situação, e **histórico de movimentações** com filtro por tipo, mostrando a saída automática por venda concluída como lançamento do sistema.

**Atualização de 31/07/2026**: com a pendência 13 resolvida, o botão de excluir movimentação foi construído e publicado, restrito ao administrador, com confirmação em duas etapas. O Ajuste continua sem campo de custo. A saída automática de venda concluída nunca mostra o botão, porque é calculada, não gravada.

**Ajustes de uso, mesma sessão (31/07/2026)**: a aba Produtos ganhou um painel de "Movimentações de hoje" no rodapé, juntando entradas e ajustes reais com as saídas de venda calculadas do dia. A aba Histórico passou a carregar os últimos 7 dias por padrão (antes só o dia atual), a mostrar também as saídas de venda do período junto dos lançamentos reais, e ganhou um indicador visual (ponto verde para entrada, vermelho para saída). As datas de movimentação passaram a mostrar só dia e mês, sem horário. O filtro de período da aba Histórico passou a usar o mesmo padrão de pills do Dashboard e de Vendas, e a carregar sozinho ao abrir a tela, sem precisar clicar em Buscar.

#### Clientes

Busca por nome ou telefone, cadastro e edição. Sem histórico de pedidos por cliente nem indicadores, porque dependem de consultas que ainda não existem, previstas na seção 9.4.

**Ajuste de 30/07/2026**: a busca por telefone usava um campo (`telefone_norm`) que o endpoint não devolvia. Corrigido para extrair os dígitos do campo `telefone`, que existe na resposta. **Atualização de 31/07/2026**: o endpoint passou a devolver `telefone_norm` pronto (pendência 15, seção 3.11); simplificar a busca do painel para usar o campo pronto é limpeza opcional, sem urgência.

**Ajuste de 31/07/2026**: adicionada a coluna Endereço na tabela de clientes. Ordem final das colunas: nome, telefone, endereço, bairro, observação.

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

#### Painel v5 (01/08/2026, entregue e ainda não publicado)

**Interruptor de bandeira "Outras"**: aparece abaixo da linha de pagamento, só quando a forma é Crédito ou Débito, junto do seletor de parcelas, que só aparece no Crédito. Não é seletor de bandeiras, é interruptor de duas posições, como a 3.12 decidiu. O seletor oferece 1x, 2x e 3x, que é o que existe cadastrado; oferecer 4x geraria venda sem taxa. Quando a função devolve aviso, o painel mostra uma faixa âmbar dizendo que o pedido entrou mas com ressalva.

**Tela de Usuários**: seção 5.5.

**Edição completa de pedido**: o botão Editar passou a abrir **o mesmo formulário do lançamento**, preenchido. Reaproveitar em vez de escrever um segundo formulário é o que garante que a regra de pagamento e o interruptor de bandeira valham igual ao lançar e ao corrigir na entrega.

**Nove correções de uso pedidas pelo Rogério no primeiro dia de operação:**

| Ponto | O que era |
|---|---|
| Botão de fechar | Adicionado nos sete formulários, reaproveitando o Cancelar de cada um. Esc também fecha |
| Formulário voltava ao topo, e o Tab não andava | **Era um bug só**: `renderizar()` reconstruía a página inteira a cada campo, o que destrói foco e rolagem. Campos numéricos passaram a corrigir só os números na tela; onde reconstruir é inevitável, foco e rolagem voltam ao lugar |
| Cadastrar cliente de dentro do pedido | Abre o formulário de Clientes por cima, aproveita o que já foi digitado na busca e ao salvar já deixa o cliente escolhido |
| Telas não atualizavam sozinhas | Saldo e custo vinham da carga do login. Agora toda gravação marca o que envelheceu e a tela busca de novo ao abrir |
| Menu e topo rolavam junto | Presos. O menu rola por dentro quando não couber, para o Sair nunca sumir |
| Tela de Usuários mal formatada | Faltava `usuarios` no mapa de títulos do topo, e os botões estavam numa grade de filtro que os esticava |

Corrigido também o `tabindex` do interruptor de bandeira e do seletor de parcelas, que estavam fora da sequência do Tab por engano: a regra de 30/07 valia para botões de ação, não para campos.

**Defeito encontrado fora da lista**: `totaisDoPeriodo` devolvia nulo quando a resposta não trazia os totais, e o Dashboard lia o campo direto. Uma resposta sem totais derrubava a tela inteira em branco. Blindado.

**Código morto**: o formulário de edição leve deixou de ser usado. Pendência 23.

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

#### Desenho iniciado em 31/07/2026

**Ordem confirmada**: o desenho das regras é feito agora, mas **os endpoints só são construídos depois da etapa 9.7**, conforme a seção 10 já determinava. Construir antes significaria escrever a regra de permissão duas vezes, e o financeiro é justamente o módulo que o setor "financeiro" define. Ver 9.7.

**Decisão 1: baixa parcial existe, com tabela própria.**

Hoje `contas_receber.status` só aceita Em aberto ou Recebido, o que não representa a realidade do crediário: o cliente deve R$ 200 e paga R$ 50. Marcar como recebido é mentira, e deixar aberto perde os R$ 50.

Nasce uma tabela de **baixas**, ligada ao recebível ou à conta a pagar, com valor, data e forma. É o mesmo desenho que fez o pagamento dividido funcionar na seção 3.5: quem tem várias linhas é o filho, não o pai. O status passa a ser calculado pela soma das baixas contra o valor, em vez de ser digitado.

**Decisão 2: nomes das duas telas de caixa.**

O menu tinha "Fluxo de Caixa" no grupo Financeiro e existia "Controle de Caixa" como etapa 9.5. São coisas diferentes e o nome parecido levaria a procurar sangria no lugar errado.

| Tela | O que é | Etapa |
|---|---|---|
| **Fluxo de Caixa** | Projeção de entradas e saídas: o que entra e o que sai, previsto e realizado | 9.4 |
| **Caixa Diário** | A gaveta do balcão: abertura, sangria, reforço e fechamento por operador | 9.5 |

**Decisão 3: conta a pagar nascida da entrada de estoque é opcional.**

A Entrada já registra fornecedor e custo, então dá para gerar a conta a pagar junto. Mas nem toda compra é a prazo, e muita é paga na entrega. Portanto o vencimento é perguntado no momento do lançamento e a geração **não é automática**.

**Decisão 4: cadastros próprios, editáveis pelo administrador.**

Seis tabelas criadas em 31/07/2026, detalhadas na seção 3.12: contas, maquininhas, bandeiras, taxas de cartão, formas de pagamento e categorias de contas a pagar. Motivo dado pelo Rogério: esses dados mudam com o tempo, e ele precisa poder cadastrar, editar e excluir sem depender de sessão com IA.

**Taxas da Rede levantadas e gravadas** em 31/07/2026, com Pix confirmado sem taxa. Quadro completo na seção 3.12.

#### Decidido: o destino contábil da taxa da maquininha

Levantado em 31/07/2026 e **fechado em 01/08/2026**.

O documento trata Crédito, Débito e Gás do Povo como "dinheiro que já é da empresa mas ainda não chegou". **Só que não chega inteiro**: a operadora desconta a taxa antes de creditar. Uma venda de R$ 100 no crédito Elo vira R$ 96,49 na conta.

Três efeitos que hoje não estão previstos:

| Efeito | Consequência |
|---|---|
| Recebível gravado pelo valor cheio | Superestima o caixa futuro |
| Taxa não entra no custo da venda | **A margem da `vw_pedidos` está otimista em toda venda no cartão** |
| Valor esperado diferente do creditado | A conciliação nunca fecha |

**Uma restrição técnica já eliminou uma das saídas.** Gravar o `pagamentos.valor` pelo líquido é impossível: a trava adiada da seção 3.10 confere se a soma dos pagamentos bate com o total do pedido, e o líquido quebraria todo pedido no cartão. Portanto **`valor` continua sendo o valor cheio, o que o cliente pagou**, e entram colunas novas em `pagamentos` para maquininha, grupo, parcelas, taxa aplicada, valor da taxa e valor líquido. Os dois números são verdade e os dois ficam guardados.

**O que falta decidir é o destino contábil da taxa**, e ele muda o número que aparece na tela:

| Opção | Efeito |
|---|---|
| **Custo da venda** | A `vw_pedidos` subtrai a taxa do lucro. A margem passa a dizer quanto rendeu de verdade um botijão vendido no crédito Elo |
| **Despesa operacional** | A margem segue bruta e a taxa aparece como conta a pagar, no resultado do mês |

**Decisão do Rogério em 01/08/2026: guardar as duas margens.**

As duas opções acima fechavam uma porta cada. A terceira saída não estava na mesa: como a taxa passa a ser gravada em coluna própria de qualquer jeito, a view calcula as duas margens ao mesmo tempo. Qual delas aparece no card vira **decisão de tela, reversível**, em vez de decisão de banco, e a função de maior risco do sistema não precisa ser operada duas vezes.

A `vw_pedidos` passou a trazer `lucro` e `margem_pct` brutos, com o mesmo sentido de antes, mais `taxa_total`, `lucro_liquido`, `margem_liquida_pct` e `recebido_liquido`. Quadro numérico na seção 6.

**Fechou antes da primeira venda no cartão**, então a margem nasceu certa em vez de nascer torta e ser corrigida depois.

#### Decidido: cartão não vira recebível (01/08/2026)

A pendência 9 tratava Crédito, Débito e Gás do Povo como um bloco só, todos virando linha em `contas_receber`.

Depois da captura da taxa, cartão **não precisa** disso: a taxa está gravada, o valor líquido calculado e o prazo de crédito cadastrado. O Fluxo de Caixa projeta direto da tabela de pagamentos, mostrando o que a operadora vai creditar e quando. Ninguém dá baixa manual em algo que a operadora paga sozinha.

**Gás do Povo continua virando recebível de verdade**, porque o repasse do programa pode atrasar ou vir diferente do previsto.

#### Ordem alterada: o financeiro foi antecipado (01/08/2026)

A seção 10 determinava que os endpoints do financeiro só seriam construídos depois da etapa 9.7, para não escrever a regra de permissão duas vezes.

**O que mudou o argumento**: a tela de Cadastros já foi antecipada em 31/07 com o raciocínio de que a permissão ali é simples, administrador ou nada, sem depender do desenho de setores. O mesmo vale para Contas a Receber e Contas a Pagar. Quando a 9.7 chegar, entra um check a mais, não uma reescrita.

**A 9.7 continua existindo** e continua sendo pré-requisito da pendência 11. Ela só deixa de bloquear o financeiro.

#### O que já está pronto para o Dashboard da Fase 2

| Indicador que faltava | Situação em 01/08/2026 |
|---|---|
| **Valores por forma de pagamento** | **Pronto no servidor.** `por_forma` vem nos totais, e a lista de pagamentos vem linha a linha. Falta a tela |
| **Gráfico de evolução comparado ao mês anterior** | **Nenhum trabalho de servidor.** O painel já busca o período atual e o anterior. Falta a tela |
| Lista "Atenção hoje", parte de estoque | Já vem, pelo saldo e pelo mínimo |
| Lista "Atenção hoje", parte financeira | Depende da pendência 18 e do endpoint financeiro |

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

**Alterado em 01/08/2026**: esta etapa **deixou de bloquear o módulo financeiro**, que foi antecipado com permissão de administrador ou nada, pelo mesmo raciocínio que liberou a tela de Cadastros em 31/07. Ela continua sendo pré-requisito da pendência 11, que é o valor do pedido exposto ao colaborador. Motivo na seção 9.4.

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
| 2º | Base de dados no Supabase | **Concluído.** Esquema, produtos, clientes, endpoints e segurança fechados. Virada feita em 31/07/2026 |
| 3º | **Fase 1: painel operacional novo** | **No ar como sistema único desde 31/07/2026**, em painel.olvdistribuidora.com.br. Pendência 17 e Estoque Inicial resolvidos em 01/08/2026. Falta publicar o painel v5 |
| 4º | **Contas a pagar e receber** | **Antecipado em 01/08/2026**, com permissão de administrador ou nada. Começa por Contas a Receber, que depende da pendência 18 (tabela de baixas). Ver 9.4 |
| 5º | Setores e permissões | **Deixou de bloquear o financeiro em 01/08/2026.** Continua pré-requisito da pendência 11 |
| 6º | Controle de caixa | Fase 2 |
| 7º | Módulo fiscal | Visão de futuro |

**Fechadas em 01/08/2026:**

- ~~Pendência 17, gestão de usuários~~: resolvida em três frentes, seção 8.3.
- ~~Destino contábil da taxa da maquininha~~: **guardar as duas margens**, seção 9.4.
- ~~Edição de valores de pedido~~: existe, e **preserva o custo original** dos itens que já estavam no pedido, carimbando o custo de hoje só nos produtos novos. Seção 3.10.
- ~~Recebíveis de terceiros~~: **cartão não vira recebível**, o Fluxo de Caixa projeta direto de `pagamentos`. Gás do Povo continua virando. Seção 9.4.
- ~~Ordem entre setores e financeiro~~: financeiro antecipado, seção 9.4.

**Decisões a fechar:**

- **Anexo de boleto (pendência 22)**: por onde o arquivo sobe para o Storage, já que o navegador nunca fala direto com o Supabase. Trava a tela de Contas a Pagar.
- Setores: lista final, níveis internos e tratamento do histórico.
- Pendência 11: quem atribui o entregador e como tratar Retirada e Aguardando.
- Financeiro: categorias de contas a pagar.
- Caixa: único ou por operador; o que entra como sangria e reforço; como tratar diferenças no fechamento.
- Fiscal: ST do gás e vasilhame com o contador; CSC na SEFAZ-ES; inscrição estadual no provedor.

**Ações do Rogério, fora de sessão:**

- Publicar o painel v5, redefinir a senha da Gabriele e conferir o CORS da tela de Usuários no navegador.
- Preencher conta de destino nas 8 formas e na maquininha (pendência 20).
- Preencher saldo inicial e data inicial das 4 contas (pendência 21).

---

## 11. REFERÊNCIAS RÁPIDAS

| Item | Valor |
|---|---|
| URL do painel | **https://painel.olvdistribuidora.com.br**, único desde a virada de 31/07/2026 |
| URL do painel antigo | https://n8n-wmtt.srv1830312.hstgr.cloud/webhook/olv-painel. **Fora do ar desde 31/07/2026**, workflow desativado e guardado |
| Autenticação | Login com papéis (administrador e colaborador), token de 12h |
| Instância n8n | n8n-wmtt.srv1830312.hstgr.cloud |
| Workflows de dados, ativos | OLV2 Dados, OLV2 Pedido, OLV2 Estoque, OLV2 Clientes, ativos desde 30/07/2026; OLV2 Cadastros, ativo desde 31/07/2026 |
| Workflows de acesso, ativos | OLV Login `W4qgfRIna8BIRUVz`, OLV Contas `UBMRIgBy2jcBESBw`. Camada compartilhada, não saem na virada. Seção 4.1 |
| Workflows desativados na virada | OLV Painel Mobile `WAiagumIwB8viELn`; OLV Vendas `9Fx0Y4zvPq7PcHKK`; OLV Estoque `fgmMA5a6hiEZr9z2`. **Desativados, não excluídos.** Ponto de retorno |
| Tabelas internas do n8n | OLV Usuarios `T0MSOwzSF6VH4ngX` e OLV Painel HTML |
| Pontos de restauração (v1.4) | Painel 63bf15bb; Vendas 348ed17a; Estoque 2adfcba0 |
| Identificadores dos workflows novos | Dados fi2DPaA6qL7MxwIV; Pedido aPr6vx4oesVfkLis; Estoque 3l15lOfGCeYqLEu7; Clientes ClsDIM8jVRB5fijC (v3.1); Cadastros fV9fzAM81cT2Pvma (31/07/2026) |
| Banco de dados | Supabase PostgreSQL 17, projeto olv-distribuidora_sistema, região São Paulo. 8 tabelas de movimento, 6 de cadastro, 2 views, RLS ativa. **Papéis públicos sem acesso ao schema `public` desde 31/07/2026** |
| Rotina de fechamento de acesso | `fechar_acesso_publico()` e `auditar_acesso_publico()`, criadas em 01/08/2026. Rodar depois de **toda** migração. Seção 8.6 |
| Funções de gravação de pedido | `registrar_venda` (criar) e `atualizar_venda` (editar por completo). As duas numa transação só, as duas capturando a taxa do cartão |
| Cadastros do financeiro | `contas`, `maquininhas`, `bandeiras`, `taxas_cartao`, `formas_pagamento`, `categorias_conta_pagar`. Criados em 31/07/2026, com workflow (OLV2 Cadastros) e tela publicados no mesmo dia. Seções 3.12, 4.3 e 5.4 |
| Maquininha e taxas | Rede. Padrão (Mastercard e Visa) e Outras (Elo). Débito 0,69 e 1,49; crédito à vista 2,71 e 3,51; 2x 3,78 e 4,58; 3x 4,35 e 5,15. Tudo em 1 dia útil. Pix sem taxa |
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
| POST /webhook/olv2-pedido | `{acao, token, venda, itens, pagamentos, idempotency_key}` ou `{acao, token, venda_id, ...}`. Ações: `criar`, `editar`, `status`, `editar_leve`, `excluir` |
| POST /webhook/olv-contas | `{action, token, usuario, senha, papel, ativo}`. Ações: `listar`, `criar`, `papel`, `ativo`, `trocar_senha`, `resetar_senha`. **Usa `action`, não `acao`**, por ser anterior aos endpoints OLV2 |
| POST /webhook/olv2-estoque | `{acao, token, tipo, produto_id, quantidade, custo_unitario, fornecedor}` |
| POST /webhook/olv2-clientes | `{acao, token, nome, endereco, bairro, telefone}` ou `{acao, token, termo}` |
| POST /webhook/olv2-cadastros | `{acao, token, tabela, ...campos da tabela}` |

Toda resposta traz `ok` verdadeiro ou falso. Quando falso, traz `erro` com mensagem pronta para exibir ao usuário.

### Cuidado com cache ao carregar o documento

O `raw.githubusercontent.com` entrega por rede de distribuição que guarda cópias. Os caminhos `refs/heads/main/...` e `main/...` são endereços diferentes e podem ter cópias de idades diferentes. Em 29/07/2026 o primeiro entregou a v1.6 enquanto o segundo entregava a v1.7, três dias de diferença.

**Reincidência em 31/07/2026, pior que a primeira.** O caminho curto entregou a v3.0 e o `refs/heads/main` entregou a **v1.6**, de seis dias antes. A sessão parou pela regra 7, e a leitura só foi possível depois.

**O caminho que resolve: carregar pelo código do commit.**

```
https://raw.githubusercontent.com/rogeriosandre/olv-distribuidora/<commit>/Painel_OLV_Documentacao_e_Evolucao.md
```

O código do commit sai da tela do repositório no GitHub, ao lado da última alteração. Endereço de commit é único e não tem cópia guardada, então entrega sempre a versão certa. Foi assim que a v3.4 foi lida.

**Ordem recomendada**: tentar o caminho curto; se a versão vier menor que a esperada, ir direto para o caminho do commit. Sempre conferir a versão lida, conforme a regra 7.

**Ajuste pendente nas instruções do projeto**: elas mandam carregar por `refs/heads/main`, que é justamente o caminho mais defasado. Alinhar com esta seção, senão a regra 7 continuará disparando a cada sessão.

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
| 9.4 Financeiro | Tabela de baixas e cálculo de status | Opus 5 |
| 9.4 Financeiro | Endpoint e telas do financeiro | Sonnet 5 |
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
| 30/07/2026 | 3.0 | **Modelo de pedido com itens, 8 formas de pagamento e função `registrar_venda`.** (a) `venda_itens` criada: `produto_id`, `quantidade` e `custo_unitario` saíram de `vendas`, que ganhou `desconto` e `idempotency_key`. (b) Definidas as 8 formas de pagamento em lista fechada, com a regra de recebível por linha de pagamento e o tratamento da Gratuidade, que zera o pedido. (c) Criada a função `registrar_venda`, que grava pedido, itens, pagamentos e recebíveis numa transação só, encerrando as pendências 1 e 4. (d) Criadas as travas de consistência do pedido, adiadas para o fim da transação. (e) `contas_receber` ganhou `pagamento_id` e cascata, encerrando a pendência 8. (f) Custo do produto passou a nascer da entrada de estoque, por média ponderada, com o custo da última compra visível ao lado; `estoque_movimentacoes` ganhou custo e fornecedor. (g) `vw_estoque_atual` reconstruída: abatia estoque na data do pedido em vez da conclusão, contrariando a regra da seção 6. (h) Criada `vw_pedidos`, que calcula lucro e margem já com o desconto, corrigindo distorção descoberta nos testes. (i) 6 produtos cadastrados e 910 clientes preparados a partir de 1.598 contatos, com 24 bairros padronizados. (j) Simulação visual do painel novo aprovada, com Dashboard exclusivo do administrador e o indicador de ticket médio removido, encerrando a pendência 12. (k) Decidido reescrever o HTML do painel como arquivo versionado, em construção paralela ao painel atual. |
| 30/07/2026 | 3.1 | **Endpoints do painel novo construídos e desativados (tarefa 4).** (a) Criados os quatro workflows OLV2, com bloco de autenticação idêntico, permissão positiva sobre `admin` e nó Crypto com `type` e `encoding` fixados desde o rascunho. Seção 4.2. (b) Registradas as seis decisões de desenho aprovadas, incluindo itens do pedido por segunda consulta e a divisão da edição de pedido em `criar`, `status`, `editar_leve` e `excluir`. Seção 9.2. (c) Nova convenção 8.4: toda gravação viaja como um único parâmetro em base64, porque o nó Postgres separa parâmetros por vírgula e texto livre tem vírgula. Fecha também a porta para injeção de SQL. (d) Nova regra de permissão na seção 7: Entrada e Estoque Inicial exclusivos do administrador, porque alimentam o custo médio; Ajuste liberado a qualquer usuário ativo. (e) Pendência 13 registrada: sem exclusão de movimentação de estoque na Fase 1. (f) Registrado que a Gratuidade combinada com Crediário grava recebível com o valor cheio. (g) Pendência 11 reduzida e detalhada: totais agregados deixam de ser calculados para quem não é admin, e ficou decidida a direção de entregar valor apenas dos pedidos em rota atribuídos ao colaborador, para executar na etapa 9.7. (h) Endpoint de leitura nasceu com duas ações em vez de cinco, por peso no celular. (i) Registrado que `responsavel` guarda quem lançou, não quem entrega, e que não existe campo de entregador. |
| 30/07/2026 | 3.2 | **Painel novo construído, publicado e ligado aos endpoints; workflows OLV2 ativados.** (a) `index.html` do painel novo escrito e publicado em https://painel.olvdistribuidora.com.br via Cloudflare Workers (Cloudflare Drop, domínio próprio anexado). (b) Corrigido bloqueio de CORS nos cinco webhooks usados pelo painel (OLV Login e os quatro OLV2): nenhum tinha `allowedOrigins` configurado, o que impedia qualquer chamada do navegador a partir do domínio novo. Corrigido nó a nó e publicado; os quatro workflows OLV2 ativados. (c) Filtro de período (Dashboard e Concluídos) movido para o topo da página, ao lado do título. (d) Corrigida busca de cliente por telefone: usava o campo `telefone_norm`, que o endpoint `olv2-dados` não devolve; passou a extrair os dígitos do campo `telefone`. Nova pendência 15 registrada na seção 3.11. (e) Ícones próprios no menu lateral, no lugar de um marcador genérico. (f) Ordem de navegação por Tab corrigida no formulário de novo pedido. (g) Tela inicial pós-login passa a ser o Dashboard para administrador, e Vendas para os demais papéis. (h) Nova pendência registrada: cores da marca no painel novo ainda são valores de trabalho, não confirmados como oficiais. (i) DNS de painel.olvdistribuidora.com.br confirmado configurado e no ar. |
| 31/07/2026 | 3.3 e 3.4 | **Validação técnica da virada, correções de bug e ajustes visuais no painel novo.** (a) Conferido o status real dos workflows n8n (5 antigos + 4 OLV2, todos ativos, ponto de retorno intacto) e o estado do Supabase (dados batendo com o documentado). (b) Nova pendência 16 registrada na seção 8.3: `vw_pedidos`, `vw_estoque_atual` e cinco funções (incluindo `registrar_venda`) são SECURITY DEFINER com privilégio liberado para `anon`/`authenticated`, permitindo acesso direto à API do Supabase sem passar pelo n8n; correção fica para sessão em Opus 5, por alterar trava de acesso. A aprovação final da virada depende dessa decisão. (c) Pendência 15 resolvida: `telefone_norm` incluído no SELECT do nó `Consultar Clientes T` (OLV2 Dados), workflow publicado. (d) Pendência 14 resolvida: função `registrar_venda` corrigida para zerar `contas_receber.valor` quando o pedido é Gratuidade, mesmo combinado com Crediário; testado com pedido real e dado de teste removido. (e) Testada a troca das cores do painel pelas cores extraídas da logo (azul-marinho #013090, azul médio #0041BC, verde #82D602); Rogério avaliou e decidiu manter as cores de trabalho originais (#0b2545/#1d6fd6/#1c9c5b). Decisão fechada. (f) Corrigidos dois bugs do menu lateral: sobreposição do conteúdo em telas ≤720px ao expandir o menu, e a logo "OLV Distribuidora" não sumindo por completo ao recolher. (g) Filtro de período do Dashboard e de Vendas/Concluídos redesenhado em pills de atalho (Hoje, Ontem, 7 dias, Este mês, Mês ant., Tudo). (h) Favicon, apple-touch-icon e ícone de manifest (PWA) adicionados a partir da logo, com cantos recortados com transparência real; tudo embutido como data URI no próprio HTML. |
| 31/07/2026 | 3.5 | **Pendência 16 resolvida e virada feita.** (a) Avaliado o impacto de revogar o acesso dos papéis públicos do Supabase, com cinco conferências antes de tocar no banco: o n8n conecta pelo papel `postgres`, que ignora RLS e privilégio; a instância inteira tem uma única credencial de Supabase, do tipo Postgres, e nenhuma de API REST; o log da API do Supabase nas últimas 24h estava vazio; zero usuários no Supabase Auth; zero buckets de Storage. Conclusão: revogar era seguro. (b) **Pendência 16 corrigida em duas migrações.** A primeira revogou só os privilégios nominais de `anon` e `authenticated` e **não fechou a brecha**: a conferência mostrou `registrar_venda` ainda chamável sem login, porque função nasce com execução liberada para o pseudo-papel `PUBLIC`, do qual `anon` herda. A segunda migração revogou o privilégio herdado de `PUBLIC`, fechou `USAGE` no schema `public` e os privilégios padrão para objeto novo, e passou as duas views para `security_invoker`. Incluída também a função `set_data_conclusao`, exposta e ausente do diagnóstico original. (c) Auditor de segurança do Supabase rodado depois: restaram 8 avisos, todos informativos e todos do tipo `rls_enabled_no_policy`, que são o desenho pretendido e não falha. (d) **Descoberta a separação entre fluxos de dados e fluxos de acesso** (seção 4.1): OLV Login e OLV Contas não são "sistema antigo", são a camada de acesso compartilhada com o painel novo, e desligá-los derrubaria o login e a gestão de usuários. A lista de cinco a aposentar estava errada; os desligáveis eram três. (e) **Virada feita.** OLV Painel Mobile, OLV Vendas e OLV Estoque desativados, não excluídos, com ponto de retorno intacto. A recomendação técnica era adiar até um domingo real de operação, com o banco ainda em 0 pedidos; Rogério decidiu seguir porque o painel antigo já não estava em uso. (f) **Pendência 17 criada**: o webhook do OLV Contas está sem `allowedOrigins`, então o painel novo não alcança a gestão de usuários nem a troca de senha do primeiro acesso. (g) Registrada na seção 11 o carregamento do documento pelo código do commit, que resolve o cache do GitHub em definitivo, depois da reincidência de 31/07, quando o caminho curto entregou a v3.0 e o `refs/heads/main` entregou a v1.6. |
| 31/07/2026 | 3.6 | **Pendência 13 resolvida e desenho do financeiro iniciado.** (a) Ao abrir o código apareceram **quatro falhas no custo médio, não uma**: além da exclusão que não desfazia, o lançamento retroativo já calculava contra o saldo de hoje, mudanças em venda concluída não recalculavam, e **erro de digitação no custo era incorrigível**, porque Ajuste não toca em custo, não havia exclusão e o cadastro não aceita custo digitado. Causa raiz: valor incremental sobre histórico editável. (b) Criada a função `recalcular_custo_produto`, que refaz a conta do zero lendo o histórico em ordem cronológica; removidos o gatilho `trg_custo_produto` e a função `atualiza_custo_produto`; criados três gatilhos que disparam em movimentação de estoque (INSERT, UPDATE e DELETE), item de pedido e mudança de status do pedido. Mecânica na seção 6. (c) Registrado que o lucro histórico não muda, porque `venda_itens.custo_unitario` é copiado no momento da venda. (d) Avaliado e descartado no mesmo dia: Ajuste com custo, por abrir um segundo caminho para alterar a base do lucro. (e) Criadas as seis tabelas de cadastro do financeiro (seção 3.12): `contas`, `maquininhas`, `bandeiras`, `taxas_cartao`, `formas_pagamento` e `categorias_conta_pagar`, com as taxas reais da Rede levantadas e gravadas. Regra de exclusão comum: sem uso, exclui; com uso, desativa, pelo mesmo motivo de cliente e produto. Taxa de cartão nunca se sobrescreve, se sucede, pelo mesmo motivo da pendência 13. (f) Criada `produtos.preco_venda`, preço sugerido que vai preencher o formulário de pedido, sem travar a negociação item a item. (g) Criada `vendas.acrescimo`, ao lado do desconto, **em coluna separada e não como desconto negativo**, para que o relatório de descontos concedidos não se anule com os acréscimos cobrados. Gatilhos de total e `registrar_venda` atualizados; testado com pedido real de R$ 60 em itens, R$ 5 de desconto e R$ 8 de acréscimo resultando em R$ 63, com a trava de consistência aceitando; dado de teste removido. (h) Registrada na seção 6 a limitação testada da contagem no mesmo dia: Estoque Inicial em produto que já teve Entrada no mesmo dia duplica o saldo, e o caminho correto ali é o Ajuste. (i) Detalhada na 9.4 a decisão em aberto do destino contábil da taxa, com a restrição técnica de que `pagamentos.valor` tem de continuar sendo o valor cheio, sob pena de quebrar a trava adiada. Essa decisão **bloqueia a religação da `registrar_venda`**. |
| 31/07/2026 | 3.7 | Cabeçalho realinhado ao log (estava parado em 3.4) e a regra 7 detalhada com o caminho de resolução por commit, depois da reincidência de cache do mesmo dia. Sem mudança de conteúdo além disso. Seção 11. |
| 31/07/2026 | 3.8 | **Sessão Sonnet 5: quatro tarefas do painel, três bugs de produção pré-existentes corrigidos, ajustes de uso.** (a) **Pendência 13 encerrada por completo**: o endpoint OLV2 Estoque ganhou a ação `excluir`, restrita ao administrador, com botão e confirmação em duas etapas no painel; a saída automática de venda concluída nunca mostra o botão, por ser calculada, não gravada. Seções 3.11, 7 e 9.2. (b) A coluna `vendas.acrescimo`, criada na v3.6 mas ainda sem uso no formulário, foi ligada à tela: seletor único de Desconto ou Acréscimo, nunca os dois ao mesmo tempo, validado também no servidor. Seção 9.2. (c) O campo de valor unitário do pedido passou a ser preenchido com `produtos.preco_venda` ao escolher o produto, continuando editável e negociável item a item. Seção 9.2. (d) **Construído e publicado o workflow OLV2 Cadastros** (`fV9fzAM81cT2Pvma`), antecipando parte da etapa 9.7 a pedido de Rogério: cobre as sete tabelas de cadastro do financeiro, com tela exclusiva do administrador e o mesmo bloco de entrada dos outros quatro endpoints. Seções 3.12, 4.2 e 5.4. (e) **Três bugs de produção encontrados e corrigidos**, sem relação com as quatro tarefas acima: a publicação dos quatro workflows OLV2 estava atrasada em relação ao painel já no ar, causando divergência entre o acréscimo enviado pelo painel e o que o backend antigo esperava, corrigida publicando os quatro; o `queryReplacement` do nó Postgres do n8n corta valor por vírgula e descarta parâmetro que resolve para string vazia, quebrando `Consultar Itens T` e `Consultar Itens P` do OLV2 Dados (lista de pedidos do período) e `Consultar Histórico` do OLV2 Estoque (filtro Tipo em branco); documentado como armadilha nova na seção 8.5, corrigido nos três nós e publicado. (f) **Ajustes de uso**: todas as telas com filtro de período passaram a carregar sozinhas ao abrir, sem precisar clicar em Buscar; Estoque ganhou painel de "Movimentações de hoje" na aba Produtos e passou a mostrar 7 dias por padrão na aba Histórico, com indicador colorido (verde para entrada, vermelho para saída) e datas sem horário; campo de data retroativa no formulário de pedido, corrigindo bug em que a data enviada ao servidor era sempre a de hoje; valor do pagamento preenchido automaticamente quando há uma única linha; cards de pedido mostrando só a data; coluna Endereço adicionada à tabela de Clientes. Seções 9.1 e 9.2. (g) Todos os dados de teste gerados na sessão, incluindo os do próprio Rogério durante a validação em produção, removidos do banco ao final, conferido por SQL. |
| 01/08/2026 | 3.9 | **Sessão Opus 5: religação da `registrar_venda`, edição completa de pedido, pendência 17 resolvida e primeiro dia de operação real.** (a) **Decisão da seção 9.4 fechada: guardar as duas margens.** As duas opções do documento fechavam uma porta cada; a terceira saída faz a `vw_pedidos` calcular margem bruta e líquida ao mesmo tempo, deixando a escolha de qual mostrar como decisão de tela, reversível. `pagamentos` ganhou oito colunas de cartão, com `valor` continuando cheio pela trava adiada. (b) **`registrar_venda` religada** à tabela `formas_pagamento`, com conferência prévia campo a campo provando que o comportamento não mudou, e com captura da taxa do cartão. Nunca bloqueia a venda por taxa ausente: grava sem taxa e devolve aviso. (c) **Interruptor de bandeira "Outras"** na tela de venda, com seletor de parcelas só no crédito. (d) **Função `atualizar_venda` criada**: edição completa de pedido, inclusive já concluído. A proposta original de criar uma situação "Finalizado" foi descartada, porque criaria uma janela em que o produto já saiu e o sistema ainda o conta no depósito; descobriu-se que os gatilhos de 31/07 já resolviam o recálculo, e faltava só a função. Custo dos itens preserva o original e carimba o de hoje só nos produtos novos. (e) **Pendência 17 resolvida em três frentes indissociáveis**: CORS no OLV Contas, ação `resetar_senha` para o administrador, e troca de senha obrigatória dentro do painel novo. O diagnóstico da v3.8 subestimava o problema: o login **bloqueava** quem tinha troca pendente e mandava usar o painel antigo, desativado em 31/07, deixando a pessoa trancada fora do sistema. Aconteceu com a usuária Gabriele. (f) **Regra da seção 3.10 corrigida**: "objeto novo nasce fechado" está errado para funções. Tabela nasce fechada, **função nasce aberta**, testado três vezes, e `ALTER DEFAULT PRIVILEGES` não resolve. Criadas `fechar_acesso_publico()` e `auditar_acesso_publico()`, e a regra 9 das instruções. Nova seção 8.6. (g) **A regra de exclusão de cadastro não funcionava**: ela depende de o banco recusar o DELETE, e `formas_pagamento` e `categorias_conta_pagar` não são referenciadas por chave nenhuma, enquanto `taxas_cartao` apontava para maquininhas com CASCADE. Corrigido com RESTRICT e dois gatilhos. (h) **Erro de digitação corrigido no custo do Gás 13kg**: o Estoque Inicial foi lançado com quantidade 37 e custo 37,00, quando o real é 87,00, inflando o lucro do dia em R$ 200,00. Exigiu correção em duas frentes, porque o custo do item é congelado na venda. (i) **Faturamento passou a contar só pedido concluído**, mesmo critério do estoque, encerrando a divergência de dois critérios para o mesmo pedido. (j) **Cartão deixou de virar recebível** (pendência 9 redesenhada): o Fluxo de Caixa projeta direto de `pagamentos`. Gás do Povo continua virando. (k) **Financeiro antecipado**, deixando de esperar a etapa 9.7, pelo mesmo raciocínio que liberou os Cadastros em 31/07. (l) **Painel v5** com nove correções de uso pedidas no primeiro dia de operação, entre elas a perda de foco e de rolagem ao digitar, que eram um bug só. Cinco pendências novas registradas, 18 a 23, sendo duas de cadastro que dependem só do Rogério. |

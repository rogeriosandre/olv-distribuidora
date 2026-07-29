# OLV DISTRIBUIDORA
## Painel de Vendas, Estoque e Financeiro

**Documentação do funcionamento e plano de evolução**

Versão 2.9 · 29/07/2026
Preparado para Rogério

---

## INSTRUÇÕES PARA CLAUDE (leia antes de editar)

Este documento é a fonte única de verdade do projeto. Quando você o mencionar em nova conversa, carregue-o como contexto. Regras de atualização:

1. **Seção com asterisco (*)**: Rogério atualiza via conversa. Exemplo: "*9.1 Fundação: Base de dados no Supabase" significa que quando há mudança, ele pede "atualize essa seção com X".
2. **Sem asterisco**: Já foi decidido e finalizado. Não altere sem permissão explícita.
3. **Status e Andamento**: Atualize conforme Rogério relata progresso. Use a data do dia quando mudar status.
4. **Falta/Pendência**: Sempre que uma tarefa ficar pronta, mude "Falta:" para "Concluído:" e inclua a data.
5. **Data de última atualização**: Sempre que editar, atualize o "Versão" no topo (formato V.X · DD/MM/AAAA).
6. **Modelo indicado**: ao iniciar qualquer tarefa deste projeto, informe na primeira linha o modelo indicado conforme a seção 12, e avise quando o modelo em uso for mais pesado que o necessário.
7. **Conferência de versão (obrigatória, criada em 29/07/2026)**: sempre que carregar este documento, informe na primeira linha a versão lida. Se ela for menor que a última versão registrada no log de mudanças ou que a versão informada por Rogério, **pare e avise**, porque o conteúdo provavelmente veio de cache. Nunca edite nem numere uma versão nova sem essa confirmação. Esta regra vale mesmo quando ler o documento não fazia parte do plano da tarefa. O motivo está registrado no log da v2.9.

---

## 1. VISÃO GERAL

O projeto é um painel operacional da OLV Distribuidora que roda no navegador (celular ou computador). Ele centraliza três frentes: registro e consulta de vendas, controle de estoque, e (na evolução planejada) o controle financeiro de contas a pagar e a receber.

O objetivo é dar autonomia para lançar e consultar o dia a dia do negócio em um só lugar, com cálculos automáticos de lucro e de saldo de estoque, e sem depender de nenhum aplicativo instalado.

**Estado atual**: existe uma única versão do painel, acessível pela web. A antiga versão desktop (que rodava dentro do Cowork) foi aposentada, e agora tudo é mantido em um só lugar. Desde 25/07/2026, o acesso passou a ser por login individual com papéis (administrador e colaborador), substituindo a chave única anterior.

**Decisão de 25/07/2026 (banco de dados)**: o sistema é construído do zero sobre o Supabase, um banco de dados profissional (PostgreSQL), o mesmo tipo usado em aplicativos de mercado. Não há migração de dados antigos: o sistema começa vazio e vai sendo preenchido pelo uso, com apenas os clientes e os usuários de login inseridos na abertura.

---

## 2. ARQUITETURA

O painel é uma página web servida pelo n8n. Toda leitura e gravação de dados passa pelo n8n, que por sua vez fala com o banco de dados. O fluxo é direto e não depende de nenhum computador ligado:

```
Navegador (celular/PC) → n8n (servidor) → Supabase
```

O celular abre a página do painel e envia as ações (nova venda, edição, lançamento de estoque) para o n8n. O n8n valida o login e o token de sessão, grava no banco e devolve a resposta. A leitura dos dados vem de um endpoint do n8n que consulta o banco e devolve um JSON compacto.

Como é autônomo, o painel funciona 24/7 mesmo com o computador desligado. As dependências são o servidor n8n e o Supabase estarem no ar.

Os anexos do financeiro (boletos e comprovantes) ficarão guardados no Supabase Storage, um cofre de arquivos do próprio Supabase.

**Status em 26/07/2026**: a conexão entre n8n e Supabase está ativa e testada. A religação dos fluxos de vendas e estoque para gravar no Supabase é a pendência aberta da seção 9.1; até ela ser concluída, esses dois fluxos ainda gravam na base anterior, que está em desativação.

### 2.1 Camada de login e usuários

Três peças sustentam o login multiusuário, sem alterar o fluxo principal Navegador → n8n → Supabase:

**Tabela de usuários ("OLV Usuarios")**: uma Data Table interna do n8n (uma tabelinha do próprio n8n, separada da base do negócio) que guarda usuário, nome, senha embaralhada, papel e se está ativo. Manter os acessos apartados dos dados do negócio é mais seguro, como ter a portaria num cômodo separado do cofre.

**Fluxo de Login ("OLV Login")**: recebe usuário e senha, confere na tabela e devolve um token de sessão (um crachá temporário assinado) que carrega o papel e o nome e expira sozinho em 12 horas.

**Fluxo de Contas ("OLV Contas")**: cria usuário, troca senha, lista, muda papel e ativa/desativa acessos. As ações de administração só funcionam se quem chamou for administrador.

**HTML servido de forma segura**: o código da página é grande, então fica guardado codificado (base64, uma forma de escrever o conteúdo como texto para transporte) numa Data Table chamada "OLV Painel HTML", e o fluxo do painel decodifica e monta a página ao servir. Para mudar o visual no futuro, atualiza-se essa tabela.

---

## 3. ESTRUTURA DE DADOS (SUPABASE)

Cada assunto vive em uma tabela própria, ligada às outras por um código (ID). É a diferença entre um caderno único onde tudo é anotado na mesma página e um arquivo com pastas separadas que se referenciam.

Sete tabelas criadas em 25/07/2026, todas com a tranca de segurança (RLS, proteção por linha) ativada. O esquema abaixo foi lido direto do banco em 26/07/2026.

### Como ler as tabelas

- **ID**: número que o banco gera sozinho a cada novo registro, sem repetir nunca. É a identidade permanente daquele registro.
- **Obrigatório**: o banco recusa a gravação se o campo vier vazio.
- **Número**: aceita casas decimais (valores e quantidades).
- **Data e hora**: registrada com fuso horário.
- **Calculada**: o banco calcula sozinho a partir de outros campos. Não é digitada e não sai errada.
- **Validação**: lista fechada de valores aceitos. O banco recusa qualquer coisa fora dela.

### 3.1 clientes

| Campo | Tipo | Regra |
|-------|------|-------|
| id | ID | Chave do registro |
| nome | Texto | Obrigatório |
| endereco | Texto | Opcional |
| bairro | Texto | Opcional |
| telefone | Texto | Opcional. Guardado como digitado |
| telefone_norm | Texto | **Gerado pelo banco**: o telefone só com dígitos. **Único**, impede cliente duplicado |
| observacao | Texto | Opcional |
| criado_em | Data e hora | Preenchido sozinho na criação |

**Trava de duplicidade (26/07/2026)**: o banco cria sozinho uma versão do telefone só com números e não permite dois clientes com o mesmo. Assim, "(27) 99999-8888" e "27999998888" são reconhecidos como o mesmo contato, mesmo digitados de formas diferentes.

Dois limites conhecidos e aceitos:

- **Cliente sem telefone não é protegido.** Vários cadastros sem número são permitidos, porque o banco não tem como saber se são a mesma pessoa. A defesa aqui é o painel avisar quando já existe cliente com nome parecido.
- **Telefone compartilhado bloqueia o segundo cadastro.** Casal ou portaria de condomínio com o mesmo número. Na prática costuma ser o mesmo cliente de entrega; se precisar separar, cadastre o segundo sem telefone.

### 3.2 produtos

| Campo | Tipo | Regra |
|-------|------|-------|
| id | ID | Chave do registro |
| nome | Texto | Obrigatório e **único**: o banco impede cadastrar o mesmo produto duas vezes |
| unidade | Texto | Opcional (ex.: unidade de medida do produto) |
| custo_atual | Número | Opcional |
| estoque_minimo | Número | Começa em 0 |
| ativo | Sim/Não | Começa como ativo. Permite aposentar um produto sem apagar o histórico |
| criado_em | Data e hora | Preenchido sozinho na criação |

### 3.3 vendas

| Campo | Tipo | Regra |
|-------|------|-------|
| id | ID | Chave do registro |
| cliente_id | Ligação | Aponta para clientes. Opcional |
| produto_id | Ligação | Aponta para produtos. Opcional |
| quantidade | Número | Obrigatório |
| valor_total | Número | Obrigatório |
| custo_unitario | Número | Opcional |
| data | Data | Começa com a data de hoje |
| status | Texto | Obrigatório, **validado**. Começa como Aguardando. Ver o fluxo abaixo |
| tipo_entrega | Texto | Obrigatório, **validado**: Entrega ou Retirada. Começa como Entrega |
| pre_pago | Sim/Não | Obrigatório. Começa como Não. Marca o pedido pago antes do resgate |
| data_conclusao | Data | **Preenchida pelo banco** quando o pedido vira Concluído. Volta a ficar vazia se o status recuar |
| observacao | Texto | Opcional |
| responsavel | Texto | Quem lançou, preenchido pelo sistema |
| criado_em | Data e hora | Preenchido sozinho na criação |
| custo_total | **Calculada** | quantidade × custo_unitario (custo vazio conta como zero) |
| lucro | **Calculada** | valor_total − (quantidade × custo_unitario) |

**Dois campos, duas perguntas** (definido em 26/07/2026):

O pedido tem dois eixos independentes. Misturar os dois num campo só impede representar situações reais, como uma retirada que ainda vai ser buscada.

| Campo | Valores | Responde |
|-------|---------|----------|
| tipo_entrega | Entrega, Retirada | Como o pedido sai |
| status | Aguardando, Em rota, Concluído | Em que etapa ele está |

**Fluxo por tipo:**

| Tipo | Caminho |
|------|---------|
| Entrega | Aguardando → Em rota → Concluído |
| Retirada | Aguardando → Concluído |

O banco recusa a combinação Retirada com Em rota, porque pedido de balcão não sai para rota. O status trata apenas do andamento; se o cliente pagou ou não é assunto da tabela contas_receber.

**Pedido pré-pago** (definido em 26/07/2026): o cliente paga hoje e resgata depois, podendo levar dias ou meses. O campo pre_pago é um terceiro eixo, independente dos outros dois, porque um pré-pago continua sendo entrega ou retirada.

| Momento | O que se movimenta | O que não se movimenta |
|---------|--------------------|------------------------|
| Pedido criado e pago | Financeiro | Estoque |
| Pedido concluído | Estoque, na data da conclusão | Financeiro |

É por isso que existe a data_conclusao: sem ela, o estoque cairia no dia do pagamento, com o produto ainda no depósito. O banco preenche essa data sozinho quando o pedido é concluído, sem depender de ninguém lembrar.

**Ponto importante**: custo_total e lucro são calculados pelo próprio banco. Se o valor ou a quantidade mudarem, os dois se atualizam sozinhos. Isso resolve de forma definitiva o problema antigo de números que não recalculavam depois de editados.

### 3.4 pagamentos

Uma venda pode ter mais de uma linha aqui, o que permite pagamento dividido (parte em dinheiro, parte no crédito).

| Campo | Tipo | Regra |
|-------|------|-------|
| id | ID | Chave do registro |
| venda_id | Ligação | Aponta para vendas. Obrigatório |
| forma | Texto | Obrigatório (Pix, Dinheiro, Crédito, Crediario, Em aberto) |
| valor | Número | Obrigatório |
| criado_em | Data e hora | Preenchido sozinho na criação |

**Atenção (registrado em 29/07/2026)**: diferente de `vendas.status` e de `estoque_movimentacoes.tipo`, o campo `forma` **não tem validação** no banco. Ele é obrigatório, mas aceita qualquer texto. A lista entre parênteses acima é a intenção, não uma trava. Ver pendência 7 da seção 3.9.

### 3.5 estoque_movimentacoes

| Campo | Tipo | Regra |
|-------|------|-------|
| id | ID | Chave do registro |
| produto_id | Ligação | Aponta para produtos. Obrigatório |
| tipo | Texto | **Validado**: só aceita Entrada, Ajuste ou Estoque Inicial |
| quantidade | Número | Obrigatório |
| data | Data | Começa com a data de hoje |
| responsavel | Texto | Quem lançou |
| observacao | Texto | Opcional |
| criado_em | Data e hora | Preenchido sozinho na criação |

### 3.6 contas_pagar

| Campo | Tipo | Regra |
|-------|------|-------|
| id | ID | Chave do registro |
| fornecedor | Texto | Opcional |
| descricao | Texto | Opcional |
| valor | Número | Obrigatório |
| vencimento | Data | Opcional |
| status | Texto | **Validado**: Em aberto ou Pago. Começa como Em aberto |
| categoria | Texto | Opcional |
| pago_em | Data | Preenchido na baixa |
| anexo_url | Texto | Endereço do boleto ou comprovante no Supabase Storage |
| criado_em | Data e hora | Preenchido sozinho na criação |

### 3.7 contas_receber

| Campo | Tipo | Regra |
|-------|------|-------|
| id | ID | Chave do registro |
| cliente_id | Ligação | Aponta para clientes. Opcional |
| venda_id | Ligação | Aponta para vendas. Opcional |
| descricao | Texto | Opcional |
| valor | Número | Obrigatório |
| vencimento | Data | Opcional |
| status | Texto | **Validado**: Em aberto ou Recebido. Começa como Em aberto |
| recebido_em | Data | Preenchido na baixa |
| criado_em | Data e hora | Preenchido sozinho na criação |

### 3.8 Como as tabelas se ligam

```
clientes ──┬─→ vendas ──┬─→ pagamentos
           │            └─→ contas_receber
           └─→ contas_receber

produtos ──┬─→ vendas
           └─→ estoque_movimentacoes

contas_pagar (independente)
```

O banco impede apagar um cliente ou produto que tenha lançamento ligado a ele, o que protege o histórico contra exclusão acidental.

**Anexos**: boletos e comprovantes ficam no Supabase Storage, com o endereço guardado no campo anexo_url.

### 3.9 Pendências e decisões em aberto *

Levantadas em 26/07/2026 na leitura do banco, com acréscimos em 29/07/2026.

| # | Ponto | Por que importa |
|---|-------|-----------------|
| 1 | **A forma de pagamento saiu de vendas e foi para pagamentos** | Toda venda passa a gravar em duas tabelas. Muda o desenho da religação do fluxo de vendas. |
| 2 | ~~clientes.nome não é único~~ | **Resolvido em 26/07/2026**: a trava passou a ser pelo telefone, normalizado pelo banco (só dígitos). O nome segue livre, porque dois clientes podem ter o mesmo nome legitimamente. |
| 3 | ~~vendas.status aceita qualquer texto~~ | **Resolvido em 26/07/2026, texto corrigido em 29/07/2026**: campo obrigatório, com lista fechada de **três** valores (Aguardando, Em rota, Concluído) e início em Aguardando. Entrega e Retirada saíram do status e viraram o campo próprio `tipo_entrega` na v2.5. O valor "Recebido" foi descartado, já que o controle de pagamento do crediário passa a viver em Contas a Receber. |
| 4 | **Nada garante que a soma dos pagamentos bata com o valor da venda** | Regra de negócio ainda não existe no banco. |
| 5 | **Não existe tabela de caixa nem de usuários no Supabase** | Caixa é a etapa 9.5. O login segue na Data Table do n8n; falta decidir se migra. |
| 6 | ~~A regra de baixa de estoque precisa mudar de gatilho~~ | **Resolvido em 26/07/2026**, autorizado por Rogério: a baixa passou a ocorrer na conclusão do pedido, pela data da conclusão. Seção 6 atualizada. |
| 7 | **pagamentos.forma não é validada** (novo em 29/07/2026) | Aceita qualquer texto. Sem lista fechada, erro de digitação cria uma forma de pagamento nova em silêncio e quebra os totais por forma. Precisa da mesma trava que `vendas.status` recebeu. |
| 8 | **contas_receber não aponta para a linha de pagamento** (novo em 29/07/2026) | Existe `venda_id`, mas não `pagamento_id`. Numa venda com duas linhas de crediário, não dá para saber qual recebível corresponde a qual pagamento, e a baixa fica ambígua. Depende da decisão do item 1. |

### 3.10 Verificações concluídas (26/07/2026)

**Índices: em ordem.** Um índice funciona como o sumário de um livro: sem ele, o banco lê tudo para achar uma linha. Já existem em vendas (data, cliente_id, produto_id), estoque_movimentacoes (data, produto_id), pagamentos (venda_id), contas_receber (vencimento, venda_id) e contas_pagar (vencimento). A performance está coberta para o crescimento previsto.

Duas ausências de baixo impacto, registradas sem urgência: contas_receber.cliente_id e clientes.nome. O segundo só passa a pesar se a busca por nome no autocompletar ficar lenta com a base cheia.

**RLS: configuração correta, com um esclarecimento importante.** A tranca está ativa nas 7 tabelas e **não existe nenhuma política liberando acesso**. O efeito prático é:

- As chaves públicas do Supabase (usadas por navegador) ficam **totalmente bloqueadas**. Ninguém lê nem grava nada por fora.
- O n8n conecta com um usuário que ignora a tranca por natureza, então tem acesso completo.

Ou seja, a porta da frente está fechada e só o n8n tem a chave de serviço. **A proteção por papel (administrador x colaborador) depende inteiramente da validação dentro do n8n**, não do banco. Isso é aceitável no desenho atual, em que o navegador nunca fala direto com o Supabase. Se um dia o painel passar a acessar o banco diretamente, será obrigatório escrever políticas antes.

---

## 4. OS WORKFLOWS N8N

O painel é sustentado por cinco workflows ativos na instância n8n-wmtt.srv1830312.hstgr.cloud. São todos necessários:

| Workflow | O que faz | Endpoint |
|----------|-----------|----------|
| OLV Painel Mobile (web) | Serve a página HTML do painel e o endpoint de leitura de dados (vendas + estoque + clientes). | /webhook/olv-painel e /webhook/olv-dados |
| OLV Vendas – Lançamento (painel) | Cria, edita e exclui vendas. Calcula Custo Total, Lucro e Mês. | POST /webhook/olv-venda |
| OLV Estoque – Lançamento (painel) | Cria, edita e exclui lançamentos de estoque. | POST /webhook/olv-estoque |
| OLV Login | Recebe usuário e senha, confere na tabela de usuários e devolve o token de sessão (crachá temporário de 12 horas). | POST /webhook/olv-login |
| OLV Contas | Cria usuário, troca senha, lista, muda papel e ativa/desativa acessos. Ações de administração exigem papel de administrador. | POST /webhook/olv-contas |

**Pendente**: os fluxos de vendas e estoque ainda precisam ser religados para gravar no Supabase (ver seção 9.1).

O workflow "OLV Atendimento - Agente WhatsApp" pertence a outro projeto e não faz parte do painel.

**Workflow arquivado (28/07/2026)**: o `TEMP - Leitura OLV Painel HTML`, criado para ler e editar o HTML durante a renomeação do papel, foi arquivado depois de cumprir a função. Ele não roda mais e não aparece na lista ativa, mas continua guardado na seção de arquivados do n8n, porque as ferramentas usadas não têm exclusão definitiva. Excluir de vez, se desejado, é manual.

---

## 5. O PAINEL (FUNCIONALIDADES ATUAIS)

### 5.1 Aba Vendas

- Filtros de período: hoje, ontem, últimos 7 dias, este mês, mês anterior, tudo e período personalizado.
- Indicadores do período: itens vendidos, faturamento, lucro e número de vendas, com médias e margem.
- Quantidade por produto e valores por forma de pagamento.
- Gráfico de evolução por produto (por dia ou por mês, alternando entre quantidade e faturamento).
- Nova venda: formulário com autocompletar de cliente (puxa endereço, bairro e telefone da tabela clientes) e cálculo automático de custo total e lucro.
- Editar e excluir vendas direto na lista, com confirmação em dois cliques na exclusão.
- Filtros de consulta: por produto, forma de pagamento e status, além da busca por texto (cliente/produto).

### 5.2 Aba Estoque

- Tabela de estoque atual por produto, calculado automaticamente.
- Lançamento de movimentações: Entrada, Ajuste (aceita negativo) e Estoque Inicial (contagem que zera o histórico anterior daquele produto).
- Histórico de lançamentos com editar e excluir.

### 5.3 Login, usuários e papéis

- Tela de login com usuário e senha, no lugar da chave única.
- Crachá no topo com o nome e o papel de quem está logado, e os botões Trocar senha e Sair.
- Troca de senha obrigatória no primeiro acesso e disponível a qualquer momento.
- Área de Usuários (só administrador): listar, criar usuário, tornar administrador ou colaborador e ativar/desativar acessos.
- Responsável automático: o campo "quem lançou" saiu do formulário; o sistema preenche sozinho com o usuário logado.
- Trava de dono: o colaborador só vê Editar e Excluir nas próprias linhas, e o servidor confere o dono antes de gravar.

**Não existe hoje**: redefinição de senha de outro usuário pelo administrador. Se alguém esquecer a senha, o caminho é criar um usuário novo ou alterar direto na Data Table. Registrado como melhoria desejada.

---

## 6. REGRAS DE CÁLCULO

### Estoque atual

Estoque atual = último Estoque Inicial + entradas ± ajustes − vendas concluídas, considerando apenas as datas iguais ou posteriores à data da contagem inicial. Vendas sem quantidade preenchida não abatem estoque.

**Gatilho da baixa (alterado em 26/07/2026, autorizado por Rogério)**: a venda abate estoque quando o pedido é **concluído**, pela data da conclusão. Antes, a baixa acontecia na data do lançamento do pedido.

Motivo: pedido lançado não é produto que saiu. Num pré-pago, a diferença entre pagar e resgatar pode ser de meses, e a regra antiga derrubaria o estoque com o produto ainda no depósito. A regra antiga também errava, em menor grau, em qualquer pedido lançado e entregue depois; só não aparecia porque quase tudo era do mesmo dia.

**O que o número passa a significar**: produto disponível no depósito hoje. Pedidos em Aguardando e Pré-pagos não abateram estoque, porque o produto continua lá.

**Ponto de atenção**: pedidos em Em rota também não abatem, embora o produto já esteja no caminhão. É proposital, para que uma entrega não realizada não precise de estorno. Em compensação, durante a rota o estoque do sistema fica um pouco acima do que existe fisicamente na loja.

### Lucro

Lucro = Valor Total − (Quantidade × Custo Unitário). O custo unitário é informado no momento da venda e o lucro é gravado como valor numérico fixo.

---

## 7. ACESSO E SEGURANÇA ATUAL

Acesso pela URL https://n8n-wmtt.srv1830312.hstgr.cloud/webhook/olv-painel.

O acesso é por login individual: cada pessoa tem usuário e senha próprios e um papel. A chave única OLV2026 foi removida.

### Papéis

**Administrador**: vê tudo (faturamento, lucro, margem, valores por forma de pagamento e gráfico em R$), edita e exclui qualquer lançamento e gerencia usuários (cria, muda papel, ativa e desativa).

**Colaborador**: vê o operacional (itens vendidos, número de vendas e estoque atual), sem faturamento, lucro nem margem; só edita e exclui as próprias vendas e lançamentos, com a trava conferida também no servidor.

**Contas iniciais**: Rogério (administrador), Gabriele (administradora) e Colaborador (colaborador). Todas começam com senha temporária, trocada no primeiro acesso. A criação de contas é feita só pelo administrador (não há autocadastro na tela de login).

### Renomeação do papel aplicada no sistema (28/07/2026)

O papel antes chamado "vendedor" foi renomeado para "colaborador" em produção. Os alvos alterados:

| Alvo | Pontos alterados |
|------|------------------|
| OLV Painel HTML (Data Table) | 6 pontos: a comparação do papel, o valor e o texto da opção no formulário de novo usuário, o rótulo do crachá, o rótulo na lista de usuários, o cálculo do próximo papel e o texto do botão de alternar |
| OLV Contas (painel) | 4 pontos, todos valores padrão de normalização. Publicado em produção |
| OLV Usuarios (Data Table) | Papel do usuário existente atualizado. Rogério e Gabriele permanecem admin |

**Sem alteração**: OLV Login (não mencionava o papel) e as travas de dono em OLV Vendas e OLV Estoque (comparação positiva sobre "admin").

**O nome de usuário de login continua sendo "vendedor".** Só o papel mudou. A pessoa entra com o mesmo usuário e a mesma senha. O campo "nome" do cadastro também segue como "Vendedor"; trocar para "Colaborador" é cosmético e está pendente.

### Como as senhas e a sessão são protegidas (siglas explicadas)

**Senha embaralhada (hash)**: a senha nunca é guardada como texto. Guardamos um resumo irreversível dela (hash SHA-256, sigla de Secure Hash Algorithm, o algoritmo que gera esse resumo) somado a um sal (salt: um tempero aleatório, único por usuário, que impede que senhas iguais gerem o mesmo código).

**Token assinado**: o crachá de sessão é validado por uma assinatura (um selo que só o servidor consegue gerar, com um segredo interno). Crachá forjado não passa.

**Sessão com validade**: o crachá expira em 12 horas; depois o painel pede login de novo.

**Nota técnica para manutenção**: este servidor n8n bloqueia criptografia dentro do nó de código, então o embaralhamento da senha e a assinatura do token usam o nó Crypto nativo do n8n (hash SHA-256 com segredo interno), que dispensa credencial e é seguro contra falsificação.

---

## 8. CONVENÇÕES E CUIDADOS

- Cada venda e cada lançamento é identificado por um ID próprio do banco, estável e independente da ordem em que aparecem na tela.
- Os nomes de produto precisam bater exatamente (a comparação é sem diferenciar maiúsculas/minúsculas); por isso o produto é escolhido em menu suspenso.
- O endpoint de leitura foi configurado com cabeçalho no-store e um parâmetro anti-cache, para o painel sempre puxar dados frescos após uma gravação.
- **Custo do colaborador (pendência)**: o campo "Custo unitário" foi ocultado do colaborador. Enquanto a busca automática do último custo do produto não for implementada, as vendas lançadas por colaboradores ficam sem custo, e o lucro dessas vendas fica igual ao valor. Vale priorizar essa melhoria ou reavaliar mostrar o campo.

### 8.1 Como o painel esconde os dados financeiros (criado em 29/07/2026)

Esta seção existe porque o mecanismo é indireto e já causou um incidente. Quem for mexer em papéis precisa ler antes.

**O que acontece no navegador**, em duas etapas:

1. Se o papel do usuário logado for exatamente a string `'colaborador'`, o HTML aplica a classe CSS `vend` no `<body>`.
2. A regra `body.vend .hideVend{display:none!important}` esconde todo elemento marcado com `hideVend`.

**Os 7 pontos protegidos por `hideVend`**: card Faturamento, card Lucro, bloco Valores por forma de pagamento, botão de alternar o gráfico para R$, campo Custo unitário no formulário de venda, dica de lucro calculado e as colunas de faturamento nas tabelas.

**O que acontece no servidor**: o nó `Montar Payload` do workflow OLV Painel Mobile decide se envia lucro e custo unitário. Desde 29/07/2026 a decisão é `papel !== 'admin'`, ou seja, quem não for administrador recebe `null` nesses dois campos.

**Três avisos que precisam ser respeitados:**

**Primeiro**: os nomes internos `EH_VEND`, `.vend` e `.hideVend` **não foram renomeados** na mudança de 28/07. Foi decisão consciente de não mexer no que não precisava, mas deixa o código com nomes que dizem "vend" enquanto o papel se chama "colaborador". Não confunda o nome interno com o rótulo.

**Segundo**: esconder no navegador **não é controle de acesso**. O CSS apenas deixa de exibir; o dado, se enviado, continua no navegador e pode ser lido por qualquer pessoa com as ferramentas de desenvolvedor abertas. A proteção real é a filtragem no servidor.

**Terceiro**: o `Valor Total` de cada venda **é enviado para qualquer papel**, sem condicional. Ou seja, o faturamento nunca esteve protegido no servidor, só escondido na tela. Registrado como pendência 9 abaixo.

### 8.2 Incidente de 28/07/2026 e correção (registrado em 29/07/2026)

Registro do que aconteceu, para não se repetir.

A renomeação de 28/07 auditou os fluxos OLV Vendas e OLV Estoque procurando comparações negativas de papel, e concluiu corretamente que ali as travas eram positivas sobre `admin`. Mas **não auditou o OLV Painel Mobile**, que é justamente onde vive o endpoint de leitura de dados.

Naquele workflow havia a única comparação negativa do lado servidor: `papel === 'vendedor'`. Com o papel renomeado para `'colaborador'`, a comparação deixou de ser verdadeira, e o servidor **passou a enviar lucro e custo unitário de todas as vendas para o colaborador**. A tela continuava correta, porque o CSS escondia, mas o dado trafegava.

**Corrigido em 29/07/2026**, com autorização de Rogério: a comparação virou `papel !== 'admin'` e o workflow foi publicado. Verificado no navegador com o usuário colaborador: lucro e custo chegam como `null`.

**Ajuste adicional aplicado na mesma publicação**: o nó `Assinar D` estava com os parâmetros `type: SHA256` e `encoding: hex` presentes só na versão publicada, ausentes no rascunho. Publicar o rascunho sem tratar isso faria o nó cair no padrão do n8n e quebraria a validação de todos os tokens de sessão. Os dois valores foram fixados explicitamente, iguais aos que já rodavam. Comportamento inalterado.

**Regra que fica**: toda verificação de permissão no servidor deve ser **positiva sobre `admin`**. Assim, qualquer papel novo nasce restrito por padrão, em vez de nascer com acesso total por acidente. Comparação negativa contra um rótulo específico é proibida.

### 8.3 Pendências de segurança em aberto

| # | Ponto | Situação |
|---|-------|----------|
| 9 | **Ticket médio exposto ao colaborador** | O card "Nº de vendas" mostra o ticket médio em reais. Multiplicado pelo número de vendas, revela o faturamento do período. Contraria a regra da seção 7. Não foi causado pela renomeação; nunca esteve protegido. **Decisão pendente**: ou marcar o trecho com `hideVend` e filtrar no servidor, ou alterar a seção 7 para permitir que o colaborador veja ticket médio. Hoje o código e o documento discordam. |
| 10 | **Valor Total enviado a todos os papéis** | Ver seção 8.1, terceiro aviso. A correção completa depende do desenho de setores (seção 9.7). |

---

## 9. PLANO DE EVOLUÇÃO *

A evolução será feita por partes, uma de cada vez, seguindo sempre o mesmo ciclo para reduzir risco:

1. Definir: fechar o escopo e as decisões da etapa.
2. Implementar: construir a mudança sem afetar o que já funciona.
3. Testar: validar com dados reais, incluindo casos de erro.
4. Validar: você confere e aprova.
5. Produção: publica e segue para a próxima etapa.

Antes de cada etapa que mexe no painel, guardamos uma cópia da versão anterior, para poder voltar rápido se precisar.

### 9.1 Fundação: Base de dados no Supabase (construção do zero) *

**Status**: em andamento (iniciado em 25/07/2026). Construir a base de dados do sistema no Supabase (PostgreSQL), do zero, sem trazer histórico de vendas.

#### Andamento (concluído em 25/07/2026)

- Banco de dados criado no Supabase (projeto olv-distribuidora_sistema, região São Paulo).
- As 7 tabelas no ar, ainda vazias: clientes, produtos, vendas, pagamentos, estoque_movimentacoes, contas_pagar e contas_receber.
- Tranca de segurança (RLS) ativada em todas as tabelas.
- n8n ligado ao Supabase pela conexão Session Pooler (IPv4), credencial "Supabase OLV". Teste de consulta de ponta a ponta feito com sucesso.

#### Falta

- Especificar as colunas, tipos, chaves e índices de cada tabela: **concluído em 26/07/2026**, documentado na seção 3.
- Importar os clientes (Google Contatos), com limpeza e padronização de nomes antes de subir. **Atenção**: a trava de telefone recusa contatos repetidos, então a importação precisa tratar o erro de duplicidade em vez de parar no primeiro conflito.
- Religar os fluxos de vendas e estoque para gravar no Supabase.

#### O que entra na abertura

- **Clientes**: importados do Google Contatos (nome e endereço já salvos), com limpeza e padronização antes de subir.
- **Usuários de login**: recriados os acessos que já existem (Rogério e Gabriele como administradores e o Colaborador).
- **O resto, manual**: estoque, caixa, contas a pagar e a receber são lançados por você, pelo próprio painel, a partir da virada. Recomendação: no dia da virada, faça a contagem física do estoque (lançamento Estoque Inicial) e cadastre as vendas ainda em aberto, para os números começarem certos.

#### Por que fazer isso

- Segurança dos dados: o Supabase faz backup automático (no plano pago, com recuperação a um ponto no tempo).
- As funções novas (pagamento múltiplo, histórico do cliente, financeiro integrado) são naturais em um banco de dados relacional.
- A virada só acontece depois que você validar, com ponto de retorno guardado.

#### Domínio próprio

O painel passa a abrir em painel.olvdistribuidora.com.br, no lugar do endereço técnico atual.

O domínio olvdistribuidora.com.br está registrado no Registro.br, onde a configuração do DNS será feita. **Configuração pendente.**

### 9.2 Upgrade 1: Novo formato do painel *

Repaginar a interface para ficar mais moderna, mais rápida de usar e mais organizada conforme o painel cresce (com o módulo financeiro chegando).

#### Ideias em aberto

- Nova identidade visual: cores, tipografia e cards mais limpos; modo claro/escuro.
- Navegação: hoje são abas Vendas/Estoque no topo; avaliar um menu inferior fixo (estilo app) que comporte também Financeiro.
- Tela inicial com resumo do dia como primeira coisa que aparece (indicadores definidos abaixo).
- Transformar o painel em atalho na tela inicial do celular (PWA), abrindo como se fosse um aplicativo, inclusive com ícone próprio.
- Alertas visuais: estoque mínimo por produto e contas vencendo.

#### Estrutura de seções definida (25/07/2026)

O painel passa a ter seis seções: Dashboard (métricas), Vendas, Estoque, Clientes, Contas a Pagar e Contas a Receber.

#### Dashboard: indicadores da tela inicial *

| Indicador | O que mostra | Por que importa |
|-----------|--------------|-----------------|
| Vendas de hoje | Quantidade e faturamento do dia | Pulso do dia |
| A receber vencendo | Contas a receber no vencimento ou vencidas | Cobrança |
| Estoque baixo | Produtos abaixo do estoque mínimo | Reposição |
| **Pré-pagos a entregar** | Valor total e quantidade de pedidos pagos ainda não resgatados | Definido em 26/07/2026. Ver explicação abaixo |

**Sobre o indicador de pré-pagos**: a soma dos pré-pagos pendentes é produto que a empresa já recebeu e ainda deve entregar. É uma obrigação assumida, não lucro realizado. Serve para dois usos práticos: saber quanto de produto está comprometido antes de fazer uma compra, e enxergar pedidos esquecidos há meses. Hoje esse número não existe em lugar nenhum.

**Acesso ao Dashboard (definido em 29/07/2026)**: o Dashboard é exclusivo do administrador. Colaboradores não têm acesso, porque a tela consolida informação financeira e operacional de toda a empresa. Ver seção 9.7.

#### Seção Vendas: quadro de operação por status (definido em 26/07/2026)

A seção de Vendas deixa de ser só um histórico e passa a ser a tela de trabalho do dia. Os pedidos aparecem em lista, dividida em abas por status, e o operador vai movendo cada pedido conforme o andamento.

- **Quatro abas**, com esta regra de roteamento (cada pedido aparece em uma só):

| Aba | Regra |
|-----|-------|
| Aguardando | Não concluído, não pré-pago. O trabalho do dia |
| Pré-pagos | Pago e ainda não resgatado, sem prazo definido |
| Em rota | Saiu para entrega |
| Concluídos | Finalizados, mostrando o dia atual por padrão |

- O tipo Entrega ou Retirada aparece como marca visual no pedido, não como aba.
- Um pré-pago sai da sua aba assim que entra no fluxo do dia, e segue o caminho normal.
- A aba Pré-pagos é ordenada do mais antigo para o mais novo e permite busca por cliente, porque o atendimento começa quando o cliente aparece no balcão.
- **No cadastro**: o operador informa o status e o pedido já entra direto na aba correspondente.
- **Botão de mudança rápida**: cada pedido na lista muda de status com um toque, sem abrir a tela de edição completa.

É o mesmo princípio de um quadro de tarefas: cada coluna é uma etapa, e o cartão anda de coluna conforme o trabalho avança.

**Pontos em aberto desta tela:**

| # | Ponto | Alternativas |
|---|-------|--------------|
| A | Quantas abas | **Resolvido**: quatro (Aguardando, Pré-pagos, Em rota, Concluídos) |
| B | O que a aba de concluídos mostra | Só o dia atual por padrão, para não carregar milhares de pedidos antigos |
| C | Quais mudanças de status são permitidas | Livre em qualquer direção, ou só avançar, com exceção para corrigir engano |
| D | Contador na aba | Mostrar ou não a quantidade de pedidos em cada aba |
| E | Registrar a hora de cada mudança | Necessário se um dia você quiser medir o tempo entre pedido e entrega |
| F | Quem pode mudar o status | **Resolvido**: qualquer usuário ativo |

**Resolvido (F), 26/07/2026**: as permissões passam a ser separadas por tipo de ação, e não por tela.

| Ação | Quem pode |
|------|-----------|
| Mudar o status do pedido | Qualquer usuário ativo, em qualquer pedido |
| Editar valores, quantidade, cliente ou produto | Só o dono do lançamento, e o administrador |
| Excluir lançamento | Só o dono do lançamento, e o administrador |

O raciocínio: mudar status é ação operacional e reversível com um toque; alterar valor ou excluir é irreversível na prática. A proteção fica onde está o risco, que é o dinheiro, sem travar a operação do dia.

#### Cuidados

- Performance: manter o carregamento leve conforme o volume cresce (paginação, carregar só o necessário).
- Não quebrar os fluxos de gravação já validados ao trocar o visual.
- A aba de pedidos concluídos é a que mais cresce; sem filtro de período padrão, vira o ponto de lentidão da tela.

### 9.3 Upgrade 2: Login para múltiplos usuários (concluído)

**Status**: concluído e no ar (25/07/2026). Foi implementada a Opção A (contas individuais com papéis), detalhada abaixo. As duas opções ficam registradas como histórico da decisão.

**Atualização (28/07/2026)**: o rótulo do papel "vendedor" foi renomeado para "colaborador" em todo o sistema. Detalhes na seção 7.

#### Opção A: Contas individuais com papéis (ESCOLHIDA)

Cada pessoa tem usuário e senha próprios e um papel (por exemplo: administrador vê tudo; colaborador só lança vendas).

**Prós**: rastreabilidade (registra quem fez cada lançamento), permissões por função, desligar o acesso de uma pessoa sem afetar as outras.

**Contras**: mais complexo de construir e manter; exige gestão de usuários e de permissões.

#### Opção B: Vários logins simples (NÃO ESCOLHIDA)

Alguns pares de usuário/senha válidos, todos com o mesmo acesso total, sem distinção de papel.

**Prós**: simples e rápido de implementar; já resolve o "cada um com sua senha".

**Contras**: sem controle de permissão; rastreio de quem fez o quê fica limitado.

#### Pontos de segurança resolvidos

- Guardar os usuários fora do código: em uma tabela Data Table do n8n, não fixos no HTML.
- Nunca guardar senha em texto puro: usar senha com hash.
- Sessão com validade: token que expira, em vez da chave permanente salva no navegador.
- Registrar o usuário em cada lançamento (campo Responsável/Usuário) para rastreio.

### 9.4 Upgrade 3: Contas a pagar e a receber *

Adicionar um módulo financeiro ao painel, com duas seções: o que a empresa tem a receber e o que tem a pagar.

#### Contas a receber

- Puxar automaticamente as vendas com forma de pagamento "Em aberto" e "Crediario", que são recebíveis naturais.
- Controlar por cliente, valor, vencimento e status (em aberto / recebido).
- Baixa de recebimento pelo painel e alertas de vencimento enviados automaticamente no Telegram.

#### Contas a pagar

- Cadastro de contas com fornecedor, descrição, valor, vencimento, status e categoria.
- Ligar com as entradas de estoque (compras de gás, água, etc.), aproveitando o custo já registrado.
- Anexar boletos e comprovantes a cada conta, guardados no Supabase Storage.
- Alertas de vencimento enviados automaticamente no Telegram.

#### No painel

- Duas seções próprias, Contas a Pagar e Contas a Receber (definido em 25/07/2026, no lugar de sub-abas de um único Financeiro).
- Indicadores: total a vencer, total vencido, saldo projetado do período.
- Alertas de vencimento e filtros por período e status.

#### Vendas e Clientes (ligados ao financeiro)

- Vendas: uma venda pode ter mais de uma forma de pagamento (o gás pago parte em dinheiro e parte no crédito), registradas na tabela de pagamentos.
- Clientes: seção própria com cadastro, edição e o histórico de pedidos de cada cliente ao pesquisar pelo nome.

### 9.5 Upgrade 4: Controle de caixa *

Registrar o movimento de dinheiro do caixa no dia a dia: abertura, entradas e saídas ao longo do dia e fechamento com conferência. Fecha o ciclo financeiro operacional, junto com as contas a pagar e a receber.

#### Ideias em aberto

- Abertura de caixa: saldo inicial do dia (troco), por operador.
- Entradas: vendas recebidas em dinheiro (puxadas automaticamente das vendas com forma "Dinheiro") e reforços de caixa.
- Saídas: sangrias (retiradas), pagamentos feitos em dinheiro (ligados às contas a pagar) e despesas avulsas.
- Fechamento: saldo esperado x saldo contado, com a diferença (quebra ou sobra) destacada.
- Histórico por dia e por operador, e relatório de caixa do dia e do período.

#### No painel

- Nova seção "Caixa" com abertura, lançamentos de entrada/saída e fechamento.
- Indicadores: saldo atual em caixa, total de entradas e saídas do dia, e diferença no fechamento.

#### Integrações e cuidados

- Vendas em dinheiro alimentam as entradas; pagamentos em dinheiro alimentam as saídas, evitando digitar duas vezes.
- Depende do login para saber qual operador abriu e fechou o caixa, e para a conferência ter dono.
- Regra de um caixa aberto por vez (ou por operador) e bloqueio de lançamento em caixa já fechado.

### 9.6 Upgrade 5: Módulo fiscal via API (visão de futuro) *

Emitir automaticamente o documento fiscal da venda a partir do painel, integrando com a API de um provedor de emissão. Registrado como visão de futuro. Diferente das outras etapas, aqui já existe um contexto fiscal levantado (estado, regime, tributação e provedor), resumido abaixo.

#### Contexto fiscal da OLV (já levantado)

- **Estado**: Espírito Santo; emissão pela SEFAZ-ES.
- **Regime tributário**: Lucro Presumido (a nota precisa trazer PIS e COFINS).
- **Gás (GLP)**: tem Substituição Tributária; o ICMS é recolhido na origem, então sai como ICMS ST/substituto.
- **Água**: ICMS normal (sem ST).
- **Certificado digital (e-CNPJ)**: a empresa já possui.
- **Provedor avaliado**: NFe.io (emissor gratuito homologado, com API).

#### Qual documento emitir

- **NFC-e (modelo 65)**: a maioria das vendas (consumidor final, botijão e galão por WhatsApp ou balcão). É o cupom fiscal eletrônico, com QR Code.
- **NF-e (modelo 55)**: vendas para CNPJ (condomínio, comércio) que precisam do documento para a contabilidade.

#### Como funcionaria no painel

Ao confirmar a venda no painel, o n8n chama a API do provedor (ex.: NFe.io), que gera o XML assinado, transmite para a SEFAZ-ES e devolve a chave de acesso, o QR Code e o PDF. O painel guarda a chave/QR Code junto da venda e permite enviar a nota ao cliente por WhatsApp.

A conversa original sugeria Google Apps Script; no nosso projeto a integração vai pelo n8n, aproveitando os workflows que já existem.

#### Pendências para habilitar

- Gerar na SEFAZ-ES o CSC (Código de Segurança do Contribuinte) e o Identificador do Token, obrigatórios para NFC-e.
- Concluir o cadastro no provedor (houve erro na inscrição estadual 083592253; testar com zeros à esquerda, ex.: 00083592253, ou validar com o contador).
- Cadastro fiscal dos produtos: NCM, CFOP e tributação correta (ST no gás, normal na água).

#### Cuidados

- Vasilhame retornável (garrafão e botija): tem tratamento fiscal específico (comodato/troca) e normalmente não entra como venda; confirmar com o contador para não distorcer faturamento e estoque.
- Ordem correta do fluxo: venda confirmada, depois atualiza estoque, depois emite a nota.
- O certificado digital vence; renovar com antecedência. A NFC-e é obrigatória mesmo quando o cliente não pede.
- Pico de vendas no domingo: a emissão automática evita gargalo de digitação.

Estes pontos vêm da conversa "Diferença entre nota fiscal e cupom fiscal" e devem ser confirmados com o contador da empresa antes da execução.

### 9.7 Upgrade 6: Setores e permissões por área * (criado em 29/07/2026)

**Status**: definido em princípio, não construído. Decidido em 29/07/2026 que a execução acontece **depois da religação do Supabase (9.1)**, para não construir controle de acesso sobre uma estrutura que será substituída.

Substituir o modelo atual de dois papéis por um modelo em que a pessoa pertence a um ou mais setores da empresa, e o que ela enxerga decorre disso.

#### O modelo

Separar duas coisas que hoje estão misturadas em um campo só:

| Dimensão | Valores | Função |
|----------|---------|--------|
| papel | admin ou comum | Nível de acesso. Admin vê e faz tudo, ignora setores |
| setores | Lista: vendas, financeiro, operacional | Escopo. Só se aplica a quem é comum |

**Por que separar em vez de trocar**: todas as travas do sistema hoje são comparações positivas sobre `admin`. Mantendo o campo `papel` como está, essas travas continuam válidas sem nenhuma alteração, e os setores entram como camada adicional. Isso evita repetir o incidente de 28/07 (seção 8.2), em que renomear um rótulo derrubou uma proteção em silêncio.

**Vários setores por pessoa** (decidido em 29/07/2026). A permissão é a soma dos setores. Numa empresa pequena a mesma pessoa acumula funções, e forçar um setor único obrigaria a criar dois logins ou a promover alguém a administrador sem necessidade.

#### O que cada setor alcança

| Setor | Vê |
|-------|-----|
| vendas | Pedidos, clientes e valores de venda. Sem lucro, custo nem margem |
| financeiro | Contas a pagar e a receber, fluxo de caixa, faturamento, lucro e margem |
| operacional | Estoque, movimentações e alertas de mínimo |
| comum sem setor | Nada além da própria troca de senha |

**Dashboard**: exclusivo do administrador. Setores acessam a própria área, não o consolidado da empresa.

#### A regra inegociável

**A filtragem acontece no servidor, antes de o dado sair.** Esconder no navegador não conta como controle de acesso.

O incidente de 28/07 é a demonstração prática: a proteção visual continuou intacta enquanto o dado sensível trafegava livremente. Com setores, prometer separação de áreas e entregar apenas ocultação visual seria pior do que não prometer nada, porque cria confiança sem base.

Na prática, o nó `Montar Payload` deixa de montar um payload único e passa a montar o payload conforme os setores presentes no token.

#### O que muda em cada peça

| Peça | Mudança |
|------|---------|
| OLV Usuarios | Campo novo para a lista de setores |
| OLV Login | O token passa a carregar os setores |
| OLV Contas | Criação e edição de setores por usuário |
| OLV Painel Mobile | Payload filtrado por setor, no servidor |
| Painel HTML | Menu e telas conforme os setores do usuário |
| Endpoints novos | Já nascem com validação por setor |

#### Por que nesta posição da fila

Depois do Supabase, porque os fluxos ainda gravam na base anterior e o trabalho seria refeito. Antes do módulo financeiro, porque os endpoints novos previstos já nasceriam com a validação por setor embutida, o que é mais barato do que adaptá-los depois.

#### Decisões ainda em aberto

- Lista final de setores e os nomes exatos.
- Se um setor pode ter níveis internos (por exemplo, financeiro que consulta x financeiro que dá baixa).
- Como tratar o histórico: quem lançou uma venda continua vendo o próprio lançamento mesmo se sair do setor?
- Se o campo `papel` continua com dois valores ou se admin vira apenas mais um setor com tudo marcado.

---

## 10. ORDEM SUGERIDA E DECISÕES EM ABERTO

**Sequência recomendada** (flexível, você decide):

| Ordem | Etapa | Por quê nessa posição |
|-------|-------|----------------------|
| 1º | Login multiusuário (concluído e no ar) | Base de segurança e de rastreio; pré-requisito para liberar o financeiro e o caixa com responsabilidade. |
| 2º | Base de dados no Supabase (do zero), banco criado 25/07/2026 | Fundação nova; sustenta as funções novas (pagamento múltiplo, histórico do cliente, financeiro integrado) com backup automático. Sem migração histórica. |
| 3º | Setores e permissões por área | Definido em 29/07/2026. Depois do Supabase para não refazer o trabalho; antes do financeiro para que os endpoints novos já nasçam com validação por setor. |
| 4º | Contas a pagar e receber | Alto valor de negócio; usa dados que o próprio sistema já gera (vendas em aberto e crediário). |
| 5º | Controle de caixa | Fecha o ciclo financeiro do dia; usa vendas em dinheiro e pagamentos já registrados; precisa do operador (login). |
| 6º | Novo formato do painel | Polimento visual e de navegação; absorve as novas seções (financeiro e caixa) já prontas. |
| 7º | Módulo fiscal via API | Visão de futuro; o certificado já existe. Depende de gerar o CSC na SEFAZ-ES, do cadastro fiscal dos produtos (ST no gás) e da integração com o provedor (NFe.io). |

**Decisões a fechar quando chegarmos em cada etapa:**

- Login: opção A ou B; quais pessoas e (se A) quais papéis. **RESOLVIDO: Opção A, Rogério e Gabriele administradores, Colaborador colaborador.**
- Setores: lista final de setores, níveis internos e tratamento do histórico (ver 9.7).
- Ticket médio: o colaborador pode ver ou não (ver pendência 9 da seção 8.3).
- Financeiro: quais categorias de contas a pagar.
- Caixa: um caixa único ou um por operador; o que entra como sangria e reforço; como tratar diferenças no fechamento.
- Fiscal: confirmar ST do gás e tratamento do vasilhame com o contador; gerar o CSC na SEFAZ-ES; resolver o cadastro da inscrição estadual no provedor (NFe.io).
- Formato: estilo de navegação (abas no topo x menu inferior) e se vira atalho/app na tela inicial.

---

## 11. REFERÊNCIAS RÁPIDAS

### Infraestrutura

| Item | Valor |
|------|-------|
| URL do painel | https://n8n-wmtt.srv1830312.hstgr.cloud/webhook/olv-painel |
| Autenticação | Login por usuário e senha, com papéis (administrador e colaborador). Token de sessão de 12h. Chave única OLV2026 removida. |
| Instância n8n | n8n-wmtt.srv1830312.hstgr.cloud |
| Workflow do painel | OLV Painel Mobile (web) |
| Workflow de vendas | OLV Vendas – Lançamento (painel), endpoint /webhook/olv-venda |
| Workflow de estoque | OLV Estoque – Lançamento (painel), endpoint /webhook/olv-estoque |
| Workflow de login | OLV Login (autenticação; devolve o token de sessão) |
| Workflow de contas | OLV Contas (gestão de usuários; só administrador) |
| Tabelas internas do n8n | OLV Usuarios (usuários e papéis) e OLV Painel HTML (HTML do painel em base64) |
| Pontos de restauração (v1.4) | Painel 63bf15bb; Vendas 348ed17a; Estoque 2adfcba0 |
| Banco de dados | Supabase (PostgreSQL), projeto olv-distribuidora_sistema, região São Paulo. 7 tabelas criadas em 25/07/2026, RLS ativa. Sem migração histórica. |
| Domínio do painel | olvdistribuidora.com.br (Registro.br). Endereço planejado: painel.olvdistribuidora.com.br. DNS pendente. |
| Fonte de clientes | Google Contatos (nome e endereço), com limpeza antes de importar. |
| Conexão n8n → Supabase | Session Pooler (IPv4). Host aws-0-sa-east-1.pooler.supabase.com, porta 5432, base postgres, usuário postgres.ggvfrnympdrqyqxgcyex, SSL ativo. Credencial no n8n: "Supabase OLV". (A senha fica guardada só no n8n.) |
| Documento oficial | Repositório GitHub rogeriosandre/olv-distribuidora, arquivo Painel_OLV_Documentacao_e_Evolucao.md |

### Cuidado com cache ao carregar o documento

O endereço `raw.githubusercontent.com` entrega o arquivo por uma rede de distribuição que guarda cópias. Os caminhos `refs/heads/main/...` e `main/...` são tratados como endereços diferentes e podem ter cópias de idades diferentes. Em 29/07/2026 o primeiro entregou a v1.6 enquanto o segundo entregava a v1.7, com três dias de diferença.

**Use o caminho curto** (`.../main/arquivo.md`) e sempre confira a versão lida, conforme a regra 7 das Instruções para Claude.

---

## 12. GUIA DE MODELOS POR ETAPA *

Criado em 26/07/2026. Define qual modelo de IA usar em cada tipo de tarefa do projeto, para reduzir consumo sem aumentar risco.

### 12.1 Os modelos disponíveis

Um modelo de IA funciona como um profissional contratado por hora. O mais experiente resolve problema difícil com menos erro, mas custa mais caro por hora. O mais rápido resolve tarefa repetitiva por uma fração do preço. Contratar o sênior para arquivar papel é desperdício; contratar o júnior para desenhar a fundação da casa é risco.

| Modelo | Perfil | Uso no projeto |
|--------|--------|----------------|
| Haiku 4.5 | Rápido e econômico | Tarefa repetitiva e automação rodando dentro do n8n |
| Sonnet 5 | Equilíbrio entre custo e capacidade | Execução do que já foi especificado |
| Opus 5 | Raciocínio profundo e contexto longo | Decisão estrutural, regra de negócio, segurança |
| Fable 5 | Topo de linha da Anthropic | Reserva, só se o Opus 5 travar em algo específico |

### 12.2 Regra de acionamento

O gatilho é o tipo de tarefa, não uma avaliação caso a caso:

- **Opus 5**: quando a decisão é difícil de desfazer (esquema de banco, regra financeira, segurança, autenticação), quando exige cruzar várias seções deste documento, ou quando é planejamento de etapa.
- **Sonnet 5**: quando o "o quê" já está definido e falta o "como" (escrever SQL já especificado, ajustar nó do n8n, montar tela, revisar texto, depurar erro pontual).
- **Haiku 4.5**: tarefa repetitiva, classificação simples, e qualquer chamada de IA que rode dentro de fluxo em produção, onde custo por chamada e velocidade pesam mais que profundidade.

### 12.3 Modelo indicado por etapa

| Etapa | Tarefa | Modelo | Por quê |
|-------|--------|--------|---------|
| 9.1 Supabase | Desenho do esquema (tabelas, colunas, chaves, índices, RLS) | Opus 5 | Erro aqui só aparece meses depois |
| 9.1 Supabase | Escrever e aplicar o SQL já especificado | Sonnet 5 | Execução de escopo fechado |
| 9.1 Supabase | Lógica de limpeza e padronização dos nomes de clientes | Opus 5 | Agrupar nomes parecidos sem juntar clientes diferentes |
| 9.1 Supabase | Rodar o script de limpeza e revisar a lista | Sonnet 5 | Trabalho mecânico |
| 9.1 Supabase | Plano de virada e ponto de retorno | Opus 5 | Risco alto, muitas dependências |
| 9.1 Supabase | Religar os fluxos n8n: desenho | Opus 5 | Muda o contrato dos 5 workflows ao mesmo tempo |
| 9.1 Supabase | Religar os fluxos n8n: aplicação nó a nó | Sonnet 5 | Repetitivo |
| 9.2 Novo painel | Arquitetura de navegação e performance | Opus 5 | Decisão estrutural |
| 9.2 Novo painel | Telas, estilo, componentes | Sonnet 5 | Execução visual |
| 9.4 Financeiro | Regras, casos de borda e pagamento múltiplo | Opus 5 | Regra errada gera número errado |
| 9.4 Financeiro | Telas e operações de cadastro | Sonnet 5 | Padrão conhecido |
| 9.5 Caixa | Regras de abertura, sangria e fechamento | Opus 5 | Envolve conferência e responsabilidade |
| 9.5 Caixa | Telas e histórico | Sonnet 5 | Execução |
| 9.6 Fiscal | Mapeamento fiscal e desenho da integração | Opus 5 | Alto risco, exige marcar pendência em vez de supor |
| 9.6 Fiscal | Testes de chamada da API | Sonnet 5 | Tentativa e erro controlado |
| **9.7 Setores** | **Desenho do modelo de permissões e da filtragem no servidor** | **Opus 5** | **Segurança. Erro aqui expõe dado sem avisar** |
| **9.7 Setores** | **Aplicação nos nós e nas telas, com o desenho fechado** | **Sonnet 5** | **Execução** |
| Qualquer etapa | Alteração de papel, permissão ou trava de acesso | Opus 5 | Ver seção 8.2. Já houve incidente |
| Documento | Merge, revisão cruzada, mudança de versão | Opus 5 | Documento longo com regras próprias |
| Documento | Ajuste de texto e tabela | Sonnet 5 | Edição pontual |

**Regra fixa**: nada do módulo fiscal vai para produção sem validação do contador, independentemente do modelo usado.

**Regra fixa (criada em 29/07/2026)**: qualquer tarefa que altere papel, permissão ou trava de acesso deve auditar **todos os cinco workflows**, não apenas os que parecem relacionados. O incidente da seção 8.2 aconteceu porque a auditoria deixou um workflow de fora.

### 12.4 Como o Claude vai avisar

Ao iniciar qualquer tarefa do projeto, a primeira linha da resposta traz:

> **Modelo indicado: [X]. Motivo: [tipo de tarefa].**

Quando o modelo em uso for mais pesado que o necessário, o aviso é explícito:

> **Você está no Opus 5, mas esta tarefa roda bem no Sonnet 5. Se quiser economizar, abra uma conversa nova no Sonnet com este trecho.**

### 12.5 Limites desta prática

Registrado para não criar expectativa errada:

- O Claude **não troca de modelo sozinho**. A troca é manual, no seletor da interface.
- Trocar de modelo no meio da conversa **carrega todo o histórico** para o novo modelo. A economia é parcial.
- A economia maior vem de **abrir conversa nova e curta** no modelo leve, com só o trecho necessário deste documento.
- O aviso do modelo é baseado em **regra escrita** (tipo de tarefa), não em autoavaliação do Claude, que não é confiável para julgar a própria necessidade.
- **Cuidado aprendido em 29/07/2026**: economizar contexto instruindo uma sessão a **não carregar o documento** removeu a única defesa contra o cache e causou a perda descrita no log da v2.9. Se a sessão puder acabar lendo ou editando o documento por qualquer motivo, a regra 7 das Instruções para Claude precisa ir junto no prompt.
- **Pendência**: dividir este documento em um núcleo enxuto (estado atual, arquitetura, decisões, referências) mais anexos por etapa, carregados só quando a etapa estiver em execução. É a economia estrutural, maior que a troca de modelo. Sugerido para depois da virada do Supabase.

---

## LOG DE MUDANÇAS

| Data | Versão | Mudança |
|------|--------|---------|
| 25/07/2026 | 1.6 | Documento criado em .md; estrutura de atualização definida; login multiusuário implementado; banco Supabase criado; decisão tomada de construção do zero sem migração de histórico. |
| 26/07/2026 | 1.7 | Criada a seção 12, Guia de Modelos por Etapa, com regra de acionamento, modelo indicado por tarefa e protocolo de aviso no início de cada tarefa. Incluída a regra 6 nas Instruções para Claude. |
| 26/07/2026 | 1.8 | Removidas todas as referências à base anterior em planilha, conforme a decisão de sistema independente. A antiga seção 3 (estrutura da planilha) foi substituída pela seção 3, Estrutura de Dados (Supabase). Ajustadas as seções 1, 2, 2.1, 4, 5.1, 6, 8, 9.1, 9.4, 10, 11 e 12.3. |
| 26/07/2026 | 1.9 | Seção 3 reescrita com o esquema real das 7 tabelas do Supabase (colunas, tipos, chaves, relacionamentos e validações), lido direto do banco. Criada a 3.9 com 5 decisões em aberto e 2 verificações. Corrigido o nome do projeto no Supabase. |
| 26/07/2026 | 2.0 | Índices e políticas de RLS verificados no banco e documentados na nova seção 3.10. Confirmado que os índices necessários já existem e que a tranca do banco bloqueia acesso externo, com a proteção por papel dependendo do n8n. |
| 26/07/2026 | 2.1 | Aplicada a validação de status em vendas. Decisão 3 da seção 3.9 concluída. |
| 26/07/2026 | 2.2 | Status da venda passou a obrigatório, com valor inicial Aguardando e o novo estado Em rota. Fluxo dos status documentado na seção 3.3. |
| 26/07/2026 | 2.3 | Definida a seção Vendas como quadro de operação: lista dividida em abas por status, entrada direta na aba escolhida no cadastro e botão de mudança rápida. Registrados 6 pontos em aberto. |
| 26/07/2026 | 2.4 | Papel "Vendedor" renomeado para "Colaborador" em todo o documento. Ponto F da seção 9.2 resolvido. |
| 26/07/2026 | 2.5 | Separados os eixos do pedido: nova coluna tipo_entrega (Entrega ou Retirada) e status reduzido a Aguardando, Em rota e Concluído, com o banco recusando Retirada em rota. Ponto F revisto: mudança de status liberada para qualquer usuário ativo, mantendo a trava de dono para editar valores e excluir. |
| 26/07/2026 | 2.6 | Criado o pedido pré-pago: campos pre_pago e data_conclusao, esta preenchida pelo banco na conclusão. Quadro de vendas passa a ter quatro abas. Registrada a pendência 6, mudança do gatilho de baixa de estoque. |
| 26/07/2026 | 2.7 | Seção 6 alterada com autorização: a baixa de estoque passou da data do pedido para a data da conclusão. Pendência 6 encerrada. Definidos os indicadores do Dashboard, incluindo Pré-pagos a entregar. |
| 26/07/2026 | 2.8 | Criada a trava de cliente duplicado por telefone, com normalização automática para só dígitos. Adicionado índice de busca por nome. Pendência 2 encerrada. |
| 29/07/2026 | 2.9 | **Restauração e correção de segurança.** (a) Restaurada a v2.8, que havia sido sobrescrita em 28/07 pelo commit 639642a. Aquele commit partiu da v1.6 em vez da v2.8, porque a sessão que o gerou leu o documento pela URL com `refs/heads/main` e recebeu uma cópia em cache de três dias antes. O conteúdo da v2.8 foi recuperado do commit 9b906a7 e nada se perdeu. (b) Incorporada a renomeação do papel aplicada no sistema em 28/07, detalhada na seção 7. (c) Criada a seção 8.1 documentando o mecanismo de ocultação financeira (classe `vend` e `hideVend`), que não estava registrado em lugar nenhum. (d) Criada a seção 8.2 registrando o incidente: a renomeação quebrou a filtragem no servidor do nó Montar Payload, que usava comparação negativa contra "vendedor", expondo lucro e custo unitário ao colaborador por cerca de 16 horas. Corrigido para comparação positiva sobre "admin", publicado e verificado. Fixados também os parâmetros SHA256 e hex do nó Assinar D, ausentes no rascunho, que teriam quebrado a validação de token na publicação. (e) Criada a seção 8.3 com duas pendências de segurança: ticket médio exposto ao colaborador e Valor Total enviado a todos os papéis. (f) Criada a etapa 9.7, Setores e permissões por área, com execução decidida para depois da religação do Supabase. (g) Corrigido o texto da pendência 3 da seção 3.9, que descrevia quatro status e ficou desatualizado desde a v2.5. (h) Acrescentadas as pendências 7 e 8 na seção 3.9. (i) Criada a regra 7 nas Instruções para Claude, de conferência obrigatória de versão. |

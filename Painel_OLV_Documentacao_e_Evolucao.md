# OLV DISTRIBUIDORA
## Painel de Vendas, Estoque e Financeiro

**Documentação do funcionamento e plano de evolução**

Versão 1.6 · 25/07/2026
Preparado para Rogério

---

## INSTRUÇÕES PARA CLAUDE (leia antes de editar)

Este documento é a fonte única de verdade do projeto. Quando você o mencionar em nova conversa, carregue-o como contexto. Regras de atualização:

1. **Seção com asterisco (*)**: Rogério atualiza via conversa. Exemplo: "*9.1 Fundação: Base de dados no Supabase" significa que quando há mudança, ele pede "atualize essa seção com X".
2. **Sem asterisco**: Já foi decidido e finalizado. Não altere sem permissão explícita.
3. **Status e Andamento**: Atualize conforme Rogério relata progresso. Use a data do dia quando mudar status.
4. **Falta/Pendência**: Sempre que uma tarefa ficar pronta, mude "Falta:" para "Concluído:" e inclua a data.
5. **Data de última atualização**: Sempre que editar, atualize o "Versão" no topo (formato V.X · DD/MM/AAAA).

---

## 1. VISÃO GERAL

O projeto é um painel operacional da OLV Distribuidora que roda no navegador (celular ou computador) e trabalha diretamente sobre a planilha Google Sheets "Vendas_OLV Distribuidora". Ele centraliza três frentes: registro e consulta de vendas, controle de estoque, e (na evolução planejada) o controle financeiro de contas a pagar e a receber.

O objetivo é dar autonomia para lançar e consultar o dia a dia do negócio sem abrir a planilha, com cálculos automáticos de lucro e de saldo de estoque, e sem depender de nenhum aplicativo instalado.

**Estado atual**: existe uma única versão do painel, acessível pela web. A antiga versão desktop (que rodava dentro do Cowork) foi aposentada, e agora tudo é mantido em um só lugar. Desde 25/07/2026, o acesso passou a ser por login individual com papéis (administrador e vendedor), substituindo a chave única anterior.

**Decisão de 25/07/2026 (banco de dados)**: o projeto vai adotar o Supabase como novo banco de dados, construído do zero. O Supabase é um banco de dados profissional (PostgreSQL), o mesmo tipo usado em aplicativos de mercado. A planilha atual deixa de ser a base do sistema e passa a ser um arquivo histórico, mantido só para consulta. Não haverá migração dos dados antigos: o sistema começa vazio e vai sendo preenchido pelo uso, com apenas os clientes e os usuários de login inseridos na abertura.

---

## 2. ARQUITETURA ATUAL

O painel é uma página web servida pelo n8n. Toda leitura e gravação de dados passa pelo n8n, que por sua vez fala com o Google Sheets. O fluxo é direto e não depende de nenhum computador ligado:

```
Navegador (celular/PC) → n8n (servidor) → Google Sheets
```

O celular abre a página do painel e envia as ações (nova venda, edição, lançamento de estoque) para o n8n. O n8n valida o login e o token de sessão, grava na planilha e devolve a resposta. A leitura dos dados vem de um endpoint do n8n que lê a planilha e devolve um JSON compacto.

Como é autônomo, o painel funciona 24/7 mesmo com o computador desligado. A única dependência é o servidor n8n estar no ar.

**Evolução planejada (banco de dados)**: a arquitetura vai passar de Navegador → n8n → Google Sheets para Navegador → n8n → Supabase. O n8n é mantido como camada de trás (continua validando o login, calculando e gravando); só troca a planilha pelo Supabase como base de dados. Os anexos do financeiro (boletos e comprovantes) ficarão guardados no Supabase Storage, um cofre de arquivos do próprio Supabase. Esta mudança está planejada, ainda não concluída; o detalhamento está na seção 9.

### 2.1 Camada de login e usuários (novo)

Com o login multiusuário no ar, três peças novas passaram a fazer parte da arquitetura, sem mudar o fluxo principal Navegador → n8n → Google Sheets:

**Tabela de usuários ("OLV Usuarios")**: uma Data Table interna do n8n (uma tabelinha do próprio n8n, separada da planilha do negócio) que guarda usuário, nome, senha embaralhada, papel e se está ativo. Ficar fora da planilha é mais seguro, como manter a portaria num cômodo separado do cofre.

**Fluxo de Login ("OLV Login")**: recebe usuário e senha, confere na tabela e devolve um token de sessão (um crachá temporário assinado) que carrega o papel e o nome e expira sozinho em 12 horas.

**Fluxo de Contas ("OLV Contas")**: cria usuário, troca senha, lista, muda papel e ativa/desativa acessos. As ações de administração só funcionam se quem chamou for administrador.

**HTML servido de forma segura**: o código da página é grande, então fica guardado codificado (base64, uma forma de escrever o conteúdo como texto para transporte) numa Data Table chamada "OLV Painel HTML", e o fluxo do painel decodifica e monta a página ao servir. Para mudar o visual no futuro, atualiza-se essa tabela.

---

## 3. A PLANILHA (FONTE DE DADOS)

Toda a informação vive na planilha Google Sheets "Vendas_OLV Distribuidora". As abas relevantes para o painel são:

### 3.1 Estrutura da aba Vendas Geral

Cada venda é uma linha, com as colunas A a R. As colunas de cálculo (Custo Total, Lucro, Mês) são fórmulas nas linhas históricas.

| Coluna | Campo | Observação |
|--------|-------|-----------|
| B | Cliente | Nome do cliente |
| C / D / E | Endereço / Bairro / Telefone | Dados de entrega, preenchidos pelo autocompletar |
| F | Produto | Ex.: Água 20L, Gás 13kg |
| G | Qtd. | Quantidade vendida (abate estoque) |
| H | Valor Total | Valor da venda |
| I | Data | Data da venda |
| J | Forma de Pagamento | Pix, Crédito, Dinheiro, Crediario, Em aberto, etc. |
| M | Status | Entregue, Aguardando, Recebido, Retirada |
| N | Obs | Observação livre |
| O | Custo Unitário | Custo por unidade (base do lucro) |
| P / Q / R | Custo Total / Lucro / Mês | Calculados (Custo Total = Qtd × Custo; Lucro = Valor − Custo Total) |
| S | Responsavel | Quem lançou a venda, preenchido automaticamente com o usuário logado. Vendas antigas ficam em branco. |

---

## 4. OS WORKFLOWS N8N

O painel é sustentado por cinco workflows ativos na instância n8n-wmtt.srv1830312.hstgr.cloud. Todos usam a mesma credencial do Google Sheets. São todos necessários:

| Workflow | O que faz | Endpoint |
|----------|-----------|----------|
| OLV Painel Mobile (web) | Serve a página HTML do painel e o endpoint de leitura de dados (vendas + estoque + clientes). | /webhook/olv-painel e /webhook/olv-dados |
| OLV Vendas – Lançamento (painel) | Cria, edita e exclui vendas na aba Vendas Geral. Calcula Custo Total, Lucro e Mês. | POST /webhook/olv-venda |
| OLV Estoque – Lançamento (painel) | Cria, edita e exclui lançamentos de estoque na aba Estoque. | POST /webhook/olv-estoque |
| OLV Login | Recebe usuário e senha, confere na tabela de usuários e devolve o token de sessão (crachá temporário de 12 horas). | POST (webhook do fluxo OLV Login) |
| OLV Contas | Cria usuário, troca senha, lista, muda papel e ativa/desativa acessos. Ações de administração exigem papel de administrador. | POST (webhook do fluxo OLV Contas) |

Há ainda um workflow "OLV Estoque – Setup aba" que já foi arquivado (cumpriu a função de criar a aba Estoque). O "OLV Atendimento - Agente WhatsApp" pertence a outro projeto e não faz parte do painel.

---

## 5. O PAINEL (FUNCIONALIDADES ATUAIS)

### 5.1 Aba Vendas

- Filtros de período: hoje, ontem, últimos 7 dias, este mês, mês anterior, tudo e período personalizado.
- Indicadores do período: itens vendidos, faturamento, lucro e número de vendas, com médias e margem.
- Quantidade por produto e valores por forma de pagamento.
- Gráfico de evolução por produto (por dia ou por mês, alternando entre quantidade e faturamento).
- Nova venda: formulário com autocompletar de cliente (puxa endereço, bairro e telefone da aba Clientes) e cálculo automático de custo total e lucro.
- Editar e excluir vendas direto na lista, com confirmação em dois cliques na exclusão.
- Filtros de consulta: por produto, forma de pagamento e status, além da busca por texto (cliente/produto).

### 5.2 Aba Estoque

- Tabela de estoque atual por produto, calculado automaticamente.
- Lançamento de movimentações: Entrada, Ajuste (aceita negativo) e Estoque Inicial (contagem que zera o histórico anterior daquele produto).
- Histórico de lançamentos com editar e excluir.

### 5.3 Login, usuários e papéis (novo)

- Tela de login com usuário e senha, no lugar da chave única.
- Crachá no topo com o nome e o papel de quem está logado, e os botões Trocar senha e Sair.
- Troca de senha obrigatória no primeiro acesso e disponível a qualquer momento.
- Área de Usuários (só administrador): listar, criar usuário, tornar administrador ou vendedor e ativar/desativar acessos.
- Responsável automático: o campo "quem lançou" saiu do formulário; o sistema preenche sozinho com o usuário logado.
- Trava de dono: o vendedor só vê Editar e Excluir nas próprias linhas, e o servidor confere o dono antes de gravar.

---

## 6. REGRAS DE CÁLCULO

### Estoque atual

Estoque atual = último Estoque Inicial + entradas ± ajustes − vendas, considerando apenas as datas iguais ou posteriores à data da contagem inicial. Vendas sem quantidade preenchida não abatem estoque.

### Lucro

Lucro = Valor Total − (Quantidade × Custo Unitário). O custo unitário é informado no momento da venda. Importante: vendas gravadas pelo painel gravam o lucro como valor numérico fixo; as linhas antigas usam fórmula. Se o valor for editado direto na planilha depois, aquela linha não recalcula sozinha, por isso o caminho recomendado de edição é sempre pelo painel.

---

## 7. ACESSO E SEGURANÇA ATUAL

Acesso pela URL https://n8n-wmtt.srv1830312.hstgr.cloud/webhook/olv-painel.

O acesso agora é por login individual: cada pessoa tem usuário e senha próprios e um papel. A chave única OLV2026 foi removida.

### Papéis

**Administrador**: vê tudo (faturamento, lucro, margem, valores por forma de pagamento e gráfico em R$), edita e exclui qualquer lançamento e gerencia usuários (cria, muda papel, ativa e desativa).

**Vendedor**: vê o operacional (itens vendidos, número de vendas e estoque atual), sem faturamento, lucro nem margem; só edita e exclui as próprias vendas e lançamentos, com a trava conferida também no servidor.

**Contas iniciais**: Rogério (administrador), Gabriele (administradora) e Vendedor (vendedor). Todas começam com senha temporária, trocada no primeiro acesso. A criação de contas é feita só pelo administrador (não há autocadastro na tela de login).

### Como as senhas e a sessão são protegidas (siglas explicadas)

**Senha embaralhada (hash)**: a senha nunca é guardada como texto. Guardamos um resumo irreversível dela (hash SHA-256, sigla de Secure Hash Algorithm, o algoritmo que gera esse resumo) somado a um sal (salt: um tempero aleatório, único por usuário, que impede que senhas iguais gerem o mesmo código).

**Token assinado**: o crachá de sessão é validado por uma assinatura (um selo que só o servidor consegue gerar, com um segredo interno). Crachá forjado não passa.

**Sessão com validade**: o crachá expira em 12 horas; depois o painel pede login de novo.

**Nota técnica para manutenção**: este servidor n8n bloqueia criptografia dentro do nó de código, então o embaralhamento da senha e a assinatura do token usam o nó Crypto nativo do n8n (hash SHA-256 com segredo interno), que dispensa credencial e é seguro contra falsificação.

---

## 8. CONVENÇÕES E CUIDADOS

- Vendas e lançamentos são identificados pela posição da linha na planilha (row_number); depois de mexer manualmente na aba, recarregue o painel antes de editar ou excluir por ele.
- Os nomes de produto precisam bater exatamente (a comparação é sem diferenciar maiúsculas/minúsculas); por isso o produto é escolhido em menu suspenso.
- O cadastro de clientes tem cerca de 226 nomes, mas a aba de vendas tem cerca de 2.078 variações de nome (erros de digitação históricos). Padronizar isso é uma oportunidade de melhoria.
- O endpoint de leitura foi configurado com cabeçalho no-store e um parâmetro anti-cache, para o painel sempre puxar dados frescos após uma gravação.
- **Custo do vendedor (pendência)**: o campo "Custo unitário" foi ocultado do vendedor. Enquanto a busca automática do último custo do produto não for implementada, as vendas lançadas por vendedores ficam sem custo, e o lucro dessas vendas fica igual ao valor. Vale priorizar essa melhoria ou reavaliar mostrar o campo.

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

**Status**: em andamento (iniciado em 25/07/2026). Mover a base de dados da planilha Google Sheets para o Supabase (banco de dados PostgreSQL), construindo o sistema do zero, sem trazer o histórico de vendas. A planilha continua existindo como arquivo de consulta.

#### Andamento (concluído em 25/07/2026)

- Banco de dados criado no Supabase (projeto olv-distribuidora, região São Paulo).
- As 7 tabelas no ar, ainda vazias: clientes, produtos, vendas, pagamentos, estoque_movimentacoes, contas_pagar e contas_receber.
- Tranca de segurança (RLS) ativada em todas as tabelas.
- n8n ligado ao Supabase pela conexão Session Pooler (IPv4), credencial "Supabase OLV". Teste de consulta de ponta a ponta feito com sucesso.

#### Falta

- Importar os clientes (Google Contatos) e religar os fluxos de vendas e estoque para gravar no Supabase.

#### Como fica organizado (as tabelas)

No lugar de uma única aba com tudo, cada coisa vira uma tabela própria, ligada às outras por um código (ID):

- **clientes**: nome, endereço, bairro e telefone, cada um com ID próprio.
- **produtos**: nome padronizado, custo atual e estoque mínimo.
- **vendas**: cada venda ligada ao cliente e ao produto pelo ID.
- **pagamentos**: ligada à venda; permite uma venda com mais de uma forma de pagamento (por exemplo, o gás pago parte em dinheiro e parte no crédito).
- **estoque**: movimentações de entrada, ajuste e estoque inicial, como já funciona hoje.
- **contas a pagar e contas a receber**: base do módulo financeiro.
- **anexos (Supabase Storage)**: boletos e comprovantes guardados junto das contas.

#### O que entra na abertura

- **Clientes**: importados do Google Contatos (nome e endereço já salvos), com limpeza e padronização antes de subir.
- **Usuários de login**: recriados os acessos que já existem (Rogério e Gabriele como administradores e o Vendedor).
- **O resto, manual**: estoque, caixa, contas a pagar e a receber são lançados por você, pelo próprio painel, a partir da virada. Recomendação: no dia da virada, faça a contagem física do estoque (lançamento Estoque Inicial) e cadastre as vendas ainda em aberto, para os números começarem certos.

#### Por que fazer isso

- Tira o receio de perder a planilha: o Supabase faz backup automático (no plano pago, com recuperação a um ponto no tempo).
- As funções novas (pagamento múltiplo, histórico do cliente, financeiro integrado) são naturais em um banco de dados e ficam forçadas numa planilha.
- A planilha original é preservada e a virada só acontece depois que você validar, então dá para voltar atrás.

#### Domínio próprio

O painel passa a abrir em painel.olvdistribuidora.com.br, no lugar do endereço técnico atual.

O domínio olvdistribuidora.com.br está registrado no Registro.br, onde a configuração do DNS será feita. **Configuração pendente.**

### 9.2 Upgrade 1: Novo formato do painel *

Repaginar a interface para ficar mais moderna, mais rápida de usar e mais organizada conforme o painel cresce (com o módulo financeiro chegando).

#### Ideias em aberto

- Nova identidade visual: cores, tipografia e cards mais limpos; modo claro/escuro.
- Navegação: hoje são abas Vendas/Estoque no topo; avaliar um menu inferior fixo (estilo app) que comporte também Financeiro.
- Tela inicial com resumo do dia (vendas de hoje, a receber vencendo, estoque baixo) como primeira coisa que aparece.
- Transformar o painel em atalho na tela inicial do celular (PWA), abrindo como se fosse um aplicativo, inclusive com ícone próprio.
- Alertas visuais: estoque mínimo por produto e contas vencendo.

#### Estrutura de seções definida (25/07/2026)

O painel passa a ter seis seções: Dashboard (métricas), Vendas, Estoque, Clientes, Contas a Pagar e Contas a Receber.

#### Cuidados

- Performance: a base tem milhares de vendas; manter o carregamento leve (paginação, carregar só o necessário).
- Não quebrar os fluxos de gravação já validados ao trocar o visual.

### 9.3 Upgrade 2: Login para múltiplos usuários (concluído)

**Status**: concluído e no ar (25/07/2026). Foi implementada a Opção A (contas individuais com papéis), detalhada abaixo. As duas opções ficam registradas como histórico da decisão.

#### Opção A: Contas individuais com papéis (ESCOLHIDA)

Cada pessoa tem usuário e senha próprios e um papel (por exemplo: administrador vê tudo; vendedor só lança vendas).

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

- Aproveitar a aba Recebiveis que já existe na planilha, hoje fora do painel.
- Puxar automaticamente as vendas com forma de pagamento "Em aberto" e "Crediario", que são recebíveis naturais.
- Controlar por cliente, valor, vencimento e status (em aberto / recebido), com baixa de recebimento pelo painel.
- Baixa de recebimento pelo painel e alertas de vencimento enviados automaticamente no Telegram.

**Atualização (25/07/2026)**: com a base no Supabase, os recebíveis passam a viver na tabela Contas a Receber; a aba Recebiveis da planilha deixa de ser usada.

#### Contas a pagar

- Nova aba "Contas a Pagar" (fornecedor, descrição, valor, vencimento, status, categoria).
- Ligar com as entradas de estoque (compras de gás, água, etc.), aproveitando o custo já registrado.
- Anexar boletos e comprovantes a cada conta, guardados no Supabase Storage.
- Alertas de vencimento enviados automaticamente no Telegram.

#### No painel

- Nova seção "Financeiro" com sub-abas A Receber e A Pagar.
- Indicadores: total a vencer, total vencido, saldo projetado do período.
- Alertas de vencimento e filtros por período e status.

**Atualização (25/07/2026)**: as contas viram duas seções próprias no painel, Contas a Pagar e Contas a Receber, no lugar de sub-abas de um único Financeiro.

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

---

## 10. ORDEM SUGERIDA E DECISÕES EM ABERTO

**Sequência recomendada** (flexível, você decide):

| Ordem | Etapa | Por quê nessa posição |
|-------|-------|----------------------|
| 1º | Login multiusuário (concluído e no ar) | Base de segurança e de rastreio; pré-requisito para liberar o financeiro e o caixa com responsabilidade. |
| 2º | Base de dados no Supabase (do zero), banco criado 25/07/2026 | Fundação nova; tira o risco de perder a planilha e sustenta as funções novas (pagamento múltiplo, histórico do cliente, financeiro integrado). Sem migração histórica. |
| 3º | Contas a pagar e receber | Alto valor de negócio; usa dados que já existem (vendas em aberto, aba Recebiveis). |
| 4º | Controle de caixa | Fecha o ciclo financeiro do dia; usa vendas em dinheiro e pagamentos já registrados; precisa do operador (login). |
| 5º | Novo formato do painel | Polimento visual e de navegação; absorve as novas seções (financeiro e caixa) já prontas. |
| 6º | Módulo fiscal via API | Visão de futuro; o certificado já existe. Depende de gerar o CSC na SEFAZ-ES, do cadastro fiscal dos produtos (ST no gás) e da integração com o provedor (NFe.io). |

**Decisões a fechar quando chegarmos em cada etapa:**

- Login: opção A ou B; quais pessoas e (se A) quais papéis. **RESOLVIDO: Opção A, Rogério e Gabriele administradores, Vendedor vendedor.**
- Financeiro: usar a aba Recebiveis como está ou reestruturar; quais categorias de contas a pagar.
- Caixa: um caixa único ou um por operador; o que entra como sangria e reforço; como tratar diferenças no fechamento.
- Fiscal: confirmar ST do gás e tratamento do vasilhame com o contador; gerar o CSC na SEFAZ-ES; resolver o cadastro da inscrição estadual no provedor (NFe.io).
- Formato: estilo de navegação (abas no topo x menu inferior) e se vira atalho/app na tela inicial.

---

## 11. REFERÊNCIAS RÁPIDAS

### Abas da planilha (função no painel)

| Aba | Papel no painel |
|-----|-----------------|
| Vendas Geral | Registro de todas as vendas. É a base do painel de vendas e do abatimento de estoque. |
| Clientes | Cadastro de clientes (nome, endereço, bairro, telefone). Alimenta o autocompletar. |
| Estoque | Movimentações de estoque (entradas, ajustes, contagem inicial). |
| Recebiveis | Aba já existente, ainda não usada pelo painel. Base natural para o módulo financeiro. |
| Resumo Mensal / Dashboard / etc. | Abas de análise da própria planilha, fora do escopo do painel. |

### Infraestrutura (referência rápida)

| Item | Valor |
|------|-------|
| Planilha | Vendas_OLV Distribuidora (Google Sheets) |
| URL do painel | https://n8n-wmtt.srv1830312.hstgr.cloud/webhook/olv-painel |
| Autenticação | Login por usuário e senha, com papéis (administrador e vendedor). Token de sessão de 12h. Chave única OLV2026 removida. |
| Instância n8n | n8n-wmtt.srv1830312.hstgr.cloud |
| Workflow do painel | OLV Painel Mobile (web) |
| Workflow de vendas | OLV Vendas – Lançamento (painel), endpoint /webhook/olv-venda |
| Workflow de estoque | OLV Estoque – Lançamento (painel), endpoint /webhook/olv-estoque |
| Workflow de login | OLV Login (autenticação; devolve o token de sessão) |
| Workflow de contas | OLV Contas (gestão de usuários; só administrador) |
| Tabelas internas do n8n | OLV Usuarios (usuários e papéis) e OLV Painel HTML (HTML do painel em base64) |
| Pontos de restauração (v1.4) | Painel 63bf15bb; Vendas 348ed17a; Estoque 2adfcba0 |
| Banco de dados | Supabase (PostgreSQL), projeto olv-distribuidora, região São Paulo. 7 tabelas criadas em 25/07/2026, RLS ativa. Sem migração histórica; a planilha vira arquivo de consulta. |
| Domínio do painel | olvdistribuidora.com.br (Registro.br). Endereço planejado: painel.olvdistribuidora.com.br. DNS pendente. |
| Fonte de clientes | Google Contatos (nome e endereço), com limpeza antes de importar. |
| Conexão n8n → Supabase | Session Pooler (IPv4). Host aws-0-sa-east-1.pooler.supabase.com, porta 5432, base postgres, usuário postgres.ggvfrnympdrqyqxgcyex, SSL ativo. Credencial no n8n: "Supabase OLV". (A senha fica guardada só no n8n.) |

---

## LOG DE MUDANÇAS

| Data | Versão | Mudança |
|------|--------|---------|
| 25/07/2026 | 1.6 | Documento criado em .md; estrutura de atualização definida; login multiusuário implementado; banco Supabase criado; decisão tomada de construção do zero sem migração de histórico. |

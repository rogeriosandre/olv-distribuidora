# OLV DISTRIBUIDORA
## Painel de Vendas, Estoque e Financeiro

**Documentação do funcionamento e plano de evolução**

Versão 1.8 · 26/07/2026
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

---

## 1. VISÃO GERAL

O projeto é um painel operacional da OLV Distribuidora que roda no navegador (celular ou computador). Ele centraliza três frentes: registro e consulta de vendas, controle de estoque, e (na evolução planejada) o controle financeiro de contas a pagar e a receber.

O objetivo é dar autonomia para lançar e consultar o dia a dia do negócio em um só lugar, com cálculos automáticos de lucro e de saldo de estoque, e sem depender de nenhum aplicativo instalado.

**Estado atual**: existe uma única versão do painel, acessível pela web. A antiga versão desktop (que rodava dentro do Cowork) foi aposentada, e agora tudo é mantido em um só lugar. Desde 25/07/2026, o acesso passou a ser por login individual com papéis (administrador e vendedor), substituindo a chave única anterior.

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

Sete tabelas criadas em 25/07/2026, com a tranca de segurança (RLS, proteção por linha) ativada:

| Tabela | O que guarda | Campos já definidos |
|--------|--------------|---------------------|
| clientes | Cadastro de clientes | nome, endereço, bairro, telefone, ID próprio |
| produtos | Catálogo | nome padronizado, custo atual, estoque mínimo |
| vendas | Cada venda | ligada ao cliente e ao produto pelo ID; guarda o responsável (quem lançou) |
| pagamentos | Formas de pagamento da venda | ligada à venda; permite mais de uma forma por venda |
| estoque_movimentacoes | Movimentações | entrada, ajuste e estoque inicial |
| contas_pagar | Base do financeiro | fornecedor, descrição, valor, vencimento, status, categoria |
| contas_receber | Base do financeiro | cliente, valor, vencimento, status |

**Anexos**: boletos e comprovantes ficam no Supabase Storage, ligados à conta correspondente.

**Pendente**: a especificação completa de colunas, tipos, chaves e índices de cada tabela ainda não foi escrita. É a próxima tarefa da etapa 9.1 e deve ser detalhada aqui quando concluída.

---

## 4. OS WORKFLOWS N8N

O painel é sustentado por cinco workflows ativos na instância n8n-wmtt.srv1830312.hstgr.cloud. São todos necessários:

| Workflow | O que faz | Endpoint |
|----------|-----------|----------|
| OLV Painel Mobile (web) | Serve a página HTML do painel e o endpoint de leitura de dados (vendas + estoque + clientes). | /webhook/olv-painel e /webhook/olv-dados |
| OLV Vendas – Lançamento (painel) | Cria, edita e exclui vendas. Calcula Custo Total, Lucro e Mês. | POST /webhook/olv-venda |
| OLV Estoque – Lançamento (painel) | Cria, edita e exclui lançamentos de estoque. | POST /webhook/olv-estoque |
| OLV Login | Recebe usuário e senha, confere na tabela de usuários e devolve o token de sessão (crachá temporário de 12 horas). | POST (webhook do fluxo OLV Login) |
| OLV Contas | Cria usuário, troca senha, lista, muda papel e ativa/desativa acessos. Ações de administração exigem papel de administrador. | POST (webhook do fluxo OLV Contas) |

**Pendente**: os fluxos de vendas e estoque ainda precisam ser religados para gravar no Supabase (ver seção 9.1).

O workflow "OLV Atendimento - Agente WhatsApp" pertence a outro projeto e não faz parte do painel.

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
- Área de Usuários (só administrador): listar, criar usuário, tornar administrador ou vendedor e ativar/desativar acessos.
- Responsável automático: o campo "quem lançou" saiu do formulário; o sistema preenche sozinho com o usuário logado.
- Trava de dono: o vendedor só vê Editar e Excluir nas próprias linhas, e o servidor confere o dono antes de gravar.

---

## 6. REGRAS DE CÁLCULO

### Estoque atual

Estoque atual = último Estoque Inicial + entradas ± ajustes − vendas, considerando apenas as datas iguais ou posteriores à data da contagem inicial. Vendas sem quantidade preenchida não abatem estoque.

### Lucro

Lucro = Valor Total − (Quantidade × Custo Unitário). O custo unitário é informado no momento da venda e o lucro é gravado como valor numérico fixo.

---

## 7. ACESSO E SEGURANÇA ATUAL

Acesso pela URL https://n8n-wmtt.srv1830312.hstgr.cloud/webhook/olv-painel.

O acesso é por login individual: cada pessoa tem usuário e senha próprios e um papel. A chave única OLV2026 foi removida.

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

- Cada venda e cada lançamento é identificado por um ID próprio do banco, estável e independente da ordem em que aparecem na tela.
- Os nomes de produto precisam bater exatamente (a comparação é sem diferenciar maiúsculas/minúsculas); por isso o produto é escolhido em menu suspenso.
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

**Status**: em andamento (iniciado em 25/07/2026). Construir a base de dados do sistema no Supabase (PostgreSQL), do zero, sem trazer histórico de vendas.

#### Andamento (concluído em 25/07/2026)

- Banco de dados criado no Supabase (projeto olv-distribuidora, região São Paulo).
- As 7 tabelas no ar, ainda vazias: clientes, produtos, vendas, pagamentos, estoque_movimentacoes, contas_pagar e contas_receber.
- Tranca de segurança (RLS) ativada em todas as tabelas.
- n8n ligado ao Supabase pela conexão Session Pooler (IPv4), credencial "Supabase OLV". Teste de consulta de ponta a ponta feito com sucesso.

#### Falta

- Especificar as colunas, tipos, chaves e índices de cada tabela (detalhar na seção 3).
- Importar os clientes (Google Contatos), com limpeza e padronização de nomes antes de subir.
- Religar os fluxos de vendas e estoque para gravar no Supabase.

#### O que entra na abertura

- **Clientes**: importados do Google Contatos (nome e endereço já salvos), com limpeza e padronização antes de subir.
- **Usuários de login**: recriados os acessos que já existem (Rogério e Gabriele como administradores e o Vendedor).
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
- Tela inicial com resumo do dia (vendas de hoje, a receber vencendo, estoque baixo) como primeira coisa que aparece.
- Transformar o painel em atalho na tela inicial do celular (PWA), abrindo como se fosse um aplicativo, inclusive com ícone próprio.
- Alertas visuais: estoque mínimo por produto e contas vencendo.

#### Estrutura de seções definida (25/07/2026)

O painel passa a ter seis seções: Dashboard (métricas), Vendas, Estoque, Clientes, Contas a Pagar e Contas a Receber.

#### Cuidados

- Performance: manter o carregamento leve conforme o volume cresce (paginação, carregar só o necessário).
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

---

## 10. ORDEM SUGERIDA E DECISÕES EM ABERTO

**Sequência recomendada** (flexível, você decide):

| Ordem | Etapa | Por quê nessa posição |
|-------|-------|----------------------|
| 1º | Login multiusuário (concluído e no ar) | Base de segurança e de rastreio; pré-requisito para liberar o financeiro e o caixa com responsabilidade. |
| 2º | Base de dados no Supabase (do zero), banco criado 25/07/2026 | Fundação nova; sustenta as funções novas (pagamento múltiplo, histórico do cliente, financeiro integrado) com backup automático. Sem migração histórica. |
| 3º | Contas a pagar e receber | Alto valor de negócio; usa dados que o próprio sistema já gera (vendas em aberto e crediário). |
| 4º | Controle de caixa | Fecha o ciclo financeiro do dia; usa vendas em dinheiro e pagamentos já registrados; precisa do operador (login). |
| 5º | Novo formato do painel | Polimento visual e de navegação; absorve as novas seções (financeiro e caixa) já prontas. |
| 6º | Módulo fiscal via API | Visão de futuro; o certificado já existe. Depende de gerar o CSC na SEFAZ-ES, do cadastro fiscal dos produtos (ST no gás) e da integração com o provedor (NFe.io). |

**Decisões a fechar quando chegarmos em cada etapa:**

- Login: opção A ou B; quais pessoas e (se A) quais papéis. **RESOLVIDO: Opção A, Rogério e Gabriele administradores, Vendedor vendedor.**
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
| Autenticação | Login por usuário e senha, com papéis (administrador e vendedor). Token de sessão de 12h. Chave única OLV2026 removida. |
| Instância n8n | n8n-wmtt.srv1830312.hstgr.cloud |
| Workflow do painel | OLV Painel Mobile (web) |
| Workflow de vendas | OLV Vendas – Lançamento (painel), endpoint /webhook/olv-venda |
| Workflow de estoque | OLV Estoque – Lançamento (painel), endpoint /webhook/olv-estoque |
| Workflow de login | OLV Login (autenticação; devolve o token de sessão) |
| Workflow de contas | OLV Contas (gestão de usuários; só administrador) |
| Tabelas internas do n8n | OLV Usuarios (usuários e papéis) e OLV Painel HTML (HTML do painel em base64) |
| Pontos de restauração (v1.4) | Painel 63bf15bb; Vendas 348ed17a; Estoque 2adfcba0 |
| Banco de dados | Supabase (PostgreSQL), projeto olv-distribuidora, região São Paulo. 7 tabelas criadas em 25/07/2026, RLS ativa. Sem migração histórica. |
| Domínio do painel | olvdistribuidora.com.br (Registro.br). Endereço planejado: painel.olvdistribuidora.com.br. DNS pendente. |
| Fonte de clientes | Google Contatos (nome e endereço), com limpeza antes de importar. |
| Conexão n8n → Supabase | Session Pooler (IPv4). Host aws-0-sa-east-1.pooler.supabase.com, porta 5432, base postgres, usuário postgres.ggvfrnympdrqyqxgcyex, SSL ativo. Credencial no n8n: "Supabase OLV". (A senha fica guardada só no n8n.) |

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
| Documento | Merge, revisão cruzada, mudança de versão | Opus 5 | Documento longo com regras próprias |
| Documento | Ajuste de texto e tabela | Sonnet 5 | Edição pontual |

**Regra fixa**: nada do módulo fiscal vai para produção sem validação do contador, independentemente do modelo usado.

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
- **Pendência**: dividir este documento em um núcleo enxuto (estado atual, arquitetura, decisões, referências) mais anexos por etapa, carregados só quando a etapa estiver em execução. É a economia estrutural, maior que a troca de modelo. Sugerido para depois da virada do Supabase.

---

## LOG DE MUDANÇAS

| Data | Versão | Mudança |
|------|--------|---------|
| 25/07/2026 | 1.6 | Documento criado em .md; estrutura de atualização definida; login multiusuário implementado; banco Supabase criado; decisão tomada de construção do zero sem migração de histórico. |
| 26/07/2026 | 1.7 | Criada a seção 12, Guia de Modelos por Etapa, com regra de acionamento, modelo indicado por tarefa e protocolo de aviso no início de cada tarefa. Incluída a regra 6 nas Instruções para Claude. |
| 26/07/2026 | 1.8 | Removidas todas as referências à base anterior em planilha, conforme a decisão de sistema independente. A antiga seção 3 (estrutura da planilha) foi substituída pela seção 3, Estrutura de Dados (Supabase). Ajustadas as seções 1, 2, 2.1, 4, 5.1, 6, 8, 9.1, 9.4, 10, 11 e 12.3. |

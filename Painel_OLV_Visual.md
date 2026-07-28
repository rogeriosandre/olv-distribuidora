# OLV DISTRIBUIDORA
## Painel OLV Visual

**Documentação visual, funcional e técnica para construção do novo painel**

Versão 1.0 · 28/07/2026  
Preparado para Rogério

---

## 1. OBJETIVO

Este documento registra o padrão visual aprovado e a estrutura necessária para construir o novo Painel OLV.

O painel será uma aplicação web responsiva, usada em celular e computador, com o Dashboard como tela inicial e navegação por menu lateral. A solução deve centralizar:

- Dashboard operacional e financeiro.
- Vendas e acompanhamento de pedidos.
- Estoque.
- Clientes.
- Contas a pagar.
- Contas a receber.
- Fluxo de caixa.
- Usuários e permissões.

O módulo financeiro seguirá o padrão visual da simulação aprovada em 28/07/2026: interface limpa, indicadores resumidos, tabelas objetivas, filtros simples e ações rápidas.

---

## 2. DECISÕES VISUAIS APROVADAS

### 2.1 Estrutura geral

- Dashboard será a tela inicial após o login.
- Navegação principal em menu lateral.
- Cabeçalho superior com o nome da tela, alertas e botão de novo registro.
- Conteúdo dividido em indicadores, visualização principal e tabela ou lista complementar.
- A mesma identidade visual será usada em todas as seções.
- O sistema deve funcionar bem em computador e celular.

### 2.2 Organização do menu

O menu lateral será dividido em três grupos:

#### Operação

1. Dashboard
2. Vendas
3. Estoque
4. Clientes

#### Financeiro

5. Contas a Pagar
6. Contas a Receber
7. Fluxo de Caixa

#### Administração

8. Usuários

### 2.3 Comportamento do menu

No computador:

- Menu lateral expandido.
- Ícone e nome de cada seção visíveis.
- Usuário logado e papel exibidos na parte inferior.
- Item selecionado recebe destaque visual.

No celular:

- Menu lateral compacto.
- Somente ícones permanecem visíveis.
- Ao tocar ou manter o foco, o sistema identifica a função do ícone.
- O conteúdo utiliza todo o espaço restante da tela.
- Nenhuma tabela deve ultrapassar a tela sem tratamento responsivo.

---

## 3. IDENTIDADE VISUAL

### 3.1 Cores

Usar a identidade da OLV Distribuidora:

| Uso | Diretriz |
|---|---|
| Cor principal | Azul-marinho da OLV |
| Cor secundária | Azul médio |
| Destaque positivo | Verde |
| Destaque de atenção | Amarelo ou dourado |
| Erro ou vencimento | Vermelho |
| Fundo principal | Claro, com contraste suficiente |
| Fundo do menu | Azul-marinho ou variação próxima |
| Texto principal | Escuro em fundo claro |
| Texto secundário | Cinza com contraste adequado |

As cores devem manter boa leitura em telas pequenas. Nunca depender somente da cor para comunicar um status. Usar também texto e ícone.

### 3.2 Tipografia

- Fonte sem serifa.
- Títulos com peso médio.
- Textos comuns com peso regular.
- Valores financeiros com alinhamento consistente.
- Não usar fontes excessivamente pequenas.
- Informações secundárias devem continuar legíveis no celular.

### 3.3 Componentes visuais

- Cards para indicadores principais.
- Tabelas sem excesso de linhas ou bordas.
- Badges para status.
- Botão principal para a ação mais importante.
- Ícones simples e consistentes.
- Formulários divididos em blocos.
- Campos opcionais dentro de “Mais opções” quando necessário.

---

## 4. LAYOUT BASE

### 4.1 Menu lateral

Largura recomendada no computador: entre 180 e 220 pixels.

Elementos:

- Símbolo ou logotipo compacto da OLV.
- Grupos de navegação.
- Ícones.
- Nome das seções.
- Nome do usuário logado.
- Papel do usuário.
- Ação para sair da conta.

### 4.2 Cabeçalho

Elementos:

- Nome da seção atual.
- Botão de alertas.
- Botão “Novo registro”.
- Ações específicas da tela quando necessário.

### 4.3 Área de conteúdo

Ordem padrão:

1. Título e descrição curta.
2. Filtros.
3. Até três indicadores principais.
4. Gráfico, quadro operacional ou resumo.
5. Tabela ou lista detalhada.

---

## 5. DASHBOARD

### 5.1 Objetivo

Oferecer uma visão rápida da operação e das finanças ao entrar no sistema.

### 5.2 Indicadores principais

O Dashboard deve apresentar, inicialmente:

| Indicador | Conteúdo |
|---|---|
| Vendas de hoje | Quantidade de pedidos e faturamento |
| Saldo disponível | Total realizado nas contas financeiras |
| Pré-pagos pendentes | Valor e quantidade de pedidos pagos ainda não entregues |

### 5.3 Visualização principal

Gráfico com:

- Vendas acumuladas do período.
- Saldo realizado e projetado.
- Alternância entre período diário e mensal.

### 5.4 Alertas

Lista “Atenção hoje”:

- Contas vencidas.
- Contas que vencem hoje.
- Clientes inadimplentes.
- Estoque abaixo do mínimo.
- Pedidos aguardando.
- Pré-pagos antigos ainda não entregues.

### 5.5 Movimentos prioritários

Tabela com:

- Data.
- Descrição.
- Área de origem.
- Valor.
- Situação.

---

## 6. VENDAS

### 6.1 Objetivo

Transformar a seção de vendas em uma tela de operação diária, e não apenas em um histórico.

### 6.2 Quadro de pedidos

Quatro grupos:

1. Aguardando.
2. Pré-pagos.
3. Em rota.
4. Concluídos.

Cada pedido deve mostrar:

- Cliente.
- Produto e quantidade.
- Valor.
- Entrega ou retirada.
- Responsável.
- Horário.
- Forma de pagamento.
- Status de pagamento quando necessário.

### 6.3 Ações

- Criar pedido.
- Alterar status com poucos toques.
- Editar lançamento.
- Excluir, respeitando as permissões.
- Pesquisar cliente.
- Filtrar por produto, status, responsável e data.

### 6.4 Regras

- Qualquer usuário ativo pode mudar o status operacional.
- Somente o responsável pelo lançamento e o administrador podem alterar valores ou excluir.
- Retirada não pode entrar no status “Em rota”.
- Pedido pré-pago não pode movimentar o caixa novamente na entrega.
- Estoque deve baixar somente quando o pedido for concluído.

---

## 7. ESTOQUE

### 7.1 Indicadores

- Quantidade atual de gás P13.
- Quantidade atual de água 20L.
- Quantidade de produtos abaixo do mínimo.

### 7.2 Tabela

Colunas:

- Produto.
- Estoque atual.
- Estoque mínimo.
- Última entrada.
- Situação.

### 7.3 Movimentações

Tipos:

- Entrada.
- Ajuste.
- Estoque inicial.
- Saída automática por venda concluída.

### 7.4 Alertas

- Normal.
- Próximo do mínimo.
- Estoque baixo.
- Sem estoque.

---

## 8. CLIENTES

### 8.1 Indicadores

- Total de clientes ativos.
- Clientes que compraram no período.
- Clientes com crediário.
- Clientes inadimplentes.

### 8.2 Tabela

Colunas:

- Nome.
- Telefone.
- Bairro.
- Último pedido.
- Perfil.

### 8.3 Perfil do cliente

Ao abrir um cliente, mostrar:

- Dados cadastrais.
- Endereço.
- Telefone.
- Observação.
- Histórico de pedidos.
- Total comprado.
- Contas em aberto.
- Perfil: consumidor, recorrente, revenda ou crediário.

### 8.4 Duplicidade

- Telefone deve ser normalizado para somente números.
- O sistema deve impedir dois clientes com o mesmo telefone normalizado.
- Cliente sem telefone pode ser cadastrado, mas o painel deve alertar sobre nomes semelhantes.

---

## 9. CONTAS A PAGAR

### 9.1 Indicadores

- Total em aberto.
- Total vencido.
- Total pago no mês.

### 9.2 Abas ou filtros de situação

- Em aberto.
- Vencendo hoje.
- Vencidas.
- Pagas.

### 9.3 Campos essenciais

| Campo | Obrigatório |
|---|---|
| Fornecedor | Sim |
| Descrição | Sim |
| Categoria | Sim |
| Valor | Sim |
| Data de competência | Sim |
| Vencimento | Sim |
| Status | Sim |
| Forma de pagamento | Não |
| Código de referência | Não |
| Recorrência | Não |
| Parcelamento | Não |
| Observação | Não |
| Anexo | Não |

### 9.4 Ações

- Cadastrar conta.
- Editar.
- Marcar como paga.
- Reabrir pagamento.
- Anexar boleto ou comprovante.
- Criar lançamento recorrente.
- Dividir em parcelas.
- Filtrar por fornecedor, categoria, período e status.

---

## 10. CONTAS A RECEBER

### 10.1 Indicadores

- Total em aberto.
- Total vencido.
- Total recebido no mês.

### 10.2 Abas ou filtros de situação

- Em aberto.
- Vencendo hoje.
- Vencidas.
- Recebidas.

### 10.3 Campos essenciais

| Campo | Obrigatório |
|---|---|
| Cliente | Sim |
| Descrição | Sim |
| Valor | Sim |
| Data de competência | Sim |
| Vencimento | Sim |
| Status | Sim |
| Venda relacionada | Não |
| Forma de recebimento | Não |
| Parcelamento | Não |
| Observação | Não |
| Anexo | Não |

### 10.4 Origem automática

Criar conta a receber automaticamente quando uma venda for:

- Crediário.
- Em aberto.
- Parcialmente recebida.

Não criar conta a receber para:

- Dinheiro já recebido.
- Pix já recebido.
- Cartão já confirmado, quando tratado como pagamento realizado.
- Pedido pré-pago.

---

## 11. FLUXO DE CAIXA

### 11.1 Objetivo

Mostrar as entradas e saídas realizadas e projetadas, sem exigir lançamento manual duplicado.

### 11.2 Indicadores

- Saldo inicial.
- Entradas realizadas.
- Saídas realizadas.
- Saldo atual.
- Total a receber.
- Total a pagar.
- Saldo projetado.

### 11.3 Filtros

- Hoje.
- Próximos 7 dias.
- Mês atual.
- Próximos 30 dias.
- Período personalizado.
- Realizado.
- Projetado.
- Realizado e projetado.
- Conta financeira.
- Categoria.

### 11.4 Visões

#### Diária

- Saldo inicial.
- Entradas do dia.
- Saídas do dia.
- Saldo final.
- Projeção.

#### Mensal

- Entradas.
- Saídas.
- Resultado.
- Saldo acumulado.
- Comparação com o mês anterior.

### 11.5 Regras

- Fluxo de caixa usa a data efetiva do pagamento ou recebimento.
- Visão de competência usa o mês ao qual a conta pertence.
- Transferência entre contas não é receita nem despesa.
- Um mesmo evento financeiro não pode aparecer duas vezes.
- Recebimento de crediário deve gerar uma única entrada.
- Baixa de uma conta a pagar deve gerar uma única saída.
- Entrega de pedido pré-pago não movimenta o caixa novamente.

---

## 12. USUÁRIOS E PERMISSÕES

### 12.1 Papéis

#### Administrador

- Visualiza todos os dados.
- Visualiza faturamento, lucro e margem.
- Cria e gerencia usuários.
- Edita e exclui qualquer lançamento.
- Acessa os módulos financeiros.

#### Colaborador

- Visualiza a operação.
- Lança vendas e estoque.
- Muda status dos pedidos.
- Edita e exclui somente os próprios lançamentos.
- Não visualiza lucro, margem ou informações financeiras restritas.

### 12.2 Tela de usuários

Colunas:

- Nome.
- Usuário.
- Papel.
- Último acesso.
- Situação.

Ações:

- Criar usuário.
- Alterar papel.
- Ativar ou desativar.
- Redefinir senha.
- Consultar último acesso.

---

## 13. COMPONENTES REUTILIZÁVEIS

Construir os seguintes componentes para evitar duplicação:

| Componente | Uso |
|---|---|
| Sidebar | Navegação principal |
| Topbar | Título, alertas e ação principal |
| MetricCard | Indicadores |
| StatusBadge | Situação de pedidos, contas e estoque |
| DataTable | Listagens |
| FilterBar | Período, status, busca e categorias |
| FormDrawer ou FormPage | Cadastro e edição |
| ConfirmDialog | Exclusões e ações críticas |
| EmptyState | Tela sem registros |
| AlertList | Pendências do Dashboard |
| CashFlowChart | Realizado e projetado |
| OrderBoard | Quadro de vendas |
| AttachmentField | Boletos e comprovantes |

---

## 14. ESTRUTURA DE DADOS

### 14.1 Tabelas atuais do Supabase

- clientes
- produtos
- vendas
- pagamentos
- estoque_movimentacoes
- contas_pagar
- contas_receber

### 14.2 Campos adicionais em `contas_pagar`

- `data_competencia`
- `forma_pagamento`
- `codigo_referencia`
- `recorrente`
- `recorrencia_id`
- `numero_parcela`
- `total_parcelas`
- `responsavel`
- `observacao`
- `conta_financeira_id`

### 14.3 Campos adicionais em `contas_receber`

- `data_competencia`
- `forma_recebimento`
- `numero_parcela`
- `total_parcelas`
- `responsavel`
- `observacao`
- `conta_financeira_id`

### 14.4 Nova tabela `contas_financeiras`

| Campo | Finalidade |
|---|---|
| id | Identificador |
| nome | Caixa físico, banco, cartão etc. |
| tipo | Caixa, banco, cartão ou outro |
| saldo_inicial | Valor inicial |
| data_saldo_inicial | Data de referência |
| ativo | Permite desativar sem apagar histórico |
| criado_em | Auditoria |

Contas iniciais sugeridas:

- Caixa físico.
- Conta bancária principal.
- Cartões a receber.

### 14.5 Nova tabela `movimentacoes_financeiras`

| Campo | Finalidade |
|---|---|
| id | Identificador |
| tipo | Entrada, Saída ou Transferência |
| origem | Venda, Conta a Pagar, Conta a Receber, Caixa ou Manual |
| origem_id | Registro que gerou a movimentação |
| conta_financeira_id | Conta afetada |
| categoria | Classificação financeira |
| descricao | Identificação |
| valor | Valor movimentado |
| data_movimento | Data efetiva |
| status | Previsto ou Realizado |
| responsavel | Usuário da ação |
| criado_em | Auditoria |

Criar uma trava de unicidade baseada na origem para impedir duplicidade de movimentações.

### 14.6 Nova tabela `transferencias_financeiras`

Campos:

- `id`
- `conta_origem_id`
- `conta_destino_id`
- `valor`
- `data`
- `descricao`
- `responsavel`
- `criado_em`

Uma transferência deve gerar:

- Saída na conta de origem.
- Entrada na conta de destino.
- Impacto total igual a zero no fluxo consolidado.

---

## 15. WORKFLOWS N8N

### 15.1 Workflows existentes

- OLV Painel Mobile (web).
- OLV Vendas – Lançamento (painel).
- OLV Estoque – Lançamento (painel).
- OLV Login.
- OLV Contas.

### 15.2 Endpoints novos propostos

Os nomes abaixo são propostas e ainda precisam ser implementados:

| Endpoint | Função |
|---|---|
| `/webhook/olv-dashboard` | Indicadores e alertas |
| `/webhook/olv-clientes` | Consultar e manter clientes |
| `/webhook/olv-conta-pagar` | Criar, editar, pagar e reabrir despesas |
| `/webhook/olv-conta-receber` | Criar, editar, receber e reabrir recebíveis |
| `/webhook/olv-fluxo-caixa` | Consultar realizado e projetado |
| `/webhook/olv-contas-financeiras` | Manter contas financeiras |
| `/webhook/olv-transferencia` | Transferir valores entre contas |
| `/webhook/olv-alertas` | Consultar pendências operacionais e financeiras |

### 15.3 Regras comuns

Todo endpoint deve:

- Validar token de sessão.
- Validar papel do usuário.
- Registrar responsável.
- Validar os campos obrigatórios.
- Rejeitar valores inválidos.
- Evitar duplicidade.
- Retornar mensagens claras.
- Registrar erros para manutenção.

---

## 16. ESTRUTURA DO FRONT-END

Estrutura lógica sugerida:

```text
Painel OLV
├── Layout
│   ├── Sidebar
│   ├── Topbar
│   └── Área de conteúdo
├── Dashboard
├── Vendas
│   ├── Quadro de pedidos
│   ├── Formulário
│   └── Histórico
├── Estoque
│   ├── Posição atual
│   └── Movimentações
├── Clientes
│   ├── Lista
│   └── Perfil do cliente
├── Financeiro
│   ├── Contas a Pagar
│   ├── Contas a Receber
│   ├── Fluxo de Caixa
│   └── Contas Financeiras
└── Administração
    └── Usuários
```

O HTML atualmente servido pelo n8n pode continuar como base, mas a manutenção será mais segura se o código for dividido internamente em:

- Configurações.
- Estado da aplicação.
- Chamadas de API.
- Componentes.
- Regras de permissão.
- Renderização de cada tela.
- Estilos responsivos.

---

## 17. RESPONSIVIDADE

### 17.1 Computador

- Menu lateral expandido.
- Indicadores em três colunas.
- Gráfico e lista lado a lado.
- Tabelas completas.
- Quadro de pedidos em quatro colunas.

### 17.2 Celular

- Menu lateral compacto com ícones.
- Indicadores empilhados.
- Gráfico acima da lista.
- Tabelas com colunas essenciais.
- Detalhes abertos ao tocar no registro.
- Quadro de pedidos em uma ou duas colunas.
- Botão de novo registro sempre acessível.

### 17.3 Colunas prioritárias no celular

#### Vendas

- Cliente.
- Produto.
- Valor.
- Status.

#### Estoque

- Produto.
- Atual.
- Situação.

#### Contas

- Cliente ou fornecedor.
- Vencimento.
- Valor.
- Status.

---

## 18. SEGURANÇA E AUDITORIA

- Nunca guardar senha no HTML.
- Validar todas as permissões no servidor.
- Token com validade de 12 horas.
- Registrar usuário responsável por ações financeiras.
- Não permitir exclusão definitiva de registros financeiros já realizados sem regra de estorno.
- Alterações de pagamento devem manter histórico.
- Anexos devem ficar no Supabase Storage.
- Chaves de serviço devem permanecer somente no n8n.
- O navegador nunca deve acessar o Supabase com credencial privilegiada.

---

## 19. ORDEM DE IMPLEMENTAÇÃO

### Etapa 1: Estrutura visual

- Criar layout com menu lateral.
- Criar cabeçalho.
- Criar navegação entre telas.
- Implementar responsividade.
- Definir componentes reutilizáveis.

### Etapa 2: Operação atual

- Conectar Dashboard.
- Adaptar Vendas ao quadro operacional.
- Adaptar Estoque.
- Criar seção Clientes.
- Manter login e permissões funcionando.

### Etapa 3: Fundação financeira

- Criar contas financeiras.
- Atualizar tabelas de pagar e receber.
- Criar movimentações financeiras.
- Implementar regra contra duplicidade.
- Implementar transferências.

### Etapa 4: Contas a Pagar

- Cadastro.
- Edição.
- Baixa.
- Reabertura.
- Parcelamento.
- Recorrência.
- Anexos.

### Etapa 5: Contas a Receber

- Cadastro manual.
- Geração automática por crediário.
- Recebimento total ou parcial.
- Inadimplência.
- Histórico por cliente.

### Etapa 6: Fluxo de Caixa

- Realizado.
- Projetado.
- Visão diária.
- Visão mensal.
- Filtro por conta.
- Saldo projetado.

### Etapa 7: Testes e publicação

- Testar no computador.
- Testar em diferentes celulares.
- Testar permissões.
- Testar valores e cálculos.
- Testar registros duplicados.
- Criar ponto de restauração.
- Publicar após aprovação.

---

## 20. CRITÉRIOS DE ACEITAÇÃO

### Layout

- Dashboard abre após o login.
- Menu lateral funciona em computador e celular.
- Item ativo é claramente identificado.
- Nenhuma informação importante fica cortada.

### Vendas

- Pedido entra na coluna correta.
- Mudança de status funciona.
- Retirada não entra em rota.
- Pré-pago não duplica movimentação financeira.
- Estoque baixa na conclusão.

### Estoque

- Estoque atual é calculado corretamente.
- Produtos abaixo do mínimo aparecem em alerta.
- Ajuste negativo funciona.
- Estoque inicial redefine a contagem.

### Contas a Pagar

- Conta vencida é destacada.
- Pagamento gera uma saída.
- Reabrir pagamento remove ou reverte a saída corretamente.
- Parcelas têm vencimentos e valores consistentes.

### Contas a Receber

- Crediário cria recebível.
- Pix ou dinheiro já recebido não cria recebível.
- Recebimento gera uma única entrada.
- Cliente inadimplente aparece em alerta.

### Fluxo de Caixa

- Saldo inicial mais entradas menos saídas resulta no saldo final.
- Valores previstos não são misturados com realizados.
- Transferências têm impacto consolidado igual a zero.
- Nenhum evento é contado duas vezes.

### Permissões

- Colaborador não visualiza informações restritas.
- Colaborador não exclui registros de outro usuário.
- Administrador gerencia usuários e acessa todos os dados.

---

## 21. DECISÕES AINDA PENDENTES

- Confirmar a largura final do menu lateral.
- Confirmar se haverá modo escuro.
- Confirmar se o menu compacto ficará sempre lateral no celular ou poderá abrir sobre o conteúdo.
- Definir categorias financeiras iniciais.
- Definir se o caixa será único ou separado por operador.
- Definir tratamento de recebíveis de cartão: realizado na venda ou previsto até o crédito bancário.
- Definir regras de estorno de pagamentos e recebimentos.
- Definir limites para anexos.
- Definir quais alertas serão enviados pelo Telegram.

---

## 22. CHECKLIST ANTES DE PROGRAMAR

- [ ] Validar esta documentação.
- [ ] Fechar as decisões pendentes necessárias para a primeira etapa.
- [ ] Criar ponto de restauração dos workflows atuais.
- [ ] Confirmar o esquema final do Supabase.
- [ ] Definir os contratos JSON dos endpoints.
- [ ] Separar dados administrativos e operacionais por papel.
- [ ] Definir dados fictícios para testes.
- [ ] Implementar primeiro em ambiente de teste.
- [ ] Validar no celular antes de publicar.

---

## 23. RESULTADO ESPERADO

O novo Painel OLV deve permitir que Rogério e Gabriele entendam a situação da operação em poucos segundos e executem as tarefas diárias sem alternar entre planilhas, sistemas ou telas desconectadas.

O sistema deve responder claramente:

- Quanto foi vendido hoje?
- Quais pedidos ainda precisam ser entregues?
- Qual é o estoque disponível?
- Quais contas vencem?
- Quem está devendo?
- Quanto dinheiro está disponível?
- O caixa será suficiente nos próximos dias?
- Quem fez cada lançamento?

O visual aprovado deve ser mantido como padrão para toda a evolução do projeto.


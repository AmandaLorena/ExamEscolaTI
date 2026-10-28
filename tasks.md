# Tasks — API da Pizzaria

## 1. Objetivo

Decompor a implementação da API da pizzaria em tarefas verificáveis, organizadas por dependência e prioridade.

As tarefas devem ser executadas respeitando `constitution.md`, `spec.md`, `tests.md`, `plan.md`, `ENUNCIADO.md`, `contrato.json` e `variante/params.json`.

Uma tarefa só deve ser considerada concluída quando sua implementação estiver integrada, seus critérios de conclusão forem atendidos e os testes aplicáveis forem executados.

## 2. Fase 1 — Estrutura e persistência

### TASK-01 — Preparar o projeto e a inicialização

**Objetivo:** criar a estrutura mínima para executar a API.

Atividades:
- Configurar Python e FastAPI.
- Criar o arquivo de dependências.
- Configurar a aplicação para escutar na porta definida por `PORTA_SERVICO`.
- Preparar os arquivos necessários para execução em container.
- Documentar como iniciar o serviço e executar os testes.

Critérios de conclusão:
- O serviço inicia sem erros.
- A API fica acessível em `http://localhost:8002`.
- As dependências e as instruções de execução estão declaradas.

### TASK-02 — Implementar persistência e modelo de dados

**Objetivo:** armazenar os dados necessários para os casos de uso.

Atividades:
- Configurar SQLite e inicialização automática do banco.
- Modelar pedidos e itens.
- Modelar mesas e comandas.
- Modelar sabores e disponibilidade do cardápio.
- Armazenar os dados necessários para histórico, relatórios e fidelidade.
- Definir os relacionamentos e as restrições de integridade.

Critérios de conclusão:
- As entidades necessárias podem ser persistidas e consultadas.
- Os dados permanecem consistentes entre requisições durante a execução.
- Operações relacionadas podem ser confirmadas ou revertidas em conjunto quando necessário.

## 3. Fase 2 — Pedidos e regras centrais

### TASK-03 — Implementar validação de pedidos e cálculos monetários

**Objetivo:** centralizar as regras de criação e cálculo dos pedidos.

Atividades:
- Validar cliente, itens, sabor, tamanho, quantidade, tipo e `recebido_em`.
- Validar a presença e a situação da comanda para pedidos de mesa.
- Calcular subtotal, desconto promocional, taxa de entrega e total.
- Utilizar parâmetros de `variante/params.json`.
- Implementar cálculos com centavos inteiros e arredondamento correto.
- Padronizar os erros e os códigos HTTP previstos no contrato.

Critérios de conclusão:
- Pedidos válidos produzem os valores esperados.
- Entradas inválidas retornam os erros contratuais.
- Os cálculos não utilizam valores monetários em ponto flutuante.

### TASK-04 — Implementar criação e consulta de pedidos

**Objetivo:** disponibilizar as operações básicas de pedidos.

Atividades:
- Implementar `POST /pedidos`.
- Implementar `GET /pedidos/ativos`.
- Implementar `GET /pedidos?cliente={nome}`.
- Aplicar a ordenação exigida nas listagens.
- Garantir que os campos das respostas correspondam ao contrato.

Critérios de conclusão:
- A criação retorna `201` para pedidos válidos.
- As consultas retornam os registros corretos e na ordem especificada.
- Consultas sem resultados retornam listas vazias quando previsto.

### TASK-05 — Implementar ciclo de vida e cancelamento

**Objetivo:** controlar a evolução e o cancelamento dos pedidos.

Atividades:
- Implementar `POST /pedidos/{id}/status`.
- Implementar `POST /pedidos/{id}/cancelamento`.
- Restringir as transições à sequência permitida.
- Permitir cancelamento somente no estado `recebido`.
- Atualizar a situação do pedido sem deixar dados inconsistentes.

Critérios de conclusão:
- A sequência `recebido → preparo → pronto → entregue` funciona corretamente.
- O estado `entregue` é terminal.
- Cancelamentos inválidos retornam os erros previstos.
- Pedidos cancelados deixam de ser considerados ativos.

### TASK-06 — Implementar restrição de pedidos ativos por cliente

**Objetivo:** impedir pedidos simultâneos incompatíveis com a regra de negócio.

Atividades:
- Verificar pedidos ativos do cliente durante a criação.
- Aplicar a restrição a pedidos de entrega e balcão.
- Excluir pedidos de mesa dessa restrição.
- Liberar o cliente após entrega ou cancelamento permitido.
- Respeitar a precedência das validações.

Critérios de conclusão:
- Um cliente com pedido ativo de entrega ou balcão recebe `409` com `pedido_em_aberto` ao tentar criar outro pedido desses tipos.
- Pedidos de mesa não são bloqueados por essa regra.
- Requisições rejeitadas não criam pedidos nem consomem benefícios de fidelidade.

## 4. Fase 3 — Cardápio e fidelidade

### TASK-07 — Implementar cardápio e disponibilidade

**Objetivo:** permitir consultar e alterar a disponibilidade dos sabores.

Atividades:
- Implementar `GET /cardapio`.
- Implementar `PUT /cardapio/{sabor}`.
- Registrar sabores conhecidos por pedidos ou atualização explícita.
- Normalizar a comparação sem distinção entre maiúsculas e minúsculas.
- Preservar a capitalização do primeiro registro.
- Impedir pedidos que contenham sabores indisponíveis.

Critérios de conclusão:
- O cardápio apresenta os sabores conhecidos em ordem alfabética.
- A atualização de disponibilidade retorna a representação exigida pelo contrato.
- Um pedido com sabor indisponível retorna `409` com `sabor_indisponivel`.
- Nenhum item é gravado parcialmente quando o pedido é rejeitado.

### TASK-08 — Implementar fidelidade e resgate de benefícios

**Objetivo:** conceder e controlar descontos com base nos pedidos entregues.

Atividades:
- Contabilizar pedidos entregues por cliente, independentemente do tipo.
- Excluir pedidos cancelados da contagem.
- Conceder o benefício a cada cinco pedidos entregues.
- Aplicar o desconto no pedido premiado durante a criação.
- Impedir o uso duplicado do benefício.
- Devolver o benefício quando o pedido premiado for cancelado em `recebido`.
- Garantir que pedidos de mesa contem para a fidelidade e que o desconto não seja reaplicado no fechamento.

Critérios de conclusão:
- O sexto pedido após cinco entregues recebe o benefício.
- O campo `desconto_fidelidade_centavos` aparece com o valor correto na resposta.
- O cancelamento permitido devolve o resgate.
- O total nunca fica negativo.
- A contagem e o resgate permanecem consistentes entre pedidos de diferentes tipos.

## 5. Fase 4 — Mesas e comandas

### TASK-09 — Implementar abertura e consulta de comandas

**Objetivo:** gerenciar a ocupação das mesas e o histórico de comandas.

Atividades:
- Implementar `POST /mesas/{numero}/comandas`.
- Implementar `GET /comandas/abertas`.
- Implementar `GET /mesas/{numero}/comandas`.
- Validar números de mesa entre 1 e 50.
- Impedir duas comandas abertas simultaneamente para a mesma mesa.
- Aplicar a ordenação correta nas listagens.

Critérios de conclusão:
- A abertura válida retorna `201`.
- Mesa ocupada retorna `409` com `mesa_ocupada`.
- Número de mesa inválido retorna `422` com `mesa_invalida`.
- As listagens incluem os registros corretos e respeitam a ordenação contratual.

### TASK-10 — Implementar pedidos vinculados a comandas

**Objetivo:** integrar pedidos de mesa ao fluxo de atendimento.

Atividades:
- Exigir comanda válida e aberta para pedidos do tipo `mesa`.
- Associar cada pedido à comanda correspondente.
- Permitir vários pedidos na mesma comanda.
- Manter o ciclo de vida independente de cada pedido.
- Garantir que apenas pedidos entregues e não cancelados componham a conta.

Critérios de conclusão:
- Pedidos de mesa são associados corretamente à comanda.
- Pedidos vinculados preservam seus próprios estados e valores.
- A comanda não inclui pedidos cancelados no subtotal do fechamento.

### TASK-11 — Implementar fechamento, gorjeta e divisão da conta

**Objetivo:** calcular e registrar o fechamento de uma comanda.

Atividades:
- Implementar `POST /comandas/{id}/fechamento`.
- Validar existência e situação da comanda.
- Impedir fechamento enquanto houver pedidos ativos.
- Calcular subtotal, gorjeta e total.
- Aplicar o percentual padrão quando a gorjeta estiver ausente.
- Validar `gorjeta_percentual` e `dividir_em`.
- Gerar `parcelas_centavos` quando solicitado.
- Registrar o fechamento e liberar a mesa.

Critérios de conclusão:
- O fechamento soma os pedidos entregues e não cancelados.
- Gorjeta e total são calculados com arredondamento correto.
- Fechamentos inválidos não alteram o estado da comanda.
- A divisão produz a quantidade solicitada de parcelas, com soma exata do total e diferença máxima de um centavo.
- O fechamento bem-sucedido permite abrir outra comanda para a mesa.

## 6. Fase 5 — Relatório e consistência temporal

### TASK-12 — Implementar relatório diário

**Objetivo:** calcular os indicadores de vendas por data.

Atividades:
- Implementar `GET /relatorios/diario?data=AAAA-MM-DD`.
- Validar o formato da data.
- Filtrar pedidos por `recebido_em`, respeitando o fuso `-03:00`.
- Excluir pedidos cancelados.
- Calcular quantidade, faturamento e ticket médio.
- Excluir gorjetas do faturamento.
- Tratar corretamente dias sem pedidos.

Critérios de conclusão:
- O relatório apresenta os campos e os valores exigidos pelo contrato.
- Datas inválidas retornam `422` com `data_invalida`.
- Dias sem pedidos retornam `200` com indicadores numéricos zerados.
- O ticket médio utiliza o arredondamento especificado.
- Pedidos de mesa são considerados normalmente.

### TASK-13 — Revisar datas, fuso e consistência dos dados

**Objetivo:** garantir coerência temporal e integridade entre as funcionalidades.

Atividades:
- Revisar a validação de `recebido_em`.
- Garantir que os horários retornados usem ISO 8601 com `-03:00`.
- Verificar a coerência entre criação, histórico e relatório.
- Revisar transações em criação, cancelamento, resgate de fidelidade e fechamento.
- Corrigir efeitos parciais identificados nos testes.

Critérios de conclusão:
- Datas explícitas produzem resultados reproduzíveis.
- As operações mantêm os dados consistentes.
- Requisições rejeitadas não deixam alterações parciais.

## 7. Fase 6 — Testes e entrega

### TASK-14 — Implementar testes automatizados

**Objetivo:** verificar o contrato e os cenários definidos em `tests.md`.

Atividades:
- Criar testes unitários para cálculos e regras de negócio.
- Criar testes de integração para os endpoints.
- Testar códigos HTTP, corpos JSON e ordenação.
- Testar limites, campos ausentes e valores inválidos.
- Testar precedência de erros e ausência de efeitos parciais.
- Testar os fluxos completos de fidelidade, mesas e comandas.

Critérios de conclusão:
- Os cenários aplicáveis de `tests.md` possuem testes automatizados.
- Os testes podem ser executados conforme as instruções do projeto.
- Falhas identificadas são corrigidas antes da entrega.

### TASK-15 — Revisar contrato e cobertura dos 16 casos de uso

**Objetivo:** verificar se a implementação contempla todos os requisitos obrigatórios.

Atividades:
- Comparar os endpoints implementados com `contrato.json`.
- Conferir métodos HTTP, parâmetros, campos, status e erros.
- Revisar todos os 16 casos de uso e seus critérios de aceitação.
- Conferir os parâmetros reais de `variante/params.json`.
- Garantir que nenhum requisito tenha sido omitido por ter parâmetro configurado como zero.

Critérios de conclusão:
- Todos os endpoints obrigatórios estão implementados.
- Todos os casos de uso possuem cobertura correspondente.
- As respostas e os erros respeitam o contrato.

### TASK-16 — Validar inicialização e entrega

**Objetivo:** preparar a solução para execução no ambiente de avaliação.

Atividades:
- Verificar dependências e arquivos de inicialização.
- Confirmar a execução em container.
- Confirmar que o serviço escuta na porta `8002`.
- Executar os testes disponíveis.
- Revisar as instruções de execução.
- Conferir a documentação Markdown e o ponto de entrada `.prova/entrada`.

Critérios de conclusão:
- O serviço inicia e responde em `http://localhost:8002`.
- Os testes disponíveis são executáveis.
- O arquivo indicado em `.prova/entrada` existe e direciona para o README de especificação.
- A documentação permite navegar até todos os arquivos necessários para gerar a implementação.

## 8. Ordem de execução e dependências

As tarefas devem seguir a ordem numérica, respeitando estas dependências principais:

- `TASK-01` precede todas as demais.
- `TASK-02` deve estar pronta antes das funcionalidades que persistem dados.
- `TASK-03` e `TASK-04` são pré-requisitos para o ciclo de vida, fidelidade e relatórios.
- `TASK-05` e `TASK-06` dependem da estrutura de pedidos.
- `TASK-07` e `TASK-08` dependem da criação e persistência de pedidos.
- `TASK-09` precede a integração dos pedidos de mesa.
- `TASK-10` precede o fechamento completo de comandas.
- `TASK-11` deve estar concluída antes da validação completa das contas.
- `TASK-12` depende dos dados e cálculos de pedidos.
- `TASK-13` a `TASK-16` devem consolidar a consistência, a cobertura e a entrega.

A implementação pode agrupar tarefas tecnicamente relacionadas, desde que nenhum requisito seja omitido e todos os critérios de conclusão sejam verificados.

## 9. Definição de concluído

A implementação estará pronta para entrega quando:

- Todos os 16 casos de uso estiverem contemplados.
- Todos os endpoints e respostas respeitarem o contrato.
- Os cálculos monetários, descontos, gorjetas e divisões forem corretos.
- As regras de fidelidade e cancelamento preservarem a consistência.
- Mesas e comandas respeitarem seus estados e relacionamentos.
- Os testes de `tests.md` tiverem sido executados e os problemas encontrados corrigidos.
- O serviço puder ser iniciado no ambiente previsto.
- A documentação estiver completa e acessível a partir do ponto de entrada.

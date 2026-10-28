# Constitution — API da Pizzaria

## 1. Objetivo e escopo

A solução deve implementar uma API REST para gerenciamento de pedidos, preparo, mesas, comandas, cardápio, fechamento de contas, relatórios diários e fidelidade.

O escopo contempla exclusivamente o backend. Não implementar frontend.

O `ENUNCIADO.md` e o `contrato.json` são as referências normativas para endpoints, métodos HTTP, campos, códigos de resposta, validações e regras de negócio. Os parâmetros específicos devem ser lidos de `variante/params.json`.

## 2. Regras monetárias

### 2.1. Representação e preços

- Todo valor monetário deve ser representado e retornado como inteiro em centavos, tanto nos cálculos quanto nas respostas da API. Nunca retornar valores monetários em ponto flutuante.
- O preço unitário da pizza broto é `2750` centavos e o da pizza grande é `5000` centavos.
- A taxa de entrega desta variante é `0` centavos. Ela só se aplica a pedidos do tipo `entrega`; pedidos de `balcao` e `mesa` sempre possuem taxa zero.
- O desconto promocional desta variante é `0%`, mas sua regra geral deve continuar implementada: `desconto_centavos = ceil(subtotal_pizzas_centavos × DESCONTO_PROMO_PERCENTUAL / 100)`.
- O desconto promocional incide exclusivamente sobre as pizzas, nunca sobre a taxa de entrega.
- O desconto de fidelidade corresponde a `PRECO_BROTO_CENTAVOS` no pedido premiado e a zero nos demais.

### 2.2. Fórmula do total do pedido

O total deve ser calculado pela seguinte fórmula:

`total_centavos = max(0, subtotal_pizzas_centavos - desconto_centavos + taxa_entrega_centavos - desconto_fidelidade_centavos)`

O desconto promocional e o desconto de fidelidade devem ser subtraídos do subtotal, enquanto a taxa de entrega deve ser somada integralmente. O piso zero é obrigatório: os descontos nunca podem produzir um total negativo.

A gorjeta não faz parte do total do pedido. Ela é calculada somente no fechamento da comanda.

### 2.3. Regras de arredondamento

- Desconto promocional e gorjeta: arredondar para cima, utilizando `ceil`.
- Ticket médio do relatório diário: arredondar para o inteiro mais próximo, com valores terminados em 0,5 arredondados para cima.
- Divisão da conta: distribuir os centavos restantes entre as primeiras parcelas, sem perder ou criar centavos.
- Percentual zero resulta em desconto ou gorjeta igual a zero.

## 3. Pedidos e ciclo de vida

- Todo pedido novo deve iniciar em `recebido`.
- A sequência de estados é `recebido → preparo → pronto → entregue`. O estado `entregue` é terminal.
- Somente pedidos em `recebido` podem ser cancelados. Tentativas de cancelar pedidos em `preparo`, `pronto` ou `entregue` devem retornar `409` com `{"erro":"pedido_nao_cancelavel"}`.
- Pedidos cancelados não entram no relatório diário nem contam como pedidos entregues para a fidelidade.
- Clientes com pedido ativo (`recebido`, `preparo` ou `pronto`) não podem criar outro pedido de `entrega` ou `balcao`. Pedidos do tipo `mesa` não estão sujeitos a essa restrição.
- O cancelamento de um pedido deve liberar o cliente para criar outro pedido quando a regra de pedido ativo for aplicável.
- O campo opcional `recebido_em` deve permitir registrar um instante informado em ISO 8601 com fuso. Quando ausente, deve ser utilizado o instante atual.

## 4. Precedência de validações e erros

A API deve respeitar a precedência definida no contrato. Quando mais de uma condição inválida estiver presente, deve ser retornado o erro correspondente à primeira etapa aplicável, sem executar efeitos de negócio antes da validação necessária.

### 4.1. Criação de pedidos

No `POST /pedidos`, a sequência obrigatória é:

1. **Formato do body (422):** validar cliente, itens, sabor, tamanho, quantidade, tipo e formato de `recebido_em`.
2. **Comanda (422, para pedidos de mesa):** validar a presença de `comanda_id` e sua existência e situação aberta. Se o tipo não for `mesa`, a presença de `comanda_id` também deve ser rejeitada com `comanda_invalida`.
3. **Disponibilidade do cardápio (409):** verificar se todos os sabores solicitados estão disponíveis. Se algum estiver indisponível, retornar `{"erro":"sabor_indisponivel"}` e não gravar parcialmente o pedido.
4. **Pedido ativo do cliente (409):** para `entrega` e `balcao`, verificar a regra de um pedido ativo por cliente. Em caso de conflito, retornar `{"erro":"pedido_em_aberto"}`.

O pedido só deve ser criado depois que todas as validações aplicáveis forem aprovadas.

### 4.2. Fechamento de comandas

No `POST /comandas/{id}/fechamento`:

1. Validar o formato do body, incluindo `gorjeta_percentual` e `dividir_em`, quando presentes.
2. Se `gorjeta_percentual` estiver fora de `0..30`, retornar `422` com `{"erro":"gorjeta_invalida"}`.
3. Se `dividir_em` estiver fora de `2..10`, retornar `422` com `{"erro":"divisao_invalida"}`.
4. Validar a existência da comanda; se inexistente, retornar `404` com `{"erro":"comanda_nao_encontrada"}`.
5. Se já estiver fechada, retornar `409` com `{"erro":"comanda_ja_fechada"}`.
6. Se houver pedidos em `recebido`, `preparo` ou `pronto`, retornar `409` com `{"erro":"comanda_com_pedidos_abertos"}`.
7. Somente após todas as verificações, efetivar o fechamento e calcular os valores da conta.

As respostas de erro devem respeitar os nomes e formatos definidos no contrato. Uma requisição inválida não deve produzir fechamento, alteração de estado ou resgate indevido de benefícios.

## 5. Mesas, comandas e fechamento de conta

- Os números de mesa válidos são inteiros de `1` a `50`, inclusive. Valores fora desse intervalo devem retornar `422` com `{"erro":"mesa_invalida"}`.
- Cada mesa pode ter somente uma comanda aberta por vez. Uma tentativa de abrir outra comanda para uma mesa ocupada deve retornar `409` com `{"erro":"mesa_ocupada"}`.
- Uma comanda pode reunir vários pedidos de mesa, e cada pedido mantém seu próprio ciclo de estados.
- O fechamento só pode ocorrer quando não houver pedidos ativos na comanda. Todos os pedidos devem estar concluídos ou cancelados.
- O campo `pedidos_entregues` deve contabilizar os pedidos entregues e não cancelados da comanda.
- O campo `subtotal_centavos` do fechamento deve ser a soma de `total_centavos` dos pedidos entregues e não cancelados vinculados à comanda. Esse valor já contempla os descontos, a taxa aplicável e os benefícios de fidelidade dos pedidos.
- A gorjeta padrão desta variante é `0%`. Quando `gorjeta_percentual` estiver ausente, utilizar `GORJETA_PADRAO_PERCENTUAL`; quando informado, utilizar o percentual válido da requisição.
- Calcular `gorjeta_centavos = ceil(subtotal_centavos × gorjeta_percentual / 100)` e `total_centavos = subtotal_centavos + gorjeta_centavos`.
- A gorjeta não integra o faturamento da pizzaria no relatório diário.
- Um fechamento bem-sucedido deve registrar `fechada_em`, mudar a comanda para fechada e liberar a mesa para uma nova comanda.
- Uma comanda fechada não pode ser fechada novamente.

## 6. Divisão da conta

- O campo opcional `dividir_em` aceita um inteiro entre `2` e `10`, inclusive.
- Quando informado, a resposta do fechamento deve incluir `parcelas_centavos` com exatamente `dividir_em` valores inteiros.
- A soma das parcelas deve ser exatamente igual a `total_centavos`, incluindo a gorjeta.
- A diferença entre a maior e a menor parcela deve ser de, no máximo, um centavo.
- Os centavos restantes da divisão inteira devem ser atribuídos às primeiras parcelas.
- Quando `dividir_em` estiver ausente, o campo `parcelas_centavos` não deve aparecer na resposta.
- Uma divisão inválida deve retornar `422` com `{"erro":"divisao_invalida"}` antes de qualquer alteração no estado da comanda.

## 7. Fidelidade por pedidos entregues

- A cada cinco pedidos entregues de um cliente, o próximo pedido criado por esse cliente recebe o benefício de fidelidade. O benefício corresponde a `PRECO_BROTO_CENTAVOS`.
- Somente pedidos com status `entregue` contam para os selos. Pedidos cancelados não contam, independentemente do tipo.
- Pedidos de `entrega`, `balcao` e `mesa` contam para a fidelidade pelo campo `cliente`.
- O benefício deve ser aplicado individualmente ao pedido premiado durante sua criação, nunca novamente no fechamento da comanda.
- No `201` de criação, o campo `desconto_fidelidade_centavos` deve conter `PRECO_BROTO_CENTAVOS` quando o pedido for premiado e `0` nos pedidos não premiados.
- O resgate ocorre uma única vez, na criação do pedido premiado. O benefício não pode ser aplicado novamente a outro pedido enquanto estiver resgatado.
- Se o pedido premiado for cancelado enquanto estiver em `recebido`, o resgate deve ser devolvido e o cliente deve voltar a ser elegível para o benefício.
- Se o pedido premiado estiver em `preparo`, `pronto` ou `entregue`, o benefício não deve ser devolvido. Pedidos nessas condições não podem ser cancelados pela regra de cancelamento do contrato.
- O total do pedido deve respeitar a fórmula:

  `total_centavos = max(0, subtotal_pizzas_centavos - desconto_centavos + taxa_entrega_centavos - desconto_fidelidade_centavos)`

- O desconto promocional e o desconto de fidelidade devem ser aplicados antes do piso zero. O total nunca pode ser negativo.
- O controle de elegibilidade, resgate e devolução deve permanecer consistente ao longo do histórico do cliente, incluindo pedidos de mesa e pedidos cancelados.

## 8. Cardápio e disponibilidade

- O cardápio é aberto: qualquer sabor com string de `2` a `30` caracteres é aceito e deve nascer disponível.
- Um sabor passa a ser conhecido quando aparece em um pedido ou é explicitamente registrado pelo endpoint `PUT /cardapio/{sabor}`.
- A comparação dos sabores não deve distinguir maiúsculas de minúsculas. A capitalização do primeiro registro deve ser preservada.
- `PUT /cardapio/{sabor}` deve exigir `disponivel` como booleano e retornar a representação do sabor e sua disponibilidade.
- `GET /cardapio` deve listar os sabores conhecidos em ordem alfabética, incluindo sua disponibilidade.
- Um pedido que contenha sabor indisponível deve retornar `409` com `{"erro":"sabor_indisponivel"}` e não pode gravar o pedido nem parte de seus itens.
- As regras de disponibilidade também se aplicam aos pedidos de mesa.

## 9. Relatório diário

- O endpoint `GET /relatorios/diario?data=AAAA-MM-DD` deve validar a data informada. Se o formato for inválido, retornar `422` com `{"erro":"data_invalida"}`.
- O relatório deve considerar todos os pedidos cujo campo `recebido_em` corresponda à data solicitada e que não estejam cancelados.
- A data deve ser determinada respeitando o fuso contratual `-03:00`.
- `total_pedidos` deve representar a quantidade de pedidos elegíveis, incluindo pedidos de mesa.
- `faturamento_centavos` deve ser a soma dos campos `total_centavos` dos pedidos elegíveis. A gorjeta do fechamento de comandas não integra esse faturamento.
- `ticket_medio_centavos` deve ser calculado dividindo `faturamento_centavos` por `total_pedidos`, arredondando o resultado para o inteiro mais próximo, com valores terminados em 0,5 arredondados para cima.
- **Dia sem pedidos elegíveis:** quando não houver pedidos na data consultada, o endpoint deve retornar `200` com a data solicitada e todos os indicadores numéricos zerados:

  ```json
  {
    "data": "AAAA-MM-DD",
    "total_pedidos": 0,
    "faturamento_centavos": 0,
    "ticket_medio_centavos": 0
  }
  ```

- Um dia sem pedidos não é erro. A API não deve retornar `404`, `422`, `500` nem tentar dividir por zero nessa situação.
- Pedidos cancelados não devem contribuir para a contagem, o faturamento ou o ticket médio.
- O relatório deve manter os nomes de campos e os tipos definidos no contrato, usando centavos inteiros e nunca valores monetários em ponto flutuante.

## 10. Datas, fuso horário e relógio

- Datas e horários retornados pela API devem utilizar ISO 8601 com fuso `-03:00`, conforme o contrato.
- `recebido_em` permite que a suíte de correção teste datas específicas sem depender da passagem do tempo real.
- Quando `recebido_em` estiver ausente, utilizar o relógio atual.
- Os campos temporais das comandas, incluindo `aberta_em` e `fechada_em`, devem registrar instantes válidos no fuso contratual.
- As regras de data devem ser consistentes entre criação de pedidos, histórico e relatório diário.

## 11. Servidor e execução

- O serviço gerado deve disponibilizar a API na base URL usada pela suíte de correção: `http://localhost:8002`.
- A porta da variante é `PORTA_SERVICO = 8002` e deve ser respeitada na configuração de execução e na exposição do serviço em container.
- Não exigir variáveis de ambiente obrigatórias que não estejam previstas no contrato.
- A solução deve ser executável em container e incluir as instruções e os arquivos de dependências necessários para sua inicialização.
- A escolha de linguagem, framework, persistência e configuração interna deve ser justificada em `plan.md`, sem alterar o contrato externo.

## 12. Qualidade, testes e documentação

- Os 16 casos de uso devem ser cobertos em `spec.md` com critérios de aceitação mensuráveis.
- `tests.md` deve contemplar pelo menos um caso de borda para cada regra de negócio, incluindo cálculos, validações, precedência de erros, fechamento, divisão e fidelidade.
- `tasks.md` deve decompor o trabalho em tarefas verificáveis e coerentes com os requisitos.
- A solução gerada deve possuir dependências declaradas, instruções de execução, testes próprios e organização adequada, evitando segredos e credenciais no repositório.
- Os arquivos de especificação devem orientar a geração da solução sem substituir a especificação por uma implementação extensa.

## 13. Resolução de conflitos

Em caso de divergência, prevalecem o contrato normativo de `ENUNCIADO.md` e `contrato.json` e os parâmetros de `variante/params.json`. Exemplos identificados como inconsistentes não devem ser tratados como requisitos.

Os valores específicos da variante não anulam as regras gerais: percentuais configurados como zero devem continuar respeitando os comportamentos previstos para valores diferentes quando a requisição permitir substituí-los.

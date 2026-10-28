# Specification — API da Pizzaria

## 1. Objetivo

Especificar os requisitos funcionais e os critérios de aceitação da API REST da pizzaria, abrangendo pedidos, estados de preparo, cancelamentos, relatórios, mesas, comandas, cardápio e fidelidade.

A implementação deve seguir o `ENUNCIADO.md`, o `contrato.json`, o `variante/params.json` e as regras definidas em `constitution.md`.

## 2. Configuração da variante

A solução deve utilizar os seguintes parâmetros:

| Parâmetro | Valor |
|---|---:|
| `PRECO_BROTO_CENTAVOS` | 2750 |
| `PRECO_GRANDE_CENTAVOS` | 5000 |
| `TAXA_ENTREGA_CENTAVOS` | 0 |
| `DESCONTO_PROMO_PERCENTUAL` | 0 |
| `GORJETA_PADRAO_PERCENTUAL` | 0 |
| `PORTA_SERVICO` | 8002 |

Todos os valores monetários devem ser calculados e retornados em centavos inteiros. Os percentuais configurados como zero não eliminam a necessidade de implementar as regras gerais para percentuais válidos diferentes de zero.

## 3. Convenções gerais da API

- A base URL deve ser `http://localhost:8002`.
- As respostas devem respeitar os métodos HTTP, caminhos, códigos de status, campos e formatos estabelecidos no contrato.
- Os erros devem utilizar o formato JSON e os identificadores definidos no contrato.
- Datas e horários devem utilizar ISO 8601 com fuso `-03:00`.
- As validações devem ocorrer na ordem de precedência estabelecida no contrato.
- Uma operação rejeitada não deve produzir alterações parciais nos dados nem consumir benefícios indevidamente.
- Listagens devem respeitar a ordenação especificada em cada caso de uso.
- A API deve manter consistência entre pedidos, clientes, cardápio, mesas, comandas, relatórios e fidelidade.

## 4. Casos de uso e critérios de aceitação

### UC01 — Criar pedido

**Endpoint:** `POST /pedidos`

**Objetivo:** registrar um pedido de pizza para entrega, retirada no balcão ou consumo em mesa.

**Critérios de aceitação:**

- **CA01.1:** dado um body válido com cliente, itens e tipo, a API deve retornar `201` e criar um pedido com status `recebido`.
- **CA01.2:** o cliente deve ser uma string com comprimento entre 2 e 60 caracteres.
- **CA01.3:** a lista de itens deve conter entre 1 e 20 itens. Cada item deve ter sabor com 2 a 30 caracteres, tamanho `broto` ou `grande` e quantidade inteira entre 1 e 10.
- **CA01.4:** o tipo deve ser `entrega`, `balcao` ou `mesa`.
- **CA01.5:** cada item deve apresentar `valor_unitario_centavos` correspondente ao tamanho: 2750 para broto e 5000 para grande.
- **CA01.6:** `subtotal_pizzas_centavos` deve ser a soma dos preços unitários multiplicados pelas quantidades.
- **CA01.7:** a taxa de entrega deve ser aplicada somente ao tipo `entrega`; para `balcao` e `mesa`, deve ser zero.
- **CA01.8:** o desconto promocional deve ser calculado sobre o subtotal das pizzas, arredondado para cima, sem incluir a taxa de entrega.
- **CA01.9:** o total deve seguir a fórmula definida na constituição, incluindo o desconto de fidelidade e o piso zero.
- **CA01.10:** pedidos de mesa devem informar `comanda_id` válido, vinculado a uma comanda aberta. Pedidos de outros tipos não podem informar `comanda_id`.
- **CA01.11:** quando `recebido_em` for informado, deve ser validado como data e hora ISO 8601 com fuso. Quando ausente, deve ser registrado o instante atual.
- **CA01.12:** quando houver mais de uma condição inválida, a API deve respeitar a precedência: formato, comanda, disponibilidade do sabor e pedido ativo do cliente.
- **CA01.13:** o pedido deve incluir os campos definidos no contrato, incluindo cliente, itens, tipo, comanda, data de recebimento, status, subtotal, taxa, descontos e total.

### UC02 — Atualizar status do pedido

**Endpoint:** `POST /pedidos/{id}/status`

**Objetivo:** avançar o pedido pelas etapas de preparo e atendimento.

**Critérios de aceitação:**

- **CA02.1:** um pedido em `recebido` pode avançar para `preparo`.
- **CA02.2:** um pedido em `preparo` pode avançar para `pronto`.
- **CA02.3:** um pedido em `pronto` pode avançar para `entregue`.
- **CA02.4:** um pedido em `entregue` é terminal e não pode avançar novamente.
- **CA02.5:** ao solicitar uma transição para um pedido inexistente, a API deve retornar `404` com `pedido_nao_encontrado`.
- **CA02.6:** ao tentar avançar um pedido já entregue, a API deve retornar `409` com `pedido_ja_finalizado`.

### UC03 — Listar pedidos ativos

**Endpoint:** `GET /pedidos/ativos`

**Objetivo:** consultar os pedidos que ainda não foram concluídos ou cancelados.

**Critérios de aceitação:**

- **CA03.1:** a resposta deve incluir pedidos nos estados `recebido`, `preparo` e `pronto`.
- **CA03.2:** pedidos `entregue` e cancelados não devem aparecer.
- **CA03.3:** os pedidos devem ser apresentados do mais recente para o mais antigo, conforme o critério de ordenação do contrato.
- **CA03.4:** quando não houver pedidos ativos, a API deve retornar uma lista vazia, sem erro.

### UC04 — Consultar relatório diário

**Endpoint:** `GET /relatorios/diario?data=AAAA-MM-DD`

**Objetivo:** consultar os indicadores de pedidos e faturamento de uma data.

**Critérios de aceitação:**

- **CA04.1:** para uma data válida, a API deve retornar `total_pedidos`, `faturamento_centavos` e `ticket_medio_centavos`, além dos campos exigidos pelo contrato.
- **CA04.2:** somente pedidos recebidos na data consultada e não cancelados devem ser contabilizados.
- **CA04.3:** pedidos de entrega, balcão e mesa devem ser incluídos, desde que atendam ao critério de data e não estejam cancelados.
- **CA04.4:** o faturamento deve ser a soma de `total_centavos` dos pedidos elegíveis, sem incluir gorjetas de fechamento de comandas.
- **CA04.5:** o ticket médio deve ser calculado dividindo o faturamento pela quantidade de pedidos, com arredondamento para o inteiro mais próximo e empate de meio centavo para cima.
- **CA04.6:** quando não houver pedidos elegíveis, a API deve retornar `200`, com a data solicitada e os indicadores numéricos zerados.
- **CA04.7:** uma data com formato inválido deve retornar `422` com `data_invalida`.
- **CA04.8:** a data do pedido deve ser interpretada conforme o fuso `-03:00`.

### UC05 — Cancelar pedido

**Endpoint:** `POST /pedidos/{id}/cancelamento`

**Objetivo:** cancelar um pedido que ainda não entrou em preparo.

**Critérios de aceitação:**

- **CA05.1:** um pedido em `recebido` deve poder ser cancelado.
- **CA05.2:** um pedido cancelado deve deixar de ser considerado ativo e não deve contribuir para o relatório diário.
- **CA05.3:** um pedido cancelado não deve contar para a fidelidade.
- **CA05.4:** um pedido inexistente deve retornar `404` com `pedido_nao_encontrado`.
- **CA05.5:** pedidos em `preparo`, `pronto` ou `entregue` devem retornar `409` com `pedido_nao_cancelavel`.
- **CA05.6:** se o pedido cancelado tiver consumido um benefício de fidelidade, o benefício deve ser devolvido quando o cancelamento permitido ocorrer em `recebido`.

### UC06 — Consultar pedidos por cliente

**Endpoint:** `GET /pedidos?cliente={nome}`

**Objetivo:** consultar o histórico de pedidos de um cliente.

**Critérios de aceitação:**

- **CA06.1:** a consulta deve retornar os pedidos associados ao cliente informado, independentemente do status.
- **CA06.2:** os resultados devem ser ordenados do mais recente para o mais antigo.
- **CA06.3:** quando não houver pedidos para o cliente, a API deve retornar uma lista vazia.
- **CA06.4:** se o parâmetro `cliente` estiver ausente ou tiver menos de 2 caracteres, a API deve retornar `422` com `cliente_invalido`.
- **CA06.5:** os pedidos retornados devem preservar os campos e os tipos definidos no contrato.

### UC07 — Aplicar desconto promocional

**Aplicação:** cálculo realizado durante `POST /pedidos`.

**Objetivo:** aplicar o percentual promocional configurado ao subtotal das pizzas.

**Critérios de aceitação:**

- **CA07.1:** o desconto deve ser calculado como `ceil(subtotal_pizzas_centavos × DESCONTO_PROMO_PERCENTUAL / 100)`.
- **CA07.2:** o desconto deve incidir somente sobre as pizzas, sem incluir a taxa de entrega.
- **CA07.3:** para subtotal de 10000 centavos e percentual de 10%, o desconto deve ser 1000 centavos.
- **CA07.4:** para subtotal de 9999 centavos e percentual de 10%, o desconto deve ser 1000 centavos.
- **CA07.5:** para subtotal de 333 centavos e percentual de 10%, o desconto deve ser 34 centavos.
- **CA07.6:** com percentual igual a zero, o desconto deve ser zero.
- **CA07.7:** o valor do desconto deve ser informado no campo correspondente da resposta de criação do pedido.

### UC08 — Impedir pedidos ativos duplicados por cliente

**Aplicação:** validação realizada durante `POST /pedidos`.

**Objetivo:** impedir que um cliente tenha mais de um pedido ativo de entrega ou balcão.

**Critérios de aceitação:**

- **CA08.1:** se o cliente possuir um pedido de `entrega` ou `balcao` em `recebido`, `preparo` ou `pronto`, uma nova tentativa de pedido de entrega ou balcão deve retornar `409` com `pedido_em_aberto`.
- **CA08.2:** pedidos do tipo `mesa` não devem ser bloqueados por essa regra.
- **CA08.3:** pedidos entregues ou cancelados não devem bloquear um novo pedido.
- **CA08.4:** a validação deve ocorrer depois das validações de formato, comanda e disponibilidade do sabor.
- **CA08.5:** uma requisição bloqueada não deve criar um novo pedido nem consumir um benefício de fidelidade.

### UC09 — Abrir comanda de mesa

**Endpoint:** `POST /mesas/{numero}/comandas`

**Objetivo:** abrir uma comanda para registrar pedidos de uma mesa.

**Critérios de aceitação:**

- **CA09.1:** para um número de mesa inteiro entre 1 e 50, sem comanda aberta, a API deve retornar `201`.
- **CA09.2:** a comanda criada deve conter identificador, número da mesa, data de abertura em ISO 8601 com fuso `-03:00` e status `aberta`.
- **CA09.3:** uma mesa com comanda aberta não pode receber outra comanda simultaneamente; a tentativa deve retornar `409` com `mesa_ocupada`.
- **CA09.4:** um número de mesa fora do intervalo de 1 a 50 deve retornar `422` com `mesa_invalida`.
- **CA09.5:** depois do fechamento da comanda anterior, a mesa deve poder receber uma nova comanda.

### UC10 — Registrar pedidos de mesa

**Endpoint:** `POST /pedidos`

**Objetivo:** vincular pedidos do tipo `mesa` a uma comanda aberta.

**Critérios de aceitação:**

- **CA10.1:** um pedido do tipo `mesa` deve estar vinculado a uma comanda existente e aberta.
- **CA10.2:** um pedido de mesa deve seguir o ciclo de status `recebido`, `preparo`, `pronto` e `entregue`.
- **CA10.3:** pedidos de mesa não devem receber taxa de entrega.
- **CA10.4:** o desconto promocional e o benefício de fidelidade devem seguir as mesmas regras de cálculo aplicadas aos demais pedidos.
- **CA10.5:** uma comanda pode conter vários pedidos de mesa, cada um com ciclo de status independente.
- **CA10.6:** somente pedidos entregues e não cancelados devem integrar o subtotal do fechamento da comanda.
- **CA10.7:** pedidos de mesa devem contar para a fidelidade usando o campo `cliente`, sem reaplicar o desconto de fidelidade no fechamento da comanda.

### UC11 — Fechar comanda

**Endpoint:** `POST /comandas/{id}/fechamento`

**Objetivo:** calcular a conta final da comanda e liberar a mesa.

**Critérios de aceitação:**

- **CA11.1:** a API deve retornar `404` com `comanda_nao_encontrada` quando o identificador não existir.
- **CA11.2:** a API deve retornar `409` com `comanda_ja_fechada` quando a comanda já estiver fechada.
- **CA11.3:** a API deve retornar `409` com `comanda_com_pedidos_abertos` se houver pedidos em `recebido`, `preparo` ou `pronto`.
- **CA11.4:** o subtotal deve somar o `total_centavos` dos pedidos entregues e não cancelados vinculados à comanda.
- **CA11.5:** `gorjeta_percentual`, quando informado, deve ser um inteiro entre 0 e 30. Fora do intervalo, retornar `422` com `gorjeta_invalida`.
- **CA11.6:** quando o percentual de gorjeta estiver ausente, utilizar `GORJETA_PADRAO_PERCENTUAL`.
- **CA11.7:** a gorjeta deve ser calculada com arredondamento para cima: `ceil(subtotal_centavos × gorjeta_percentual / 100)`.
- **CA11.8:** o total da conta deve ser `subtotal_centavos + gorjeta_centavos`.
- **CA11.9:** o fechamento deve registrar `fechada_em`, mudar o status da comanda para fechada e liberar a mesa.
- **CA11.10:** o campo `pedidos_entregues` deve contabilizar os pedidos entregues e não cancelados da comanda.
- **CA11.11:** o fechamento não deve incluir gorjeta no faturamento diário dos pedidos.

### UC12 — Listar comandas abertas

**Endpoint:** `GET /comandas/abertas`

**Objetivo:** consultar as comandas que ainda não foram fechadas.

**Critérios de aceitação:**

- **CA12.1:** a resposta deve conter somente comandas com status `aberta`.
- **CA12.2:** as comandas devem ser ordenadas da mais antiga para a mais recente.
- **CA12.3:** quando não houver comandas abertas, a API deve retornar uma lista vazia.
- **CA12.4:** cada registro deve apresentar os campos exigidos pelo contrato.

### UC13 — Consultar comandas de uma mesa

**Endpoint:** `GET /mesas/{numero}/comandas`

**Objetivo:** consultar o histórico de comandas de uma mesa.

**Critérios de aceitação:**

- **CA13.1:** o número da mesa deve ser um inteiro entre 1 e 50.
- **CA13.2:** um número inválido deve retornar `422` com `mesa_invalida`.
- **CA13.3:** a resposta deve incluir comandas abertas e fechadas associadas à mesa.
- **CA13.4:** os resultados devem ser ordenados da comanda mais recente para a mais antiga.
- **CA13.5:** quando a mesa ainda não possuir comandas, a API deve retornar uma lista vazia.

### UC14 — Consultar e atualizar disponibilidade do cardápio

**Endpoints:** `GET /cardapio` e `PUT /cardapio/{sabor}`

**Objetivo:** consultar sabores conhecidos e controlar sua disponibilidade.

**Critérios de aceitação:**

- **CA14.1:** qualquer sabor válido entre 2 e 30 caracteres deve ser aceito inicialmente como disponível.
- **CA14.2:** um sabor passa a ser conhecido quando aparece em um pedido ou é explicitamente registrado pela atualização do cardápio.
- **CA14.3:** a atualização deve exigir o campo `disponivel` como booleano e retornar `200` com o sabor e sua disponibilidade.
- **CA14.4:** a comparação de sabores deve ignorar diferenças entre maiúsculas e minúsculas, preservando a capitalização do primeiro registro.
- **CA14.5:** `GET /cardapio` deve retornar os sabores conhecidos em ordem alfabética, incluindo sua disponibilidade.
- **CA14.6:** pedidos que contenham algum sabor indisponível devem retornar `409` com `sabor_indisponivel`.
- **CA14.7:** um pedido rejeitado por sabor indisponível não deve criar pedido nem gravar parcialmente seus itens.
- **CA14.8:** a disponibilidade deve ser validada também para pedidos do tipo `mesa`.

### UC15 — Dividir a conta

**Aplicação:** campo opcional `dividir_em` no body de `POST /comandas/{id}/fechamento`.

**Objetivo:** dividir o valor final da conta entre um número definido de parcelas.

**Critérios de aceitação:**

- **CA15.1:** `dividir_em`, quando informado, deve ser um inteiro entre 2 e 10, inclusive.
- **CA15.2:** um valor inválido deve retornar `422` com `divisao_invalida`, antes de alterar o estado da comanda.
- **CA15.3:** a resposta deve conter `parcelas_centavos` com exatamente a quantidade solicitada de parcelas inteiras.
- **CA15.4:** a soma de todas as parcelas deve ser igual ao `total_centavos`, incluindo a gorjeta.
- **CA15.5:** a diferença entre a maior e a menor parcela deve ser de, no máximo, um centavo.
- **CA15.6:** os centavos restantes devem ser atribuídos às primeiras parcelas.
- **CA15.7:** quando `dividir_em` não for informado, a resposta não deve conter o campo `parcelas_centavos`.

### UC16 — Aplicar fidelidade por pedidos entregues

**Aplicação:** cálculo realizado durante `POST /pedidos`.

**Objetivo:** conceder desconto equivalente ao preço de uma pizza broto após cada cinco pedidos entregues por cliente.

**Critérios de aceitação:**

- **CA16.1:** a cada cinco pedidos entregues pelo cliente, o próximo pedido criado por esse cliente deve receber o benefício de fidelidade.
- **CA16.2:** o benefício deve corresponder a `PRECO_BROTO_CENTAVOS`, que nesta variante equivale a 2750 centavos.
- **CA16.3:** somente pedidos entregues contam para a fidelidade. Pedidos cancelados não contam.
- **CA16.4:** pedidos de entrega, balcão e mesa devem ser contabilizados por cliente, usando o campo `cliente`.
- **CA16.5:** o campo `desconto_fidelidade_centavos` deve aparecer na resposta de criação, contendo 2750 para um pedido premiado e zero para os demais.
- **CA16.6:** o benefício deve ser resgatado uma única vez durante a criação do pedido e aplicado ao próprio pedido, não novamente no fechamento da comanda.
- **CA16.7:** se o pedido premiado for cancelado enquanto estiver em `recebido`, o benefício deve ser devolvido e o cliente deve voltar a ser elegível.
- **CA16.8:** o total deve respeitar o piso zero após a aplicação dos descontos promocional e de fidelidade.
- **CA16.9:** a contagem de pedidos entregues e o estado do benefício devem permanecer consistentes entre pedidos de diferentes tipos.

## 5. Requisitos não funcionais

### RNF01 — Compatibilidade contratual

A API deve respeitar integralmente os caminhos, métodos HTTP, nomes de campos, formatos de resposta, códigos de status e identificadores de erro estabelecidos no contrato.

### RNF02 — Consistência dos dados

As operações devem preservar a integridade entre pedidos, clientes, cardápio, comandas, mesas, relatórios e fidelidade. Requisições rejeitadas não devem deixar alterações parciais.

### RNF03 — Persistência durante a execução

Os dados devem permanecer disponíveis entre as requisições realizadas durante a execução do serviço, de modo que consultas e operações subsequentes reflitam as alterações confirmadas.

### RNF04 — Execução do serviço

O serviço deve iniciar na porta `8002`, disponibilizar a API em `http://localhost:8002` e incluir os arquivos de dependências e as instruções necessárias para sua execução.

### RNF05 — Testabilidade

A implementação deve permitir testar os casos de uso, os limites das validações, os erros, a precedência das regras, os cálculos monetários e os fluxos completos de pedidos e comandas.

## 6. Regras transversais de validação

- Strings e números devem respeitar os limites definidos para cada campo.
- Valores monetários devem utilizar inteiros em centavos.
- Campos opcionais devem seguir o comportamento especificado quando ausentes.
- Datas devem ser interpretadas no fuso contratual `-03:00`.
- O erro retornado deve respeitar a precedência estabelecida na constituição e no contrato.
- Listagens sem resultados devem retornar arrays vazios quando especificado, sem transformar ausência de registros em erro.
- O comportamento deve permanecer consistente entre pedidos de entrega, balcão e mesa.

## 7. Referências normativas

- `ENUNCIADO.md`: descrição dos casos de uso e regras gerais da prova.
- `contrato.json`: contrato da API, endpoints, respostas, erros e precedência.
- `variante/params.json`: parâmetros específicos desta variante.
- `constitution.md`: princípios operacionais, regras monetárias, ciclo de vida, fidelidade e execução do serviço.

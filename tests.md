# Test Specification — API da Pizzaria

## 1. Objetivo

Definir cenários de teste funcionais, casos de borda e resultados esperados para validar a API da pizzaria antes da entrega.

Os testes devem verificar os 16 casos de uso, os cálculos monetários, as validações, a precedência de erros, a consistência dos dados e os fluxos completos de pedidos e comandas.

A implementação deve seguir `ENUNCIADO.md`, `contrato.json`, `variante/params.json`, `constitution.md` e `spec.md`.

## 2. Convenções de execução

- Cada cenário deve ser executado com dados conhecidos e independentes dos demais testes, evitando interferência entre pedidos, clientes, comandas e benefícios de fidelidade.
- Os testes que dependem de pedidos anteriores devem preparar explicitamente o histórico necessário.
- As respostas devem ser verificadas quanto ao código HTTP, aos campos obrigatórios, aos tipos e aos valores esperados.
- Quando o cenário produzir erro, verificar também que nenhum efeito parcial foi persistido.
- Valores monetários devem ser comparados em centavos inteiros.
- Os testes de data devem considerar o fuso `-03:00`.
- Os testes de percentual devem incluir valores diferentes de zero, mesmo quando o percentual configurado na variante for zero.
- Os exemplos com parâmetros alternativos servem para validar as regras gerais do contrato; os testes específicos da variante devem usar os valores reais de `variante/params.json`.

## 3. Testes de criação e validação de pedidos — UC01

| ID | Cenário | Resultado esperado |
|---|---|---|
| T01 | Criar pedido válido de uma pizza broto, quantidade 1 | `201`; status `recebido`; subtotal de 2750 centavos |
| T02 | Criar pedido com pizza grande, quantidade 2 | `201`; subtotal de 10000 centavos |
| T03 | Informar cliente ausente ou com menos de 2 caracteres | `422`; erro `cliente_invalido` |
| T04 | Informar cliente com mais de 60 caracteres | `422`; erro `cliente_invalido` |
| T05 | Enviar lista de itens vazia | `422`; erro `itens_invalidos` |
| T06 | Enviar mais de 20 itens | `422`; erro `itens_invalidos` |
| T07 | Informar tamanho diferente de `broto` e `grande` | `422`; erro `itens_invalidos` |
| T08 | Informar quantidade 0, negativa, decimal ou maior que 10 | `422`; erro `itens_invalidos` |
| T09 | Informar tipo diferente de `entrega`, `balcao` ou `mesa` | `422`; erro `tipo_invalido` |
| T10 | Informar sabor com menos de 2 ou mais de 30 caracteres | `422`, conforme a validação de formato definida no contrato |
| T11 | Informar `recebido_em` com formato inválido | `422`; erro `recebido_invalido` |
| T12 | Criar pedido válido sem `recebido_em` | `201`; registrar o instante atual em ISO 8601 com fuso `-03:00` |
| T13 | Criar pedido válido com `recebido_em` explícito | `201`; preservar o instante informado conforme as regras de fuso do contrato |
| T14 | Criar pedido do tipo `mesa` sem `comanda_id` | `422`; erro `comanda_invalida` |
| T15 | Criar pedido de mesa com comanda inexistente ou fechada | `422`; erro `comanda_invalida` |
| T16 | Informar `comanda_id` em pedido de entrega ou balcão | `422`; erro `comanda_invalida` |
| T17 | Criar pedido válido de entrega com taxa configurada diferente de zero | A taxa deve ser somada ao total sem integrar o subtotal das pizzas |
| T18 | Criar pedido de balcão ou mesa com taxa de entrega configurada diferente de zero | A taxa aplicada deve ser zero |

## 4. Testes do ciclo de vida e cancelamento — UC02 e UC05

| ID | Cenário | Resultado esperado |
|---|---|---|
| T19 | Avançar pedido de `recebido` para `preparo` | Transição aceita |
| T20 | Avançar pedido de `preparo` para `pronto` | Transição aceita |
| T21 | Avançar pedido de `pronto` para `entregue` | Transição aceita |
| T22 | Tentar avançar pedido já entregue | `409`; erro `pedido_ja_finalizado` |
| T23 | Atualizar status de pedido inexistente | `404`; erro `pedido_nao_encontrado` |
| T24 | Cancelar pedido em `recebido` | Cancelamento aceito |
| T25 | Cancelar pedido em `preparo`, `pronto` ou `entregue` | `409`; erro `pedido_nao_cancelavel` |
| T26 | Consultar relatório após cancelar um pedido | O pedido cancelado não deve contribuir para os indicadores |
| T27 | Cancelar pedido de cliente que tinha um pedido ativo de entrega ou balcão | O cliente deve poder criar outro pedido, respeitadas as demais validações |

## 5. Testes de listagem e histórico — UC03 e UC06

| ID | Cenário | Resultado esperado |
|---|---|---|
| T28 | Consultar pedidos ativos com pedidos em `recebido`, `preparo` e `pronto` | Retornar os pedidos ativos, ordenados do mais recente para o mais antigo |
| T29 | Consultar pedidos ativos quando todos estiverem entregues ou cancelados | `200`; lista vazia |
| T30 | Consultar histórico de cliente com pedidos em vários estados | Retornar todos os estados, ordenados do mais recente para o mais antigo |
| T31 | Consultar histórico de cliente sem pedidos | `200`; lista vazia |
| T32 | Consultar histórico sem parâmetro `cliente` | `422`; erro `cliente_invalido` |
| T33 | Consultar histórico com `cliente` de um caractere | `422`; erro `cliente_invalido` |

## 6. Testes de cálculos monetários e desconto promocional — UC01 e UC07

Para os cenários percentuais, utilizar uma configuração de teste com `DESCONTO_PROMO_PERCENTUAL = 10`.

| ID | Subtotal das pizzas | Percentual | Resultado esperado |
|---|---:|---:|---|
| T34 | 10000 centavos | 10% | Desconto de 1000 centavos |
| T35 | 9999 centavos | 10% | Desconto de 1000 centavos |
| T36 | 333 centavos | 10% | Desconto de 34 centavos |
| T37 | 10000 centavos | 0% | Desconto de 0 centavos |
| T38 | 1 centavo | 10% | Desconto de 1 centavo |

### Cenários complementares

- **T39:** verificar que o desconto promocional incide apenas sobre as pizzas, nunca sobre a taxa de entrega.
- **T40:** verificar que o total utiliza somente inteiros em centavos, sem valores monetários em ponto flutuante.
- **T41:** configurar descontos promocional e de fidelidade cuja soma supere o subtotal acrescido da taxa. O total deve ser zero, nunca negativo.
- **T42:** conferir que o preço unitário retornado corresponde ao tamanho de cada item e que o subtotal considera todas as quantidades.

## 7. Testes de pedido ativo por cliente — UC08

| ID | Cenário | Resultado esperado |
|---|---|---|
| T43 | Cliente com pedido em `recebido` tenta criar pedido de entrega | `409`; erro `pedido_em_aberto` |
| T44 | Cliente com pedido em `preparo` tenta criar pedido de balcão | `409`; erro `pedido_em_aberto` |
| T45 | Cliente com pedido em `pronto` tenta criar outro pedido de entrega ou balcão | `409`; erro `pedido_em_aberto` |
| T46 | Cliente cujo pedido anterior foi entregue tenta criar outro pedido | A criação deve ser permitida, se as demais regras forem atendidas |
| T47 | Cliente cujo pedido anterior foi cancelado tenta criar outro pedido | A criação deve ser permitida, se as demais regras forem atendidas |
| T48 | Cliente com pedido ativo de entrega cria pedido de mesa | A regra de pedido ativo não deve bloquear o pedido de mesa |
| T49 | Cliente tenta criar novo pedido quando o sabor está indisponível e já possui pedido ativo | Prevalece `409` com erro `sabor_indisponivel`, conforme a precedência do contrato |

## 8. Testes de relatório diário — UC04

| ID | Cenário | Resultado esperado |
|---|---|---|
| T50 | Consultar data com pedidos não cancelados | Contagem, faturamento e ticket médio corretos |
| T51 | Consultar data sem pedidos | `200`; data solicitada e indicadores numéricos iguais a zero |
| T52 | Consultar data em que todos os pedidos foram cancelados | `200`; indicadores numéricos iguais a zero |
| T53 | Consultar data com pedidos de entrega, balcão e mesa | Todos os pedidos elegíveis devem ser considerados |
| T54 | Consultar data com pedidos recebidos em datas diferentes | Contabilizar somente os pedidos cuja data de `recebido_em`, interpretada em `-03:00`, corresponda à data consultada |
| T55 | Consultar relatório com data em formato inválido | `422`; erro `data_invalida` |
| T56 | Calcular ticket médio de faturamento de 3 centavos dividido por 2 pedidos | Ticket médio de 2 centavos, com empate arredondado para cima |
| T57 | Fechar comanda com gorjeta e consultar o relatório diário | A gorjeta não deve integrar o faturamento dos pedidos |
| T58 | Consultar relatório sem pedidos elegíveis | Não retornar `404`, não dividir por zero e não gerar erro interno |

## 9. Testes de mesas e comandas — UC09, UC10, UC11, UC12 e UC13

### 9.1. Abertura e histórico de mesas

| ID | Cenário | Resultado esperado |
|---|---|---|
| T59 | Abrir comanda para mesa 1 | `201`; comanda aberta |
| T60 | Abrir comanda para mesa 50 | `201`; comanda aberta |
| T61 | Tentar abrir comanda para mesa 0 ou 51 | `422`; erro `mesa_invalida` |
| T62 | Abrir segunda comanda para mesa que já possui uma aberta | `409`; erro `mesa_ocupada` |
| T63 | Consultar comandas abertas | Retornar somente comandas abertas, da mais antiga para a mais recente |
| T64 | Consultar comandas de mesa sem histórico | `200`; lista vazia |
| T65 | Consultar histórico de mesa com comandas abertas e fechadas | Retornar ambas, da mais recente para a mais antiga |
| T66 | Fechar comanda e abrir outra para a mesma mesa | A nova abertura deve ser permitida |

### 9.2. Pedidos de mesa e fechamento

| ID | Cenário | Resultado esperado |
|---|---|---|
| T67 | Criar vários pedidos válidos na mesma comanda aberta | Todos devem ser vinculados à comanda e manter ciclos independentes |
| T68 | Tentar fechar comanda com pedido em `recebido` | `409`; erro `comanda_com_pedidos_abertos` |
| T69 | Tentar fechar comanda com pedido em `preparo` | `409`; erro `comanda_com_pedidos_abertos` |
| T70 | Tentar fechar comanda com pedido em `pronto` | `409`; erro `comanda_com_pedidos_abertos` |
| T71 | Fechar comanda cujos pedidos estejam entregues ou cancelados | Fechamento permitido; subtotal considera somente pedidos entregues e não cancelados |
| T72 | Fechar comanda inexistente | `404`; erro `comanda_nao_encontrada` |
| T73 | Fechar novamente comanda já fechada | `409`; erro `comanda_ja_fechada` |
| T74 | Fechar comanda com pedido cancelado e outro entregue | O pedido cancelado não deve entrar no subtotal nem na contagem de entregues |

## 10. Testes de gorjeta e arredondamento — UC11

| ID | Cenário | Resultado esperado |
|---|---|---|
| T75 | Fechar comanda sem informar `gorjeta_percentual` | Utilizar `GORJETA_PADRAO_PERCENTUAL` da variante |
| T76 | Informar gorjeta de 0% | `gorjeta_centavos = 0` |
| T77 | Informar gorjeta de 10% sobre subtotal de 10000 centavos | Gorjeta de 1000 centavos |
| T78 | Informar gorjeta de 10% sobre subtotal de 333 centavos | Gorjeta de 34 centavos |
| T79 | Informar gorjeta de 30% | Requisição aceita |
| T80 | Informar gorjeta negativa ou superior a 30% | `422`; erro `gorjeta_invalida` |
| T81 | Informar gorjeta decimal em vez de inteiro | `422`; erro `gorjeta_invalida`, conforme validação do contrato |
| T82 | Informar gorjeta inválida e tentar fechar comanda inexistente | Prevalece a validação do body, conforme a precedência definida no contrato |

## 11. Testes de divisão da conta — UC15

| ID | Cenário | Resultado esperado |
|---|---|---|
| T83 | Fechar comanda com `dividir_em = 2` | Retornar exatamente 2 parcelas inteiras |
| T84 | Fechar comanda com `dividir_em = 10` | Retornar exatamente 10 parcelas inteiras |
| T85 | Dividir total de 10000 centavos em 3 parcelas | `[3334, 3333, 3333]` |
| T86 | Dividir total de 1 centavo em 2 parcelas | `[1, 0]` |
| T87 | Dividir total de 0 centavos em 3 parcelas | Três parcelas iguais a zero |
| T88 | Dividir conta cujo total inclui gorjeta | A soma das parcelas deve incluir integralmente a gorjeta |
| T89 | Informar `dividir_em = 1` ou `11` | `422`; erro `divisao_invalida` |
| T90 | Informar `dividir_em` decimal ou não inteiro | `422`; erro `divisao_invalida` |
| T91 | Omitir `dividir_em` | A resposta não deve conter `parcelas_centavos` |
| T92 | Dividir um total que não seja múltiplo do número de parcelas | A diferença entre as parcelas deve ser no máximo 1 centavo, com o resto atribuído às primeiras |

## 12. Testes de cardápio e disponibilidade — UC14

| ID | Cenário | Resultado esperado |
|---|---|---|
| T93 | Criar pedido com sabor ainda desconhecido e válido | Pedido aceito; sabor passa a ser conhecido e disponível |
| T94 | Marcar sabor conhecido como indisponível | `200`; retornar sabor e `disponivel = false` |
| T95 | Tentar pedir sabor indisponível | `409`; erro `sabor_indisponivel` |
| T96 | Criar pedido com dois sabores, sendo um indisponível | Rejeitar pedido inteiro; nenhum item deve ser gravado |
| T97 | Marcar sabor como disponível novamente | `200`; novos pedidos com esse sabor devem ser aceitos |
| T98 | Consultar cardápio com sabores conhecidos | Retornar sabores em ordem alfabética, com disponibilidade |
| T99 | Consultar cardápio sem sabores conhecidos | `200`; lista vazia |
| T100 | Registrar sabor e consultá-lo com diferença de maiúsculas e minúsculas | Tratar como o mesmo sabor, preservando a capitalização do primeiro registro |
| T101 | Criar pedido de mesa com sabor indisponível | `409`; erro `sabor_indisponivel`, depois da validação da comanda |

## 13. Testes de fidelidade — UC16

Preparar o histórico do cliente com pedidos efetivamente entregues. A contagem deve considerar todos os tipos de pedido e excluir cancelamentos.

| ID | Cenário | Resultado esperado |
|---|---|---|
| T102 | Cliente com 4 pedidos entregues cria novo pedido | `desconto_fidelidade_centavos = 0` |
| T103 | Cliente com 5 pedidos entregues cria o sexto pedido | Desconto igual a `PRECO_BROTO_CENTAVOS` |
| T104 | Cliente com pedidos entregues de entrega, balcão e mesa completa cinco entregues | O próximo pedido recebe o benefício |
| T105 | Cliente com pedidos cancelados, mas menos de cinco entregues, cria pedido | Pedidos cancelados não contribuem para a contagem |
| T106 | Cliente elegível cria pedido premiado | Benefício aplicado uma única vez no momento da criação |
| T107 | Cliente cria outro pedido enquanto o benefício já está resgatado | Não aplicar o mesmo benefício novamente |
| T108 | Cancelar pedido premiado enquanto está em `recebido` | Devolver o resgate; o cliente volta a ser elegível |
| T109 | Pedido premiado avança para `preparo` | O pedido não pode ser cancelado pela regra de cancelamento; o benefício não deve ser devolvido |
| T110 | Pedido premiado é do tipo `mesa` | Desconto aplicado ao pedido, não novamente no fechamento da comanda |
| T111 | Desconto de fidelidade supera o valor disponível no pedido | O total deve permanecer em zero, nunca negativo |
| T112 | Cliente completa novamente cinco pedidos entregues após um resgate válido | O próximo pedido elegível deve receber novo benefício |

## 14. Testes de precedência de erros

Estes testes devem enviar requisições que violem mais de uma regra ao mesmo tempo. O erro esperado é o da primeira validação aplicável.

| ID | Condições combinadas | Resultado esperado |
|---|---|---|
| T113 | Pedido com formato inválido e sabor indisponível | Erro de formato `422`, sem conflito de disponibilidade |
| T114 | Pedido de mesa com body inválido e comanda inexistente | Erro de formato `422` |
| T115 | Pedido de mesa com body válido, comanda inválida e sabor indisponível | `422`; erro `comanda_invalida` |
| T116 | Pedido com formato válido, sabor indisponível e cliente com pedido ativo | `409`; erro `sabor_indisponivel` |
| T117 | Pedido de entrega com formato válido, sabores disponíveis e cliente com pedido ativo | `409`; erro `pedido_em_aberto` |
| T118 | Fechamento com `gorjeta_percentual` inválido e comanda inexistente | `422`; erro `gorjeta_invalida` |
| T119 | Fechamento com `dividir_em` inválido e comanda já fechada | `422`; erro `divisao_invalida` |
| T120 | Fechamento válido de comanda inexistente | `404`; erro `comanda_nao_encontrada` |
| T121 | Fechamento válido de comanda fechada | `409`; erro `comanda_ja_fechada` |
| T122 | Fechamento válido de comanda com pedido ainda ativo | `409`; erro `comanda_com_pedidos_abertos` |

## 15. Testes de integridade e consistência

| ID | Cenário | Resultado esperado |
|---|---|---|
| T123 | Rejeitar pedido por erro de validação | Nenhum pedido ou item parcial deve ser criado |
| T124 | Rejeitar pedido por sabor indisponível | Não consumir fidelidade nem deixar itens gravados |
| T125 | Rejeitar pedido por cliente com pedido ativo | Não criar pedido nem consumir benefício |
| T126 | Rejeitar fechamento de comanda | A comanda deve permanecer no estado anterior |
| T127 | Fechar comanda com sucesso | Registrar `fechada_em`, estado fechado e liberar a mesa |
| T128 | Criar pedidos, avançar status, cancelar e consultar histórico | As consultas devem refletir os estados confirmados |
| T129 | Criar pedidos e consultar relatório diário | Os indicadores devem refletir pedidos elegíveis, descontos e cancelamentos |
| T130 | Criar pedidos de diferentes tipos para o mesmo cliente | A contagem de fidelidade deve ser consistente entre todos os tipos |
| T131 | Repetir consultas de cardápio, histórico e comandas após alterações | Os dados retornados devem refletir as alterações persistidas durante a execução |

## 16. Critério de conclusão

A suíte de testes será considerada satisfatória quando:

- Todos os cenários aplicáveis retornarem os códigos HTTP e os corpos previstos no contrato.
- Os cálculos monetários, percentuais, gorjetas, divisão da conta e fidelidade produzirem os resultados esperados.
- As regras de precedência forem respeitadas quando houver erros simultâneos.
- Os pedidos cancelados forem excluídos dos indicadores e da contagem de fidelidade.
- O relatório de um dia sem pedidos retornar zeros, sem erro.
- As operações rejeitadas não causarem efeitos parciais.
- As consultas e operações preservarem a consistência dos dados ao longo da execução.
- Os 16 casos de uso estiverem cobertos por cenários verificáveis.

A suíte descrita orienta a validação da implementação e não substitui os testes públicos ou ocultos fornecidos pela avaliação.

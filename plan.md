# Implementation Plan — API da Pizzaria

## 1. Objetivo do plano

Definir as decisões técnicas e a estratégia de implementação da API REST da pizzaria, transformando os requisitos de `constitution.md`, `spec.md` e `tests.md` em uma solução executável, testável e consistente com o contrato.

A implementação deve priorizar a compatibilidade com os testes de avaliação, a simplicidade arquitetural, a integridade dos dados e a cobertura dos 16 casos de uso.

Este documento orienta a geração da implementação. Ele não substitui os requisitos normativos nem deve conter código-fonte completo.

## 2. Referências e prioridade dos requisitos

A geração deve considerar os documentos nesta ordem de autoridade:

1. `ENUNCIADO.md` e `contrato.json`: contrato externo, endpoints, métodos HTTP, respostas, erros e regras de negócio.
2. `variante/params.json`: valores específicos da execução da prova.
3. `constitution.md`: regras operacionais e princípios que devem ser preservados.
4. `spec.md`: critérios de aceitação dos casos de uso.
5. `tests.md`: cenários de validação, limites e comportamentos esperados.
6. Este plano: decisões de arquitetura e organização interna.

Em caso de divergência, o contrato normativo e os parâmetros da variante prevalecem. Nenhuma decisão técnica pode modificar os endpoints, os campos ou os códigos de resposta exigidos.

## 3. Stack tecnológica

### 3.1. Linguagem e framework

**Decisão:** utilizar Python com FastAPI para implementar a API REST.

**Justificativa:**

- Python permite implementar regras de negócio e cálculos monetários de maneira direta.
- FastAPI oferece suporte à definição de rotas HTTP, validação de entradas e respostas JSON.
- A separação entre modelos de requisição, regras de negócio e persistência facilita a manutenção e os testes.
- A stack é adequada para um serviço pequeno, sem frontend e sem necessidade de infraestrutura distribuída.

A implementação deve declarar suas dependências em um arquivo próprio e utilizar versões compatíveis entre si.

### 3.2. Persistência

**Decisão:** utilizar SQLite como mecanismo de persistência local, sem depender de um servidor de banco de dados externo.

**Justificativa:**

- A solução precisa manter os dados entre requisições durante a execução do serviço.
- SQLite atende ao escopo da prova com baixa complexidade operacional.
- A persistência permite consultar históricos, acompanhar estados, consolidar comandas e calcular a fidelidade com base nos pedidos registrados.
- A inicialização deve preparar o banco de dados e garantir um estado inicial previsível para cada execução de avaliação.

O banco deve ser inicializado automaticamente. O serviço não deve depender de instalação ou configuração manual de infraestrutura externa.

### 3.3. Execução em container

**Decisão:** fornecer os arquivos necessários para iniciar o serviço em container, incluindo definição da imagem, dependências e instruções de execução.

**Justificativa:**

- A execução deve ser reproduzível em um ambiente de avaliação.
- A configuração reduz diferenças entre o ambiente de desenvolvimento e o ambiente de correção.
- O serviço deve escutar na porta indicada por `PORTA_SERVICO`, cujo valor nesta variante é `8002`.

A inicialização deve permitir que a suíte de correção acesse `http://localhost:8002`. Não exigir variáveis de ambiente obrigatórias que não estejam previstas no contrato.

## 4. Organização da arquitetura

A implementação deve adotar uma arquitetura simples em camadas, evitando abstrações desnecessárias.

### 4.1. Camada HTTP

Responsável por:

- Registrar todos os endpoints definidos no contrato.
- Receber parâmetros de rota, query strings e bodies JSON.
- Validar os formatos de entrada.
- Encaminhar as operações para os serviços de negócio.
- Converter resultados e erros para os códigos HTTP e os corpos JSON exigidos.

A camada HTTP não deve concentrar toda a lógica de cálculo ou persistência.

### 4.2. Camada de regras de negócio

Responsável por:

- Criar e atualizar pedidos.
- Controlar os estados do ciclo de vida.
- Aplicar descontos, taxas e benefícios de fidelidade.
- Gerenciar mesas e comandas.
- Controlar a disponibilidade do cardápio.
- Calcular fechamento, gorjeta e divisão da conta.
- Consolidar relatórios diários.

As regras devem ser implementadas de forma centralizada, para evitar comportamentos diferentes entre endpoints que compartilham a mesma regra.

### 4.3. Camada de persistência

Responsável por armazenar e consultar:

- Pedidos e seus itens.
- Clientes identificados pelo campo `cliente`, conforme o contrato.
- Histórico de estados e cancelamentos necessários às regras de negócio.
- Mesas e comandas.
- Sabores conhecidos e sua disponibilidade.
- Dados necessários para contabilizar e resgatar benefícios de fidelidade.

As operações relacionadas devem preservar a integridade dos dados. Uma requisição rejeitada não pode deixar registros parciais.

## 5. Modelo de dados

A estrutura interna deve representar explicitamente as entidades e seus relacionamentos.

### 5.1. Pedidos

Cada pedido deve registrar seu identificador, cliente, itens, tipo, comanda associada quando aplicável, data de recebimento, estado, subtotal, taxa, descontos e total.

Cada item deve registrar sabor, tamanho, quantidade e preço unitário em centavos. O preço unitário aplicado deve ser preservado no pedido para manter o histórico consistente.

### 5.2. Mesas e comandas

Uma mesa pode possuir várias comandas ao longo do tempo, mas somente uma comanda aberta por vez.

Uma comanda deve registrar identificador, número da mesa, abertura, fechamento e estado. Os pedidos associados devem manter seus próprios ciclos de vida.

### 5.3. Cardápio

Os sabores conhecidos devem possuir uma representação normalizada para comparação sem distinção entre maiúsculas e minúsculas e uma representação de exibição que preserve a capitalização do primeiro registro.

A disponibilidade deve ser mantida independentemente do histórico de pedidos, permitindo que a cozinha altere o estado de um sabor.

### 5.4. Fidelidade

O modelo deve permitir identificar:

- Quantos pedidos do cliente foram efetivamente entregues.
- Se o cliente possui um benefício disponível.
- Se um benefício foi resgatado em um pedido.
- Se o cancelamento permitido exige a devolução de um resgate.

A contagem deve considerar pedidos de entrega, balcão e mesa. Pedidos cancelados não contam como entregues.

## 6. Estratégia para cálculos monetários

### 6.1. Inteiros em centavos

**Decisão:** todos os valores monetários serão calculados e armazenados como inteiros em centavos.

**Justificativa:** isso evita erros de precisão associados a valores monetários em ponto flutuante e permite verificar a igualdade exata entre totais e parcelas.

Os preços unitários devem ser obtidos dos parâmetros da variante, sem duplicar valores fixos desnecessariamente na lógica de negócio.

### 6.2. Descontos e taxas

O subtotal deve resultar da soma de preço unitário multiplicado pela quantidade de cada item.

O desconto promocional deve incidir somente sobre o subtotal das pizzas, utilizando arredondamento para cima. A taxa de entrega deve ser aplicada somente ao tipo `entrega`.

O desconto de fidelidade deve ser aplicado ao pedido elegível durante sua criação. O total deve respeitar o piso zero depois da aplicação dos descontos e da taxa.

### 6.3. Gorjeta e divisão

A gorjeta deve ser calculada no fechamento da comanda, utilizando o percentual informado ou o valor padrão da variante quando ausente.

A divisão deve operar sobre o total final da conta, incluindo a gorjeta. O algoritmo deve produzir exatamente a quantidade solicitada de parcelas inteiras, distribuir o resto entre as primeiras parcelas e preservar a soma total.

### 6.4. Regras de arredondamento

O desconto promocional e a gorjeta utilizam `ceil`, arredondando para cima quando houver fração de centavo.

O ticket médio utiliza arredondamento para o inteiro mais próximo, com empates em 0,5 arredondados para cima.

Essas regras são distintas e não devem compartilhar uma estratégia de arredondamento genérica que produza resultados diferentes do contrato.

## 7. Datas e controle do tempo

**Decisão:** centralizar o tratamento de datas e horários em uma convenção única, usando ISO 8601 e o fuso contratual `-03:00`.

**Justificativa:** os pedidos podem receber datas explícitas para testes determinísticos, e o relatório diário depende da data de recebimento do pedido.

A implementação deve:

- Validar `recebido_em` quando informado.
- Utilizar o instante atual quando o campo estiver ausente.
- Registrar datas de abertura e fechamento das comandas.
- Calcular a data do relatório respeitando o fuso `-03:00`.
- Evitar misturar datas locais sem fuso com instantes que possuem fuso.
- Permitir que testes com datas explícitas sejam reproduzidos sem depender do horário real da execução.

## 8. Estratégia de validação e erros

**Decisão:** implementar as validações na ordem especificada no contrato, com respostas de erro centralizadas.

**Justificativa:** os testes podem enviar requisições que violem várias regras simultaneamente. Retornar um erro diferente do previsto pode reprovar o cenário mesmo quando todas as validações individuais existem.

### 8.1. Criação de pedidos

A ordem de validação deve ser:

1. Formato do body e dos campos.
2. Validade da comanda, quando aplicável.
3. Disponibilidade de todos os sabores solicitados.
4. Existência de pedido ativo do cliente, para entrega e balcão.

Somente após todas as verificações aplicáveis serem aprovadas o pedido deve ser persistido e o benefício de fidelidade efetivamente resgatado.

### 8.2. Fechamento de comandas

A ordem deve priorizar:

1. Validação do body, incluindo gorjeta e divisão.
2. Existência da comanda.
3. Estado da comanda e existência de pedidos ainda ativos.
4. Cálculo dos valores e confirmação do fechamento.

Os códigos HTTP e os identificadores de erro devem corresponder exatamente ao contrato.

## 9. Integridade das operações

**Decisão:** tratar cada operação de negócio como uma unidade consistente de alteração de dados.

**Justificativa:** erros de validação, cancelamentos e conflitos de estado não podem deixar o sistema parcialmente atualizado.

As seguintes operações exigem atenção especial:

- Criar pedido com vários itens: não persistir parte dos itens se algum sabor estiver indisponível.
- Resgatar fidelidade: não consumir o benefício antes de confirmar a validade do pedido.
- Cancelar pedido premiado: devolver o benefício quando o cancelamento em `recebido` for permitido.
- Atualizar status: não permitir transições incompatíveis com o ciclo de vida.
- Fechar comanda: calcular o total e alterar seu estado somente depois das validações.
- Reabrir mesa: permitir nova comanda somente após o fechamento válido da anterior.

Quando necessário, utilizar transações de banco de dados para garantir que as alterações relacionadas sejam confirmadas ou revertidas em conjunto.

## 10. Estratégia de implementação

A implementação deve ser desenvolvida nesta ordem para reduzir retrabalho e dependências entre funcionalidades.

### Etapa 1 — Estrutura e inicialização

- Configurar a aplicação e o framework HTTP.
- Preparar o banco de dados e as entidades.
- Configurar a porta da variante.
- Garantir que o serviço inicialize sem dependências externas não previstas.

### Etapa 2 — Pedidos e ciclo de vida

- Implementar criação e validação de pedidos.
- Implementar cálculo de subtotal, taxa e descontos.
- Implementar transições de status e cancelamento.
- Implementar listagens de pedidos ativos e histórico por cliente.
- Implementar a regra de um pedido ativo por cliente.

### Etapa 3 — Cardápio e fidelidade

- Implementar cadastro implícito e atualização de disponibilidade de sabores.
- Garantir comparação de sabores sem distinção entre maiúsculas e minúsculas.
- Implementar contagem de pedidos entregues, resgate e devolução de benefícios.

### Etapa 4 — Mesas e comandas

- Implementar abertura e consulta de comandas.
- Associar pedidos às comandas abertas.
- Implementar fechamento, gorjeta e divisão da conta.
- Garantir a liberação da mesa após o fechamento válido.

### Etapa 5 — Relatório diário

- Implementar filtro por data de recebimento.
- Excluir pedidos cancelados.
- Calcular contagem, faturamento e ticket médio.
- Garantir resposta zerada para dias sem pedidos.

### Etapa 6 — Validação final

- Conferir todos os endpoints e erros contra o contrato.
- Executar os cenários de `tests.md`.
- Verificar integridade dos dados e consistência entre operações.
- Confirmar a inicialização na porta correta e a execução em container.

## 11. Estratégia de testes

Os testes devem ser organizados em três níveis.

### 11.1. Testes unitários

Verificar isoladamente:

- Cálculos de subtotal, desconto, taxa e total.
- Arredondamento de percentuais.
- Cálculo do ticket médio.
- Divisão de valores em parcelas.
- Contagem, resgate e devolução da fidelidade.
- Validação de limites numéricos e textuais.

### 11.2. Testes de integração

Verificar o comportamento dos endpoints com persistência real durante a execução:

- Criação, consulta, atualização e cancelamento de pedidos.
- Disponibilidade de sabores.
- Abertura e fechamento de comandas.
- Associação de pedidos a mesas.
- Consistência dos relatórios após alterações.

### 11.3. Testes de contrato e cenários combinados

Verificar:

- Códigos HTTP e corpos de erro.
- Campos obrigatórios e opcionais.
- Ordenação das listagens.
- Precedência das validações.
- Ausência de efeitos parciais em operações rejeitadas.
- Casos de borda descritos em `tests.md`.

A solução não deve depender exclusivamente de testes manuais ou de exemplos de sucesso.

## 12. Execução e entrega

A implementação gerada deve incluir:

- Código organizado por responsabilidades.
- Arquivo de dependências.
- Arquivos necessários para execução em container.
- Inicialização automática do banco de dados.
- Instruções claras de execução e testes.
- Configuração de serviço compatível com a porta `8002`.

A API deve estar acessível em `http://localhost:8002`, conforme a variante, e responder de acordo com o contrato sem exigir configuração manual não documentada.

## 13. Restrições de implementação

- Não implementar frontend.
- Não alterar o contrato externo para simplificar a implementação.
- Não substituir centavos inteiros por valores monetários em ponto flutuante.
- Não ignorar a precedência de erros.
- Não criar endpoints adicionais como substitutos dos endpoints obrigatórios.
- Não retornar estruturas genéricas quando o contrato especificar campos ou erros.
- Não utilizar dados fixos de exemplo como se fossem resultados reais.
- Não excluir regras apenas porque seus parâmetros atuais estão configurados como zero.
- Não colar a implementação nos documentos Markdown; os arquivos devem especificar o comportamento esperado.

## 14. Critérios de conclusão

O plano será considerado executado quando:

- Todos os 16 casos de uso estiverem implementados.
- Os endpoints e os formatos de resposta corresponderem ao contrato.
- Os cálculos monetários e de fidelidade estiverem corretos.
- As validações e a precedência de erros forem respeitadas.
- As operações preservarem a integridade dos dados.
- Os testes definidos em `tests.md` forem executáveis e os cenários aplicáveis passarem.
- O serviço iniciar na porta configurada e estiver pronto para avaliação automatizada.

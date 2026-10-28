# API da Pizzaria — Especificação SDD

## 1. Objetivo

Este repositório contém a documentação de especificação de uma API para gerenciamento de pedidos de uma pizzaria, utilizando a abordagem **Spec-Driven Development (SDD)**.

O objetivo é definir os requisitos funcionais, as regras de negócio, os critérios de aceitação, os cenários de teste e o planejamento necessário para orientar a implementação da solução.

## 2. Documentação do projeto

Os documentos estão organizados da seguinte forma:

| Arquivo | Descrição |
|---|---|
| [`constitution.md`](constitution.md) | Define os princípios, as regras obrigatórias e as diretrizes do projeto. |
| [`spec.md`](spec.md) | Descreve os requisitos funcionais, os casos de uso e os critérios de aceitação da API. |
| [`plan.md`](plan.md) | Apresenta a arquitetura proposta, as decisões técnicas e as etapas de implementação. |
| [`tests.md`](tests.md) | Define os cenários de teste para verificar o atendimento aos requisitos e às regras de negócio. |
| [`tasks.md`](tasks.md) | Organiza as tarefas de desenvolvimento e suas dependências. |
| [`FONTES.md`](FONTES.md) | Registra as fontes consultadas e as declarações de uso de inteligência artificial. |

## 3. Escopo funcional

A especificação contempla as seguintes funcionalidades:

- Criação, consulta, atualização de status e cancelamento de pedidos.
- Consulta de pedidos ativos e histórico por cliente.
- Validação de itens e disponibilidade dos sabores no cardápio.
- Aplicação de descontos promocionais e descontos de fidelidade.
- Gerenciamento de mesas e comandas.
- Fechamento de comandas, cálculo de gorjetas e divisão de valores.
- Geração de relatório diário de faturamento.

## 4. Abordagem de desenvolvimento

A documentação segue a abordagem **Spec-Driven Development (SDD)**, na qual os requisitos e as regras de negócio são definidos antes da implementação.

Os documentos devem ser utilizados de forma complementar, mantendo a consistência entre os requisitos, os testes, o planejamento e as tarefas.

Em caso de divergência, devem prevalecer o enunciado e o contrato oficial da atividade, conforme as orientações da seção seguinte.

## 5. Instruções para o agente gerador

1. Leia os documentos na seguinte ordem: `constitution.md`, `spec.md`, `plan.md`, `tests.md` e `tasks.md`.
2. Em caso de conflito, prevalecem `ENUNCIADO.md`, `contrato.json` e `variante/params.json`. O exemplo do relatório com `faturamento: 123.40` é inválido.
3. Gere o projeto completo, incluindo:
   - Código da API.
   - Testes próprios.
   - `Containerfile`.
   - Arquivo de dependências.
   - README com instruções de execução.
   - `.gitignore` e `.dockerignore`.
   - Configuração de linter.
   - Nenhum segredo armazenado no repositório.
4. O serviço deve escutar na porta `8002`, utilizar valores monetários em centavos inteiros e respeitar a precedência dos erros `422` antes dos erros `409`.

## 6. Referências e transparência

As fontes utilizadas na elaboração da documentação e as consultas de inteligência artificial estão registradas em [`FONTES.md`](FONTES.md).

A implementação deve respeitar os requisitos e o contrato oficial da atividade, e o conteúdo entregue deve ser compreendido e revisado antes da submissão.

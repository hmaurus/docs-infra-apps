# Contexto

Resumo do Que estamos fazendo e Status Atual

Objetivo: Implementar o primeiro módulo (experimento) do aplicativo Lab App. O LabApp vai seguir o Clean Architecture, começando como um monolito modular preparado para futura migração para microsserviços.

Onde Estamos:

1. Criamos a estrutura base de diretórios seguindo Clean Architecture:

lab-app/backend/src/

├── domain/entities/experimento/

├── application/experimento/{commands,queries,dtos}/

├── infrastructure/repositories/

└── interfaces/http/

2. Implementamos a primeira entidade de domínio:

* Arquivo: src/domain/entities/experimento/experimento.entity.ts

* Contém a entidade Experimento com suas regras de negócio

* Implementa padrão Either para tratamento de erros

* Define estados e transições válidas

1. Hoje (Monolito Modular):

* Tudo roda em um único processo

* Módulos bem definidos e isolados

* Comunicação via memória (mais rápida)

* Um único banco de dados (mas schemas separados)

2. Futuro (Microsserviços):

* Cada módulo vira um serviço

* Comunicação via rede (HTTP/gRPC)

* Bancos de dados separados

* Deploy independente

A migração será facilitada porque:

1. Domínio já está isolado

2. Interfaces já definem contratos claros

3. Eventos já são usados para comunicação

4. Dados já estão logicamente separados
# 02. Testes de Integração (Integration Testing)

Os testes de integração verificam a comunicação e o fluxo de dados entre os diferentes módulos do Julgador Automático.

---

## 2.1 Integração Não Incremental (Big Bang)

### 1. Diagrama de Sequência UML

```mermaid
sequenceDiagram
    autonumber
    actor Usuario as Usuário/Atleta
    participant API as SubmissionAPI
    participant Queue as QueueManager
    participant Sandbox as SandboxRunner
    participant Evaluator as VerdictEvaluator
    participant DB as Database

    Usuario->>API: Envia código C++ (POST /submit)
    API->>Queue: Enfileira submissão (push task)
    Queue->>Sandbox: Consome submissão da fila
    Note over Sandbox: Compila e executa o código em container isolado
    Sandbox->>Evaluator: Envia métricas (tempo/memória) e saídas brutas (stdout)
    Note over Evaluator: Compara a saída com o gabarito oficial
    Evaluator->>DB: Persiste veredito final (AC, WA, TLE, etc.)
    DB-->>API: Notifica conclusão do processamento
    API-->>Usuario: Retorna status e veredito final
```
* **Contexto:** Na abordagem Big Bang, todos os subsistemas do Julgador Automático (`SubmissionAPI`, `QueueManager`, `SandboxRunner`, `VerdictEvaluator` e `Database`) são acoplados e testados simultaneamente de ponta a ponta, sem a criação prévia de *stubs* ou *drivers* para isolamento.

* **Como a abordagem é aplicada:**
  * Uma requisição HTTP simulando o envio de um arquivo C++ é disparada na `SubmissionAPI`.
  * A API repassa a tarefa diretamente para o `QueueManager`, que avisa o `SandboxRunner`.
  * O `SandboxRunner` executa o binário em ambiente isolado e envia as métricas de tempo/memória junto ao `stdout` para o `VerdictEvaluator`.
  * O resultado é gravado no `Database` e retornado ao usuário.

* **Objetivo do Teste e Defeitos Revelados:**
  * **Objetivo:** Validar o fluxo completo da aplicação em um único ciclo e verificar a aderência dos módulos aos contratos de interface.
  * **Incompatibilidades de Payload/Tipagem:** Detecta divergências de formato de dados trafegados entre serviços (ex: IDs passados como `string` pela API e esperados como `integer` pela fila).
  * **Dessincronização de Timeouts:** Revela se o tempo de resposta e retenção da mensagem no `QueueManager` é incompatível com o tempo de compilação/execução no `SandboxRunner`.
  * **Dificuldade de Isolamento:** Demonstra a principal limitação da abordagem: caso ocorra um erro genérico durante o teste, é complexo rastrear se a falha se deu no transporte de dados, na execução do sandbox ou no banco de dados.

## 2.2 Integração Incremental Top-Down (Descendente) com uso de Stubs

### 1. Diagrama de Classes UML

```mermaid
classDiagram
    class SubmissionController {
        +submitCode(userId, problemId, code) SubmissionResponse
    }

    class IQueueManager {
        <<interface>>
        +pushToQueue(payload) Boolean
    }

    class QueueManagerStub {
        <<stub>>
        +pushToQueue(payload) Boolean
        +simularFilaCheia() void
    }

    class IDatabase {
        <<interface>>
        +saveSubmission(data) String
    }

    class DatabaseStub {
        <<stub>>
        +saveSubmission(data) String
    }

    SubmissionController --> IQueueManager : utiliza
    SubmissionController --> IDatabase : utiliza
    QueueManagerStub ..|> IQueueManager : implementa
    DatabaseStub ..|> IDatabase : implementa
```
### 2. Explicação Textual do Cenário

* **Contexto:** Na abordagem Top-Down, a verificação começa no componente de mais alto nível da hierarquia (`SubmissionController`), que gerencia o recebimento de códigos C++. Como os serviços de fila e banco de dados podem estar indisponíveis ou em desenvolvimento, utilizamos *Stubs* para simular o comportamento dessas camadas inferiores.

* **Como a abordagem é aplicada:**
  * O controlador `SubmissionController` é instanciado em ambiente de teste real.
  * Injetam-se as instâncias de `QueueManagerStub` e `DatabaseStub` no construtor do controlador através de suas interfaces (`IQueueManager` e `IDatabase`).
  * O teste dispara chamadas para `submitCode()` e os *Stubs* retornam respostas pré-programadas em memória (ex: status de enfileiramento positivo e IDs de submissão sintéticos), permitindo exercitar a lógica do controlador sem dependências externas.

* **Objetivo do Teste e Defeitos Revelados:**
  * **Objetivo:** Garantir que o fluxo de decisão e roteamento da camada superior funcione perfeitamente antes do desenvolvimento completo da infraestrutura de baixo nível.
  * **Falhas no Tratamento de Erros da API:** Permite testar como o controlador reage quando dependências inferiores falham (ex: configurando `simularFilaCheia()` no *Stub* para garantir que a API retorne código HTTP 503 adequadamente).
  * **Incompatibilidade de Contrato de Interface:** Revela se os métodos chamados pelo controlador divergem das assinaturas definidas nas interfaces das dependências.

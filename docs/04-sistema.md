# 04. Testes de Sistema (System Testing)

Os testes de sistema avaliam a solução completa e integrada sob requisitos não funcionais rigorosos de infraestrutura, resiliência, segurança e alto desempenho.

---

## 4.1 Teste de Recuperação (Recovery Testing)

### 1. Diagrama de Sequência UML

```mermaid
sequenceDiagram
    autonumber
    actor User as Competidor
    participant App as API / Load Balancer
    participant DB1 as Banco Primário (Master)
    participant DB2 as Banco Secundário (Replica)

    User->>App: Consulta status da submissão (GET /status)
    App->>DB1: Leitura de dados do veredito
    Note over DB1: ❌ Queda Abrupta (Crash / Hardware Fault)
    DB1--xApp: Timeout / Conexão recusada
    Note over App: Mecanismo de Failover detecta falha e redireciona
    App->>DB2: Promove réplica / Redireciona consulta
    DB2-->>App: Retorna dados do veredito com sucesso
    App-->>User: Retorna status 200 OK sem interrupção perceida

```
### 2. Explicação Textual do Cenário

* **Contexto:** O Teste de Recuperação avalia a capacidade do Julgador Automático de tolerar falhas físicas ou de software severas em ambiente de produção, garantindo que o sistema restabeleça sua operação normal e mantenha a integridade dos dados sem impacto perceptível ao usuário final.

* **Como a abordagem é aplicada:**
  * Durante uma bateria ativa de julgamento de submissões, força-se uma falha crítica de infraestrutura (como a interrupção abrupta do contêiner do banco de dados primário ou o encerramento do processo do servidor).
  * O mecanismo de monitoramento e *failover* (balanceador de carga / orquestrador de contêineres) deve detectar a perda de conectividade com o nó primário e reorientar o tráfego em milissegundos para um nó secundário/réplica.
  * O teste checa se as requisições dos competidores continuam sendo atendidas e se submissões em andamento não foram corrompidas ou perdidas na transição.

* **Objetivo do Teste e Defeitos Revelados:**
  * **Objetivo:** Verificar o RTO (*Recovery Time Objective*) e o RPO (*Recovery Point Objective*) do sistema diante de falhas de hardware e rede.
  * **Incapacidade de Failover Automático:** Revela se a perda do nó primário trava toda a API do julgador exigindo intervenção manual da equipe de infraestrutura.
  * **Inconsistência de Estado/Perda de Dados:** Detecta se submissões processadas imediatamente antes da queda sumiram da base de dados ou ficaram presas em estado indeterminado (ex: mantidas como "Em Processamento" indefinidamente).
 
## 4.2 Teste de Segurança (Security Testing)

### 1. Diagrama de Sequência UML

```mermaid
sequenceDiagram
    autonumber
    actor Attacker as Atacante / Malicioso
    participant API as SubmissionAPI
    participant Sec as Security/Sandbox Shield
    participant OS as Sistema Operacional (Hospedeiro)

    Attacker->>API: Submete C++ com chamada 'system("cat /etc/passwd")'
    API->>Sec: Envia binário para execução isolada
    Note over Sec: Inspeciona syscalls via seccomp / cgroups
    Sec->>OS: Tenta executar chamada proibida (SYS_execve)
    OS-->>Sec: Bloqueia operação (Permission Denied)
    Sec-->>API: Registra violação de segurança
    API-->>Attacker: Retorna veredito 'Restricted System Call / Security Violation'
```
### 2. Explicação Textual do Cenário

* **Contexto:** O Teste de Segurança valida se o Julgador Automático é capaz de neutralizar tentativas de ataques maliciosos embarcados nos códigos submetidos pelos usuários, garantindo a proteção dos dados do sistema hospedeiro, dos gabaritos fechados e da rede interna.

* **Como a abordagem é aplicada:**
  * Submetem-se binários compilados em C++ contendo instruções maliciosas intencionais, tais como: leitura de arquivos sensíveis do SO hospedeiro, alocação abusiva de processos (*fork bomb*), abertura de sockets para conexões externas não autorizadas ou acesso a arquivos de teste de outros problemas.
  * O teste avalia se as camadas de isolamento (*seccomp profiles*, namespaces e *Linux cgroups*) interceptam as chamadas de sistema restritas (*syscalls*) e abortam a execução do programa imediatamente.

* **Objetivo do Teste e Defeitos Revelados:**
  * **Objetivo:** Garantir a confidencialidade, integridade e isolamento absoluto do ambiente do julgador contra execuções arbitrárias de código.
  * **Brechas no Isolamento de Processos (Sandbox Escape):** Revela se permissões do ambiente do container estão frouxas, permitindo escalada de privilégios ou navegação pela árvore de diretórios do servidor.
  * **Vazamento de Gabaritos:** Detecta se o código do usuário consegue acessar os caminhos de memória ou arquivos temporários do servidor onde ficam armazenadas as saídas oficiais (*output files*) do problema durante a validação.
 
## 4.3 Teste de Estresse (Stress Testing)

### 1. Diagrama de Sequência UML

```mermaid
sequenceDiagram
    autonumber
    actor Gen as Gerador de Carga (k6 / Locust)
    participant API as SubmissionAPI
    participant Queue as Fila (Redis)
    participant Workers as Sandbox Workers Cluster

    Gen->>API: Injeta 5.000 requisições/seg (Pico Extremo)
    API->>Queue: Enfileira submissões rapidamente
    Note over Queue: Fila atinge 95% da capacidade do buffer
    API-->>Gen: Retorna 202 Accepted (Em Fila)
    Note over Workers: Workers operam a 100% de CPU/Memória sem travar
    Gen->>API: Cessa a sobrecarga
    Queue->>Workers: Drena requisições acumuladas gradualmente
    Note over Queue: Sistema retorna ao estado estável normal
```
### 2. Explicação Textual do Cenário

* **Contexto:** O Teste de Estresse avalia o comportamento do Julgador Automático em condições limite de carga, excedendo os parâmetros normais de operação (ex: disparos massivos de submissões no início ou final de maratonas de programação), para identificar o ponto de quebra da infraestrutura e verificar a capacidade de autorrecuperação.

* **Como a abordagem é aplicada:**
  * Ferramentas de geração de carga (como k6 ou Locust) injetam milhares de requisições concorrentes de submissão em um curto intervalo de tempo.
  * O teste força o esgotamento dos recursos do servidor (CPU, memória RAM, conexões do banco de dados e capacidade do buffer da fila).
  * Avalia-se se o sistema aplica *degradação graciosa* (mantendo requisições em fila com status `202 Accepted`) ou se sofre queda catastrófica (*crash* sem resposta).

* **Objetivo do Teste e Defeitos Revelados:**
  * **Objetivo:** Determinar o limite máximo de escalabilidade do julgador e garantir que o sistema recupere seu estado normal assim que a sobrecarga for removida.
  * **Vazamentos de Memória sob Alta Carga (Memory Leaks):** Revela se contêineres de execução do *Sandbox* não destruídos adequadamente acumulam memória no servidor até paralisar o sistema operacional.
  * **Esgotamento do Pool de Conexões:** Identifica gargalos de banco de dados ou mensageria que causam erros fatais (`HTTP 500`) em vez de enfileiramento controlado.
 

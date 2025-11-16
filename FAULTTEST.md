## Teste de Tolerância a Falhas no Hadoop

Além do benchmark de desempenho, foi realizado um experimento específico para avaliar o comportamento do Hadoop/HDFS e do YARN diante de falhas em componentes do cluster (DataNodes e NameNode) durante a execução de um job **WordCount** de longa duração sobre o arquivo `/jobs/big.txt` (61 splits).

> **Importante:** assim como nos demais testes, todos os serviços (NameNode, DataNodes, ResourceManager, NodeManagers) estão em **containers rodando em um único host físico**. Isso significa que as falhas simuladas são de processos/containers (e não de hardware real ou rede física). Os resultados são válidos para fins **didáticos**, mas **não representam fielmente** todas as nuances de um cluster distribuído em produção.

### Metodologia do Experimento

O teste seguiu as seguintes etapas:

1. Remoção da saída anterior e submissão do job WordCount com o comando:  
   ``` bash
   hdfs dfs -rm -r -f /jobs-out-baseline  
   hadoop jar "$EX" wordcount /jobs/big.txt /jobs-out-baseline
   ```
2. Aguardar o job atingir aproximadamente **10% de progresso em map**.

3. Simular falhas de componentes na seguinte ordem:
   - Derrubar **apenas o DataNode 1 (dn1)**.  
   - Derrubar **também o DataNode 2 (dn2)**, deixando o cluster sem DataNodes disponíveis.  
   - Subir novamente os dois DataNodes.  
   - Derrubar o **NameNode**, com todos os DataNodes já estáveis.  
   - Subir novamente o NameNode.

4. Observar o comportamento do job: congelamento, erros emitidos, retomada e avanço do progresso.

5. Verificar se a tarefa finaliza com sucesso após as falhas.

### Observações Experimentais

A tabela a seguir resume o comportamento observado em cada evento de falha:

| Etapa | Ação realizada                             | Estado do job / progresso | Comportamento observado                                                                                      |
| ----: | ------------------------------------------ | ------------------------- | ------------------------------------------------------------------------------------------------------------ |
|     1 | Job submetido normalmente                  | map 0% → 10%              | Execução normal, avanço gradual sem erros aparentes.                                                         |
|     2 | Queda do **dn1** (um DataNode fora)        | ~map 10–20%               | Job continua executando, porém mais lento; não ocorre falha imediata.                                        |
|     3 | Queda do **dn2** (ambos DataNodes offline) | ~map 23%                  | Execução congela; surgem erros `BlockMissingException` devido à ausência de réplicas disponíveis.            |
|     4 | Subida de **dn1** e **dn2**                | ~map 23% → 30%+           | Job imprime avisos de falha, mas **retoma** a execução a partir do ponto em que estava.                      |
|     5 | Queda do **NameNode**                      | progresso em map/reduce   | Job congela imediatamente; são emitidos erros relacionados ao acesso aos metadados do HDFS.                  |
|     6 | Subida do **NameNode**                     | mesmo progresso inicial   | Job é retomado; progresso avança normalmente após o retorno do serviço.                                      |
|     7 | Finalização do job                         | 100% map / 100% reduce    | Apesar das falhas, a tarefa **finaliza com sucesso**, com várias tentativas marcadas como FAILED/RELAUNCHED. |

Durante a queda dos DataNodes, múltiplas tasks Map falharam com exceções `BlockMissingException`, indicando diretamente que nenhum nó vivo continha os blocos necessários para continuar lendo partes do arquivo de entrada. Após a recuperação dos DataNodes, as tasks foram reexecutadas normalmente.

### Análise Crítica do Comportamento

#### 1. Falha parcial de DataNode (dn1)

Quando apenas **um** DataNode é derrubado, o job continua rodando — ainda que mais lentamente. Isso ocorre porque:

- Havia réplicas acessíveis nos demais DataNodes.
- O NameNode simplesmente redirecionou as leituras.
- Algumas tasks perderam localidade de dados e precisaram ler remotamente.

Essa situação é coerente com o funcionamento esperado do HDFS: a perda de um DataNode degrada performance, mas **não interrompe a execução** desde que existam réplicas disponíveis.

#### 2. Falha total de DataNodes (dn1 + dn2)

Com ambos os DataNodes fora do ar:

- O NameNode não consegue localizar réplicas vivas dos blocos necessários.
- O cliente HDFS lança `BlockMissingException` repetidamente.
- O job congela por volta de 23% de map.
- Tasks são marcadas como FAILED devido à impossibilidade de leitura.

Quando os DataNodes voltam ao ar:

- O NameNode detecta novamente os blocos.
- YARN reroda as tasks necessárias.
- O job volta a avançar normalmente.

Isso mostra corretamente que:

> O Hadoop tolera falhas temporárias nos DataNodes, desde que eventualmente os dados voltem a ficar disponíveis.

Se essa falha ocorresse em um ambiente real sem réplicas suficientes, o job não se recuperaria.

#### 3. Falha do NameNode

A queda do NameNode causa **congelamento imediato** da execução, mesmo com todos os DataNodes ativos. Isso acontece porque:

- O NameNode é responsável por todos os metadados, incluindo localização de blocos.
- Nenhuma operação de leitura, escrita ou abertura de splits pode avançar sem ele.
- O job apenas aguarda o NameNode voltar.

Após a subida do NameNode, o job retoma do ponto exato em que havia parado.

Isso demonstra que, nesta configuração:

> O NameNode é um **ponto único de falha (SPOF)** — comportamento esperado em setups sem HA.

#### 4. Consistência com o modelo Hadoop

Os resultados observados são absolutamente coerentes com a arquitetura do Hadoop:

- Falhas de tasks são esperadas e toleradas: o YARN as reexecuta automaticamente.
- DataNodes podem cair temporariamente sem interromper permanentemente um job.
- O NameNode é essencial; sem ele, o cluster fica “cego”.
- A recuperação depende da disponibilidade futura dos dados.

#### 5. Limitações do ambiente de laboratório

Como todos os serviços rodam dentro de **containers em um único host**, há limitações importantes:

- A latência entre nós é artificialmente baixa (rede virtual).
- Perda de localidade de dados não gera o mesmo impacto que num cluster distribuído real.
- A queda de um DataNode não envolve perda de disco real nem falhas complexas.
- A recuperação é mais rápida e previsível que em hardware distribuído.

Ainda assim, o teste demonstra de forma prática e clara:

- A resiliência do Hadoop a falhas temporárias.
- A maneira como erros de leitura de blocos se manifestam.
- A importância crítica do NameNode.
- A eficácia do YARN em reexecutar tasks automaticamente.

O experimento confirma que o Hadoop consegue **garantir continuidade e confiabilidade** mesmo sob falhas parciais — desde que os serviços críticos retornem dentro de um intervalo razoável de tempo.

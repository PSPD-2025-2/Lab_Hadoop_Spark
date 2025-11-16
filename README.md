# Relatório da Atividade Extraclasse – PSPD:

# Experimentos com Hadoop e Spark

- **Curso:** Engenharia de Software – UnB
- **Disciplina:** Programação de Sistemas Paralelos e Distribuídos (PSPD)
- **Professor:** Fernando W. Cruz
- **Integrantes do Grupo:** Gabryel Nicolas Soares de Sousa; Guilherme Westphall de Queiroz; Joel Soares Rangel; Lucas Martins Gabriel
- **Data:** 15 de Novembro de 2025

> Para informações sobre a configuração do ambiente Hadoop e Spark utilizados, consulte os arquivos `hadoop-lab/README.md` e `spark-lab/README.md` neste repositório.

## Introdução

Este relatório descreve uma atividade prática extraclasse da disciplina PSPD (Programação de Sistemas
Paralelos e Distribuídos), na qual exploramos dois principais frameworks de Big Data: **Apache Hadoop**
e **Apache Spark**. O objetivo foi identificar na prática as características de cada plataforma, através de
dois experimentos: (1) montagem e teste de um _cluster_ Hadoop para processamento distribuído
(aplicação **MapReduce** clássica de contagem de palavras, **WordCount** ), incluindo avaliações de
performance e tolerância a falhas; e (2) desenvolvimento de uma solução de fluxo de dados em **tempo
real** utilizando Spark Structured Streaming, integrando múltiplas ferramentas (Discord, Apache Kafka,
ElasticSearch, Kibana) para coletar dados de uma rede social e exibí-los em um dashboard.

No experimento com **Hadoop** , construímos um cluster completo usando containers Docker para
simular múltiplos nós (NameNode, DataNodes, etc.) e executamos jobs MapReduce sobre o HDFS,
medindo o desempenho sob diferentes configurações e simulando falhas de nós durante a execução.
Em seguida, no experimento com **Spark** , implementamos uma arquitetura de processamento de
_streaming_ onde mensagens de um canal do **Discord** são enviadas a um tópico **Kafka** , processadas por
um aplicativo Spark em _stream_ (contagem de palavras em janela temporal) e os resultados são enviados
a outro tópico Kafka, do qual são consumidos e indexados no **ElasticSearch** via **Kafka Connect** ,
permitindo visualização em tempo real de uma nuvem de palavras no **Kibana**.

## Experimento com Apache Hadoop

### Arquitetura do Cluster Hadoop com Docker Compose

Para o experimento com Hadoop, montamos um _cluster_ multi-nó utilizando **Docker Compose** , o que
nos permitiu criar múltiplos contêineres simulando os papéis distintos dentro de um ecossistema
Hadoop. A arquitetura final consistiu em **8 contêineres Docker** interconectados em uma mesma rede
virtual , conforme descrito na tabela abaixo:

| **Contêiner** | **Papel no Hadoop**                                                                                                                 | **Portas Expostas**                 |
| ------------- | ----------------------------------------------------------------------------------------------------------------------------------- | ----------------------------------- |
| `nn`          | **NameNode** – Nó mestre do HDFS, gerencia o sistema de arquivos distribuído.                                                       | 9870 (UI NameNode), 9000 (RPC HDFS) |
| `dn1`, `dn2`  | **DataNode** – Nós de dados do HDFS, armazenam blocos dos arquivos distribuídos.                                                    | 9864 (DataNode)                     |
| `rm`          | **ResourceManager** – Nó mestre do YARN, coordena o agendamento de aplicações/containers.                                           | 8088 (UI RM), 8032 (RPC YARN)       |
| `nm1`, `nm2`  | **NodeManager** – Nós trabalhadores do YARN, executam tasks (mappers/reducers) em contêineres.                                      | 8042 (UI NM)                        |
| `jhs`         | **JobHistoryServer** – Serviço de histórico de jobs MapReduce finalizados.                                                          | 19888 (UI JHS)                      |
| `edge`        | **Edge/Client Node** – Nó de borda para acesso dos clientes. Usado para executar comandos `hdfs`, `yarn`, `mapred` e submeter jobs. | (sem porta específica)              |

Execução de Jobs MapReduce (WordCount)

Com o cluster configurado, partimos para a execução de um job MapReduce clássico: a contagem de palavras (WordCount). Inicialmente, realizamos um teste básico para validar o funcionamento do ambiente: criamos um pequeno arquivo de texto e o processamos com o WordCount. Dentro do contêiner edge, executamos o 

```bash
# Cria arquivo de teste e envia ao HDFS
printf "um\ndois\ndois\ntres tres tres\n" > /tmp/input.txt  
hdfs dfs -mkdir -p /jobs  
hdfs dfs -put -f /tmp/input.txt /jobs/input.txt  

# Executa WordCount usando o jar de exemplos do Hadoop
EXAMPLE_JAR=$(ls $HADOOP_HOME/share/hadoop/mapreduce/hadoop-mapreduce-examples-*.jar | head -n1)  
OUT_DIR=/jobs-out-$(date +%s)  
$HADOOP_HOME/bin/hadoop jar "$EXAMPLE_JAR" wordcount /jobs/input.txt "$OUT_DIR"  

# Verifica resultado
hdfs dfs -cat "$OUT_DIR/part-r-00000"
```

A saída obtida do job WordCount foi:

```
dois    2  
tres    3  
um      1  
```

## Teste de performance

Com o ambiente validado, prosseguimos para o experimento principal de performance e tolerância a falhas, utilizando um volume de dados grande para o WordCount. Conforme a orientação da atividade, preparamos um arquivo de texto de tamanho suficientemente grande para manter o cluster ocupado por vários minutos. Para isso, geramos um arquivo sintético diretamente no contêiner edge usando o comando yes do Linux, que repete indefinidamente uma string. Geramos 300 milhões de linhas (~8 GB de texto) com conteúdo aleatório repetido, por exemplo:

```bash
yes "lorem ipsum dolor sit amet" | head -n 300000000 > /tmp/big.txt  
hdfs dfs -put /tmp/big.txt /jobs/big.txt
```

Ao submeter o job, acompanhamos sua execução tanto via linha de comando (log do Hadoop MapReduce) quanto pela interface web do ResourceManager (localhost:8088). O Hadoop dividiu o trabalho de map-reduce em uma quantidade significativa de map tasks (devido ao arquivo massivo ser particionado em muitos blocos HDFS de 128 MB) e algumas reduce tasks. Observamos na UI do YARN que os mappers foram distribuídos entre os dois NodeManagers (nm1 e nm2), executando em paralelo conforme os recursos permitiam. Cada NodeManager pôde rodar múltiplos containers de map em paralelo (limitado pela configuração de núcleos/memória que definimos). Durante a fase shuffle e reduce, os resultados intermediários foram gerenciados pelo ApplicationMaster, e enfim a contagem final de palavras foi escrita no HDFS no diretório de saída especificado.

Após a conclusão, validamos o resultado contando o total de linhas no output (part-r-00000). Como esperado, dado que todas as linhas continham a mesma frase, o wordcount resultou na contagem dessa frase ou de cada palavra dela. Por exemplo, presumindo que cada linha tinha 5 palavras, o output deveria conter “lorem” com 300 milhões de ocorrências, e assim por diante para cada palavra distinta na frase.

## Benchmark

A Tabela a seguir resume os principais parâmetros e métricas observadas em cada execução.
| Cenário                | Job ID                 | Replicação | Blocksize | Memória NM | Tempo (s) | Throughput (MB/s) | CPU total (s) | GC total (s) | Memória física snapshot (GB)\* |
| ---------------------- | ---------------------- | ---------- | --------- | ---------- | --------- | ----------------- | ------------- | ------------ | ------------------------------ |
| **Baseline**           | job_1763235544715_0002 | 1          | 128 MB    | 2048 MB    | 598,616   | **12,90**         | ≈ 1843,6      | ≈ 87,7       | ≈ 19,0                         |
| **2 réplicas**         | job_1763236505643_0001 | 2          | 128 MB    | 2048 MB    | 396,503   | **19,48**         | ≈ 1833,2      | ≈ 84,3       | ≈ 19,2                         |
| **Blocksize = 64 MB**  | job_1763237020860_0001 | 1          | 64 MB     | 2048 MB    | 374,766   | **20,61**         | ≈ 1853,1      | ≈ 93,7       | ≈ 19,1                         |
| **Blocksize = 256 MB** | job_1763237525385_0001 | 1          | 256 MB    | 2048 MB    | 382,375   | **20,20**         | n/d           | n/d          | n/d                            |
| **NodeManager = 4 GB** | job_1763242547198_0001 | 1          | 128 MB    | 4096 MB    | 373,000   | **20,71**         | ≈ 1872,3      | ≈ 83,2       | ≈ 17,4                         |


- **Tempo (s):** conversão direta do tempo reportado (`minutos + segundos`).
- **Throughput (MB/s):** aproximado com base em ≈ 7725 MB processados (8,1 bilhões de bytes / 1024²) divididos pelo tempo total.
- **CPU total (s):** soma do tempo de CPU em milissegundos (`CPU_MILLISECONDS`) convertido para segundos.
- **GC total (s):** soma do tempo de coleta de lixo (`GC_TIME_MILLIS`) em segundos.
- **Memória física snapshot (GB):** valor do contador `PHYSICAL_MEMORY_BYTES` convertido para gigabytes.


### Análise dos Resultados

#### Tempo de execução e throughput

Em termos de tempo de execução, o cenário **baseline** é o mais lento (≈ 598 s), enquanto os demais convergem para uma faixa entre **373 s e 396 s**. Isso corresponde a um ganho de desempenho entre **~34% e ~38%** em relação ao baseline.

O throughput segue a mesma tendência:

- **Baseline:** ~12,9 MB/s  
- **Demais cenários:** entre ~19,5 e ~20,7 MB/s

Ou seja, **qualquer ajuste** (replicação 2, mudança de blocksize ou aumento de memória do NodeManager) já coloca o sistema em um patamar de throughput significativamente maior do que a configuração inicial.

#### Fator de replicação

Comparando **Baseline (replicação = 1, 128 MB)** com **2 réplicas (replicação = 2, 128 MB)**:

- O throughput sobe de **12,90 MB/s** para **19,48 MB/s**.
- O tempo total cai de **598,6 s** para **396,5 s**, uma redução de **~34%**.
- O volume de dados lidos do HDFS permanece o mesmo (≈ 8,1 GB em todos os casos), assim como o número de registros de entrada e saída.

Os contadores de CPU e GC são muito próximos entre os dois cenários (diferenças de poucos segundos no total), o que indica que:

> O custo de CPU do job é essencialmente o mesmo; o que muda é **quão rápido o cluster consegue alimentar os mapas com dados**, graças à maior disponibilidade de réplicas para leitura local (melhor data locality e menor dependência de rede).

Mesmo num ambiente de containers compartilhando o mesmo disco, esse efeito aparece claramente nos tempos de execução.

#### Tamanho de bloco (blocksize)

Os três cenários com replicação 1 e blocksize variando:

- **64 MB:** 374,8 s (20,61 MB/s)
- **128 MB (baseline):** 598,6 s (12,90 MB/s)
- **256 MB:** 382,4 s (20,20 MB/s)

Aqui acontecem duas coisas ao mesmo tempo:

1. **Blocksize muito pequeno (64 MB)**  
   - Aumenta o número de splits, logo aumenta também o número potencial de tarefas Map (mais paralelismo).  
   - Isso melhora o balanceamento entre containers, reduzindo o tempo de fila e ociosidade de recursos.  
   - O custo agregado de CPU não diminui (pelo contrário, o cenário de 64 MB é até ligeiramente mais “caro” em CPU total do que o baseline), mas o *tempo de parede* (elapsed) cai bastante, pois o trabalho é melhor distribuído.

2. **Blocksize maior (256 MB)**  
   - Reduz o número de splits e de tarefas Map.  
   - Ainda assim, neste ambiente de laboratório, o resultado foi **melhor que o baseline** e próximo ao cenário 64 MB.  
   - A pequena diferença entre 64 MB e 256 MB sugere que, com apenas alguns containers competindo no mesmo host, o limite de paralelismo já foi atingido e o sistema não consegue explorar totalmente a granularidade extra de 64 MB.

O contraste forte é entre **baseline (128 MB original)** e os cenários ajustados. Isso indica que a configuração inicial estava longe do “ótimo local” para esse ambiente específico; tanto blocos menores quanto maiores, combinados com outros ajustes (como replicação ou memória), já trazem ganhos relevantes.

#### Memória do NodeManager (2 GB → 4 GB)

O cenário **NodeManager = 4 GB** é particularmente interessante:

- O tempo total cai para **373 s**, o melhor valor entre todos.
- O throughput sobe levemente para **20,71 MB/s**, praticamente empatado com o cenário 64 MB.
- Entretanto:
  - A memória máxima por tarefa Map (`MAP_PHYSICAL_MEMORY_BYTES_MAX`) continua na casa de **~355 MB**, muito abaixo dos 2 GB ou 4 GB configurados.
  - O snapshot de memória física total do cluster cai de ~19 GB para ~17,4 GB, sugerindo inclusive melhor utilização/compactação dos containers, não “mais consumo”.

Isso mostra que:

> Aumentar a memória máxima por NodeManager **não muda radicalmente o comportamento do job** (que não é memory-bound), mas ajuda a reduzir pequenas contenções internas de YARN/containers, gerando uma melhoria discreta no tempo final.

Em outras palavras, **não houve “salto” de performance** com 4 GB, mas um ajuste fino que melhora um pouco o throughput.

#### Uso de CPU e GC

Analisando os contadores de CPU e GC:

- O **tempo de CPU total** fica sempre na faixa de **~1833 s a ~1872 s**, uma variação de menos de 3% entre os cenários.
- O tempo de **GC** fica entre **~83 s e ~94 s** — também muito próximo em todos os testes.
- Isso reforça que:
  - O custo computacional intrínseco do WordCount é estável para essa massa de dados.
  - As diferenças observadas no tempo de parede (elapsed time) são principalmente de **agendamento, paralelismo efetivo e acesso a dados**, não de “trabalho extra” feito pelo algoritmo.

#### Espaço e uso de memória

Em termos de espaço:

- O **volume de dados lidos do HDFS** é **idêntico** em todos os cenários (~8,1 GB), bem como o número de registros de entrada e saída. Ou seja, as alterações de configuração **não mudam a quantidade de dados processados**, apenas a forma como eles são distribuídos no cluster.
- O snapshot de **memória física** do cluster fica entre **~17 GB e ~19 GB** em todos os casos, independentemente de 2 GB ou 4 GB de memória por NodeManager. Isso indica que:
  - O job não está consumindo toda a memória alocada para YARN.
  - Há um “overhead fixo” da plataforma (Hadoop + JVM + containers) que domina o uso de memória neste ambiente de testes limitado.

#### Considerações sobre o ambiente de laboratório

Por fim, é importante destacar que todos esses resultados foram obtidos em um **cluster containerizado em um único host físico**. Na prática, isso implica:

- **Contenção de recursos**: CPU, disco e memória são compartilhados por todos os containers, o que pode introduzir variações de tempo difíceis de controlar (por exemplo, interferência entre containers, I/O concorrente no mesmo disco).
- **Latência de rede subestimada**: em um cluster real, a comunicação entre nós passa por rede física, com latência e banda limitadas; aqui, a comunicação entre containers é muito mais rápida e previsível.
- **Replicação “barata”**: o ganho com fator de replicação 2 pode estar **superestimado** em relação a um cluster distribuído real, onde escrever múltiplas réplicas em discos e hosts diferentes tem custo maior.

Mesmo com essas limitações, o benchmark é útil para fins didáticos, pois evidencia:

- O impacto de parâmetros como **replicação** e **blocksize** na **localidade de dados e no paralelismo**.
- Como **ajustes de memória** podem trazer ganhos marginais em jobs que não são estritamente bound a RAM.
- Que o **overhead de gerenciamento do Hadoop** é relevante, principalmente em ambientes pequenos, e deve ser considerado ao dimensionar clusters de produção.

> [!IMPORTANT] 
> #### Detalhes
> Para mais informações sobre as execuções dos jobs, consulte os PDFs gerados pelo JobHistoryServer, disponíveis na seção de [anexos](./hadoop-lab/assets/) deste relatório.

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

#### Falha parcial de DataNode (dn1)

Quando apenas **um** DataNode é derrubado, o job continua rodando — ainda que mais lentamente. Isso ocorre porque:

- Havia réplicas acessíveis nos demais DataNodes.
- O NameNode simplesmente redirecionou as leituras.
- Algumas tasks perderam localidade de dados e precisaram ler remotamente.

Essa situação é coerente com o funcionamento esperado do HDFS: a perda de um DataNode degrada performance, mas **não interrompe a execução** desde que existam réplicas disponíveis.

#### Falha total de DataNodes (dn1 + dn2)

Com ambos os DataNodes fora do ar:

- O NameNode não consegue localizar réplicas vivas dos blocos necessários.
- O cliente HDFS lança `BlockMissingException` repetidamente.
- O job congela por volta de 23% de map.
- Tasks são marcadas como FAILED devido à impossibilidade de leitura.

Quando os DataNodes voltam ao ar:

- O NameNode detecta novamente os blocos.
- YARN reexecuta as tasks necessárias.
- O job volta a avançar normalmente.

Isso mostra corretamente que:

> O Hadoop tolera falhas temporárias nos DataNodes, desde que eventualmente os dados voltem a ficar disponíveis.

Se essa falha ocorresse em um ambiente real sem réplicas suficientes, o job não se recuperaria.

#### Falha do NameNode

A queda do NameNode causa **congelamento imediato** da execução, mesmo com todos os DataNodes ativos. Isso acontece porque:

- O NameNode é responsável por todos os metadados, incluindo localização de blocos.
- Nenhuma operação de leitura, escrita ou abertura de splits pode avançar sem ele.
- O job apenas aguarda o NameNode voltar.

Após a subida do NameNode, o job retoma do ponto exato em que havia parado.

Isso demonstra que, nesta configuração:

> O NameNode é um **ponto único de falha (SPOF)** — comportamento esperado em setups sem HA.

#### Consistência com o modelo Hadoop

Os resultados observados são absolutamente coerentes com a arquitetura do Hadoop:

- Falhas de tasks são esperadas e toleradas: o YARN as reexecuta automaticamente.
- DataNodes podem cair temporariamente sem interromper permanentemente um job.
- O NameNode é essencial; sem ele, o cluster fica “cego”.
- A recuperação depende da disponibilidade futura dos dados.

####  Limitações do ambiente de laboratório

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

## Experimento com Apache Spark

### Arquitetura da Solução de Streaming 

No segundo experimento, desenvolvemos uma arquitetura de processamento de **dados em streaming**
integrada a fontes e saídas externas, visando demonstrar o uso do **Apache Spark** (módulo Structured
Streaming) em conjunto com outras ferramentas do ecossistema Big Data. A figura conceitual do fluxo
é a seguinte:

`Discord → Kafka (input) → Spark → Kafka (output) → Kafka Connect → ElasticSearch → Kibana`

| Componente      | Função                                                      |
| --------------- | ----------------------------------------------------------- |
| Discord Bot     | Captura mensagens em tempo real                             |
| Apache Kafka    | Armazena e distribui mensagens (tópicos de entrada e saída) |
| Spark Streaming | Processa mensagens em janelas deslizantes de 5 segundos     |
| Kafka Connect   | Integra Kafka com ElasticSearch (sink connector)            |
| ElasticSearch   | Armazena dados indexados para consulta                      |
| Kibana          | Visualiza os dados em nuvem de palavras                     |
| Ngrok           | Exposição pública do Kibana no Colab                        |

### Configuração do Ambiente

Todos os componentes (Spark Standalone, Kafka, ElasticSearch, Kibana, Kafka Connect) foram instalados e executados em um único ambiente **Google Colab**. Devido às restrições de rede do Colab, o **Ngrok** foi usado para expor a porta do Kibana (5601) e permitir o acesso externo ao dashboard.

Criação do tópico Kafka:

```bash
# Cria o tópico de entrada (Discord -> Spark)
./kafka/bin/kafka-topics.sh --create --bootstrap-server localhost:9092 \
    --replication-factor 1 --partitions 1 --topic canalinput

# Cria o tópico de saída (Spark -> Elastic)
./kafka/bin/kafka-topics.sh --create --bootstrap-server localhost:9092 \
    --replication-factor 1 --partitions 1 --topic canaloutput
```

### Fluxo de execução

1. **Bot do Discord**: Um bot (desenvolvido em Python com discord.py) foi conectado a um servidor de teste. Ao receber uma mensagem, ele a formatava como JSON e a enviava para o tópico canalinput do Kafka.
2. **Processamento Spark**: A aplicação Spark (em PySpark) foi configurada para:

    - Ler o stream do canalinput.
    - Analisar o JSON e extrair o conteúdo textual das mensagens.
    - Dividir o texto em palavras (tokenização).
    - Aplicar uma agregação de contagem (WordCount) sobre uma janela temporal de 5 segundos.
    - Escrever os resultados (em formato JSON) no tópico canaloutput.
3. **Visualização no Kibana**: O Kafka Connect atuou como uma "cola", movendo os dados do canaloutput para o ElasticSearch. No Kibana, criamos uma visualização Tag Cloud. Quando enviávamos mensagens no Discord (ex: "PSPD" repetidamente), a palavra "PSPD" aparecia e crescia na nuvem de palavras em tempo real, confirmando o sucesso do pipeline fim-a-fim. Abaixo, uma captura de tela do dashboard Kibana exibindo a nuvem de palavras gerada a partir do *lorem ipsum* enviado via Discord:

![Kibana Tag Cloud](./spark-lab/assets/kibana_tag_cloud.png)

### Dificuldades e Aprendizado:

- Integração: O maior desafio foi integrar 5-6 tecnologias diferentes, garantindo que as versões e formatos de dados (JSON, schemas) fossem compatíveis.
- Ambiente: O Google Colab apresentou limitações de recursos (memória), exigindo ajustes para rodar tantos serviços (Spark, ES, Kibana) simultaneamente.
- Debugging: Depurar um sistema de streaming é complexo. Foi necessário inspecionar os tópicos Kafka manualmente para identificar onde o fluxo de dados estava parando (ex: um erro inicial de schema no Kafka Connect).
- Comparativo: O Spark permitiu implementar a lógica de contagem de palavras em janelas com muito menos código e com baixa latência, algo inviável com MapReduce.

## Conclusão

Os experimentos permitiram uma comparação prática direta entre os dois paradigmas:

- Hadoop (MapReduce): Revelou-se excelente para seu propósito original: processamento batch de volumes massivos de dados com alta confiabilidade e tolerância a falhas de workers. Sua principal desvantagem é a alta latência e a rigidez do modelo.
- Spark (Streaming): Mostrou-se ideal para processamento flexível e rápido de fluxos de dados em tempo real. Permitiu criar um dashboard interativo, mas ao custo de uma arquitetura de integração significativamente mais complexa (Spark + Kafka + ElasticSearch + Kibana).

Concluímos que a escolha da ferramenta depende fundamentalmente dos requisitos do problema. Hadoop é a escolha para batch offline robusto, enquanto Spark e seu ecossistema brilham em cenários que exigem baixa latência e análise contínua

> Para mais detalhes técnicos, códigos-fonte e arquivos de configuração utilizados, veja a implementação direta nos diretórios do repositório [hadoop-lab](./hadoop-lab/) e [spark-lab](./spark-lab/).

## Refêrencias

APACHE SOFTWARE FOUNDATION. *Apache Kafka – Documentation*. Disponível em: https://kafka.apache.org. Acesso em: 15 nov. 2025.

APACHE SOFTWARE FOUNDATION. *Apache Spark – Structured Streaming Guide*. Disponível em: https://spark.apache.org. Acesso em: 15 nov. 2025.

APACHE SOFTWARE FOUNDATION. *Apache Hadoop – MapReduce Tutorial*. Disponível em: https://hadoop.apache.org. Acesso em: 15 nov. 2025.

ELASTICSEARCH B.V. *Elasticsearch: The Definitive Guide*. Disponível em: https://www.elastic.co/guide. Acesso em: 15 nov. 2025.

ELASTICSEARCH B.V. *Kibana Documentation*. Disponível em: https://www.elastic.co/kibana. Acesso em: 15 nov. 2025.

DISCORD INC. *Discord Developer Portal – Bots and Gateway Documentation*. Disponível em: https://discord.com/developers. Acesso em: 15 nov. 2025.

NGROK. *Ngrok Documentation*. Disponível em: https://ngrok.com/docs. Acesso em: 15 nov. 2025.

GARCIA-MOLINA, H.; ULLMAN, J. D.; WIDOM, J. *Banco de Dados – Sistemas de Banco de Dados*. 2. ed. Rio de Janeiro: LTC, 2008.

STONEBRAKER, M.; ÇETINTEMEL, U. *“One Size Fits All”: An Idea Whose Time Has Come and Gone*. Communications of the ACM, v. 51, n. 12, 2005.

KLEPPMANN, M. *Designing Data-Intensive Applications*. 1. ed. Sebastopol: O’Reilly Media, 2017.


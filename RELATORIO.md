# 1. Introdução

## 1.1 Visão Geral  
Este projeto desenvolve um pipeline completo de **processamento de dados em tempo real**, integrando de forma coordenada Discord, Apache Kafka, Apache Spark Structured Streaming, ElasticSearch, Kibana e Ngrok.  
O sistema captura mensagens enviadas em um servidor Discord, processa essas informações continuamente e extrai palavras para contabilizar suas ocorrências em janelas temporais.  
Todo o fluxo é automatizado, permitindo monitoramento instantâneo por meio de dashboards dinâmicos no Kibana.

## 1.2 Motivação  
O trabalho tem como objetivo explorar um ecossistema moderno de Big Data, demonstrando na prática como múltiplas tecnologias podem operar juntas em um pipeline distribuído. Entre os principais pontos avaliados estão:  
- ingestão de dados em tempo real;  
- uso de mensageria para comunicação distribuída;  
- processamento contínuo (*stream processing*) com baixa latência;  
- armazenamento otimizado para indexação;  
- visualização rápida de métricas e padrões.  

Essa integração permite compreender como sistemas usados na indústria lidam com fluxos de dados contínuos e de alta demanda.

## 1.3 Objetivos do Projeto  
- Construir um pipeline completo e funcional de ponta a ponta.  
- Capturar mensagens reais provenientes do Discord de forma automatizada.  
- Realizar o processamento contínuo das mensagens usando Spark Streaming.  
- Contabilizar a frequência de palavras por janelas deslizantes.  
- Persistir os resultados no ElasticSearch para consulta eficiente.  
- Visualizar informações em dashboards no Kibana.  
- Expor todo o ambiente externamente utilizando Ngrok.

# 2. Arquitetura do Sistema

## 2.1 Pipeline Geral  
O pipeline foi desenvolvido inteiramente sobre um ecossistema de *streaming*, utilizando Kafka como mensageria e Spark Structured Streaming como motor de processamento contínuo.  
O fluxo principal do sistema é:

**Discord → Kafka (input) → Spark Streaming → Kafka (output) → Kafka Connect → ElasticSearch → Kibana**

Esse encadeamento garante a ingestão, transformação, indexação e visualização dos dados em tempo real, sem utilização de Hadoop no pipeline principal.

## 2.2 Componentes Utilizados  
- **Discord Bot** – Captura mensagens enviadas no servidor e envia ao Kafka.  
- **Apache Kafka** – Atua como sistema de mensageria distribuída para entrada e saída de eventos.  
- **Spark Structured Streaming** – Realiza o processamento contínuo das mensagens, incluindo tokenização e contagem de palavras.  
- **Kafka Connect** – Consumidor automático responsável por transferir os resultados processados para o ElasticSearch.  
- **ElasticSearch** – Armazena os documentos finais e fornece mecanismos otimizados de busca e agregação.  
- **Kibana** – Painel de visualização utilizado para monitorar os dados processados em tempo real.  
- **Ngrok** – Exposição pública do Kibana ou ElasticSearch para acesso remoto.

## 2.3 Fluxo de Dados  
1. O usuário envia uma mensagem em um canal do Discord.  
2. O bot captura a mensagem, formata como JSON e envia ao Kafka (`canalinput`).  
3. O Spark Structured Streaming consome as mensagens, aplica limpeza, tokenização e contagem por janelas.  
4. Os resultados são enviados de volta ao Kafka (`canaloutput`).  
5. O Kafka Connect lê o tópico de saída e insere os documentos no ElasticSearch.  
6. O Kibana consome o índice e exibe visualizações atualizadas em tempo real.

# 3. Captura de Dados – Discord Bot

## 3.1 Funcionamento do Bot  
O bot foi implementado com a biblioteca `discord.py`, ouvindo mensagens em tempo real em um canal configurado. Qualquer nova mensagem acionará imediatamente o envio de um evento para o pipeline.

## 3.2 Estrutura da Mensagem Capturada  
Cada mensagem capturada é convertida para JSON contendo:  
- id  
- author_id  
- author_name  
- content  
- channel_name  
- created_at (timestamp)

Essa formatação garante compatibilidade com o Spark e com o Kafka.

## 3.3 Envio de Eventos ao Kafka  
O bot utiliza `KafkaProducer` para enviar o JSON ao tópico **`canalinput`**, iniciando oficialmente o fluxo de dados.

# 4. Apache Kafka

## 4.1 Tópicos Criados  
- **canalinput** – recebe mensagens brutas do Discord;  
- **canaloutput** – armazena resultados processados pelo Spark.

## 4.2 Configuração do Broker  
- 1 broker Kafka;  
- 1 partição;  
- Fator de replicação = 1;  
- Comunicação realizada em ambiente local (Colab).

## 4.3 Fluxo de Mensagens  
O fluxo segue sempre a ordem:  
**Discord → canalinput → Spark → canaloutput → Kafka Connect → ElasticSearch**.

# 5. Spark Structured Streaming

## 5.1 Leitura do Kafka  
O Spark consome o tópico `canalinput` continuamente, convertendo cada evento para DataFrames estruturados.

## 5.2 Schema dos Dados  
Um schema explícito é definido para interpretar corretamente o JSON recebido, assegurando tipagem adequada.

## 5.3 Tokenização e Explode  
O campo `content` passa por:  
- divisão por espaço (`split`);  
- transformação em lista de palavras;  
- expansão (`explode`) para uma palavra por linha.

## 5.4 Janela de Tempo (Windowing)  
Foi utilizado o modelo:  
- **window de 5 segundos**  
- **slide de 5 segundos**  

Assim, cada janela representa um conjunto de mensagens processadas no mesmo intervalo temporal.

## 5.5 Escrita de Saída no Kafka  
Ao final, o Spark converte o resultado novamente em JSON e escreve no tópico **`canaloutput`**, preparando os dados para o ElasticSearch.

# 6. ElasticSearch

## 6.1 Template de Indexação  
Foi definido um template de indexação para padronizar automaticamente os tipos de dados dos documentos recebidos. Esse template garante que novos índices sigam a mesma estrutura, evitando erros de interpretação de tipos e assegurando consistência ao longo do pipeline.

## 6.2 Estrutura do Índice  
O índice recebe documentos contendo informações essenciais para análise de frequência e temporalidade das palavras processadas. Cada documento inclui:  
- `word`: termo identificado no fluxo de dados;  
- `count`: quantidade de ocorrências da palavra no intervalo definido;  
- `start_time`: timestamp que registra o momento em que o processamento daquela contagem teve início.

## 6.3 Mapeamento de Campos  
O mapeamento foi configurado explicitamente para evitar ambiguidades e garantir consultas eficientes no Kibana e no Elasticsearch:  
- `word`: armazenado como `keyword`, preservando o valor literal da palavra e permitindo agregações precisas;  
- `count`: definido como `long`, adequado para contagens inteiras e operações numéricas;  
- `start_time`: configurado como `date`, permitindo ordenações, filtros temporais e análises por janela de tempo.

# 7. Kibana

## 7.1 Criação do Index Pattern  
O Index Pattern foi configurado para permitir que o Kibana reconheça os dados enviados pelo Elasticsearch. Essa etapa habilita a navegação pelos campos, filtros e visualizações, garantindo que todas as mensagens processadas estejam acessíveis para consulta.

## 7.2 Dashboard (Discover)  
A aba Discover permite acompanhar em tempo real todas as mensagens processadas pelo pipeline. É possível aplicar filtros, buscar palavras específicas, analisar campos individuais e monitorar a chegada contínua de novos registros, facilitando a inspeção e validação dos dados.

## 7.3 Visualização – Nuvem de Palavras  
A nuvem de palavras destaca visualmente as palavras mais frequentes no conjunto de mensagens processadas. Termos que aparecem com maior repetição são exibidos em maior tamanho, permitindo identificar rapidamente padrões, tópicos dominantes e possíveis anomalias no fluxo de dados.

# 8. Kafka Connect

## 8.1 Configuração do ElasticsearchSinkConnector  
Para integrar o Kafka ao ElasticSearch, foi utilizado o conector oficial **ElasticsearchSinkConnector**, responsável por consumir mensagens do tópico de saída do Spark e indexá-las automaticamente.  
A configuração foi ajustada para garantir compatibilidade com JSON puro, além de criar documentos de forma contínua conforme novos resultados eram produzidos.

## 8.2 Conversão JSON  
A conversão foi realizada com `JsonConverter` configurado **sem schema**, permitindo que os registros fossem enviados como JSON simples.  
Essa abordagem simplifica o fluxo e reduz problemas de compatibilidade entre Kafka Connect e ElasticSearch.

## 8.3 Mapeamento de Tópico → Índice  
O mapeamento utilizado conectou diretamente o tópico de saída ao índice final:  
`canaloutput` → `discord_word_counts`  
Com isso, cada contagem de palavra processada pelo Spark passou a ser registrada automaticamente no ElasticSearch.

# 9. Ngrok

## 9.1 Objetivo  
O Ngrok foi utilizado para expor **publicamente** o Kibana e/ou o ElasticSearch, permitindo acesso remoto ao dashboard sem necessidade de configuração de rede ou IP público.  
Isso possibilitou demonstrar o pipeline funcionando em tempo real para terceiros, mesmo sem servidor dedicado.

## 9.2 Configuração  
- Ngrok executado localmente com token autenticado;  
- Porta exposta: **5601** (Kibana);  
- Comando utilizado:

```bash
ngrok http 5601
```

O Ngrok gera um endpoint público temporário, que muda a cada execução.

## 9.3 Uso no Projeto  
O endereço fornecido pelo Ngrok foi usado para:  
- acessar o Kibana fora da máquina local;  
- permitir visualização remota dos dashboards;  
- validar todo o pipeline em tempo real, inclusive em apresentações.

# 10. Resultados dos Testes

## 10.1 Teste de Pipeline Completo  
O pipeline foi testado de ponta a ponta e funcionou como esperado:

1. O bot capturou mensagens do Discord;  
2. Kafka recebeu os eventos no tópico `canalinput`;  
3. Spark processou e aplicou as janelas temporais;  
4. O resultado foi enviado ao tópico `canaloutput`;  
5. Kafka Connect indexou os dados no ElasticSearch;  
6. Kibana exibiu tudo em tempo real.

O sistema demonstrou estabilidade e baixa latência em toda a cadeia.

## 10.2 Volume de Dados  
Foram enviadas grandes quantidades de mensagens para validar:

- resiliência do Spark Structured Streaming;  
- estabilidade do Kafka;  
- velocidade de indexação do ElasticSearch;  
- impacto do volume na latência geral.

Em todos os testes, o comportamento foi consistente e dentro do esperado para um pipeline distribuído.

## 10.3 Visualização no Kibana  
O Kibana conseguiu exibir corretamente:

- registros individuais no Discover;  
- séries temporais com contagens por janela;  
- nuvem de palavras atualizada continuamente;  
- dashboards reagindo aos dados quase em tempo real.

Isso confirma o funcionamento completo do ciclo ingestão → processamento → indexação → visualização.

# 11. Benchmark dos Jobs MapReduce

Nesta seção comparamos cinco execuções do job **WordCount** variando três parâmetros principais: fator de replicação do HDFS, tamanho de bloco (blocksize) e memória disponível para cada NodeManager (YARN). Todos os jobs processam a mesma massa de dados (≈ 8,1 bilhões de bytes lidos do HDFS, cerca de 7,7 GB), com o mesmo código de aplicação.

> **Importante:** todos os serviços (NameNode, DataNodes, ResourceManager, NodeManagers) estão em **containers no mesmo host físico**. Isso significa que o agendamento de CPU, o uso de disco e de memória competem no mesmo hardware. Portanto, as diferenças de tempo devem ser interpretadas com cautela, pois **não representam um cluster distribuído “real”**, mas sim um ambiente de laboratório controlado.

## 11.1 Resumo dos Experimentos

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


## 11.2 Análise dos Resultados

### 11.2.1 Tempo de Execução e Throughput

Em termos de tempo de execução, o cenário **baseline** é o mais lento (≈ 598 s), enquanto os demais convergem para uma faixa entre **373 s e 396 s**. Isso corresponde a um ganho de desempenho entre **~34% e ~38%** em relação ao baseline.

O throughput segue a mesma tendência:

- **Baseline:** ~12,9 MB/s  
- **Demais cenários:** entre ~19,5 e ~20,7 MB/s

Ou seja, **qualquer ajuste** (replicação 2, mudança de blocksize ou aumento de memória do NodeManager) já coloca o sistema em um patamar de throughput significativamente maior do que a configuração inicial.

### 11.2.2 Fator de Replicação

Comparando **Baseline (replicação = 1, 128 MB)** com **2 réplicas (replicação = 2, 128 MB)**:

- O throughput sobe de **12,90 MB/s** para **19,48 MB/s**.
- O tempo total cai de **598,6 s** para **396,5 s**, uma redução de **~34%**.
- O volume de dados lidos do HDFS permanece o mesmo (≈ 8,1 GB em todos os casos), assim como o número de registros de entrada e saída.

Os contadores de CPU e GC são muito próximos entre os dois cenários (diferenças de poucos segundos no total), o que indica que:

> O custo de CPU do job é essencialmente o mesmo; o que muda é **quão rápido o cluster consegue alimentar os mapas com dados**, graças à maior disponibilidade de réplicas para leitura local (melhor data locality e menor dependência de rede).

Mesmo num ambiente de containers compartilhando o mesmo disco, esse efeito aparece claramente nos tempos de execução.

### 11.2.3 Tamanho de Bloco (blocksize)

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

### 11.2.4 Memória do NodeManager

O cenário **NodeManager = 4 GB** é particularmente interessante:

- O tempo total cai para **373 s**, o melhor valor entre todos.
- O throughput sobe levemente para **20,71 MB/s**, praticamente empatado com o cenário 64 MB.
- Entretanto:
  - A memória máxima por tarefa Map (`MAP_PHYSICAL_MEMORY_BYTES_MAX`) continua na casa de **~355 MB**, muito abaixo dos 2 GB ou 4 GB configurados.
  - O snapshot de memória física total do cluster cai de ~19 GB para ~17,4 GB, sugerindo inclusive melhor utilização/compactação dos containers, não “mais consumo”.

Isso mostra que:

> Aumentar a memória máxima por NodeManager **não muda radicalmente o comportamento do job** (que não é memory-bound), mas ajuda a reduzir pequenas contenções internas de YARN/containers, gerando uma melhoria discreta no tempo final.

Em outras palavras, **não houve “salto” de performance** com 4 GB, mas um ajuste fino que melhora um pouco o throughput.

### 11.2.5 Uso de CPU e GC

Analisando os contadores de CPU e GC:

- O **tempo de CPU total** fica sempre na faixa de **~1833 s a ~1872 s**, uma variação de menos de 3% entre os cenários.
- O tempo de **GC** fica entre **~83 s e ~94 s** — também muito próximo em todos os testes.
- Isso reforça que:
  - O custo computacional intrínseco do WordCount é estável para essa massa de dados.
  - As diferenças observadas no tempo de parede (elapsed time) são principalmente de **agendamento, paralelismo efetivo e acesso a dados**, não de “trabalho extra” feito pelo algoritmo.

### 11.2.6 Espaço e Uso de Memória

Em termos de espaço:

- O **volume de dados lidos do HDFS** é **idêntico** em todos os cenários (~8,1 GB), bem como o número de registros de entrada e saída. Ou seja, as alterações de configuração **não mudam a quantidade de dados processados**, apenas a forma como eles são distribuídos no cluster.
- O snapshot de **memória física** do cluster fica entre **~17 GB e ~19 GB** em todos os casos, independentemente de 2 GB ou 4 GB de memória por NodeManager. Isso indica que:
  - O job não está consumindo toda a memória alocada para YARN.
  - Há um “overhead fixo” da plataforma (Hadoop + JVM + containers) que domina o uso de memória neste ambiente de testes limitado.

### 11.2.7 Considerações sobre o Ambiente de Laboratório

Por fim, é importante destacar que todos esses resultados foram obtidos em um **cluster containerizado em um único host físico**. Na prática, isso implica:

- **Contenção de recursos**: CPU, disco e memória são compartilhados por todos os containers, o que pode introduzir variações de tempo difíceis de controlar (por exemplo, interferência entre containers, I/O concorrente no mesmo disco).
- **Latência de rede subestimada**: em um cluster real, a comunicação entre nós passa por rede física, com latência e banda limitadas; aqui, a comunicação entre containers é muito mais rápida e previsível.
- **Replicação “barata”**: o ganho com fator de replicação 2 pode estar **superestimado** em relação a um cluster distribuído real, onde escrever múltiplas réplicas em discos e hosts diferentes tem custo maior.

Mesmo com essas limitações, o benchmark é útil para fins didáticos, pois evidencia:

- O impacto de parâmetros como **replicação** e **blocksize** na **localidade de dados e no paralelismo**.
- Como **ajustes de memória** podem trazer ganhos marginais em jobs que não são estritamente bound a RAM.
- Que o **overhead de gerenciamento do Hadoop** é relevante, principalmente em ambientes pequenos, e deve ser considerado ao dimensionar clusters de produção.

# 12. Conclusão

O pipeline desenvolvido demonstrou que a integração entre Kafka, Spark Structured Streaming, Elasticsearch e Kibana é altamente eficiente para processamento de dados em tempo real, apresentando baixa latência, alta estabilidade e boa capacidade de escalabilidade conforme o volume de mensagens aumenta. Apesar dos resultados positivos, algumas limitações foram identificadas, como a necessidade de ajustes finos de configuração (memória, particionamento, batch sizes), a dependência de hardware adequado para manter o desempenho e a sensibilidade à qualidade dos dados recebidos no início do fluxo. Como trabalhos futuros, propõe-se a integração com Spark MLlib para análises avançadas, aprimoramento das estratégias de otimização do pipeline, implementação de monitoramento contínuo, desenvolvimento de dashboards mais completos no Kibana e a migração da solução para um ambiente em nuvem, utilizando ferramentas de orquestração como Apache Airflow ou Prefect.

# 13. Referências

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


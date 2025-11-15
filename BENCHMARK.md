# Benchmark dos Jobs MapReduce

Nesta seção comparamos cinco execuções do job **WordCount** variando três parâmetros principais: fator de replicação do HDFS, tamanho de bloco (blocksize) e memória disponível para cada NodeManager (YARN). Todos os jobs processam a mesma massa de dados (≈ 8,1 bilhões de bytes lidos do HDFS, cerca de 7,7 GB), com o mesmo código de aplicação.

> **Importante:** todos os serviços (NameNode, DataNodes, ResourceManager, NodeManagers) estão em **containers no mesmo host físico**. Isso significa que o agendamento de CPU, o uso de disco e de memória competem no mesmo hardware. Portanto, as diferenças de tempo devem ser interpretadas com cautela, pois **não representam um cluster distribuído “real”**, mas sim um ambiente de laboratório controlado.

## Resumo dos Experimentos

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


## Análise dos Resultados

### 1. Tempo de execução e throughput

Em termos de tempo de execução, o cenário **baseline** é o mais lento (≈ 598 s), enquanto os demais convergem para uma faixa entre **373 s e 396 s**. Isso corresponde a um ganho de desempenho entre **~34% e ~38%** em relação ao baseline.

O throughput segue a mesma tendência:

- **Baseline:** ~12,9 MB/s  
- **Demais cenários:** entre ~19,5 e ~20,7 MB/s

Ou seja, **qualquer ajuste** (replicação 2, mudança de blocksize ou aumento de memória do NodeManager) já coloca o sistema em um patamar de throughput significativamente maior do que a configuração inicial.

### 2. Fator de replicação

Comparando **Baseline (replicação = 1, 128 MB)** com **2 réplicas (replicação = 2, 128 MB)**:

- O throughput sobe de **12,90 MB/s** para **19,48 MB/s**.
- O tempo total cai de **598,6 s** para **396,5 s**, uma redução de **~34%**.
- O volume de dados lidos do HDFS permanece o mesmo (≈ 8,1 GB em todos os casos), assim como o número de registros de entrada e saída.

Os contadores de CPU e GC são muito próximos entre os dois cenários (diferenças de poucos segundos no total), o que indica que:

> O custo de CPU do job é essencialmente o mesmo; o que muda é **quão rápido o cluster consegue alimentar os mapas com dados**, graças à maior disponibilidade de réplicas para leitura local (melhor data locality e menor dependência de rede).

Mesmo num ambiente de containers compartilhando o mesmo disco, esse efeito aparece claramente nos tempos de execução.

### 3. Tamanho de bloco (blocksize)

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

### 4. Memória do NodeManager (2 GB → 4 GB)

O cenário **NodeManager = 4 GB** é particularmente interessante:

- O tempo total cai para **373 s**, o melhor valor entre todos.
- O throughput sobe levemente para **20,71 MB/s**, praticamente empatado com o cenário 64 MB.
- Entretanto:
  - A memória máxima por tarefa Map (`MAP_PHYSICAL_MEMORY_BYTES_MAX`) continua na casa de **~355 MB**, muito abaixo dos 2 GB ou 4 GB configurados.
  - O snapshot de memória física total do cluster cai de ~19 GB para ~17,4 GB, sugerindo inclusive melhor utilização/compactação dos containers, não “mais consumo”.

Isso mostra que:

> Aumentar a memória máxima por NodeManager **não muda radicalmente o comportamento do job** (que não é memory-bound), mas ajuda a reduzir pequenas contenções internas de YARN/containers, gerando uma melhoria discreta no tempo final.

Em outras palavras, **não houve “salto” de performance** com 4 GB, mas um ajuste fino que melhora um pouco o throughput.

### 5. Uso de CPU e GC

Analisando os contadores de CPU e GC:

- O **tempo de CPU total** fica sempre na faixa de **~1833 s a ~1872 s**, uma variação de menos de 3% entre os cenários.
- O tempo de **GC** fica entre **~83 s e ~94 s** — também muito próximo em todos os testes.
- Isso reforça que:
  - O custo computacional intrínseco do WordCount é estável para essa massa de dados.
  - As diferenças observadas no tempo de parede (elapsed time) são principalmente de **agendamento, paralelismo efetivo e acesso a dados**, não de “trabalho extra” feito pelo algoritmo.

### 6. Espaço e uso de memória

Em termos de espaço:

- O **volume de dados lidos do HDFS** é **idêntico** em todos os cenários (~8,1 GB), bem como o número de registros de entrada e saída. Ou seja, as alterações de configuração **não mudam a quantidade de dados processados**, apenas a forma como eles são distribuídos no cluster.
- O snapshot de **memória física** do cluster fica entre **~17 GB e ~19 GB** em todos os casos, independentemente de 2 GB ou 4 GB de memória por NodeManager. Isso indica que:
  - O job não está consumindo toda a memória alocada para YARN.
  - Há um “overhead fixo” da plataforma (Hadoop + JVM + containers) que domina o uso de memória neste ambiente de testes limitado.

### Considerações sobre o ambiente de laboratório

Por fim, é importante destacar que todos esses resultados foram obtidos em um **cluster containerizado em um único host físico**. Na prática, isso implica:

- **Contenção de recursos**: CPU, disco e memória são compartilhados por todos os containers, o que pode introduzir variações de tempo difíceis de controlar (por exemplo, interferência entre containers, I/O concorrente no mesmo disco).
- **Latência de rede subestimada**: em um cluster real, a comunicação entre nós passa por rede física, com latência e banda limitadas; aqui, a comunicação entre containers é muito mais rápida e previsível.
- **Replicação “barata”**: o ganho com fator de replicação 2 pode estar **superestimado** em relação a um cluster distribuído real, onde escrever múltiplas réplicas em discos e hosts diferentes tem custo maior.

Mesmo com essas limitações, o benchmark é útil para fins didáticos, pois evidencia:

- O impacto de parâmetros como **replicação** e **blocksize** na **localidade de dados e no paralelismo**.
- Como **ajustes de memória** podem trazer ganhos marginais em jobs que não são estritamente bound a RAM.
- Que o **overhead de gerenciamento do Hadoop** é relevante, principalmente em ambientes pequenos, e deve ser considerado ao dimensionar clusters de produção.

> [!important] Detalhes
> Para mais informações sobre as execuções dos jobs, consulte os PDFs gerados pelo JobHistoryServer, disponíveis na seção de [anexos](./hadoop-lab/assets/) deste relatório.
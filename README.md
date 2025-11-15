# Lab_Hadoop_Spark

Este projeto implementa um **cluster Hadoop completo** usando **Docker Compose**, como parte do laboratório da disciplina **Programação de Sistemas Paralelos e Distribuídos (PSPD)**.  
O ambiente simula um cluster distribuído com múltiplos nós (NameNode, DataNodes, ResourceManager, NodeManagers, etc.) e permite executar **jobs MapReduce** — como o clássico *WordCount* — sobre o **HDFS (Hadoop Distributed File System)**.

## Arquitetura do Sistema

O ambiente é composto por **8 contêineres Docker** conectados por uma mesma rede virtual:

| Serviço      | Função                                                                     | Porta       |
| ------------ | -------------------------------------------------------------------------- | ----------- |
| `nn`         | **NameNode** – Gerencia o sistema de arquivos distribuído (HDFS).          | 9870 / 9000 |
| `dn1`, `dn2` | **DataNodes** – Armazenam blocos reais de dados do HDFS.                   | 9864        |
| `rm`         | **ResourceManager** – Coordena e agenda tarefas do YARN.                   | 8088 / 8032 |
| `nm1`, `nm2` | **NodeManagers** – Executam containers de tarefas (Map/Reduce).            | 8042        |
| `jhs`        | **JobHistoryServer** – Armazena logs e histórico de execuções.             | 19888       |
| `edge`       | **Client Node** – Ponto de entrada para comandos `hdfs`, `yarn`, `mapred`. | —           |

### Comunicação entre contêineres
Todos os serviços compartilham a rede interna criada pelo Docker Compose (`hadoop-lab_default`), permitindo comunicação direta via hostname:

```
nn, dn1, dn2, rm, nm1, nm2, jhs, edge
```

Assim, quando executamos um comando como:

```bash
hdfs dfs -ls 
```

o cliente (`edge`) contata o **NameNode** (`nn`) via `fs.defaultFS = hdfs://nn:9000`, que coordena o acesso aos **DataNodes** (`dn1`, `dn2`).


## Como o Hadoop funciona?

O Hadoop é uma plataforma composta por **quatro camadas principais**:

| Camada                    | Componente                                | Função                                                      |
| ------------------------- | ----------------------------------------- | ----------------------------------------------------------- |
| Armazenamento             | **HDFS (NameNode + DataNodes)**           | Armazena dados de forma distribuída e replicada.            |
| Gerenciamento de recursos | **YARN (ResourceManager + NodeManagers)** | Coordena onde e quando cada tarefa será executada.          |
| Execução distribuída      | **MapReduce**                             | Modelo de programação paralela baseado em “map” e “reduce”. |
| Interface de usuário      | **CLI / Web UI**                          | Ferramentas de monitoramento e controle dos jobs.           |

### Fluxo simplificado do WordCount

1. O cliente (`edge`) submete o job `wordcount` ao **ResourceManager** (`rm`);
2. O RM cria um **ApplicationMaster**, que coordena o job;
3. O AM solicita containers aos **NodeManagers** (`nm1`, `nm2`);
4. Cada container executa partes (splits) do input (Map tasks);
5. O resultado intermediário é embaralhado (Shuffle) e reduzido (Reduce);
6. O output é salvo no **HDFS**, distribuído entre os DataNodes.


## Componentes e Arquivos Importantes

| Arquivo           | Descrição                                                                 |
| ----------------- | ------------------------------------------------------------------------- |
| `compose.yml`     | Define os contêineres, volumes e rede do cluster.                         |
| `Dockerfile`      | Imagem base com Hadoop instalado em `/home/hadoop/hadoop`.                |
| `core-site.xml`   | Configura o sistema de arquivos padrão (`fs.defaultFS = hdfs://nn:9000`). |
| `hdfs-site.xml`   | Define diretórios e política de replicação do HDFS.                       |
| `yarn-site.xml`   | Configura o ResourceManager e os NodeManagers (rede, logs, memória).      |
| `mapred-site.xml` | Define o framework MapReduce e variáveis de ambiente.                     |
| `hadoop-env.sh`   | Define variáveis globais (`HADOOP_HOME`, `JAVA_HOME`, etc.).              |

---

## 🚀 Subindo o Cluster

### 1. Build e inicialização

```bash
docker compose build
docker compose up -d # flag d para rodarem como daemons
```

Verifique se todos os serviços estão **rodando**:

```bash
docker compose ps
```

## Testando o HDFS

Acesse o contêiner `edge`:

```bash
docker compose exec edge bash
```

Dentro do contêiner `edge`:
```bash
# criar diretório e enviar arquivos
hdfs dfs -mkdir -p /input
hdfs dfs -put $HADOOP_HOME/etc/hadoop/*.xml /input
# verificar no HDFS
hdfs dfs -ls /input
```

## Executando o Job WordCount

O WordCount é um exemplo oficial do Hadoop, disponível no diretório:

```
$HADOOP_HOME/share/hadoop/mapreduce/hadoop-mapreduce-examples-*.jar
```

Execute:

```bash
EX=$(ls /home/hadoop/hadoop/share/hadoop/mapreduce/hadoop-mapreduce-examples-*.jar | head -n1)
OUT=/output-$(date +%s)
/home/hadoop/hadoop/bin/hadoop jar "$EX" wordcount /input "$OUT"
hdfs dfs -cat "$OUT/part-r-00000" | head
```

## Testando com arquivo próprio

1. Crie o arquivo de entrada:

```bash
printf "um\ndois\ndois\ntres tres tres\n" > /tmp/input.txt
hdfs dfs -mkdir -p /jobs
hdfs dfs -put -f /tmp/input.txt /jobs/input.txt
```

2. Execute o WordCount sobre o arquivo:

```bash
EX=$(ls /home/hadoop/hadoop/share/hadoop/mapreduce/hadoop-mapreduce-examples-*.jar | head -n1)
OUT=/jobs-out-$(date +%s)
/home/hadoop/hadoop/bin/hadoop jar "$EX" wordcount /jobs/input.txt "$OUT"
hdfs dfs -cat "$OUT/part-r-00000"
```

**Saída esperada:**

```
dois    2
tres    3
um      1
```

## Testando com um arquivo grande:

Gere um arquivo grande o suficiente e envie para o HDFS:

```bash
yes "lorem ipsum dolor sit amet" | head -n 300000000 > /tmp/big.txt
hdfs dfs -put /tmp/big.txt /jobs/big.txt
```

Execute o WordCount:

```bash
EX=$(ls /home/hadoop/hadoop/share/hadoop/mapreduce/hadoop-mapreduce-examples-*.jar | head -n1)
hadoop jar "$EX" wordcount /jobs/big.txt /jobs-out-baseline
```

Depois veja os termos mais frequentes:
```bash
hdfs dfs -cat /jobs-out-baseline/part-r-00000
```


## 🌐 Acessando as Interfaces Web

| Interface              | URL padrão                                       | Descrição                                      |
| ---------------------- | ------------------------------------------------ | ---------------------------------------------- |
| **NameNode UI**        | [http://localhost:9870](http://localhost:9870)   | Visualiza estrutura do HDFS e blocos de dados. |
| **ResourceManager UI** | [http://localhost:8088](http://localhost:8088)   | Monitora jobs, containers e recursos YARN.     |
| **JobHistoryServer**   | [http://localhost:19888](http://localhost:19888) | Exibe histórico e logs de jobs finalizados.    |


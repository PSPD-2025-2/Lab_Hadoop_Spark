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

# 11 Testes Comparativos Benchmark – Hadoop WordCount

## 11.1 Tabela Consolidada dos Testes

| Teste                     | Job ID                   | Tempo         | Replication | Blocksize | NodeManager Mem. |
| ------------------------- | ------------------------ | ------------- | ----------- | --------- | ---------------- |
| **Baseline**              | `job_1763235544715_0002` | **9m58.616s** | 1           | 128 MB    | 2048 MB          |
| **Replicação = 2**        | `job_1763236505643_0001` | **6m36.503s** | 2           | 128 MB    | 2048 MB          |
| **Blocksize = 64 MB**     | `job_1763237020860_0001` | **6m14.766s** | 1           | 64 MB     | 2048 MB          |
| **Blocksize = 256 MB**    | `job_1763237525385_0001` | **6m22.375s** | 1           | 256 MB    | 2048 MB          |
| **NodeManager = 4096 MB** | `job_1763242547198_0001` | **6m13s**     | 1           | 128 MB    | 4096 MB          |

## 11.2 Conclusão do Benchmark

Os testes demonstraram melhorias claras no desempenho do Hadoop com ajustes simples:

* **Mais memória para o NodeManager (4096 MB)** entregou o **melhor desempenho**, reduzindo o tempo para **373s**.
* **Blocksize de 64 MB** também foi extremamente eficiente, quase empatando com o melhor cenário.
* **Replicação = 2** surpreendentemente reduziu o tempo de execução, indicando melhor distribuição dos blocos entre os DataNodes.
* **Blocksize de 256 MB** ainda teve bom desempenho, mas não superou blocos menores.
* O **baseline** foi o pior caso, reforçando a importância de configurar o cluster.

Esses resultados mostram que ajustes em **replicação, blocksize e memória disponível** podem reduzir o tempo de execução em **até 35%** em relação à configuração padrão.

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


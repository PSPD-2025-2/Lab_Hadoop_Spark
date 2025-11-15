# 1. Introdução

## 1.1 Visão Geral  
Este projeto implementa um pipeline completo de processamento de dados em tempo real utilizando Discord, Apache Kafka, Apache Spark Structured Streaming, ElasticSearch, Kibana e Ngrok.  
O sistema captura mensagens enviadas em um servidor Discord e processa essas mensagens continuamente, extraindo palavras e contabilizando suas ocorrências em janelas de tempo, para visualização em dashboards dinâmicos.


## 1.2 Motivação  
A proposta do trabalho foi explorar um ecossistema moderno de Big Data, integrando múltiplas tecnologias para demonstrar:  
- ingestão de dados em tempo real;  
- comunicação distribuída via mensageria;  
- processamento contínuo (*stream processing*);  
- indexação otimizada;  
- visualização imediata de insights.  

## 1.3 Objetivos do Projeto  
- Construir um pipeline 100% funcional de ponta a ponta.  
- Capturar mensagens reais do Discord automaticamente.  
- Processar mensagens usando Spark Streaming.  
- Contabilizar palavras usando janelas deslizantes.  
- Armazenar resultados no ElasticSearch.  
- Visualizar dados no Kibana.  
- Expor o ambiente via internet com Ngrok.

# 2. Arquitetura do Sistema

## 2.1 Pipeline Geral  
Fluxo geral do sistema:  Discord → Kafka (input) → Spark Streaming → Kafka (output)  → Kafka Connect → ElasticSearch → Kibana

## 2.2 Componentes Utilizados  
- **Discord Bot**: coleta mensagens.  
- **Apache Kafka**: mensageria para tráfego de eventos.  
- **Spark Structured Streaming**: processamento em tempo real.  
- **Kafka Connect**: envia dados processados ao ElasticSearch.  
- **ElasticSearch**: indexação e busca.  
- **Kibana**: visualização.  
- **Ngrok**: exposição pública.

## 2.3 Fluxo de Dados  
1. Usuário envia mensagem no Discord.  
2. O bot captura e envia JSON para Kafka.  
3. Spark lê, transforma e conta palavras.  
4. Spark devolve o resultado ao Kafka.  
5. Kafka Connect envia ao ElasticSearch.  
6. Kibana indexa e exibe em tempo real.

# 3. Captura de Dados – Discord Bot

## 3.1 Funcionamento do Bot  
O bot utiliza a biblioteca `discord.py` para ouvir mensagens em um canal específico em tempo real.

## 3.2 Estrutura da Mensagem Capturada  
Cada mensagem é transformada em JSON contendo:  
- id  
- author_id  
- author_name  
- content  
- channel_name  
- created_at (timestamp)

## 3.3 Envio de Eventos ao Kafka  
O bot usa `KafkaProducer` para enviar o JSON ao tópico **`canalinput`**.

# 4. Apache Kafka

## 4.1 Tópicos Criados  
- **canalinput**: mensagens brutas do Discord.  
- **canaloutput**: resultados processados pelo Spark.

## 4.2 Configuração do Broker  
- 1 broker Kafka  
- Partições = 1  
- Replicação = 1  
- Comunicação local via Colab

## 4.3 Fluxo de Mensagens  
Discord → canalinput → Spark → canaloutput → ElasticSearch.

# 5. Spark Structured Streaming

## 5.1 Leitura do Kafka  
Spark lê continuamente o tópico `canalinput` usando DataFrames estruturados.

## 5.2 Schema dos Dados  
Schema explícito para interpretar corretamente o JSON enviado pelo bot.

## 5.3 Tokenização e Explode  
O campo `content` é:  
- dividido por espaço (`split`),  
- transformado em lista,  
- expandido em várias linhas (`explode`).

## 5.4 Janela de Tempo (Windowing)  
Usado modelo:  
- *window* de 5 segundos  
- *slide* de 5 segundos  
Assim, Spark conta palavras por janela.

## 5.5 Escrita de Saída no Kafka  
Os resultados são convertidos novamente em JSON e enviados para o tópico **`canaloutput`**.

# 6. ElasticSearch

## 6.1 Template de Indexação  
Definido para configurar automaticamente os tipos corretos.

## 6.2 Estrutura do Índice  
O índice recebe documentos com os campos:  
- `word`  
- `count`  
- `start_time`  

## 6.3 Mapeamento de Campos  
- `word`: keyword  
- `count`: long  
- `start_time`: date  

# 7. Kibana

## 7.1 Criação do Index Pattern  
## 7.2 Dashboard (Discover)  
Permite visualizar mensagens processadas em tempo real.

## 7.3 Visualização – Nuvem de Palavras  
Nuvem mostra quais palavras aparecem mais no canal.

# 8. Kafka Connect

## 8.1 Configuração do ElasticsearchSinkConnector  
O conector oficial *ElasticsearchSinkConnector* foi utilizado.

## 8.2 Conversão JSON  
Configuração usando `JsonConverter` sem schema.

## 8.3 Mapeamento de Tópico → Índice  
`canaloutput` → `discord_word_counts`

# 9. Ngrok
# 10. Resultados dos Teste
# 11. Conclusão
# 12. Referências


Baseline Hadoop WordCount em arquivo grande:
- ID: 1763218053025_0001
- time: 6m15.774s

2 réplicas Hadoop WordCount em arquivo grande:
- ID: job_1763216997442_0001
- time: 6m29.778s
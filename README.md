# airflow-otel
Esse projeto faz a configuração de um monitoramento no Airflow através da integração com OpenTelemetry e Prometheus.

## Airflow e OpenTelemetry
OpenTelemetry (otel) é um framework de observabilidade open source que proporciona a geração, coleta e exportação de dados de métricas, rastreamentos e logs. O objetivo dele é enviar esses dados para alguma ferramenta de monitoramento, como Prometheus.
A integração do otel com Airflow disponibilizada na [versão 2.7](https://airflow.apache.org/docs/apache-airflow/stable/release_notes.html#airflow-2-7-0-2023-08-18) era algo bastante aguardado ([ref](https://github.com/apache/airflow/issues/12771)). Antes disso, era possível exportar algumas métricas Airflow através de statsd, mas essa opção não recebe atualizações há algum tempo, além de outras limitações e desvantagens que podem ser avaliadas na [AIP-49 OpenTelemetry Support for Apache Airflow](https://cwiki.apache.org/confluence/display/AIRFLOW/AIP-49+OpenTelemetry+Support+for+Apache+Airflow).
Por outro lado, com otel vai ser possível ter uma abrangência maior de métricas e logs de forma mais confiável e segura, que resultará em maior visibilidade sobre o que acontece com as dags. A descrição das métricas disponíveis estão na [documentação do Airflow](https://airflow.apache.org/docs/apache-airflow/stable/administration-and-deployment/logging-monitoring/metrics.html#metric-descriptions).   

## Integração Airflow, OpenTelemetry e Prometheus
As configurações necessárias acontecem no Airflow, no OpenTelemetry e no Prometheus. 

### Configurações Airflow (docker-compose.yaml)
No Airflow é necessário instalar o pacote extra otel e adicionar alguns parâmetros dele no `airflow.cfg`, através de variáveis de ambiente, por exemplo: 

    x-airflow-common:
      ...
      environment:
        ...
        AIRFLOW__METRICS__OTEL_ON: 'True'
        AIRFLOW__METRICS__OTEL_HOST: '172.17.0.1'
        AIRFLOW__METRICS__OTEL_PORT: '4318'
        AIRFLOW__METRICS__OTEL_PREFIX: 'airflow'
        AIRFLOW__METRICS__OTEL_INTERVAL_MILLISECONDS: '30000'
        AIRFLOW__METRICS__OTEL_SSL_ACTIVE: 'False'
        _PIP_ADDITIONAL_REQUIREMENTS: ${_PIP_ADDITIONAL_REQUIREMENTS:-apache-airflow[otel]}

### Configurações OpenTelemetry (otel-prometheus/collector-config.yaml)
No OpenTelemetry precisa definir um endpoint onde um coletor irá juntar as métricas airflow, como no exemplo:

    exporters:
      prometheus:
        endpoint: "0.0.0.0:8889"
        namespace: "airflow"

### Configurações Prometheus (otel-prometheus/prometheus/prometheus.yml)
O Prometheus irá ficar raspando o endpoint do coletor otel. Isso é definido assim:
  
    scrape_configs:
      - job_name: 'opentelemetry-collector'
        static_configs:
          - targets: ['172.17.0.1:8889']


## Como rodar

Pré-requisitos:
* Docker
* Docker Compose

Passo a passo pós clone do repositório:

1° Em um terminal, executa os containers Airflow do `docker-compose.yaml`:
  
    docker compose up

2° Em um novo terminal, executa os containers Prometheus e OpenTelemetry do `otel-prometheus/docker-compose.yaml`:

    docker compose up
    
3° Se tudo der certo, é possível ver a UI Airflow em `http://localhost:8080/`, as métricas Airflow em `http://localhost:8889/metrics` e o Prometheus em `http://localhost:9090/ `

## Referências

* Better Apache Airflow Observability using OpenTemeletry: https://medium.com/apache-airflow/better-apache-airflow-observability-using-opentemeletry-db9234bffeac
* AIP-49 OpenTelemetry Support for Apache Airflow: https://cwiki.apache.org/confluence/display/AIRFLOW/AIP-49+OpenTelemetry+Support+for+Apache+Airflow

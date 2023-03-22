# Funcionamento ilustrado

![image](https://user-images.githubusercontent.com/80921933/226532921-14afda67-121e-4e04-932d-53aad08e7f37.png)

# Iniciando um container do Prometheus

```bash
docker run --name prometheus -d --rm -p 9090:9090 prom/prometheus:v2.24.1
```

# Encontrando a documentação do Prometheus para o diferentes linguagens de programação

Acessamos o link da documentação: https://prometheus.io/docs/introduction/overview/

Depois, clicamos em **INSTRUMENTING -> Client Libraries**

![image](https://user-images.githubusercontent.com/80921933/226759377-d078381d-aff3-4d7d-96af-9b7dc77b879c.png)

Agora, basta selecionar a documentação para a linguagem desejada, no meu caso, Java.

![image](https://user-images.githubusercontent.com/80921933/226759514-c14b3239-23cb-4fcd-b9c9-72811e1c39a5.png)

# Adicionando Actuator e Micrometer no Java

Tutorial referenciado: https://www.tutorialworks.com/spring-boot-prometheus-micrometer/

Precisamos do **Actuator** para expor métricas e do **Micrometer** para adaptá-las para o formato que o Prometheus (ou outras ferramentas, como DataDog e Elastic) recebe.

Para isso, começamos adicionando o **Actuator** ao projeto:

```xml
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
```

Depois, inserimos a dependência do **Micrometer** que sabe exportar os dados para o Prometheus:

```xml
<dependency>
  <groupId>io.micrometer</groupId>
  <artifactId>micrometer-registry-prometheus</artifactId>
  <scope>runtime</scope>
</dependency>
```

Por fim, adicionamos a seguinte linha no `application.properties` para indicar ao **Actuator** quais endpoints ele deve expor:

```yaml
management.endpoints.web.exposure.include=health,info,prometheus
```

Pronto! As métricas básicas já estarão prontas em `<SOCKET>/actuator/prometheus`. Caso precisemos de métricas mais especializadas, podemos continuar a partir desse ponto no tutorial mencionado no começo deste tópico.

Com as métricas prontas, basta apontarmos o arquivo de configuração do Prometheus para o endpoint `/actuator/prometheus`.

Para isso, devemos pegar o arquivo de configuração do Prometheus, que pode ser facilmente encontrado (no menu do Prometheus) em **Status -> Configuration**:

```yaml
global:
  scrape_interval: 15s
  scrape_timeout: 10s
  evaluation_interval: 15s
alerting:
  alertmanagers:
  - follow_redirects: true
    enable_http2: true
    scheme: http
    timeout: 10s
    api_version: v2
    static_configs:
    - targets: []
scrape_configs:
- job_name: prometheus     #############
  honor_timestamps: true               #
  scrape_interval: 15s                 #
  scrape_timeout: 10s                  # 
  metrics_path: /metrics               # Duplicar esse bloco e apontar para a nossa aplicação!
  scheme: http                         #
  follow_redirects: true               #
  enable_http2: true                   #
  static_configs:                      #
  - targets:                           #
    - localhost:9090       #############
```

Podemos retirar configurações desnecessárias, como o **alerting**, e daí, basta duplicarmos o item do **scrape_configs** e apontarmos para a nossa aplicação:

```yaml
global:
  scrape_interval: 15s
  scrape_timeout: 10s
  evaluation_interval: 15s
scrape_configs:
- job_name: prometheus     
  honor_timestamps: true               
  scrape_interval: 15s                 
  scrape_timeout: 10s                  
  metrics_path: /metrics              
  scheme: http                         
  follow_redirects: true               
  enable_http2: true                   
  static_configs:                      
  - targets:                           
    - localhost:9090
- job_name: spring-boot # Nome do app     
  honor_timestamps: true               
  scrape_interval: 15s                 
  scrape_timeout: 10s                  
  metrics_path: /actuator/prometheus # Precisa-se adequar o caminho das métricas!              
  scheme: http                         
  follow_redirects: true               
  enable_http2: true                   
  static_configs:                      
  - targets:                           
    - host.docker.internal:8080  # Host e porta. Lembrar de passar a flag do host.docker.internal!
```

Por fim, criamos um bind-mount `-v /path/to/prometheus.yml:/etc/prometheus/prometheus.yml` para que nosso arquivo de configuração seja usado pelo Prometheus.

O comando ficará assim:

```
docker run --name prometheus --rm --add-host=host.docker.internal:host-gateway -d -p 9090:9090 -v /home/azl6/Projects/prometheus-grafana/prometheus.yml:/etc/prometheus/prometheus.yml prom/prometheus:v2.24.1
```

A primeira coisa a se verificar depois de subir o contêiner é ver se o Prometheus foi capaz de contactar a aplicação, na sessão **Status -> Targets**. O status deve estar da seguinte maneira:

![image](https://user-images.githubusercontent.com/80921933/226818898-d7925b64-f074-49af-a8aa-6fb690b85b75.png)

Caso esteja vermelho, **POR FAVOR** verifique o `ufw` em caso de Ubuntu/Mint, ou correspondente para outras distros.

Caso esteja **UP**, já podemos visualizar as métricas no Prometheus:

![Screenshot from 2023-03-22 03-29-02](https://user-images.githubusercontent.com/80921933/226820350-808bc728-9fdd-4f6d-a2b2-8c56e520a2ac.png)

# Filtrando a query no Prometheus

Uma busca inicial seria pesquisar por uma métrica, como `http_server_requests_seconds_count`:

![image](https://user-images.githubusercontent.com/80921933/226826659-7c273763-f7af-4d2c-98df-6eed3287c761.png)

Podemos especificar entre `{}` os "registros" que queremos, por exemplo, `http_server_requests_seconds_count{uri="/"}`, para buscar somente requisições feitas para o caminho `/`:

![image](https://user-images.githubusercontent.com/80921933/226827007-e706f01c-1c20-48c1-847b-dd5537efc6f1.png)

Também é possível demonstrar a busca de um range, por exemplo, de 1 em 1 minuto, especificando o parâmetro `[1]`, em `http_server_requests_seconds_count{uri="/"}[1m]`:

![image](https://user-images.githubusercontent.com/80921933/226827240-635fe51d-9e21-44f4-accf-77dda42dde56.png)

Alguns outros filtros possíveis:

![image](https://user-images.githubusercontent.com/80921933/226827627-14db221b-155c-4acb-93ca-42016cd7edf6.png)

# Counters

**Counters**, ou contadores, são números que **só aumentam**, como requisições recebidas, etc...

Uma métrica interessante para se tirar de contadores (usando o **nº de requisições recebidas** como exemplo) seria o nº de requisições por segundo/minuto/etc. Para isso, é possível utilizar a função `rate()`. (CONTINUAR 1:45 AULA 30)




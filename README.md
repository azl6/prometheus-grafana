# Funcionamento ilustrado

![image](https://user-images.githubusercontent.com/80921933/226532921-14afda67-121e-4e04-932d-53aad08e7f37.png)

# Iniciando um container do Prometheus

```bash
docker run --name prometheus -d --rm -p 9090:9090 prom/prometheus
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
- job_name: nomedomeuappaqui # Nome do app     
  honor_timestamps: true               
  scrape_interval: 15s                 
  scrape_timeout: 10s                  
  metrics_path: /actuator/prometheus # Precisa-se adequar o caminho das métricas!              
  scheme: http                         
  follow_redirects: true               
  enable_http2: true                   
  static_configs:                      
  - targets:                           
    - localhost:9090  # Host e porta
```



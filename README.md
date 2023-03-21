# Funcionamento ilustrado

![image](https://user-images.githubusercontent.com/80921933/226532921-14afda67-121e-4e04-932d-53aad08e7f37.png)

# Iniciando um container do Prometheus

```bash
docker run --name prometheus -d --rm -p 9090:9090 prom/prometheus
```

# TUTORIAL API JAVA

Primeiramente, precisamos da dependency do Actuator:

```xml
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-actuator</artifactId>
    </dependency>
```

Depois, adicionar a dependency do Micrometer:

```xml
<dependency>
  <groupId>io.micrometer</groupId>
  <artifactId>micrometer-registry-prometheus</artifactId>
  <version>${micrometer.version}</version>
</dependency>
```

Agora, adicionar a seguinte property no `application.properties`

```yaml
management.endpoints.web.exposure.incluse=info, health, metrics, prometheus 
```

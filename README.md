# Pipeline de calidad

Proyecto Java 17 con Spring Boot para automatizar compilación, análisis estático, prueba de carga y notificación.

## Requisitos locales

- Java 17
- Maven 3.9+

## Compilar y ejecutar

```bash
mvn clean verify
java -jar target/psw-pipeline-base-0.0.1-SNAPSHOT.jar --server.port=8080
```

Endpoints del ejemplo:

- `GET http://localhost:8080/products`
- `POST http://localhost:8080/login` con `Content-Type: application/json` y cuerpo `{"username":"admin","password":"123456"}`

Las credenciales y el token del login son valores demostrativos del proyecto base; no deben utilizarse en producción.

## Pipeline Jenkins

El `Jenkinsfile` está en la raíz del repositorio. El agente Linux necesita Java 17, Maven, curl y Apache JMeter 5.6.3 en `PATH`. En Jenkins instala Pipeline, Git, SonarQube Scanner for Jenkins y Slack Notification.

1. Crea un job Pipeline conectado a este repositorio y usa `Jenkinsfile` como script path.
2. Registra el servidor global de SonarQube con el nombre `SonarQube` y su token.
3. Configura en SonarQube el webhook `https://<JENKINS_URL>/sonarqube-webhook/` para que `waitForQualityGate` reciba el resultado.
4. Configura Slack Notification y el acceso al workspace/canal `#pipeline-calidad` en Jenkins. No guardes tokens en este repositorio.

El pipeline compila con `mvn clean verify`, publica el análisis y espera el Quality Gate, ejecuta la prueba JMeter y archiva el JTL y el dashboard HTML bajo `target/jmeter/report`. Envía a Slack el resultado final.

## Prueba JMeter

El plan `jmeter/pipeline-calidad.jmx` simula 50 usuarios concurrentes, rampa de 25 segundos y dos iteraciones. Prueba `GET /products` y `POST /login`, y verifica respuestas HTTP 200. El login usa datos de demostración.

Ejecución manual:

```bash
jmeter -n -t jmeter/pipeline-calidad.jmx -JbaseUrl=http://localhost:8080 -l target/jmeter/results.jtl -e -o target/jmeter/report
```

## Cobertura

JaCoCo está integrado en Maven. En la versión base no hay clases de prueba en `src/test/java`, por lo que la cobertura puede no estar disponible hasta añadir pruebas.

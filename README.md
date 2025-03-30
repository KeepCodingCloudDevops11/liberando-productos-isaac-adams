# Práctica Final - Liberando Productos DevOps

Este proyecto extiende un servidor web básico con FastAPI, implementando mejoras para producción, monitorización y despliegue en Kubernetes. A continuación, se detalla cada uno de los requisitos abordados.

## Índice

- [Descripción del proyecto original](#descripción-del-proyecto-original)
- [Requisitos de software](#requisitos-de-software)
- [Mejoras implementadas](#mejoras-implementadas)
  - [1. Nuevo endpoint](#1-nuevo-endpoint)
  - [2. Tests unitarios](#2-tests-unitarios)
  - [3. Helm Chart](#3-helm-chart)
  - [4. Pipelines CI/CD con CircleCI](#4-pipelines-cicd)
  - [5. Monitorización y alertas](#5-monitorización-y-alertas)
- [Guía de despliegue](#guía-de-despliegue)
- [Verificación y pruebas](#verificación-y-pruebas)

## Descripción del proyecto original

El proyecto base consiste en un servidor implementado con [FastAPI](https://fastapi.tiangolo.com/) que:

- Levanta un servidor en el puerto `8081` con dos endpoints iniciales:
  - `/`: Devuelve `{"message":"Hello World"}` con status code 200
  - `/health`: Devuelve `{"health": "ok"}` con status code 200

- Utiliza [prometheus-client](https://github.com/prometheus/client_python) para exponer métricas en el puerto `8000`:
  - `server_requests_total`: Contador del total de llamadas al servidor
  - `healthcheck_requests_total`: Contador de llamadas al endpoint `/health`
  - `main_requests_total`: Contador de llamadas al endpoint `/`

## Requisitos de software

Para trabajar con este proyecto es necesario:

- Python 3.11.8 o superior
- virtualenv: `pip3 install virtualenv`
- Docker
- Kubernetes (Minikube para desarrollo local)
- Helm
- Cuenta en CircleCI para la integración continua
- Cuenta en GitHub y token para GitHub Container Registry

## Mejoras implementadas

### 1. Nuevo endpoint

Se ha implementado un nuevo endpoint `/bye` que devuelve un mensaje de despedida.

#### Implementación

El endpoint ha sido añadido en `src/application/app.py`:

```python
@app.get("/bye")
async def read_bye():
    """Implement bye endpoint"""
    # Increment counter used for register the total number of calls in the webserver
    REQUESTS.inc()
    # Increment counter used for register the total number of calls in the bye endpoint
    BYE_ENDPOINT_REQUESTS.inc()
    return {"response": "bye bye"}
```

Para la monitorización, se añadió un nuevo contador en el mismo archivo:

```python
BYE_ENDPOINT_REQUESTS = Counter('app_bye_requests_total', 'Total bye endpoint requests')
```

#### Prueba del endpoint

```bash
# Realizar una petición al nuevo endpoint
curl -X 'GET' 'http://0.0.0.0:8081/bye' -H 'accept: application/json'

# Respuesta esperada:
# {"response":"bye bye"}
```

### 2. Tests unitarios

Se han desarrollado tests unitarios para el nuevo endpoint `/bye`, siguiendo el patrón de los tests existentes.

#### Implementación

Los tests se han añadido en `src/tests/app_test.py`:

```python
@pytest.mark.asyncio
async def read_bye_test(self):
    """Tests the bye endpoint"""
    response = client.get("/bye")

    assert response.status_code == 200
    assert response.json() == {"response": "bye bye"}
```

#### Ejecución de tests

```bash
# Activar entorno virtual
source venv/bin/activate

# Ejecutar todos los tests
pytest

# Ejecutar tests con cobertura
pytest --cov

# Generar informe de cobertura HTML
pytest --cov --cov-report=html
```

### 3. Helm Chart

Se ha creado un Helm Chart para facilitar el despliegue en Kubernetes, ubicado en el directorio `helm/simple-server-chart/`.

#### Estructura del chart

```
simple-server-chart/
├── Chart.yaml              # Información del chart
├── values.yaml             # Valores configurables
└── templates/
    ├── deployment.yaml     # Recursos para el Deployment
    ├── service.yaml        # Recursos para el Service
    └── _helpers.tpl        # Plantillas de ayuda
```

#### Principales configuraciones

- **Deployment**: Configura la imagen de Docker, recursos (CPU/memoria), sondas de salud y puertos.
- **Service**: Expone la aplicación dentro del cluster en los puertos 8081 y 8000.

#### Despliegue con Helm

```bash
# Instalar el chart
helm install my-server ./simple-server-chart

# Verificar la instalación
kubectl get deployments
kubectl get services
kubectl get pods

# Acceder a la aplicación (en Minikube)
minikube service my-server-simple-server-chart
```

### 4. Pipelines CI/CD

Se han configurado pipelines de CI/CD utilizando CircleCI para automatizar el proceso de testing, construcción y publicación de la imagen Docker.

#### Pipeline con CircleCI

Para la integración continua y despliegue continuo, se ha configurado CircleCI en el archivo `.circleci/config.yml`:

```yaml
version: 2.1

# Orbs son paquetes reutilizables de configuración CircleCI
orbs:
  docker: circleci/docker@2.4.0

jobs:
  build:
    docker:
      # Imagen de Python 3.12
      - image: cimg/python:3.12
    
    steps:
      # Paso 1: Checkout del código
      - checkout
      
      # Paso 2: Configuración de entorno Docker
      - setup_remote_docker
      
      # Paso 3: Restaurar cache de dependencias (si existe)
      - restore_cache:
          keys:
            - v1-dependencies-{{ checksum "requirements.txt" }}
            # Fallback a la última cache si no hay coincidencia exacta
            - v1-dependencies-
      
      # Paso 4: Instalación de dependencias
      - run:
          name: Instalar dependencias
          command: |
            # Verificar la versión de Python
            python3 --version
            
            # Instalar dependencias directamente sin entorno virtual
            python3 -m pip install --user --upgrade pip
            
            # Instalar desde requirements.txt
            python3 -m pip install --user -r requirements.txt
            
            # Instalar herramientas de desarrollo para CI/CD
            python3 -m pip install --user pytest pytest-cov flake8 httpx
      
      # Paso 5: Guardar cache de dependencias
      - save_cache:
          paths:
            - ~/.local/lib/python3.12/site-packages
          key: v1-dependencies-{{ checksum "requirements.txt" }}
          
      # Paso 6: Ejecutar linting
      - run:
          name: Ejecutar linting
          command: |
            # Solo analizar los archivos de nuestro proyecto, excluyendo venv y carpetas de dependencias
            python3 -m flake8 --exclude=venv,\
            .git,\
            __pycache__,\
            .pytest_cache,\
            .eggs,\
            *.egg,\
            build,\
            dist \
            --count --select=E9,F63,F7,F82 --show-source --statistics
      
      # Paso 7: Ejecutar tests
      - run:
          name: Ejecutar tests
          command: |
            export PYTHONPATH=$PYTHONPATH:$(pwd)
            python3 -m pytest --cov=./ --cov-report=xml
          
      # Paso 8: Almacenar resultados de tests
      - store_artifacts:
          path: coverage.xml
          destination: coverage-report

  # Job para construir y publicar imagen Docker
  build_and_push_image:
    executor: docker/docker
    steps:
      - checkout
      - setup_remote_docker
      - run:
          name: Login a GitHub Container Registry
          command: |
            echo "$GITHUB_TOKEN" | docker login ghcr.io -u $GITHUB_USERNAME --password-stdin
      - run:
          name: Construir imagen Docker
          command: |
            # Generar tag basado en el hash del commit
            DOCKER_TAG=$(echo ${CIRCLE_SHA1} | cut -c -7)
            
            # Construir la imagen con múltiples tags
            docker build -t ghcr.io/${DOCKER_IMAGE_NAME}:${DOCKER_TAG} \
                         -t ghcr.io/${DOCKER_IMAGE_NAME}:latest \
                         --label org.opencontainers.image.source=${CIRCLE_REPOSITORY_URL} \
                         --label org.opencontainers.image.revision=${CIRCLE_SHA1} \
                         .
      - run:
          name: Publicar imagen Docker
          command: |
            # Publicar todas las versiones de la imagen
            DOCKER_TAG=$(echo ${CIRCLE_SHA1} | cut -c -7)
            docker push ghcr.io/${DOCKER_IMAGE_NAME}:${DOCKER_TAG}
            docker push ghcr.io/${DOCKER_IMAGE_NAME}:latest

workflows:
  version: 2
  build_test_deploy:
    jobs:
      - build
      - build_and_push_image:
          requires:
            - build
          filters:
            branches:
              only: main
```

El pipeline de CircleCI está configurado para:

1. **Job de Build y Test**:
   - Checkout del código
   - Instala las dependencias de Python
   - Ejecuta análisis estático de código (linting)
   - Ejecuta tests unitarios con cobertura
   - Almacena el informe de cobertura como artefacto

2. **Job de Build y Push de imagen Docker**:
   - Se ejecuta solo después de que el job de build es exitoso
   - Solo se ejecuta en la rama main
   - Genera tags basados en el hash del commit
   - Publica la imagen en GitHub Container Registry

3. **Variables de entorno requeridas**:
   - `GITHUB_TOKEN`: Token de acceso a GitHub con permisos para publicar paquetes
   - `GITHUB_USERNAME`: Nombre de usuario de GitHub
   - `DOCKER_IMAGE_NAME`: Nombre de la imagen Docker (ej. "usuario/simple-server")

### 5. Monitorización y alertas

#### 5.1 Métricas Prometheus para el nuevo endpoint

Se ha configurado un contador para el nuevo endpoint `/bye` en el archivo `src/application/app.py`:

```python
BYE_ENDPOINT_REQUESTS = Counter('app_bye_requests_total', 'Total bye endpoint requests')
```

#### 5.2 Despliegue de Prometheus y Alertmanager

Se ha utilizado el chart `kube-prometheus-stack` para desplegar Prometheus, Alertmanager y Grafana en Kubernetes:

```bash
# Añadir el repositorio de Prometheus
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

# Instalar kube-prometheus-stack con configuración personalizada
helm install monitoring prometheus-community/kube-prometheus-stack -f prometheus-values.yaml
```

El archivo `prometheus-values.yaml` contiene la configuración necesaria para Prometheus y Alertmanager.

#### 5.3 Configuración de alertas

Se han configurado las siguientes alertas en Prometheus:

```yaml
# Alerta cuando el uso de CPU supera el límite configurado
- alert: HighCpuUsage
  expr: (container_cpu_usage_seconds_total{container="simple-server"} / container_spec_cpu_quota{container="simple-server"}) * 100 > 80
  for: 1m
  labels:
    severity: critical
  annotations:
    summary: "High CPU usage detected"
    description: "Container {{ $labels.container }} has high CPU usage: {{ $value }}%"
```

#### 5.4 Integración con Slack

Se ha creado un canal en Slack y configurado un webhook para recibir alertas:

```yaml
# Configuración en prometheus-values.yaml
alertmanager:
  config:
    global:
      resolve_timeout: 5m
    route:
      group_by: ['alertname', 'job']
      group_wait: 30s
      group_interval: 5m
      repeat_interval: 12h
      receiver: 'slack-notifications'
      routes:
      - match:
          severity: critical
        receiver: 'slack-notifications'
    receivers:
    - name: 'slack-notifications'
      slack_configs:
      - api_url: 'https://hooks.slack.com/services/XXXXX/YYYYY/ZZZZZ'
        channel: '#mi-nombre-prometheus-alarms'
        send_resolved: true
        title: "{{ .GroupLabels.alertname }}"
        text: >-
          {{ range .Alerts }}
            *Alert:* {{ .Annotations.summary }}
            *Description:* {{ .Annotations.description }}
            *Severity:* {{ .Labels.severity }}
          {{ end }}
```

#### 5.5 Dashboard de Grafana

Se ha creado un dashboard en Grafana para monitorizar la aplicación, que incluye:

- Número de llamadas a cada endpoint (/, /health, /bye)
- Número de veces que la aplicación ha arrancado
- Uso de recursos (CPU/Memoria)

El dashboard ha sido exportado a `grafana-dashboard.json` para facilitar su importación.

## Guía de despliegue

### 1. Despliegue local

```bash
# Clonar el repositorio
git clone https://github.com/usuario/simple-server.git
cd simple-server

# Configurar entorno virtual
python3 -m venv venv
source venv/bin/activate
pip install -r requirements.txt

# Ejecutar tests
pytest --cov

# Ejecutar la aplicación
python src/app.py
```

### 2. Despliegue con Docker

```bash
# Construir la imagen
docker build -t simple-server:0.0.1 .

# Ejecutar el contenedor
docker run -d -p 8081:8081 -p 8000:8000 --name simple-server simple-server:0.0.1

# Verificar logs
docker logs -f simple-server
```

### 3. Despliegue en Kubernetes con Helm

```bash
# Iniciar Minikube
minikube start

# Cargar imagen en Minikube (si la imagen es local)
minikube image load simple-server:0.0.1

# Instalar el chart de Helm
helm install my-server ./simple-server-chart

# Verificar la instalación
kubectl get pods
kubectl get services

# Exponer el servicio
minikube service my-server-simple-server-chart
```

### 4. Despliegue de monitorización

```bash
# Instalar stack de Prometheus
helm install monitoring prometheus-community/kube-prometheus-stack -f prometheus-values.yaml

# Verificar instalación
kubectl get pods -n monitoring

# Acceder a Grafana
kubectl port-forward svc/monitoring-grafana 3000:80 -n monitoring
# Usuario: admin, contraseña: prom-operator (por defecto)

# Importar dashboard
# En Grafana: Dashboard -> Import -> Subir grafana-dashboard.json
```

## Verificación y pruebas

### Pruebas de endpoints

```bash
# Endpoint principal
curl -X GET http://localhost:8081/

# Endpoint de salud
curl -X GET http://localhost:8081/health

# Nuevo endpoint de despedida
curl -X GET http://localhost:8081/bye
```

### Verificación de métricas

Acceder a las métricas de Prometheus en `http://localhost:8000` para ver los contadores:

```
# TYPE server_requests_total counter
server_requests_total 30.0

# TYPE healthcheck_requests_total counter
healthcheck_requests_total 10.0

# TYPE main_requests_total counter
main_requests_total 15.0

# TYPE app_bye_requests_total counter
app_bye_requests_total 5.0
```

### Pruebas de estrés

Para probar las alertas configuradas, se puede realizar una prueba de estrés:

```bash
# Instalar hey para pruebas de carga
go install github.com/rakyll/hey@latest

# Ejecutar prueba de carga
hey -z 2m -c 50 http://localhost:8081/bye

# Verificar que la alerta se ha activado en Alertmanager y en Slack
```

---

Este proyecto fue desarrollado como práctica final del curso "Liberando Productos" del bootcamp de DevOps de KeepCoding.
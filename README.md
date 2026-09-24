# donaton-bff

Backend for Frontend (BFF) de la plataforma **Donaton**. Actúa como punto de entrada único para el frontend React, enrutando las peticiones hacia los microservicios de Donaciones, Logística y Necesidades, e implementando Circuit Breaker con Resilience4j para tolerancia a fallos.

---

## Requisitos previos

| Herramienta | Versión mínima | Notas |
|-------------|---------------|-------|
| Java JDK | 17 o superior | [Descargar](https://adoptium.net/) |
| Maven | 3.8+ | Incluido en la mayoría de IDEs |
| IDE | IntelliJ IDEA / Eclipse / VS Code | Con extensión Spring Boot |

> Los tres microservicios deben estar corriendo **antes** de levantar el BFF.

---

## Microservicios dependientes

El BFF espera encontrar los siguientes servicios en estos puertos:

| Microservicio | URL esperada | Puerto |
|---------------|-------------|--------|
| MS Donaciones | `http://localhost:8081` | 8081 |
| MS Logística | `http://localhost:8082` | 8082 |
| MS Necesidades | `http://localhost:8083` | 8083 |

Si alguno de estos puertos es diferente en tu entorno, actualiza el archivo `src/main/resources/application.yaml` antes de arrancar.

---

## Instalación

### 1. Clonar el repositorio

```bash
git clone https://github.com/tu-org/donaton-bff.git
cd donaton-bff
```

### 2. Verificar la configuración

Abre `src/main/resources/application.yaml` y confirma que las URLs de los microservicios coinciden con tu entorno local:

```yaml
donaton:
  ms:
    donaciones:
      url: http://localhost:8081
    logistica:
      url: http://localhost:8082
    necesidades:
      url: http://localhost:8083
```

### 3. Compilar el proyecto

```bash
mvn clean install -DskipTests
```

---

## Ejecución

### Opción A — Desde el IDE

1. Abrir el proyecto en IntelliJ IDEA o Eclipse
2. Localizar la clase principal `DonatonBffApplication.java`
3. Clic derecho → **Run**

### Opción B — Desde la terminal

```bash
mvn spring-boot:run
```

### Opción C — Ejecutar el JAR compilado

```bash
java -jar target/donaton-bff-0.0.1-SNAPSHOT.jar
```

El BFF quedará disponible en `http://localhost:8080`.

---

## Verificar que está corriendo

En la consola debe aparecer:

```
Started DonatonBffApplication on port 8080
```

Luego puedes probar con curl o Postman:

```bash
# Listar envíos (pasa por el Circuit Breaker de logística)
GET http://localhost:8080/api/bff/logistica/envios

# Listar necesidades
GET http://localhost:8080/api/bff/necesidades

# Listar donaciones
GET http://localhost:8080/api/bff/donaciones
```

---

## Endpoints expuestos

### Logística

| Método | Endpoint | Descripción |
|--------|----------|-------------|
| `GET` | `/api/bff/logistica/envios` | Listar todos los envíos |
| `POST` | `/api/bff/logistica/envios` | Registrar nuevo envío |
| `PATCH` | `/api/bff/logistica/envios/{id}/estado` | Actualizar estado de un envío |

### Necesidades

| Método | Endpoint | Descripción |
|--------|----------|-------------|
| `GET` | `/api/bff/necesidades` | Listar todas las necesidades |
| `POST` | `/api/bff/necesidades` | Reportar nueva necesidad |
| `PATCH` | `/api/bff/necesidades/{id}/atender` | Marcar necesidad como atendida |

### Donaciones

| Método | Endpoint | Descripción |
|--------|----------|-------------|
| `GET` | `/api/bff/donaciones` | Listar todas las donaciones |
| `POST` | `/api/bff/donaciones` | Registrar nueva donación |

---

## Circuit Breaker

El BFF implementa Circuit Breaker con **Resilience4j** para cada microservicio. Si un MS no responde, el BFF retorna una respuesta de contingencia en lugar de propagar el error al frontend.

| Instancia | Ventana | Umbral de fallo | Tiempo de espera |
|-----------|---------|----------------|-----------------|
| `donacionesCB` | 5 llamadas | 50% | 10 segundos |
| `logisticaCB` | 5 llamadas | 50% | 10 segundos |
| `necesidadesCB` | 5 llamadas | 50% | 10 segundos |

**Respuesta de fallback cuando un MS está caído:**

```json
[
  {
    "error": "Servicio temporalmente no disponible",
    "estado": "FALLBACK"
  }
]
```

---

## CORS

El BFF permite peticiones desde los siguientes orígenes:

```
http://localhost:5173   (Vite — frontend en desarrollo)
http://localhost:3000   (Create React App — alternativa)
```

Para producción, actualizar `CorsConfig.java` con la URL real del frontend.

---

## Estructura del proyecto

```
donaton-bff/
├── src/main/java/com/donaton/bff/
│   ├── config/
│   │   ├── CorsConfig.java          # Configuración CORS
│   │   └── RestTemplateConfig.java  # Bean RestTemplate
│   ├── controller/
│   │   ├── LogisticaController.java
│   │   ├── NecesidadesController.java
│   │   └── DonacionesController.java
│   └── service/
│       ├── LogisticaService.java    # Circuit Breaker logisticaCB
│       ├── NecesidadesService.java  # Circuit Breaker necesidadesCB
│       └── DonacionesService.java   # Circuit Breaker donacionesCB
└── src/main/resources/
    └── application.yaml
```

---

## Solución de problemas frecuentes

**El BFF arranca pero devuelve FALLBACK en todos los endpoints**
- Verificar que los tres microservicios están corriendo antes de levantar el BFF
- Confirmar que los puertos en `application.yaml` coinciden con los puertos reales de cada MS
- Esperar 10 segundos y reintentar (tiempo de recuperación del Circuit Breaker)

**Error de conexión rechazada al arrancar**
- Algún microservicio no está disponible — el Circuit Breaker lo manejará en runtime, no impide que el BFF arranque

**Puerto 8080 en uso**
```yaml
server:
  port: 8085  # Cambiar a cualquier puerto libre
```

---

## Tecnologías utilizadas

- **Spring Boot 3.x**
- **Spring Web** — RestTemplate para llamadas HTTP
- **Resilience4j** — Circuit Breaker
- **Lombok** — Reducción de boilerplate

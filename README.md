# Infraestructura y Despliegue — Ecommerce App

Este documento describe el diseño de la infraestructura del sistema, cómo fue implementada localmente con Docker, y cómo podría migrarse a un entorno Cloud (AWS).

---

## 1. Visión General de la Arquitectura

El sistema es un conjunto de microservicios independientes que colaboran para brindar funcionalidades de e-commerce. Se diseñó bajo los siguientes principios:

- **Aislamiento de dominio**: Cada microservicio posee su propia base de datos y no comparte schema con otro servicio.
- **Comunicación interna por TCP**: Los microservicios nunca se exponen al exterior directamente; toda la comunicación externa pasa por el API Gateway.
- **Contenerización completa**: Cada servicio y base de datos corre en su propio contenedor Docker, haciendo el entorno completamente reproducible.


    ![Diagrama de Arquitectura Local](./docs/images/linktic-ecommerce-Architecture.png)

## 2. Servicios del Sistema

- `client-gateway`: API Gateway y punto de entrada público
- `products-ms`: Microservicio de productos
- `orders-ms`: Microservicio de órdenes
- `products-db`: Base de datos de productos
- `orders-db`: Base de datos de órdenes

---

## 3. Infraestructura Local con Docker Compose

Todo el entorno local está orquestado desde el repositorio `ecommerce-infra` mediante un único archivo `docker-compose.yml`.

### 3.1 Principios de la Configuración Docker

**Zona Pública vs. Zona Segura:**

Solo el `client-gateway` expone un puerto al host (`3000:3000`). Las bases de datos y microservicios internos están encapsulados en la red Docker privada; se resuelven por **nombre de servicio** (DNS interno de Docker Compose), no por IP, por ejemplo: `postgresql://postgres:pass@products-db:5432/productsdb`.

### 3.2 Gestión de configuración con `.env`

Todas las variables de entorno se centralizan en un único archivo `.env` en `ecommerce-infra/`. El `docker-compose.yml` las referencia con la sintaxis `${VARIABLE}`. Las ventajas de este enfoque son:

- **Un solo punto de cambio**: Para cambiar un puerto o credencial, solo se edita el `.env`.
- **Variables compuestas**: Las `DATABASE_URL` de cada microservicio se construyen dinámicamente a partir de otras variables (ej: `${PRODUCTS_DB_USER}`), evitando duplicar valores.
- **Nunca se versiona el `.env`**: El archivo `.env` debe estar en `.gitignore`. Se provee un `env.template` como referencia para nuevos desarrolladores.

### 3.3 Orden de arranque (`depends_on`)

El `docker-compose.yml` declara dependencias explícitas para garantizar el orden correcto de inicio:

```
products-db ───────► products-ms ─┐
                                  ├──► client-gateway
orders-db ─────────► orders-ms ───┘
```

> **Limitación actual:** `depends_on` solo garantiza que el *contenedor* esté corriendo, no que la base de datos dentro esté lista para aceptar conexiones. Una mejora es agregar `healthcheck` en cada servicio de BD.

### 3.4 Persistencia de Datos

Los datos de ambas bases de datos se mantienen en **Docker Volumes nombrados** (`products-data`, `orders-data`). Esto garantiza que los datos persistan aunque los contenedores sean eliminados y recreados con `docker-compose down` / `docker-compose up`.

### 3.5 Instalación y Ejecución

1. Clonar todos los repositorios en la misma carpeta matriz para que el orquestador encuentre los contextos locales:
   ```bash
   git clone https://github.com/Microservices-NestJS-Sadnuro/products-ms.git
   git clone https://github.com/Microservices-NestJS-Sadnuro/orders-ms.git
   git clone https://github.com/Microservices-NestJS-Sadnuro/client-gateway-ms.git
   git clone https://github.com/Microservices-NestJS-Sadnuro/ecommerce-infra.git

2. Ejecutar construccion de imagenes y ejecucion del proyecto
   ```bash
   docker-compose up --build
   ```

3. Acceder a documentacion del Api Gateway para verificar que el servicio esté activo
   ```bash
   http://localhost:3000/api/docs
   ```

4. Realizar peticiones sobre los endpoints del Api Gateway para verificar funcionalidades
   
---

## 4. Migración a AWS (Cloud Architecture)

La arquitectura actual está diseñada de manera que su migración a AWS es muy directa. Cada componente Docker tiene un equivalente AWS gestionado.

### 4.1 Diagrama de Arquitectura AWS Propuesta

![Diagrama de Arquitectura AWS Propuesta](./docs/images/linktic-ecommerce-Propuesta%20AWS.png)

### 4.2 Pipeline de CI/CD con GitHub Actions

El flujo de despliegue propuesto para cada microservicio (usando sus repos Git independientes) sería:
- `Pull Request` + `Merge` a Repositorio rama main/tag de versión
- Ejecución de GitHub Actions Workflow
    - `npm run test`        (ejecutar tests)
    - `docker build`        (construir imagen)
    - `docker push` → ECR   (subir imagen a Amazon ECR)
    - `aws ecs update-service` (actualizar el servicio en ECS con la nueva imagen)

### 4.4 Secretos y Variables de Entorno en AWS

En lugar del archivo `.env` local, en AWS se usaría:

- **AWS Secrets Manager**: Para credenciales sensibles como passwords de base de datos y connection strings.
- **SSM Parameter Store**: Para variables no secretas como puertos, hostnames de microservicios.
- Las ECS Task Definitions referencian estos secretos directamente, por lo que nunca están hardcodeados.

## 5. Mejoras Pendientes para Producción

La arquitectura e implementación actual es un buen punto de partida, sin embargo, para un entorno de producción moderno, se destacan las siguientes mejoras que se pueden implementar para asegurar la escalabilidad, seguridad y mantenibilidad del sistema:

1. **Redis o Caché Distribuido**: Indispensable si el Gateway en el futuro va a aplicar el *Rate Limiting* o el manejo de estado de sesión. Redis también ayudaría en los microservicios para caching de datos.
2. **Servicio o Base de Datos de Autenticación**: Se podría implementar un Identity Provider (Auth0, Cognito) o un microservicio de usuarios/auth (`users-ms` o `auth-ms`) específico con su propia base de datos para autenticar y autorizar usuarios/requests además de emitir y firmar JWTs.
3. **Monitoreo, Trazabilidad y Logs Centralizados**: Se pueden usar herramientas como *Prometheus + Grafana* o *ELK Stack*, además de IDs de correlación (`correlation_id`) para seguir una petición horizontalmente a través de todos los microservicios.
4. **Message Broker (RabbitMQ, Kafka. etc)**: Para comunicaciones asíncronas y eventos distribuidos o patrones Saga (por ejemplo, notificar una compra sin retener y obligarme a esperar un ciclo Request/Reply en TCP).
5. **Config Service**: Un manejador de configuración centralizada en lugar de depender únicamente de .env locales por servicio.

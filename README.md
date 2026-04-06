# Ecommerce Local Environment

Este proyecto aprovisiona toda la arquitectura local (bases de datos y microservicios) a través de Docker.

## Pasos para iniciar en el proyecto (Nuevos Desarrolladores)

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
   
# Ecommerce Local Environment

Este proyecto aprovisiona toda la arquitectura local (bases de datos y microservicios) a través de Docker.

## Pasos para iniciar en el proyecto (Nuevos Desarrolladores)

1. Clonar todos los repositorios en la misma carpeta matriz para que el orquestador encuentre los contextos locales:
   ```bash
   git clone https://github.com/MI-ORG/client-gateway.git
   git clone https://github.com/MI-ORG/products-ms.git
   git clone https://github.com/MI-ORG/orders-ms.git
   git clone https://github.com/MI-ORG/ecommerce-infra.git

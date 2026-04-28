# ISORA VPS-CORE: Global Network Anchor

Este módulo es el corazón de la infraestructura. Su única función es crear y mantener persistente la red virtual de Docker (bridge) que permite la comunicación entre los diferentes grupos de servicios (Database, Getaway, Apps).

## Función del Ancla

En arquitecturas multi-proyecto de Docker Compose, las redes suelen desaparecer si no hay contenedores activos en ellas. El network-anchor utiliza una imagen ligera de Alpine que permanece en "sleep infinity" para:

- Garantizar la existencia de la red ${GLOBAL_NETWORK_NAME}.
- Reservar el rango de IPs 172.16.0.0/24 para evitar colisiones con otras redes automáticas de Docker.
- Servir de Gateway interno (el contenedor ocupa la última IP disponible .254 para dejar las primeras libres para servicios críticos).

## Especificaciones de Red

| Parámetro       | Valor                               |
|-----------------|-------------------------------------|
| Nombre de Red   | Definido en ${GLOBAL_NETWORK_NAME}  |
| Driver          | bridge                              |
| Subnet          | 172.16.0.0/24                       |
| IP Ancla        | 172.16.0.254                        |

## Despliegue Crítico

Este grupo DEBE iniciarse antes que cualquier otro. Si este grupo no está corriendo, los demás archivos docker-compose.yml darán error al intentar encontrar una red external: true.

```bash
# 1. Configurar el nombre de la red en el .env
echo "GLOBAL_NETWORK_NAME=global_network" > .env

# 2. Levantar el ancla
docker compose up -d

# 3. Verificar que la red se creó con el direccionamiento correcto
docker network inspect global_network | grep Subnet
```

## Estructura de Archivos

- docker-compose.yml: Define el contenedor Alpine y la configuración IPAM de la red.
- .env: Contiene el nombre de la red global compartido por todos los módulos del VPS.

## Notas de Seguridad

- Este contenedor no expone puertos.
- No procesa datos; su consumo de CPU/RAM es prácticamente cero (< 5MB RAM).
- Nunca detener este contenedor mientras los grupos database o getaway estén activos, ya que podría causar inestabilidad en el ruteo interno de Docker.
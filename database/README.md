# ISORA VPS-CORE: Database Group

Este módulo gestiona el almacenamiento de datos persistentes y la capa de filtrado de tráfico (L4/L7) para el ecosistema ISORA. Utiliza una arquitectura de Gateway Aislado para asegurar que los motores de base de datos nunca estén expuestos directamente al tráfico externo.

## Arquitectura de Red

El grupo opera sobre dos redes distintas:

- **database-network (Interna):** Red aislada donde residen los motores. Sin salida a internet.
- **global-network (Externa/DMZ):** Red de interconexión donde el nginx_gateway actúa como puente y firewall.

## Stack Tecnológico

| Servicio       | Imagen           | Puerto Interno        | Función                                 |
|----------------|------------------|-----------------------|------------------------------------------|
| Nginx Gateway | nginx:alpine    | 80, 3306, 27017, 6379 | Firewall de red y Proxy TCP/UDP         |
| MariaDB       | mariadb:latest  | 3306                  | Base de datos relacional                |
| MongoDB       | mongo:latest    | 27017                 | Base de datos NoSQL (Documentos)        |
| Redis         | redis:alpine    | 6379                  | Caché y Key-Value Store                |

## Seguridad (Handshake Filtering)

El acceso a las bases de datos está restringido en el archivo nginx/streams.conf. Actualmente, el flujo de seguridad es:

- **Deny All:** Por defecto, todo tráfico es rechazado.
- **Whitelist:** Solo se permite el tráfico originado en el rango 172.16.0.0/24 (DMZ) y la IP del Host (172.16.0.1) para túneles SSH.

### Ejemplo de restricción en streams.conf

```nginx
allow 172.16.0.0/24;
deny all;
```

## Despliegue

Nota: Requiere que el grupo global-network-anchor esté en ejecución para inicializar la subred compartida.

```bash
# Levantar el grupo de bases de datos
docker compose up -d

# Verificar estado del Gateway
docker logs -f nginx_gateway

# Recargar configuración de Nginx tras cambios en streams.conf
docker exec nginx_gateway nginx -s reload
```

## Estructura de Archivos

- docker-compose.yml: Definición de servicios, volúmenes y asignación de IPs estáticas.
- nginx/nginx.conf: Configuración base de Nginx (L7).
- nginx/streams.conf: Definición de proxys TCP y reglas de acceso (L4).
- .env: Variables de entorno para credenciales y nombres de red (No incluir en el repositorio).

## Notas de Mantenimiento

- Volúmenes: Los datos son persistentes en los volúmenes declarados (mariadb_data, mongo_data, redis_data).
- IP Estática: El Gateway tiene asignada la IP 172.16.0.3. Cualquier nuevo servicio que requiera acceso a DB debe apuntar a esta IP o al nombre del servicio si está en la misma red.

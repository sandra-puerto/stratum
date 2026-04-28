# ISORA VPS-CORE: Getaway Group

Este módulo gestiona el punto de entrada único y seguro al VPS. Utiliza un túnel de salida persistente para eliminar la necesidad de abrir puertos en el firewall público.

## Componentes

- Cloudflare Tunnel (cloudflare-getaway): Establece una conexión segura saliente hacia el Edge de Cloudflare. No requiere IP pública ni puertos abiertos en el router/firewall.
- Nginx Proxy Manager (npm-getaway): Gestiona la terminación SSL (si no se usa la de Cloudflare), el balanceo de carga y la redirección de tráfico web basado en nombres de dominio.

## Seguridad de Red

- Localhost Bind: Los puertos 80, 443 y 81 del NPM están vinculados exclusivamente a 127.0.0.1. El acceso externo solo es posible a través del túnel de Cloudflare.
- Aislamiento de Red: El túnel tiene acceso a la red getaway-network para hablar con el proxy, y a la global-network para alcanzar servicios protegidos en otros grupos (como el database_group).

## Comandos de Control

```bash
# Iniciar el túnel y el proxy
docker compose up -d

# Ver logs del túnel (crítico para diagnosticar caídas de conexión)
docker logs -f cloudflare-getaway

# Acceder al panel de administración (Solo local/VPN)
# Abrir en navegador: http://127.0.0.1:81
```

## Configuración de Dominios en NPM

Al configurar un Proxy Host en el panel de NPM:

- Scheme: http o https.
- Forward Hostname: Usar la IP estática asignada en la DMZ (ej: 172.16.0.3 para el Database Gateway).
- Websockets Support: Activar si se usan aplicaciones en tiempo real.
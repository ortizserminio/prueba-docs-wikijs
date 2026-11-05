# Configuración

Opciones de configuración disponibles.

## Variables de entorno

| Variable | Descripción | Valor por defecto |
|----------|-------------|-------------------|
| `APP_PORT` | Puerto de la aplicación | `8080` |
| `APP_DEBUG` | Modo debug | `false` |
| `DB_HOST` | Host de la base de datos | `localhost` |

## Fichero de configuración

El fichero `config.yml` permite personalizar el comportamiento:

```yaml
server:
  port: 8080
  host: 0.0.0.0

database:
  host: localhost
  port: 5432
```

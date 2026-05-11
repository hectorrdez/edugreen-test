# edugreen-test

Colección de tests de API para el proyecto **Edugreen**, construida con [Bruno](https://www.usebruno.com/). Permite ejecutar la suite completa de endpoints de la API Gateway de Edugreen tanto en local como en pipelines de CI/CD (Jenkins u otros).

## Estructura

```
Edugreen/                        # Colección Bruno
├── api/
│   ├── auth/                    # Registro, login, refresh, cambio de contraseña
│   ├── user/                    # CRUD de usuarios
│   ├── institution/             # CRUD de instituciones (solo admin)
│   ├── class/                   # CRUD de clases
│   ├── user-class/              # Relación usuarios ↔ clases
│   ├── challenge/               # CRUD de desafíos
│   ├── enrollment/              # Matrículas en desafíos
│   ├── stats/                   # KPIs y estadísticas
│   └── health/                  # Health check
├── environments/
│   ├── DEV.yml                  # Local (localhost:3001)
│   ├── PRE.yml                  # Pre-producción
│   └── PRO.yml                  # Producción
└── opencollection.yml
endpoints.json                   # Especificación completa de la API
package.json                     # Dependencia: @usebruno/cli
```

## Requisitos

- Node.js >= 18
- `@usebruno/cli` (incluido en `package.json`)

```bash
npm install
```

## Variables de entorno

Cada entorno necesita las siguientes variables secretas (no se guardan en el repo):

| Variable | Descripción |
|---|---|
| `API_KEY` | Clave de autenticación global del API Gateway |
| `USER_SESSION_TOKEN` | JWT de sesión (se rellena automáticamente tras login) |
| `USER_REFRESH_TOKEN` | JWT de refresco (se rellena automáticamente tras login) |
| `USER_PASSWORD_TOKEN` | JWT de recuperación de contraseña |

Las variables no secretas (dominio, puerto, IDs de prueba) ya están definidas en cada fichero de entorno.

## Uso en local

```bash
# Ejecutar toda la colección contra el entorno Dev
npx bru run Edugreen --env Dev --output results.json

# Ejecutar una carpeta concreta
npx bru run Edugreen/api/auth --env Dev

# Pasar variables secretas por línea de comandos
npx bru run Edugreen --env Dev \
  --env-var API_KEY=<clave> \
  --env-var USER_SESSION_TOKEN=<token>
```

Los entornos disponibles son: `Dev`, `PRE`, `PRO V2`.

## Uso en Jenkins

Ejemplo de stage reutilizable para un `Jenkinsfile`:

```groovy
stage('API Tests') {
    steps {
        sh 'npm ci'
        sh """
            npx bru run Edugreen \
              --env ${ENV_NAME} \
              --env-var API_KEY=${API_KEY} \
              --output results.json
        """
    }
    post {
        always {
            archiveArtifacts artifacts: 'results.json', allowEmptyArchive: true
        }
    }
}
```

Las variables `ENV_NAME` y `API_KEY` deben estar definidas como credenciales o parámetros en el job de Jenkins. El dominio de Jenkins por entorno ya está preconfigurado en los ficheros `.yml`:

| Entorno | `JENKINS_DOMAIN` | `API_DOMAIN` |
|---|---|---|
| Dev | `http://192.168.50.200:5001` | `http://localhost` |
| PRE | `https://jenkins.hectorrdez.es` | `http://pre.edugreen.hectorrdez.es` |
| PRO V2 | `https://jenkins.hectorrdez.es` | `http://edugreen.hectorrdez.es` |

## Autenticación

Todos los endpoints requieren el header `Authorization` con el valor de `API_KEY` (middleware global). Los endpoints que además operan con datos de usuario utilizan el header `x-session-token` con un JWT de sesión.

El request `Login a user` rellena automáticamente `USER_SESSION_TOKEN`, `USER_REFRESH_TOKEN` y `USER_ID` en las variables de entorno mediante scripts post-respuesta, por lo que basta con ejecutar el login al inicio de la suite.

# Staging Environment Setup Guide

Este documento explica cómo funciona el ambiente de staging y cómo usarlo.

## Arquitectura de Staging

```
                     ┌─────────────────────────┐
                     │   GitHub Repository     │
                     │  delejove-v2-backend    │
                     └────────┬────────────────┘
                              │
                ┌─────────────┴─────────────┐
                │                           │
           main branch                 staging branch
                │                           │
                ↓                           ↓
        GitHub Actions              GitHub Actions
                │                           │
                ↓                           ↓
       ECR: ...:latest            ECR: ...:staging
                │                           │
                ↓                           │
        Production Container        Staging Container
        backend:5001                backend-v2-staging:5003
                │                           │
                ↓                           ↓
    /api/v2/*                     /api/v2-staging/*
```

## URLs de los Ambientes

### Producción
- Base URL: `https://agape.penwin.cloud/api/v2/*`
- Ejemplo: `https://agape.penwin.cloud/api/v2/users`
- Branch: `main`
- Container: `backend` (puerto 5001)

### Staging
- Base URL: `https://agape.penwin.cloud/api/v2-staging/*`
- Ejemplo: `https://agape.penwin.cloud/api/v2-staging/users`
- Branch: `staging`
- Container: `backend-v2-staging` (puerto 5003)

## Flujo de Trabajo (Workflow)

### 1. Desarrollo de Nueva Funcionalidad

```bash
# Trabajas en una feature branch
git checkout -b feature/nueva-funcionalidad

# Haces tus cambios
git add .
git commit -m "feat: nueva funcionalidad"

# Subes la feature branch
git push origin feature/nueva-funcionalidad
```

### 2. Testing en Staging

```bash
# Mergeas a staging para despliegar en staging
git checkout staging
git merge feature/nueva-funcionalidad
git push origin staging

# ⚡ GitHub Actions automáticamente:
# 1. Build Docker image
# 2. Tag como :staging
# 3. Push a ECR
# 4. Deploy en backend-v2-staging:5003
```

Ahora puedes probar en: `https://agape.penwin.cloud/api/v2-staging/*`

### 3. Deploy a Producción

Cuando ya probaste y todo funciona bien en staging:

```bash
# Mergeas a main para desplegar en producción
git checkout main
git merge staging
git push origin main

# ⚡ GitHub Actions automáticamente:
# 1. Build Docker image
# 2. Tag como :latest
# 3. Push a ECR
# 4. Deploy en backend:5001
```

Ahora está en producción: `https://agape.penwin.cloud/api/v2/*`

## Configuración de React/Frontend

### Opción 1: Variables de Entorno (Recomendada)

Crea archivos `.env` en tu proyecto React:

**.env.development**
```bash
REACT_APP_API_URL=https://agape.penwin.cloud/api/v2-staging
REACT_APP_ENV=staging
```

**.env.production**
```bash
REACT_APP_API_URL=https://agape.penwin.cloud/api/v2
REACT_APP_ENV=production
```

**En tu código React:**
```javascript
// src/services/api.js
const API_BASE_URL = process.env.REACT_APP_API_URL;

export const fetchUsers = async () => {
  const response = await fetch(`${API_BASE_URL}/users`);
  return response.json();
};
```

**Para buildear:**
```bash
# Para staging
npm run build  # usa .env.production por defecto

# Para staging manualmente
REACT_APP_API_URL=https://agape.penwin.cloud/api/v2-staging npm run build

# Para producción
npm run build
```

### Opción 2: Configuración Manual (Desarrollo Local)

```javascript
// src/config.js
export const API_CONFIG = {
  staging: 'https://agape.penwin.cloud/api/v2-staging',
  production: 'https://agape.penwin.cloud/api/v2',
};

// Puedes cambiar manualmente o usar un flag
const isDevelopment = process.env.NODE_ENV === 'development';
export const API_URL = isDevelopment
  ? API_CONFIG.staging
  : API_CONFIG.production;
```

### Opción 3: Feature Toggle (Avanzada)

```javascript
// src/services/api.js
class ApiService {
  constructor() {
    this.baseURL = process.env.REACT_APP_API_URL ||
                   'https://agape.penwin.cloud/api/v2';
  }

  // Método para cambiar dinámicamente
  setEnvironment(env) {
    if (env === 'staging') {
      this.baseURL = 'https://agape.penwin.cloud/api/v2-staging';
    } else {
      this.baseURL = 'https://agape.penwin.cloud/api/v2';
    }
  }

  async get(endpoint) {
    const response = await fetch(`${this.baseURL}/${endpoint}`);
    return response.json();
  }
}

export default new ApiService();
```

## Scripts de Ejemplo para package.json

```json
{
  "scripts": {
    "start": "react-scripts start",
    "build": "react-scripts build",
    "build:staging": "REACT_APP_API_URL=https://agape.penwin.cloud/api/v2-staging react-scripts build",
    "build:production": "REACT_APP_API_URL=https://agape.penwin.cloud/api/v2 react-scripts build"
  }
}
```

## Testing

### Verificar que Staging Funciona

```bash
# Health check
curl https://agape.penwin.cloud/api/v2-staging/health

# Comparar con producción
curl https://agape.penwin.cloud/api/v2/health
```

### Ejemplo de Test en Frontend

```javascript
// tests/api.test.js
describe('API Endpoints', () => {
  test('staging endpoint works', async () => {
    const response = await fetch('https://agape.penwin.cloud/api/v2-staging/users');
    expect(response.status).toBe(200);
  });

  test('production endpoint works', async () => {
    const response = await fetch('https://agape.penwin.cloud/api/v2/users');
    expect(response.status).toBe(200);
  });
});
```

## Troubleshooting

### El staging no se actualiza
```bash
# Verifica que GitHub Actions corrió
gh run list --repo orioletfarras/delejove-v2-backend

# Verifica que el container se actualizó
ssh ubuntu@51.94.1.69 "docker compose ps backend-v2-staging"
```

### Diferentes respuestas entre staging y producción
```bash
# Verifica qué versión está corriendo
curl https://agape.penwin.cloud/api/v2/health
curl https://agape.penwin.cloud/api/v2-staging/health

# Compara los logs
ssh ubuntu@51.94.1.69 "docker compose logs backend --tail=50"
ssh ubuntu@51.94.1.69 "docker compose logs backend-v2-staging --tail=50"
```

## Best Practices

1. **Siempre prueba en staging primero** antes de mergear a main
2. **No hagas commits directos a main**, usa staging → main workflow
3. **Usa la misma base de datos** para staging y producción (o una copia si tienes)
4. **Documenta cambios breaking** en el PR que merges de staging a main
5. **Mantén staging sincronizado** con main después de cada deploy

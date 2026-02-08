¡Por supuesto! Aquí tienes el **Manual Definitivo** (README) para configurar un monorepo Nx con NestJS + Docker que funcione rápido, estable y con soporte para Librerías Compartidas, **evitando el infierno de Webpack en macOS**.

Este resumen condensa todas nuestras horas de debugging en la solución ganadora.

---

# 🚀 Nx + NestJS + Docker (Mac Optimized)

### "The Native TSC Approach"

Guía para levantar un entorno de desarrollo robusto que soporte Hot Reload en Docker (especialmente en macOS con Apple Silicon) y librerías compartidas, utilizando el compilador nativo de TypeScript (`tsc`) en lugar de Webpack.

---

## 1. Crear el Workspace

Inicializa el monorepo vacío.

```bash
npx create-nx-workspace@latest my-org --preset=apps --packageManager=yarn --nxCloud=skip
cd my-org

```

## 2. Generar la API (NestJS)

Instala las dependencias necesarias y genera la aplicación.

```bash
# Instalamos el plugin de Nest y JS
yarn add -D @nx/nest @nx/js

# Generamos la app (sin frontend por ahora)
yarn nx g @nx/nest:app apps/api --linter=eslint --unitTestRunner=jest

```

---

## 3. El "Cambio Nativo" (CRÍTICO) ⚠️

Por defecto, Nx usa Webpack. Debemos cambiarlo a **TSC** para evitar bucles infinitos en Docker y configurar la raíz para que acepte librerías.

### A. Eliminar Webpack

Borra el archivo de configuración que no usaremos.

```bash
rm apps/api/webpack.config.js

```

### B. Editar `apps/api/project.json`

Cambia el **executor** a `@nx/js:tsc` y agrega la propiedad mágica `rootDir: "."`.

```json
"targets": {
  "build": {
    "executor": "@nx/js:tsc", 
    "outputs": ["{options.outputPath}"],
    "options": {
      "outputPath": "dist/apps/api",
      "main": "apps/api/src/main.ts",
      "tsConfig": "apps/api/tsconfig.app.json",
      "assets": ["apps/api/src/assets"],
      "generatePackageJson": true,
      
      // --- LA SOLUCIÓN MÁGICA ---
      // Fuerza a Nx a usar la raíz del workspace como contexto.
      // Esto permite importar libs sin errores de "rootDir".
      "rootDir": "."
    }
  },
  "serve": {
    "executor": "@nx/js:node",
    "options": {
      "buildTarget": "api:build",
      // Evita conflictos de debugger en Docker
      "inspect": false 
    }
  }
}

```

### C. Editar `apps/api/tsconfig.app.json`

Asegúrate de desactivar `composite` e incluir las librerías.

```json
{
  "extends": "./tsconfig.json",
  "compilerOptions": {
    "outDir": "../../dist/out-tsc",
    "types": ["node"],
    // Desactivamos reglas estrictas que chocan con monorepos simples
    "composite": false,
    "declaration": false
  },
  "include": [
    "src/**/*.ts",
    // Incluimos explícitamente las librerías compartidas
    "../../libs/**/*.ts"
  ],
  "exclude": [
    "jest.config.ts",
    "src/**/*.spec.ts",
    "src/**/*.test.ts"
  ]
}

```

---

## 4. Crear y Usar Librerías (Libs)

Genera una librería compartida (DTOs, Interfaces, Utils).

```bash
yarn nx g @nx/js:lib libs/common --importPath=@my-org/common

```

**Uso en la App:**
Simplemente impórtala en tu código NestJS:

```typescript
import { MyDto } from '@my-org/common';

```

*Gracias al paso 3B (`rootDir: "."`), esto compilará sin errores.*

---

## 5. Configuración Docker (Mac Friendly) 🐳

### Dockerfile (Raíz del proyecto)

```dockerfile
FROM node:20-alpine

WORKDIR /app

# Instalamos dependencias
COPY package.json yarn.lock ./
RUN yarn install --frozen-lockfile

# Copiamos todo el código (Nx filtrará lo necesario al compilar)
COPY . .

# Exponemos puerto
EXPOSE 3000

# Comando por defecto
CMD ["yarn", "nx", "serve", "api", "--host", "0.0.0.0"]

```

### Docker Compose (Raíz del proyecto)

La clave aquí es `CHOKIDAR_USEPOLLING`.

```yaml
version: '3.8'

services:
  api:
    build: .
    container_name: nx-nest-api
    ports:
      - "3000:3000"
      - "9229:9229" # Debugger
    environment:
      - NODE_ENV=development
      # --- CRÍTICO PARA MAC ---
      # Fuerza a NestJS a "mirar" el disco activamente en lugar de 
      # esperar eventos del sistema (que fallan en volúmenes montados).
      - CHOKIDAR_USEPOLLING=true
      - CHOKIDAR_INTERVAL=1000
    volumes:
      - .:/app
      - /app/node_modules

```

---

## 6. Ejecución 🚀

1. **Limpiar todo (opcional pero recomendado la primera vez):**
```bash
npx nx reset && rm -rf dist

```


2. **Levantar:**
```bash
docker-compose up --build

```



### Resultado Esperado

* La compilación usará `tsc` (rápido e incremental).
* Al guardar un cambio en `apps/api` o `libs/common`, el reload será casi instantáneo.
* No habrá bucles infinitos de reinicio.
* La estructura en `dist` será un espejo del repo (Nx lo maneja automáticamente).

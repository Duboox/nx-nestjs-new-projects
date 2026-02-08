# 🚀 Nx + NestJS + Docker (Esbuild Edition)

### "The Holy Grail Setup"

Guía para configurar un monorepo Nx robusto. Esta configuración soluciona los 3 grandes problemas del desarrollo moderno en NestJS:

1. **Velocidad:** Compilación en milisegundos usando **Esbuild**.
2. **Docker en Mac:** Hot Reload instantáneo sin bucles de CPU.
3. **Librerías Compartidas:** Sin errores de rutas ni `module-alias`.

---

## 1. Crear el Workspace

Inicializa el monorepo.

```bash
npx create-nx-workspace@latest my-org --preset=apps --packageManager=yarn --nxCloud=skip
cd my-org

```

## 2. Instalar el Stack de Esbuild ⚡️

A diferencia de Webpack (lento) o TSC (rutas complejas), Esbuild es el punto medio perfecto.

```bash
# 1. Instalar el plugin de Nest (si no lo tienes)
yarn add -D @nx/nest

# 2. Instalar Esbuild y su plugin de Nx
yarn add -D @nx/esbuild esbuild

```

---

## 3. Configurar la App (`apps/api`)

### A. Editar `apps/api/project.json`

Cambia el **executor** del `build` para usar esbuild. Esto empaqueta tu código local (apps + libs) en un solo archivo, pero deja los `node_modules` afuera para que Docker sea feliz.

```json
"targets": {
  "build": {
    "executor": "@nx/esbuild:esbuild",
    "outputs": ["{options.outputPath}"],
    "options": {
      "outputPath": "dist/apps/api",
      "main": "apps/api/src/main.ts",
      "tsConfig": "apps/api/tsconfig.app.json",
      "assets": ["apps/api/src/assets"],
      "generatePackageJson": true,
      
      // --- CONFIGURACIÓN ESBUILD ---
      "platform": "node",
      "format": [
          "cjs"
        ],
      "bundle": true,       // Une tu código y tus libs (@app/common)
      "thirdParty": false   // NO empaqueta node_modules (usa los de Docker)
    }
  },
  "serve": {
    "executor": "@nx/js:node",
    "options": {
      "buildTarget": "api:build",
      "watch": true
    }
  }
}

```

### B. Limpieza de `tsconfig`

Con Esbuild no necesitas hacks. Asegúrate de que tu `apps/api/tsconfig.app.json` esté limpio (sin `rootDir` forzados).

```json
{
  "extends": "./tsconfig.json",
  "compilerOptions": {
    "outDir": "../../dist/out-tsc",
    "types": ["node"]
  },
  "include": ["src/**/*.ts"],
  "exclude": ["jest.config.ts", "src/**/*.spec.ts", "src/**/*.test.ts"]
}

```

---

## 4. Ajustes de Código (Los "Gotchas") ⚠️

Como Esbuild es muy rápido, elimina cierta metadata que NestJS usa. Hay dos reglas de oro:

### Regla 1: Decoradores Explícitos (`@Prop`)

Esbuild borra los tipos de TypeScript en tiempo de ejecución. Para entidades (Mongoose/TypeORM), debes decir el tipo explícitamente.

**❌ Mal (Esbuild no sabrá qué es):**

```typescript
@Prop()
createdAt: Date;

```

**✅ Bien (Explícito):**

```typescript
@Prop({ type: Date })
createdAt: Date;

```

### Regla 2: Archivos JSON

Si importas un JSON, usa `require` para evitar errores de seguridad ESM en Node 22+.

**❌ Mal:**

```typescript
import * as data from './data.json';

```

**✅ Bien:**

```typescript
// eslint-disable-next-line @typescript-eslint/no-var-requires
const data = require('./data.json');

```

---

## 5. Configuración Docker (Mac Friendly) 🐳

Gracias a Esbuild, la estructura en `dist` ahora es plana y predecible: `dist/apps/api/main.js`.

### Dockerfile

```dockerfile
FROM node:22-alpine

WORKDIR /app

# Instalar dependencias
COPY package.json yarn.lock ./
RUN yarn install --frozen-lockfile

# Copiar código fuente
COPY . .

# Comando por defecto (será sobreescrito por docker-compose en dev)
CMD ["node", "dist/apps/api/main.js"]

```

### Docker Compose (La Estrategia "Manual Watch")

Para garantizar el **Hot Reload** más rápido posible en Mac, usamos una estrategia híbrida: Nx compila en segundo plano, y Node nativo ejecuta el archivo.

```yaml
version: '3.8'

services:
  api:
    build: .
    environment:
      - NODE_ENV=development
      # Polling para sistema de archivos de Mac
      - CHOKIDAR_USEPOLLING=true
      - CHOKIDAR_INTERVAL=1000
    volumes:
      - .:/app
      - /app/node_modules
    ports:
      - "3000:3000"
    # EL COMANDO MAESTRO:
    # 1. Nx construye en modo watch (& background)
    # 2. Esperamos 5s a que compile
    # 3. Node levanta el archivo final en modo watch
    command: >
      sh -c "yarn nx build api --watch & 
      sleep 5 && 
      node --enable-source-maps --watch dist/apps/api/main.js"

```

---

## 6. Ejecución 🚀

1. **Limpieza inicial:**
```bash
npx nx reset && rm -rf dist

```


2. **Arrancar:**
```bash
docker-compose up --build

```



### ¿Qué ganamos con esto?

* **Estructura Limpia:** El archivo final siempre está en `dist/apps/api/main.js`.
* **Velocidad:** Esbuild es 10x-50x más rápido que TSC/Webpack.
* **Simplicidad:** Sin `module-alias`, sin `rootDir` hacks, sin archivos duplicados.

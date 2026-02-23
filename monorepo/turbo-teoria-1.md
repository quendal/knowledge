Perfecto, te voy a explicar todo el ecosistema de tu monorepo. Déjame primero revisar tu configuración actual:

Perfecto, ahora te explico todo el ecosistema:

## 🏗️ Arquitectura de tu Monorepo

Tu proyecto tiene **3 capas de herramientas** trabajando juntas:

```
┌─────────────────────────────────────────────────────┐
│  CAPA 1: pnpm (Gestor de paquetes y workspaces)    │
│  - Instala dependencias                             │
│  - Gestiona workspaces (apps/* y packages/*)        │
│  - Crea enlaces simbólicos entre paquetes           │
└─────────────────────────────────────────────────────┘
                        ↓
┌─────────────────────────────────────────────────────┐
│  CAPA 2: Turborepo (Orquestador de tareas)         │
│  - Ejecuta scripts en paralelo                      │
│  - Cachea resultados                                │
│  - Respeta dependencias entre paquetes              │
└─────────────────────────────────────────────────────┘
                        ↓
┌─────────────────────────────────────────────────────┐
│  CAPA 3: npm scripts (Tareas específicas)          │
│  - jest (tests)                                     │
│  - tsc (build)                                      │
│  - eslint (lint)                                    │
└─────────────────────────────────────────────────────┘
```

---

## 📦 CAPA 1: pnpm Workspaces

**¿Qué hace?**
- Gestiona las dependencias de todo el monorepo
- Crea un único `node_modules` en la raíz (ahorro de espacio)
- Permite que `@yaestuvo/core` sea usado por otros paquetes

**Archivo clave:** `pnpm-workspace.yaml`
```yaml
packages:
  - 'apps/*'      # api-lambdas, web-admin, web-client
  - 'packages/*'  # core, adapters-db, shared-dto, ui-lib
```

**Comandos pnpm:**

```bash
# Instalar todas las dependencias del monorepo
pnpm install

# Agregar dependencia a un paquete específico
pnpm add jest --filter=@yaestuvo/core --save-dev

# Ejecutar script en un paquete específico
pnpm --filter=@yaestuvo/core test

# Ejecutar script en todos los paquetes
pnpm -r test  # -r = recursive
```

---

## ⚡ CAPA 2: Turborepo

**¿Qué hace?**
- Orquesta la ejecución de tareas (build, test, lint)
- **Cachea resultados** para no repetir trabajo
- Ejecuta tareas en **paralelo** cuando es posible
- Respeta el **grafo de dependencias** entre paquetes

**Archivo clave:** `turbo.json`
```json
{
  "tasks": {
    "test": {
      "dependsOn": ["^build"],  // ← Primero construye dependencias
      "outputs": ["coverage/**"] // ← Cachea el coverage
    }
  }
}
```

**Comandos Turborepo:**

```bash
# Ejecutar tests en TODOS los paquetes (con caché)
pnpm turbo run test

# Ejecutar tests solo en @yaestuvo/core
pnpm turbo run test --filter=@yaestuvo/core

# Forzar ejecución sin caché
pnpm turbo run test --force

# Ver qué se ejecutaría (dry-run)
pnpm turbo run test --dry-run
```

**Ventajas del caché:**
```bash
# Primera ejecución: 10 segundos
pnpm turbo run test

# Segunda ejecución: 0.1 segundos (desde caché)
pnpm turbo run test
# >>> FULL TURBO (caché hit)
```

---

## 🔧 CAPA 3: npm scripts (en cada paquete)

**¿Qué hace?**
- Define las tareas específicas de cada paquete
- Usa herramientas como Jest, TypeScript, ESLint

**Archivo clave:** `packages/core/package.json`
```json
{
  "scripts": {
    "build": "tsc",
    "test": "jest",
    "test:watch": "jest --watch",
    "test:coverage": "jest --coverage"
  }
}
```

**Comandos npm (dentro del paquete):**

```bash
cd packages/core

# Ejecutar tests
npm test

# Ejecutar tests en modo watch
npm run test:watch

# Ejecutar tests con coverage
npm run test:coverage

# Ejecutar un archivo específico
npm test -- DraftOrderUseCase.test.ts
```

---

## 🎯 Cuándo usar cada comando

### 1️⃣ **Desarrollo iterativo en UN paquete**
```bash
cd packages/core
npm run test:watch
```
✅ **Usar cuando:** Estás trabajando solo en `@yaestuvo/core`  
✅ **Ventaja:** Rápido, feedback inmediato  
❌ **Desventaja:** No verifica dependencias con otros paquetes

---

### 2️⃣ **Ejecutar tests de UN paquete desde la raíz**
```bash
# Desde la raíz
pnpm turbo run test --filter=@yaestuvo/core
```
✅ **Usar cuando:** Quieres ejecutar tests de un paquete con caché  
✅ **Ventaja:** Usa caché de Turborepo, respeta dependencias  
❌ **Desventaja:** Más lento que npm directo

---

### 3️⃣ **Ejecutar tests de TODOS los paquetes**
```bash
# Desde la raíz
pnpm turbo run test
```
✅ **Usar cuando:** CI/CD, antes de commit, validación completa  
✅ **Ventaja:** Ejecuta en paralelo, usa caché, valida todo  
❌ **Desventaja:** Más lento si hay muchos paquetes

---

### 4️⃣ **Ejecutar tests sin Turborepo (solo pnpm)**
```bash
# Desde la raíz
pnpm --filter=@yaestuvo/core test
```
✅ **Usar cuando:** Quieres evitar el overhead de Turborepo  
✅ **Ventaja:** Más directo que Turborepo  
❌ **Desventaja:** No usa caché, no respeta dependencias automáticamente

---

## 🔄 Flujo de dependencias

Tu configuración actual tiene:
```json
"test": {
  "dependsOn": ["^build"]
}
```

Esto significa:
```
1. Turborepo detecta que @yaestuvo/api-lambdas depende de @yaestuvo/core
2. Antes de ejecutar tests en api-lambdas, construye @yaestuvo/core
3. Luego ejecuta los tests
```

**Ejemplo práctico:**
```bash
pnpm turbo run test --filter=@yaestuvo/api-lambdas

# Turborepo ejecuta:
# 1. pnpm --filter=@yaestuvo/core build  (dependencia)
# 2. pnpm --filter=@yaestuvo/api-lambdas test
```

---

## 📊 Comparación de comandos

| Comando | Directorio | Caché | Paralelo | Dependencias | Velocidad |
|---------|-----------|-------|----------|--------------|-----------|
| `npm test` | `packages/core/` | ❌ | ❌ | ❌ | ⚡⚡⚡ |
| `pnpm --filter test` | Raíz | ❌ | ❌ | ❌ | ⚡⚡ |
| `pnpm turbo test --filter` | Raíz | ✅ | ❌ | ✅ | ⚡⚡ |
| `pnpm turbo test` | Raíz | ✅ | ✅ | ✅ | ⚡ (primera vez) ⚡⚡⚡ (caché) |

---

## 🎓 Recomendaciones para tu workflow

### Durante desarrollo (iteración rápida):
```bash
cd packages/core
npm run test:watch
```

### Antes de commit (validación local):
```bash
# Desde la raíz
pnpm turbo run test --filter=@yaestuvo/core
```

### En CI/CD (validación completa):
```bash
# Desde la raíz
pnpm turbo run test
```

### Para limpiar caché de Turborepo:
```bash
pnpm turbo run test --force
# o
rm -rf .turbo
```

---

## 🔍 Verificar qué hace Turborepo

```bash
# Ver el plan de ejecución
pnpm turbo run test --dry-run

# Ver logs detallados
pnpm turbo run test --verbosity=2

# Ver el grafo de dependencias
pnpm turbo run test --graph
```

---

## 📝 Scripts útiles para tu package.json raíz

Puedes agregar estos atajos:

```json
{
  "scripts": {
    "test": "turbo run test",
    "test:core": "turbo run test --filter=@yaestuvo/core",
    "test:watch": "turbo run test:watch --filter=@yaestuvo/core",
    "test:all": "turbo run test --force"
  }
}
```

Entonces podrías usar:
```bash
pnpm test:core        # Tests de core con caché
pnpm test:watch       # Tests de core en modo watch
pnpm test:all         # Todos los tests sin caché
```

¿Quieres que agregue estos scripts a tu `package.json` raíz?

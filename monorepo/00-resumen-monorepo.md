# Guía Completa de Comandos del Monorepo

## Tabla de Contenidos
1. [Introducción a los Conceptos del Monorepo](#introducción-a-los-conceptos-del-monorepo)
2. [Comandos de Instalación](#comandos-de-instalación)
3. [Comandos de Construcción (Build)](#comandos-de-construcción-build)
4. [Comandos de Pruebas (Test)](#comandos-de-pruebas-test)
5. [Comandos de Desarrollo](#comandos-de-desarrollo)
6. [Comandos de Linting](#comandos-de-linting)
7. [Gestión de Paquetes](#gestión-de-paquetes)
8. [Tabla Comparativa de Comandos](#tabla-comparativa-de-comandos)
9. [Flujos de Trabajo Comunes](#flujos-de-trabajo-comunes)
10. [Entendiendo el Flag --filter](#entendiendo-el-flag---filter)
11. [Entendiendo el Caché de Turborepo](#entendiendo-el-caché-de-turborepo)
12. [Solución de Problemas](#solución-de-problemas)
13. [Hoja de Referencia Rápida](#hoja-de-referencia-rápida)

---

## Introducción a los Conceptos del Monorepo

### ¿Qué es un Monorepo?

Un **monorepo** (repositorio monolítico) es una estrategia de desarrollo donde múltiples proyectos relacionados se almacenan en un único repositorio. En nuestro caso, tenemos:

**Aplicaciones (apps/):**
- `web-admin` - Panel de administración
- `web-client` - Aplicación del cliente
- `api-lambdas` - Funciones Lambda de AWS

**Paquetes compartidos (packages/):**
- `@yaestuvo/core` - Entidades de dominio y casos de uso
- `@yaestuvo/adapters-db` - Implementaciones de DynamoDB
- `@yaestuvo/shared-dto` - DTOs compartidos
- `@yaestuvo/ui-lib` - Componentes de UI reutilizables
- `@yaestuvo/tsconfig` - Configuraciones de TypeScript compartidas

### ¿Qué es pnpm y por qué lo usamos?

**pnpm** (performant npm) es un gestor de paquetes alternativo a npm y yarn con ventajas significativas:


**1. Eficiencia de Espacio en Disco:**
- pnpm usa un almacén global de contenido direccionable (content-addressable store)
- Los paquetes se almacenan una sola vez en tu máquina
- Los proyectos usan enlaces duros (hard links) al almacén global
- **Ejemplo:** Si 10 proyectos usan React 18.2.0, solo se almacena una copia

**2. Velocidad:**
- Instalaciones más rápidas porque reutiliza paquetes ya descargados
- No necesita copiar archivos, solo crear enlaces
- Instalaciones paralelas eficientes

**3. Gestión de Workspaces:**
- Soporte nativo para monorepos mediante `pnpm-workspace.yaml`
- Maneja dependencias entre paquetes internos automáticamente
- Permite ejecutar comandos en múltiples paquetes simultáneamente

**4. Seguridad:**
- Estructura de `node_modules` más estricta
- Los paquetes solo pueden acceder a sus dependencias declaradas
- Previene el "phantom dependencies" (usar paquetes no declarados)

### ¿Qué es Turborepo y por qué lo usamos?

**Turborepo** es un sistema de construcción de alto rendimiento para monorepos JavaScript/TypeScript:

**1. Caché Inteligente:**
- Recuerda los resultados de tareas anteriores
- Si nada cambió, reutiliza el resultado cacheado
- **Ejemplo:** Si ejecutas `test` dos veces sin cambios, la segunda es instantánea

**2. Ejecución Paralela:**
- Ejecuta tareas en múltiples paquetes simultáneamente
- Maximiza el uso de CPU
- **Ejemplo:** Puede construir `core` y `shared-dto` al mismo tiempo

**3. Grafo de Dependencias:**
- Entiende las relaciones entre paquetes
- Ejecuta tareas en el orden correcto automáticamente
- **Ejemplo:** Construye `core` antes de `adapters-db` (que depende de `core`)


**4. Configuración Declarativa:**
- Define pipelines de construcción en `turbo.json`
- Especifica dependencias entre tareas
- Configura qué archivos cachear

### Cómo Trabajan Juntos pnpm y Turborepo

```
┌─────────────────────────────────────────────────┐
│  Tu Comando: pnpm turbo run build              │
└─────────────────┬───────────────────────────────┘
                  │
                  ▼
┌─────────────────────────────────────────────────┐
│  pnpm: Gestiona dependencias y workspaces      │
│  - Resuelve qué paquetes ejecutar               │
│  - Maneja el contexto del workspace             │
└─────────────────┬───────────────────────────────┘
                  │
                  ▼
┌─────────────────────────────────────────────────┐
│  Turborepo: Orquesta la ejecución               │
│  - Analiza el grafo de dependencias             │
│  - Verifica el caché                            │
│  - Ejecuta tareas en paralelo                   │
│  - Cachea resultados                            │
└─────────────────────────────────────────────────┘
```

---

## Comandos de Instalación

### 1. Instalación Inicial del Proyecto

```bash
pnpm install
```

**¿Qué hace?**
- Instala todas las dependencias de todos los paquetes y aplicaciones
- Lee `pnpm-workspace.yaml` para identificar los workspaces
- Crea enlaces simbólicos entre paquetes internos
- Genera el archivo `pnpm-lock.yaml` con versiones exactas

**¿Cuándo usarlo?**
- Primera vez que clonas el repositorio
- Después de hacer `git pull` si hay cambios en dependencias
- Después de cambiar de rama si las dependencias difieren

**Salida esperada:**
```
Scope: all 8 workspace projects
Lockfile is up to date, resolution step is skipped
Progress: resolved 245, reused 245, downloaded 0, added 245, done
```


### 2. Agregar Dependencia a un Paquete Específico

```bash
pnpm add <paquete> --filter=@yaestuvo/core
```

**Ejemplo real:**
```bash
pnpm add zod --filter=@yaestuvo/core
```

**¿Qué hace?**
- Instala `zod` solo en el paquete `@yaestuvo/core`
- Actualiza el `package.json` de ese paquete
- Actualiza `pnpm-lock.yaml`

**¿Por qué usarlo?**
- Mantiene las dependencias organizadas por paquete
- Evita instalar dependencias innecesarias en otros paquetes
- Reduce el tamaño de los bundles finales

### 3. Agregar Dependencia de Desarrollo

```bash
pnpm add -D <paquete> --filter=@yaestuvo/core
```

**Ejemplo real:**
```bash
pnpm add -D @types/uuid --filter=@yaestuvo/core
```

**¿Qué hace?**
- Instala el paquete como `devDependency`
- Solo se usa durante desarrollo/construcción, no en producción

### 4. Agregar Dependencia a Múltiples Paquetes

```bash
pnpm add <paquete> --filter=@yaestuvo/core --filter=@yaestuvo/adapters-db
```

**Ejemplo real:**
```bash
pnpm add date-fns --filter=@yaestuvo/core --filter=@yaestuvo/adapters-db
```

### 5. Agregar Dependencia a Nivel Raíz

```bash
pnpm add -D <paquete> -w
```

**Ejemplo real:**
```bash
pnpm add -D prettier -w
```

**¿Qué hace?**
- El flag `-w` (workspace-root) instala en la raíz del monorepo
- Útil para herramientas que afectan todo el proyecto (prettier, eslint config, etc.)

---

## Comandos de Construcción (Build)


### 1. Construir Todo el Monorepo

```bash
pnpm turbo run build
# o el atajo:
pnpm build
```

**¿Qué hace?**
- Ejecuta el script `build` de todos los paquetes y aplicaciones
- Respeta el orden de dependencias (construye `core` antes de `adapters-db`)
- Ejecuta construcciones en paralelo cuando es posible
- Cachea los resultados

**¿Cuándo usarlo?**
- Antes de hacer deploy
- Para verificar que todo compila correctamente
- Después de cambios significativos en múltiples paquetes

**Flujo de ejecución:**
```
1. Turborepo analiza turbo.json
2. Identifica que build tiene "dependsOn": ["^build"]
3. Construye en orden:
   - @yaestuvo/tsconfig (no tiene build)
   - @yaestuvo/core (depende de tsconfig)
   - @yaestuvo/shared-dto
   - @yaestuvo/adapters-db (depende de core)
   - @yaestuvo/ui-lib
   - apps/web-admin, apps/web-client, apps/api-lambdas (en paralelo)
```

**Salida esperada:**
```
• Packages in scope: @yaestuvo/core, @yaestuvo/adapters-db, ...
• Running build in 8 packages
• Remote caching disabled

@yaestuvo/core:build: cache hit, replaying output [2.1s]
@yaestuvo/adapters-db:build: cache miss, executing [3.4s]
...

Tasks:    6 successful, 6 total
Cached:   4 cached, 6 total
Time:     5.2s >>> FULL TURBO
```

### 2. Construir un Paquete Específico

```bash
pnpm turbo run build --filter=@yaestuvo/core
```

**¿Qué hace?**
- Construye solo el paquete `@yaestuvo/core`
- También construye sus dependencias si es necesario
- Usa caché si nada cambió

**¿Cuándo usarlo?**
- Estás trabajando en un paquete específico
- Quieres verificar que compila sin construir todo
- Desarrollo iterativo más rápido


### 3. Construir con Dependencias

```bash
pnpm turbo run build --filter=@yaestuvo/adapters-db...
```

**¿Qué hace?**
- Los tres puntos `...` significan "incluir dependencias"
- Construye `@yaestuvo/core` primero (dependencia de adapters-db)
- Luego construye `@yaestuvo/adapters-db`

### 4. Construir sin Caché

```bash
pnpm turbo run build --force
```

**¿Qué hace?**
- Ignora el caché existente
- Ejecuta todas las construcciones desde cero
- Útil para debugging de problemas de caché

**¿Cuándo usarlo?**
- Sospechas que el caché está corrupto
- Quieres asegurar una construcción limpia
- Debugging de problemas de build

---

## Comandos de Pruebas (Test)

### 1. Ejecutar Todas las Pruebas

```bash
pnpm turbo run test
# o el atajo:
pnpm test
```

**¿Qué hace?**
- Ejecuta el script `test` de todos los paquetes
- Construye los paquetes primero (por `dependsOn: ["^build"]` en turbo.json)
- Ejecuta pruebas en paralelo cuando es posible
- Cachea resultados si el código no cambió

**¿Cuándo usarlo?**
- Antes de hacer commit
- En CI/CD pipeline
- Para verificar que nada se rompió

**Salida esperada:**
```
@yaestuvo/core:test: PASS src/entities/Order.test.ts
@yaestuvo/core:test:   ✓ should create valid order (5ms)
@yaestuvo/core:test:   ✓ should validate order items (3ms)
@yaestuvo/core:test: 
@yaestuvo/core:test: Test Suites: 5 passed, 5 total
@yaestuvo/core:test: Tests:       23 passed, 23 total
```


### 2. Ejecutar Pruebas de un Paquete Específico

```bash
pnpm turbo run test --filter=@yaestuvo/core
```

**¿Qué hace?**
- Ejecuta solo las pruebas del paquete `@yaestuvo/core`
- Más rápido que ejecutar todas las pruebas
- Útil durante desarrollo

**¿Cuándo usarlo?**
- Estás desarrollando en un paquete específico
- Quieres feedback rápido
- Debugging de pruebas específicas

### 3. Ejecutar Pruebas con Cobertura

```bash
pnpm --filter=@yaestuvo/core test:coverage
```

**¿Qué hace?**
- Ejecuta las pruebas con reporte de cobertura
- Genera archivos en `coverage/`
- Muestra porcentaje de código cubierto

**Salida esperada:**
```
----------------------|---------|----------|---------|---------|
File                  | % Stmts | % Branch | % Funcs | % Lines |
----------------------|---------|----------|---------|---------|
All files             |   87.5  |   82.3   |   90.1  |   88.2  |
 entities/            |   92.1  |   88.5   |   95.0  |   93.4  |
  Order.ts            |   95.2  |   91.2   |  100.0  |   96.1  |
  Customer.ts         |   89.3  |   85.7   |   90.0  |   90.5  |
 use-cases/           |   82.4  |   75.8   |   85.0  |   82.9  |
  CreateOrder.ts      |   88.1  |   80.2   |   90.0  |   89.0  |
----------------------|---------|----------|---------|---------|
```

### 4. Ejecutar Pruebas en Modo Watch

```bash
pnpm --filter=@yaestuvo/core test:watch
```

**¿Qué hace?**
- Ejecuta las pruebas y queda observando cambios
- Re-ejecuta automáticamente cuando guardas archivos
- Modo interactivo para desarrollo

**¿Cuándo usarlo?**
- Durante desarrollo activo con TDD
- Quieres feedback inmediato al hacer cambios


### 5. Ejecutar un Archivo de Prueba Específico

```bash
cd packages/core
pnpm test Order.test.ts
```

**¿Qué hace?**
- Ejecuta solo el archivo especificado
- Más rápido para pruebas individuales
- Útil para debugging

**Alternativa con filter:**
```bash
pnpm --filter=@yaestuvo/core test -- Order.test.ts
```

---

## Comandos de Desarrollo

### 1. Iniciar Todos los Servidores de Desarrollo

```bash
pnpm turbo run dev
# o el atajo:
pnpm dev
```

**¿Qué hace?**
- Inicia el servidor de desarrollo de todas las aplicaciones
- `web-admin` en un puerto (ej: 3000)
- `web-client` en otro puerto (ej: 3001)
- `api-lambdas` puede usar serverless-offline
- Ejecuta en modo watch (recarga automática)

**⚠️ Nota:** Este comando es persistente (no termina). Turborepo lo marca con `"persistent": true` y `"cache": false`.

**¿Cuándo usarlo?**
- Desarrollo full-stack
- Necesitas todas las apps corriendo simultáneamente
- Testing de integración entre apps

### 2. Iniciar Desarrollo de una App Específica

```bash
pnpm turbo run dev --filter=web-admin
```

**¿Qué hace?**
- Inicia solo el servidor de desarrollo de `web-admin`
- Más ligero en recursos
- Logs más limpios

**¿Cuándo usarlo?**
- Solo trabajas en una aplicación
- Quieres ahorrar recursos de CPU/memoria
- Debugging más enfocado


### 3. Desarrollo con Dependencias

```bash
pnpm turbo run dev --filter=web-admin...
```

**¿Qué hace?**
- Inicia `web-admin` y construye sus dependencias primero
- Asegura que los paquetes internos estén actualizados
- Útil si modificas `@yaestuvo/core` y `web-admin` simultáneamente

---

## Comandos de Linting

### 1. Ejecutar Lint en Todo el Monorepo

```bash
pnpm turbo run lint
# o el atajo:
pnpm lint
```

**¿Qué hace?**
- Ejecuta ESLint en todos los paquetes y aplicaciones
- Verifica estilo de código y errores comunes
- Construye dependencias primero (por `dependsOn: ["^build"]`)

**Salida esperada:**
```
@yaestuvo/core:lint: ✓ 45 files checked, 0 errors, 0 warnings
@yaestuvo/adapters-db:lint: ✓ 12 files checked, 0 errors, 0 warnings
```

### 2. Lint de un Paquete Específico

```bash
pnpm turbo run lint --filter=@yaestuvo/core
```

**¿Cuándo usarlo?**
- Verificar código antes de commit
- Después de refactorización
- Como parte de pre-commit hooks

### 3. Lint con Auto-fix

```bash
pnpm --filter=@yaestuvo/core lint --fix
```

**¿Qué hace?**
- Ejecuta lint y corrige automáticamente problemas solucionables
- Formatea código según reglas
- Ahorra tiempo en correcciones manuales

---

## Gestión de Paquetes

### 1. Ver Dependencias de un Paquete

```bash
pnpm list --filter=@yaestuvo/core
```

**¿Qué hace?**
- Muestra todas las dependencias instaladas en ese paquete
- Incluye dependencias transitivas
- Útil para auditoría


### 2. Actualizar Dependencias

```bash
pnpm update --filter=@yaestuvo/core
```

**¿Qué hace?**
- Actualiza dependencias a las últimas versiones permitidas por package.json
- Respeta rangos de versiones (^, ~)
- Actualiza pnpm-lock.yaml

### 3. Remover Dependencia

```bash
pnpm remove <paquete> --filter=@yaestuvo/core
```

**Ejemplo:**
```bash
pnpm remove lodash --filter=@yaestuvo/core
```

### 4. Ver Workspaces

```bash
pnpm list --depth 0
```

**¿Qué hace?**
- Lista todos los workspaces del monorepo
- Muestra la estructura de paquetes

### 5. Ejecutar Script Personalizado

```bash
pnpm --filter=@yaestuvo/core <nombre-script>
```

**Ejemplo:**
```bash
pnpm --filter=@yaestuvo/core test:watch
```

---

## Tabla Comparativa de Comandos

| Comando | ¿Qué hace? | Velocidad | Cuándo usar |
|---------|-----------|-----------|-------------|
| `pnpm turbo run test` | Ejecuta tests en todos los paquetes con caché y paralelización | ⚡⚡⚡ Muy rápido (con caché) | CI/CD, verificación completa |
| `pnpm test --filter=@yaestuvo/core` | Ejecuta tests solo en core, sin Turborepo | ⚡⚡ Rápido | Desarrollo enfocado en un paquete |
| `cd packages/core && pnpm test` | Ejecuta tests directamente en el directorio | ⚡⚡ Rápido | Trabajo local intensivo en un paquete |
| `pnpm --filter=@yaestuvo/core test Order.test.ts` | Ejecuta un archivo específico | ⚡⚡⚡ Muy rápido | Debugging de prueba específica |


### Diferencias Clave:

**`pnpm turbo run test`**
- ✅ Usa caché de Turborepo
- ✅ Ejecuta en paralelo
- ✅ Respeta grafo de dependencias
- ❌ Overhead de Turborepo (mínimo)

**`pnpm test --filter=@yaestuvo/core`**
- ✅ Directo, sin overhead
- ✅ Usa scripts de package.json
- ❌ Sin caché de Turborepo
- ❌ Sin paralelización

**`cd packages/core && pnpm test`**
- ✅ Más directo posible
- ✅ Útil para desarrollo local
- ❌ Requiere cambiar directorio
- ❌ No aprovecha features del monorepo

### Implicaciones de Rendimiento:

**Primera ejecución (sin caché):**
```
pnpm turbo run test:        ~15s (todos los paquetes)
pnpm test --filter=core:    ~3s  (solo core)
cd core && pnpm test:       ~3s  (solo core)
```

**Segunda ejecución (con caché, sin cambios):**
```
pnpm turbo run test:        ~0.5s (cache hit!)
pnpm test --filter=core:    ~3s   (sin caché)
cd core && pnpm test:       ~3s   (sin caché)
```

**Conclusión:** Turborepo brilla cuando ejecutas comandos repetidamente o en múltiples paquetes.

---

## Flujos de Trabajo Comunes

### Flujo 1: Desarrollo Diario

```bash
# 1. Actualizar código
git pull origin main

# 2. Instalar dependencias (si hubo cambios)
pnpm install

# 3. Iniciar desarrollo en tu app
pnpm turbo run dev --filter=web-admin

# 4. En otra terminal, ejecutar tests en watch mode
pnpm --filter=@yaestuvo/core test:watch

# 5. Hacer cambios en el código...

# 6. Antes de commit, verificar
pnpm turbo run lint
pnpm turbo run test

# 7. Commit
git add .
git commit -m "feat: nueva funcionalidad"
```


### Flujo 2: Agregar Nueva Funcionalidad

```bash
# 1. Crear rama
git checkout -b feature/nueva-funcionalidad

# 2. Agregar dependencia necesaria
pnpm add zod --filter=@yaestuvo/core

# 3. Desarrollar con TDD
pnpm --filter=@yaestuvo/core test:watch

# 4. Escribir código y pruebas...

# 5. Verificar que todo compila
pnpm turbo run build --filter=@yaestuvo/core

# 6. Ejecutar todas las pruebas
pnpm turbo run test

# 7. Verificar lint
pnpm turbo run lint --filter=@yaestuvo/core

# 8. Si usas la funcionalidad en una app, probarla
pnpm turbo run dev --filter=web-admin

# 9. Commit y push
git add .
git commit -m "feat: agregar validación con zod"
git push origin feature/nueva-funcionalidad
```

### Flujo 3: Ejecutar Pruebas Antes de Commit

```bash
# Opción 1: Rápida (solo lo que cambió)
pnpm turbo run test --filter=@yaestuvo/core

# Opción 2: Completa (todo el monorepo)
pnpm turbo run test

# Opción 3: Con cobertura
pnpm --filter=@yaestuvo/core test:coverage

# Verificar que la cobertura sea aceptable (>80%)
# Luego hacer commit
```

### Flujo 4: Debugging de un Paquete Específico

```bash
# 1. Navegar al paquete
cd packages/core

# 2. Ejecutar pruebas en modo watch
pnpm test:watch

# 3. En otra terminal, ejecutar lint
pnpm lint

# 4. Si necesitas reconstruir
pnpm build

# 5. Volver a la raíz cuando termines
cd ../..
```


### Flujo 5: Trabajar en Múltiples Paquetes Relacionados

```bash
# Escenario: Modificas @yaestuvo/core y @yaestuvo/adapters-db

# 1. Iniciar tests en watch para ambos
# Terminal 1:
pnpm --filter=@yaestuvo/core test:watch

# Terminal 2:
pnpm --filter=@yaestuvo/adapters-db test:watch

# 2. Hacer cambios en core...

# 3. Reconstruir core para que adapters-db vea los cambios
pnpm turbo run build --filter=@yaestuvo/core

# 4. Hacer cambios en adapters-db...

# 5. Verificar que todo funciona junto
pnpm turbo run test --filter=@yaestuvo/core --filter=@yaestuvo/adapters-db

# 6. Construir ambos
pnpm turbo run build --filter=@yaestuvo/core --filter=@yaestuvo/adapters-db
```

---

## Entendiendo el Flag --filter

El flag `--filter` es una de las características más poderosas de pnpm para trabajar en monorepos.

### Sintaxis Básica

```bash
--filter=<nombre-del-paquete>
```

### Ejemplos de Uso

**1. Filtrar por nombre exacto:**
```bash
pnpm --filter=@yaestuvo/core test
```
Ejecuta `test` solo en el paquete `@yaestuvo/core`.

**2. Filtrar con patrón glob:**
```bash
pnpm --filter="@yaestuvo/*" test
```
Ejecuta `test` en todos los paquetes que empiecen con `@yaestuvo/`.

**3. Filtrar por directorio:**
```bash
pnpm --filter="./packages/*" test
```
Ejecuta `test` en todos los paquetes dentro de `packages/`.

**4. Incluir dependencias (tres puntos al final):**
```bash
pnpm --filter=@yaestuvo/adapters-db... build
```
Construye `@yaestuvo/adapters-db` Y todas sus dependencias (como `@yaestuvo/core`).


**5. Incluir dependientes (tres puntos al inicio):**
```bash
pnpm --filter=...@yaestuvo/core build
```
Construye `@yaestuvo/core` Y todos los paquetes que dependen de él (como `@yaestuvo/adapters-db`).

**6. Múltiples filtros:**
```bash
pnpm --filter=@yaestuvo/core --filter=@yaestuvo/shared-dto test
```
Ejecuta `test` en ambos paquetes.

**7. Excluir paquetes:**
```bash
pnpm --filter="!@yaestuvo/core" test
```
Ejecuta `test` en todos los paquetes EXCEPTO `@yaestuvo/core`.

### Casos de Uso Prácticos

**Caso 1: Modificaste core y quieres verificar qué se rompe**
```bash
pnpm --filter=...@yaestuvo/core test
```
Esto ejecuta tests en core y en todos los paquetes que lo usan.

**Caso 2: Quieres construir una app con todas sus dependencias**
```bash
pnpm turbo run build --filter=web-admin...
```
Construye `web-admin` y todos los paquetes internos que necesita.

**Caso 3: Ejecutar comando en todas las apps pero no en packages**
```bash
pnpm --filter="./apps/*" dev
```

**Caso 4: Ejecutar en todos los paquetes excepto uno problemático**
```bash
pnpm --filter="!@yaestuvo/ui-lib" test
```

### Diferencia entre --filter y turbo --filter

```bash
# pnpm filter: Selecciona workspaces para pnpm
pnpm --filter=@yaestuvo/core test

# turbo filter: Selecciona workspaces para Turborepo
pnpm turbo run test --filter=@yaestuvo/core
```

**¿Cuál usar?**
- Usa `turbo --filter` cuando quieras aprovechar caché y paralelización
- Usa `pnpm --filter` para comandos directos sin Turborepo

---

## Entendiendo el Caché de Turborepo


### ¿Cómo Funciona el Caché?

Turborepo crea un hash único basado en:

1. **Contenido de archivos fuente**
   - Todos los archivos `.ts`, `.tsx`, `.js`, etc.
   - Archivos de configuración (`tsconfig.json`, `jest.config.js`)

2. **Dependencias**
   - Contenido de `package.json`
   - Versiones en `pnpm-lock.yaml`

3. **Variables de entorno**
   - Archivos `.env` (según `globalDependencies` en `turbo.json`)

4. **Comando ejecutado**
   - `build`, `test`, `lint`, etc.

5. **Outputs de dependencias**
   - Si `@yaestuvo/adapters-db` depende de `@yaestuvo/core`, el hash incluye el output de core

### Ejemplo de Caché en Acción

**Primera ejecución:**
```bash
$ pnpm turbo run test --filter=@yaestuvo/core

@yaestuvo/core:test: cache miss, executing
@yaestuvo/core:test: PASS src/entities/Order.test.ts
@yaestuvo/core:test: Test Suites: 5 passed, 5 total
@yaestuvo/core:test: Tests: 23 passed, 23 total
@yaestuvo/core:test: Time: 3.2s

Tasks: 1 successful, 1 total
Cached: 0 cached, 1 total
Time: 3.5s
```

**Segunda ejecución (sin cambios):**
```bash
$ pnpm turbo run test --filter=@yaestuvo/core

@yaestuvo/core:test: cache hit, replaying output
@yaestuvo/core:test: PASS src/entities/Order.test.ts
@yaestuvo/core:test: Test Suites: 5 passed, 5 total
@yaestuvo/core:test: Tests: 23 passed, 23 total
@yaestuvo/core:test: Time: 3.2s

Tasks: 1 successful, 1 total
Cached: 1 cached, 1 total
Time: 0.2s >>> FULL TURBO ⚡
```

¡De 3.5s a 0.2s! 🚀


### ¿Cuándo se Invalida el Caché?

El caché se invalida cuando cambia cualquiera de estos:

1. **Código fuente**
   ```bash
   # Modificas src/entities/Order.ts
   # Próximo test de @yaestuvo/core será cache miss
   ```

2. **Dependencias**
   ```bash
   pnpm add zod --filter=@yaestuvo/core
   # Invalida caché de core
   ```

3. **Configuración**
   ```bash
   # Modificas jest.config.js
   # Invalida caché de tests
   ```

4. **Variables de entorno**
   ```bash
   # Modificas .env.local
   # Invalida caché (si está en globalDependencies)
   ```

### Configuración del Caché en turbo.json

```json
{
  "tasks": {
    "build": {
      "dependsOn": ["^build"],
      "outputs": ["dist/**", ".next/**", "build/**"]
    },
    "test": {
      "dependsOn": ["^build"],
      "outputs": ["coverage/**"]
    },
    "dev": {
      "cache": false,
      "persistent": true
    }
  }
}
```

**Explicación:**

- `"outputs"`: Archivos que Turborepo cachea y restaura
- `"dependsOn": ["^build"]`: Ejecuta `build` de dependencias primero
- `"cache": false`: Desactiva caché (para `dev` que es interactivo)
- `"persistent": true`: Indica que el comando no termina (servidores)

### Comandos de Gestión de Caché

**Ver estadísticas de caché:**
```bash
pnpm turbo run build --summarize
```

**Limpiar caché:**
```bash
rm -rf .turbo
```

**Forzar ejecución sin caché:**
```bash
pnpm turbo run test --force
```


### Beneficios del Caché

**1. Velocidad en CI/CD:**
```bash
# Sin caché: 15 minutos
# Con caché: 2 minutos (si solo cambió 1 paquete)
```

**2. Desarrollo local más rápido:**
```bash
# Cambias entre ramas frecuentemente
git checkout feature-A  # Ejecutas tests: 15s
git checkout feature-B  # Ejecutas tests: 0.5s (caché!)
git checkout feature-A  # Ejecutas tests: 0.5s (caché!)
```

**3. Ahorro de recursos:**
- Menos uso de CPU
- Menos tiempo esperando
- Más tiempo programando

---

## Solución de Problemas

### Problema 1: "Package not found"

**Error:**
```bash
$ pnpm --filter=@yaestuvo/core test
Error: No projects matched the filters "@yaestuvo/core"
```

**Soluciones:**

1. **Verificar el nombre exacto:**
   ```bash
   cat packages/core/package.json | grep "name"
   ```

2. **Verificar que esté en pnpm-workspace.yaml:**
   ```bash
   cat pnpm-workspace.yaml
   ```

3. **Reinstalar dependencias:**
   ```bash
   pnpm install
   ```

### Problema 2: Caché Corrupto

**Síntomas:**
- Tests pasan localmente pero fallan en CI
- Builds producen resultados inconsistentes
- Cambios no se reflejan

**Soluciones:**

1. **Limpiar caché de Turborepo:**
   ```bash
   rm -rf .turbo
   pnpm turbo run build --force
   ```

2. **Limpiar node_modules:**
   ```bash
   rm -rf node_modules
   rm pnpm-lock.yaml
   pnpm install
   ```

3. **Limpiar todo:**
   ```bash
   pnpm clean
   pnpm install
   pnpm build
   ```


### Problema 3: Errores de Dependencias

**Error:**
```bash
Cannot find module '@yaestuvo/core'
```

**Soluciones:**

1. **Verificar que core esté construido:**
   ```bash
   pnpm turbo run build --filter=@yaestuvo/core
   ```

2. **Verificar dependencia en package.json:**
   ```json
   {
     "dependencies": {
       "@yaestuvo/core": "workspace:*"
     }
   }
   ```

3. **Reinstalar:**
   ```bash
   pnpm install
   ```

### Problema 4: Build Failures en Monorepo

**Error:**
```bash
@yaestuvo/adapters-db:build: Error: Cannot find module '@yaestuvo/core'
```

**Causa:** Las dependencias no se construyeron en orden.

**Solución:**

1. **Verificar turbo.json:**
   ```json
   {
     "tasks": {
       "build": {
         "dependsOn": ["^build"]
       }
     }
   }
   ```

2. **Construir con dependencias:**
   ```bash
   pnpm turbo run build --filter=@yaestuvo/adapters-db...
   ```

### Problema 5: Tests Fallan Solo en CI

**Posibles causas:**

1. **Variables de entorno diferentes:**
   - Verifica que CI tenga las mismas variables
   - Revisa `globalDependencies` en `turbo.json`

2. **Versiones de Node diferentes:**
   ```bash
   # Verifica en package.json
   "engines": {
     "node": ">=20.0.0"
   }
   ```

3. **Caché de CI corrupto:**
   - Limpia el caché de CI
   - Ejecuta build limpio


### Problema 6: pnpm install es Lento

**Soluciones:**

1. **Usar caché de pnpm:**
   ```bash
   # El caché está en ~/.pnpm-store
   # No lo borres a menos que sea necesario
   ```

2. **Verificar conexión a internet:**
   - pnpm descarga paquetes en paralelo
   - Conexión lenta afecta mucho

3. **Usar mirror local (avanzado):**
   ```bash
   # Configurar registry local
   pnpm config set registry https://registry.npmmirror.com
   ```

### Problema 7: Cambios en Paquete Interno No Se Reflejan

**Escenario:** Modificas `@yaestuvo/core` pero `web-admin` no ve los cambios.

**Soluciones:**

1. **Reconstruir el paquete:**
   ```bash
   pnpm turbo run build --filter=@yaestuvo/core
   ```

2. **Usar modo watch (si está disponible):**
   ```bash
   pnpm --filter=@yaestuvo/core build --watch
   ```

3. **Verificar que la app importe correctamente:**
   ```typescript
   // Debe ser:
   import { Order } from '@yaestuvo/core';
   
   // No:
   import { Order } from '../../packages/core/src';
   ```

---

## Hoja de Referencia Rápida

### Comandos Esenciales

| Comando | Descripción |
|---------|-------------|
| `pnpm install` | Instalar todas las dependencias |
| `pnpm build` | Construir todo el monorepo |
| `pnpm dev` | Iniciar todos los servidores de desarrollo |
| `pnpm test` | Ejecutar todas las pruebas |
| `pnpm lint` | Ejecutar lint en todo el código |


### Comandos con Filtros

| Comando | Descripción |
|---------|-------------|
| `pnpm turbo run build --filter=@yaestuvo/core` | Construir solo core |
| `pnpm turbo run test --filter=@yaestuvo/core` | Probar solo core |
| `pnpm turbo run dev --filter=web-admin` | Iniciar solo web-admin |
| `pnpm --filter=@yaestuvo/core test:watch` | Tests en modo watch |
| `pnpm --filter=@yaestuvo/core test:coverage` | Tests con cobertura |

### Gestión de Dependencias

| Comando | Descripción |
|---------|-------------|
| `pnpm add <pkg> --filter=@yaestuvo/core` | Agregar dependencia a core |
| `pnpm add -D <pkg> --filter=@yaestuvo/core` | Agregar dev dependency |
| `pnpm remove <pkg> --filter=@yaestuvo/core` | Remover dependencia |
| `pnpm add -D <pkg> -w` | Agregar a la raíz del monorepo |
| `pnpm update --filter=@yaestuvo/core` | Actualizar dependencias |

### Comandos de Caché

| Comando | Descripción |
|---------|-------------|
| `pnpm turbo run build --force` | Construir sin usar caché |
| `rm -rf .turbo` | Limpiar caché de Turborepo |
| `pnpm clean` | Limpiar builds y node_modules |

### Patrones de Filter Avanzados

| Patrón | Descripción |
|--------|-------------|
| `--filter=@yaestuvo/core...` | Core + sus dependencias |
| `--filter=...@yaestuvo/core` | Core + paquetes que lo usan |
| `--filter="@yaestuvo/*"` | Todos los paquetes @yaestuvo |
| `--filter="./apps/*"` | Todas las aplicaciones |
| `--filter="!@yaestuvo/core"` | Todos excepto core |

### Atajos Útiles

```bash
# Alias útiles para tu .bashrc o .zshrc
alias pt="pnpm turbo run"
alias pf="pnpm --filter"
alias ptb="pnpm turbo run build"
alias ptt="pnpm turbo run test"
alias ptd="pnpm turbo run dev"

# Uso:
pt test --filter=@yaestuvo/core
pf @yaestuvo/core test:watch
```


### Estructura del Proyecto

```
microempresas-gt-platform/
├── apps/
│   ├── api-lambdas/          # Funciones Lambda (Backend)
│   ├── web-admin/            # Panel de administración
│   └── web-client/           # Aplicación del cliente
├── packages/
│   ├── core/                 # @yaestuvo/core - Lógica de negocio
│   ├── adapters-db/          # @yaestuvo/adapters-db - DynamoDB
│   ├── shared-dto/           # @yaestuvo/shared-dto - DTOs
│   ├── ui-lib/               # @yaestuvo/ui-lib - Componentes UI
│   └── tsconfig/             # @yaestuvo/tsconfig - Configs TS
├── package.json              # Scripts raíz del monorepo
├── pnpm-workspace.yaml       # Configuración de workspaces
├── turbo.json                # Configuración de Turborepo
└── pnpm-lock.yaml            # Lockfile de dependencias
```

---

## Recursos Adicionales

### Documentación Oficial

- **pnpm:** https://pnpm.io/
- **Turborepo:** https://turbo.build/repo/docs
- **pnpm Workspaces:** https://pnpm.io/workspaces
- **Turborepo Filtering:** https://turbo.build/repo/docs/core-concepts/monorepos/filtering

### Conceptos Clave para Profundizar

1. **Content-Addressable Storage** - Cómo pnpm ahorra espacio
2. **Dependency Graph** - Cómo Turborepo optimiza ejecución
3. **Remote Caching** - Compartir caché entre equipo (Turborepo)
4. **Incremental Builds** - Construir solo lo que cambió

### Tips Finales

1. **Usa Turborepo para comandos repetitivos** (build, test, lint)
2. **Usa pnpm filter directo para desarrollo enfocado** (test:watch)
3. **Aprovecha el caché** - ejecuta comandos frecuentemente sin miedo
4. **Mantén paquetes pequeños y enfocados** - mejor paralelización
5. **Documenta scripts personalizados** en package.json de cada paquete

---

**¿Preguntas o problemas?** Consulta la sección de [Solución de Problemas](#solución-de-problemas) o revisa los logs detallados que Turborepo proporciona.

¡Feliz desarrollo en el monorepo! 🚀

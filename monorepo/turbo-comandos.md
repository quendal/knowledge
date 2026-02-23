Excelente pregunta. Déjame explicarte las diferencias:

## Comandos que ejecuté

Durante la implementación de los tests, ejecuté:

```bash
# Desde el directorio packages/core/
npm test
```

Este comando:
- Se ejecuta **dentro del paquete específico** (`packages/core/`)
- Usa el script `test` definido en `packages/core/package.json`
- Solo ejecuta los tests de ese paquete
- No usa Turborepo ni pnpm workspaces

## El comando que mencionaste

```bash
# Desde el directorio raíz del monorepo
pnpm test --filter=@yaestuvo/core
```

Este comando:
- Se ejecuta desde la **raíz del monorepo**
- Usa **pnpm workspaces** con el flag `--filter`
- Filtra para ejecutar tests solo del paquete `@yaestuvo/core`
- Puede aprovechar el caché de Turborepo si está configurado

## Diferencias clave

| Aspecto | `npm test` (en packages/core) | `pnpm test --filter=@yaestuvo/core` (en raíz) |
|---------|-------------------------------|------------------------------------------------|
| **Directorio** | `packages/core/` | Raíz del monorepo |
| **Herramienta** | npm directamente | pnpm workspaces |
| **Scope** | Solo ese paquete | Filtra desde todos los paquetes |
| **Caché** | No usa Turborepo | Puede usar caché de Turborepo |
| **Dependencias** | No verifica dependencias entre paquetes | Respeta el grafo de dependencias |

## Dónde ejecutar cada comando

### Opción 1: Desde el paquete específico
```bash
cd packages/core
npm test
# o
npm test -- DraftOrderUseCase  # Para un archivo específico
```

### Opción 2: Desde la raíz con pnpm
```bash
# Estando en la raíz del monorepo
pnpm test --filter=@yaestuvo/core
```

### Opción 3: Desde la raíz con Turborepo (recomendado para monorepos)
```bash
# Estando en la raíz del monorepo
pnpm turbo run test --filter=@yaestuvo/core
```

## Recomendación para tu proyecto

Para un monorepo con Turborepo, lo ideal es:

1. **Durante desarrollo** (iteración rápida):
   ```bash
   cd packages/core
   npm test -- --watch
   ```

2. **Para CI/CD o validación completa**:
   ```bash
   # Desde la raíz
   pnpm turbo run test --filter=@yaestuvo/core
   ```

3. **Para ejecutar todos los tests del monorepo**:
   ```bash
   # Desde la raíz
   pnpm turbo run test
   ```

¿Quieres que verifique la configuración de Turborepo en tu `turbo.json` para asegurarnos de que el pipeline de tests esté optimizado?

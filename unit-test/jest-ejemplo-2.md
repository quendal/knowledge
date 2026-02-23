Excelente pregunta! Te explico paso a paso cómo funcionan estas pruebas unitarias sin necesidad de tener DynamoDB creado. Es un concepto clave de Clean Architecture y testing.

## Concepto Clave: Mocking y Dependency Injection

### 1. **Separación de Interfaces e Implementaciones**

Mira la estructura del código:

```typescript
// DraftOrderUseCase recibe INTERFACES, no implementaciones concretas
constructor(
  private readonly orderRepository: IOrderRepository,  // ← Interface
  private readonly productRepository: IProductRepository  // ← Interface
) {}
```

La clase `DraftOrderUseCase` NO sabe que DynamoDB existe. Solo conoce las interfaces `IOrderRepository` e `IProductRepository`.

### 2. **¿Qué es un Mock?**

Un mock es un "objeto falso" que simula el comportamiento de una dependencia real. Jest crea estos mocks automáticamente:

```typescript
// Jest crea un objeto falso que implementa IProductRepository
let mockProductRepo: jest.Mocked<IProductRepository>;

mockProductRepo = {
  create: jest.fn(),        // ← Función falsa
  findById: jest.fn(),      // ← Función falsa
  update: jest.fn(),        // ← Función falsa
  decreaseStock: jest.fn(), // ← Función falsa
  increaseStock: jest.fn(), // ← Función falsa
  findByMerchant: jest.fn() // ← Función falsa
};
```

### 3. **Flujo Paso a Paso de una Prueba**

Tomemos este test como ejemplo:

```typescript
it('should recalculate prices from database ignoring any client-provided prices', async () => {
  // PASO 1: ARRANGE (Preparar)
  // Creamos un producto falso en memoria
  const simpleProduct: Product = {
    id: 'prod-1',
    merchantId: 'merchant-1',
    type: 'SIMPLE',
    name: 'Test Product',
    basePrice: 150, // ← Este es el precio "real" de la base de datos
    // ... otros campos
  };

  // PASO 2: Configuramos el mock
  // Le decimos: "Cuando alguien llame findById, devuelve este producto falso"
  mockProductRepo.findById.mockResolvedValue(simpleProduct);
  
  // PASO 3: ACT (Actuar)
  // Ejecutamos el caso de uso
  const order = await useCase.execute(input);
  
  // PASO 4: ASSERT (Verificar)
  // Verificamos que el caso de uso usó el precio de la "base de datos" (nuestro mock)
  expect(order.items[0].unitPrice).toBe(150); // ← Precio del mock, no del cliente
});
```

### 4. **¿Qué Pasa Internamente?**

Cuando ejecutas `useCase.execute(input)`, esto sucede:

```typescript
// Dentro de DraftOrderUseCase.execute()

// 1. El caso de uso llama al repositorio
const product = await this.productRepository.findById(
  input.merchantId,
  inputItem.productId
);

// 2. Como productRepository es un MOCK, Jest intercepta la llamada
// 3. Jest ve que configuramos: mockProductRepo.findById.mockResolvedValue(simpleProduct)
// 4. Jest devuelve el producto falso que configuramos
// 5. El caso de uso continúa con ese producto falso

// 6. El caso de uso usa el precio del producto falso
unitPrice = product.basePrice; // ← 150 del mock

// 7. Crea la orden con ese precio
const order: Order = {
  // ...
  items: [{
    unitPrice: 150, // ← Precio del mock
    // ...
  }]
};

// 8. Llama a orderRepository.create(order)
await this.orderRepository.create(order);

// 9. Como orderRepository también es un mock, Jest solo registra la llamada
// No hace nada real, solo guarda que se llamó con esos parámetros
```

### 5. **Ventajas de Este Enfoque**

```typescript
// ✅ VENTAJAS:

// 1. No necesitas DynamoDB instalado
// 2. Las pruebas son RÁPIDAS (milisegundos, no segundos)
// 3. Puedes simular cualquier escenario fácilmente:

// Simular producto no encontrado:
mockProductRepo.findById.mockResolvedValue(null);

// Simular error de red:
mockProductRepo.findById.mockRejectedValue(new Error('Network error'));

// Simular diferentes precios:
mockProductRepo.findById.mockResolvedValue({ ...product, basePrice: 999 });

// 4. Pruebas aisladas: solo testeas la LÓGICA del caso de uso
// 5. Puedes verificar que se llamaron los métodos correctos:
expect(mockProductRepo.findById).toHaveBeenCalledWith('merchant-1', 'prod-1');
expect(mockOrderRepo.create).toHaveBeenCalledWith(order);
```

### 6. **Diagrama del Flujo**

```
┌─────────────────────────────────────────────────────────┐
│                    PRUEBA UNITARIA                      │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  1. Crear mocks (objetos falsos)                       │
│     mockProductRepo = { findById: jest.fn() }          │
│                                                         │
│  2. Configurar comportamiento del mock                 │
│     mockProductRepo.findById.mockResolvedValue(...)    │
│                                                         │
│  3. Inyectar mocks al caso de uso                      │
│     useCase = new DraftOrderUseCase(                   │
│       mockOrderRepo,    ← Mock, no DynamoDB real       │
│       mockProductRepo   ← Mock, no DynamoDB real       │
│     )                                                   │
│                                                         │
│  4. Ejecutar caso de uso                               │
│     order = await useCase.execute(input)               │
│                                                         │
│     ┌─────────────────────────────────────┐            │
│     │   DraftOrderUseCase.execute()       │            │
│     │                                     │            │
│     │   productRepository.findById() ────┼───┐        │
│     │                                     │   │        │
│     │   ← Mock devuelve producto falso ──┼───┘        │
│     │                                     │            │
│     │   orderRepository.create() ────────┼───┐        │
│     │                                     │   │        │
│     │   ← Mock solo registra llamada ────┼───┘        │
│     └─────────────────────────────────────┘            │
│                                                         │
│  5. Verificar resultados                               │
│     expect(order.items[0].unitPrice).toBe(150)         │
│     expect(mockProductRepo.findById).toHaveBeenCalled()│
│                                                         │
└─────────────────────────────────────────────────────────┘

         ❌ NO SE USA DYNAMODB REAL
         ✅ TODO ES EN MEMORIA
         ⚡ SUPER RÁPIDO
```

### 7. **Comparación: Unit Tests vs Integration Tests**

```typescript
// ═══════════════════════════════════════════════════════
// UNIT TEST (Lo que estamos viendo)
// ═══════════════════════════════════════════════════════
// ✅ Usa mocks
// ✅ No necesita infraestructura
// ✅ Rápido (< 100ms)
// ✅ Testea LÓGICA de negocio
// ❌ No verifica que DynamoDB funcione

const mockRepo = { findById: jest.fn() };
const useCase = new DraftOrderUseCase(mockRepo, ...);


// ═══════════════════════════════════════════════════════
// INTEGRATION TEST (Fase 2 del proyecto)
// ═══════════════════════════════════════════════════════
// ✅ Usa DynamoDB Local o real
// ✅ Verifica que queries funcionan
// ❌ Más lento (segundos)
// ❌ Necesita setup de infraestructura

const realRepo = new DynamoProductRepository(dynamoClient);
const useCase = new DraftOrderUseCase(realRepo, ...);
```

### 8. **Ejemplo Concreto de Verificación**

```typescript
// Puedes verificar que el mock fue llamado correctamente:
expect(mockProductRepo.findById).toHaveBeenCalledWith(
  'merchant-1',  // ← merchantId correcto
  'prod-1'       // ← productId correcto
);

// Puedes verificar cuántas veces se llamó:
expect(mockProductRepo.findById).toHaveBeenCalledTimes(1);

// Puedes verificar que NO se llamó algo:
expect(mockOrderRepo.create).not.toHaveBeenCalled(); // En caso de error
```

### Resumen

Las pruebas unitarias funcionan sin DynamoDB porque:

1. **Usamos interfaces** (`IOrderRepository`, `IProductRepository`)
2. **Jest crea mocks** que implementan esas interfaces
3. **Configuramos el comportamiento** de los mocks (`mockResolvedValue`)
4. **Inyectamos los mocks** al caso de uso
5. **El caso de uso no sabe** que está usando mocks en lugar de DynamoDB real
6. **Verificamos la lógica** del caso de uso, no la infraestructura

Esto es Clean Architecture en acción: la lógica de negocio (`DraftOrderUseCase`) está completamente desacoplada de la infraestructura (DynamoDB), permitiendo testearla de forma aislada y rápida.

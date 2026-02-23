# Cómo Testear Sin Base de Datos: Una Intro a Clean Architecture

Hey! Déjame explicarte esto como si estuviéramos tomando un café. Es más simple de lo que parece.

## El Problema Clásico

Imagina que escribes esto:

```typescript
// ❌ Código acoplado (malo)
class OrderService {
  async createOrder(data) {
    // Habla directamente con DynamoDB
    const product = await dynamoDB.getItem({
      TableName: 'Products',
      Key: { id: data.productId }
    });
    
    // Más lógica...
  }
}
```

**Problema:** Para testear esto necesitas DynamoDB corriendo. Lento, complicado, frágil.

## La Solución: Interfaces

Clean Architecture dice: "Habla con interfaces, no con implementaciones concretas".

### 1. Define el Contrato (Interface)

```typescript
// IProductRepository.ts - El "contrato"
export interface IProductRepository {
  findById(merchantId: string, productId: string): Promise<Product | null>;
  create(product: Product): Promise<void>;
  // ... más métodos
}
```

Esto es como decir: "Necesito algo que sepa buscar productos. No me importa CÓMO lo haga".

### 2. Tu Lógica de Negocio Usa la Interface

```typescript
// DraftOrderUseCase.ts
export class DraftOrderUseCase {
  constructor(
    private productRepo: IProductRepository  // ← Solo conoce la interface
  ) {}

  async execute(input) {
    // Usa la interface, no sabe si es DynamoDB, MySQL, o un mock
    const product = await this.productRepo.findById(
      input.merchantId, 
      input.productId
    );
    
    if (!product) throw new ProductNotFoundError();
    
    // Lógica de negocio...
    const order = {
      items: [{
        unitPrice: product.basePrice,  // ← Precio de la "base de datos"
        quantity: input.quantity
      }]
    };
    
    return order;
  }
}
```

### 3. En Producción: Implementación Real

```typescript
// DynamoProductRepository.ts - Implementa la interface
export class DynamoProductRepository implements IProductRepository {
  async findById(merchantId: string, productId: string) {
    // Aquí SÍ hablas con DynamoDB
    const result = await this.dynamoClient.send(new GetItemCommand({
      TableName: 'SaaS_Inventory',
      Key: { PK: `MERCHANT#${merchantId}`, SK: `PRODUCT#${productId}` }
    }));
    
    return result.Item ? this.mapToDomain(result.Item) : null;
  }
}
```

### 4. En Tests: Implementación Falsa (Mock)

```typescript
// DraftOrderUseCase.test.ts
describe('DraftOrderUseCase', () => {
  it('usa el precio de la base de datos', async () => {
    // 1. Crea un repositorio FALSO
    const fakeRepo = {
      findById: jest.fn()  // ← Función espía de Jest
    };
    
    // 2. Configura qué debe devolver
    fakeRepo.findById.mockResolvedValue({
      id: 'prod-1',
      basePrice: 150  // ← Precio "de la base de datos"
    });
    
    // 3. Inyecta el fake al caso de uso
    const useCase = new DraftOrderUseCase(fakeRepo);
    
    // 4. Ejecuta
    const order = await useCase.execute({
      merchantId: 'merchant-1',
      productId: 'prod-1',
      quantity: 2
    });
    
    // 5. Verifica
    expect(order.items[0].unitPrice).toBe(150);  // ✅ Usó el precio del mock
    expect(fakeRepo.findById).toHaveBeenCalledWith('merchant-1', 'prod-1');
  });
});
```

## ¿Cómo Funciona el Mock?

Piensa en el mock como un actor de teatro:

```typescript
// El "actor" (mock) memoriza su guion
mockProductRepo.findById.mockResolvedValue({
  id: 'prod-1',
  basePrice: 150
});

// Cuando el caso de uso "actúa"...
const product = await productRepo.findById('merchant-1', 'prod-1');

// El mock responde con su línea memorizada
console.log(product.basePrice); // 150

// Y Jest registra que actuó
expect(mockProductRepo.findById).toHaveBeenCalled(); // ✅ true
```

## El Flujo Completo

```
┌─────────────────────────────────────────────────┐
│  TEST                                           │
│                                                 │
│  const fakeRepo = { findById: jest.fn() }      │
│  fakeRepo.findById.mockResolvedValue(product)  │
│                                                 │
│  const useCase = new DraftOrderUseCase(        │
│    fakeRepo  ← Inyecta el fake                 │
│  )                                              │
│                                                 │
│  await useCase.execute(...)                    │
│         │                                       │
│         ├─→ productRepo.findById()             │
│         │      │                                │
│         │      └─→ Mock devuelve producto fake │
│         │                                       │
│         └─→ Usa precio del fake                │
│                                                 │
│  expect(order.unitPrice).toBe(150) ✅          │
└─────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────┐
│  PRODUCCIÓN                                     │
│                                                 │
│  const realRepo = new DynamoProductRepository( │
│    dynamoClient                                 │
│  )                                              │
│                                                 │
│  const useCase = new DraftOrderUseCase(        │
│    realRepo  ← Inyecta el real                 │
│  )                                              │
│                                                 │
│  await useCase.execute(...)                    │
│         │                                       │
│         ├─→ productRepo.findById()             │
│         │      │                                │
│         │      └─→ DynamoDB devuelve producto  │
│         │                                       │
│         └─→ Usa precio de DynamoDB             │
└─────────────────────────────────────────────────┘
```

## Por Qué Esto Es Genial

1. **Tests rápidos**: Sin base de datos = milisegundos
2. **Tests confiables**: No dependen de red o infraestructura
3. **Fácil simular errores**: `mockRejectedValue(new Error('DB down'))`
4. **Mismo código en prod y test**: Solo cambias la implementación

## Ejemplo Real del Test

```typescript
it('lanza error si el producto no existe', async () => {
  // Mock devuelve null (producto no encontrado)
  mockProductRepo.findById.mockResolvedValue(null);
  
  const useCase = new DraftOrderUseCase(mockProductRepo, mockOrderRepo);
  
  // Verifica que lance el error correcto
  await expect(useCase.execute(input))
    .rejects
    .toThrow(ProductNotFoundError);
  
  // Y que NO haya creado la orden
  expect(mockOrderRepo.create).not.toHaveBeenCalled();
});
```

## La Magia de Jest

```typescript
// jest.fn() crea una función espía que:
const spy = jest.fn();

// 1. Puedes configurar qué devuelve
spy.mockResolvedValue({ data: 'fake' });

// 2. Puedes verificar si fue llamada
expect(spy).toHaveBeenCalled();

// 3. Puedes ver con qué argumentos
expect(spy).toHaveBeenCalledWith('arg1', 'arg2');

// 4. Puedes contar llamadas
expect(spy).toHaveBeenCalledTimes(2);
```

## Resumen en 3 Líneas

1. **Interface** = Contrato ("necesito algo que busque productos")
2. **Mock** = Actor falso que cumple el contrato en tests
3. **Implementación real** = Actor real que cumple el contrato en producción

Tu caso de uso no sabe con quién habla. Solo sabe que cumple el contrato. Eso es Clean Architecture.

# NestJS Workshop – Validation Questions

These questions require you to reason about the code, not just recall definitions. Open the files as you work through them.

---

**Q1 — Dead route diagnosis**

Look at [tasks.controller.ts:27](src/tasks/tasks.controller.ts#L27). Right now, `findAll()` has no route decorator. If you start the server and call `GET /tasks`, what response do you get — a 404, a 500, or something else? Explain *why* NestJS behaves that way, and describe exactly what you need to add to fix it.

### Obtendrías un error 404 Not Found. Aunque el método findAll() existe en el controlador, sin el decorador @Get() NestJS no sabe que debe mapear la ruta GET /tasks a ese método; el decorador es lo que le indica a NestJS qué ruta HTTP debe ejecutar qué método. Para arreglarlo, debes añadir @Get() encima del método.

---

**Q2 — When `transform: true` is not enough**

[main.ts:15](src/main.ts#L15) sets `transform: true` on the global `ValidationPipe`, which auto-converts types. Yet [products.controller.ts:32](src/products/products.controller.ts#L32) still uses `ParseIntPipe` explicitly on `@Param('id')`. If `transform: true` already handles type conversion, why is `ParseIntPipe` still there? Are they doing the same job, or is there a difference in *when* or *how* they convert and reject?

### No son lo mismo. transform: true convierte los parámetros después de la validación en el controller, mientras que ParseIntPipe convierte antes y puede validar el tipo en un paso. transform: true convierte string "42" a número 42 pero ya en el controller, mientras que ParseIntPipe valida y convierte en el middleware de parámetros y si no es un número válido lanza error 400 antes de entrar al método. Entonces ParseIntPipe es más estricto y seguro porque rechaza valores inválidos en el nivel de parámetros, no en la transformación de tipos.

---

**Q3 — Silent strip vs hard rejection**

[main.ts:13-14](src/main.ts#L13-L14) enables both `whitelist: true` and `forbidNonWhitelisted: true`. Imagine you remove `forbidNonWhitelisted: true` and keep only `whitelist: true`. Now send this request:

```bash
curl -X POST http://localhost:3000/users \
  -H "Content-Type: application/json" \
  -d '{"name": "Maria", "email": "m@m.com", "age": 20, "password": "secret"}'
```

What is the exact response status and body? What happens to `"password"` in the service? Why could this behavior be a security problem in a real app?

### Con solo whitelist: true, el status sería 201 (success), se crearía el usuario normalmente, y "password" se eliminaría silenciosamente sin llegar al servicio, lo cual es un problema de seguridad porque un cliente podría enviar campos sensibles pensando que se procesan pero se ignoran calladamente; en cambio, con forbidNonWhitelisted: true el status sería 400 Bad Request con un mensaje explícito notificando que hay un problema, lo cual es mucho más seguro que ignorar silenciosamente campos no autorizados.

---

**Q4 — Mutation side-effect**

Read [products.service.ts:44-48](src/products/products.service.ts#L44-L48). The `update` method calls `Object.assign(product, dto)`, which mutates the object in the array directly. Now read [products.service.ts:21-23](src/products/products.service.ts#L21-L23) — `findAll()` returns `this.products` directly.

If a caller modifies the object returned by `findAll()`, does that change the data stored in the service? Trace through the code and explain why. What would you change to prevent this?

### Sí, cualquier cambio al objeto retornado por findAll() afectaría directamente los datos del servicio porque findAll() retorna una referencia al array mismo, no una copia (por ejemplo, si hicieras const products = productService.findAll(); products[0].price = 0; modificarías directamente en el servicio). Para arreglarlo, deberías retornar una copia del array: return [...this.products]; o incluso una copia más profunda con return this.products.map(p => ({ ...p }));.

---

**Q5 — The optional field trap**

In [update-product.dto.ts:12-14](src/products/dto/update-product.dto.ts#L12-L14), `price` has both `@IsNumber()`, `@IsPositive()`, and `@IsOptional()`. Send this PATCH request:

```bash
curl -X PATCH http://localhost:3000/products/1 \
  -H "Content-Type: application/json" \
  -d '{"price": -50}'
```

Does validation pass or fail? Now send this:

```bash
curl -X PATCH http://localhost:3000/products/1 \
  -H "Content-Type: application/json" \
  -d '{}'
```

Does validation pass or fail? Explain the exact rule `@IsOptional()` enforces — what does "optional" actually mean to `class-validator`?

### Con el primer request {"price": -50}, la validación falla y retorna 400 Bad Request porque -50 viola @IsPositive(), pues @IsOptional() solo significa "si está presente, debe cumplir las otras reglas"; con el segundo request {}, la validación pasa retornando 200 OK y actualiza el producto pero no cambia price, porque el campo está ausente y @IsOptional() lo saltea. La regla exacta de @IsOptional() es: "Si el campo está presente, valida contra todos los otros decoradores. Si está ausente, no valida nada."

---

**Q6 — ID reuse after deletion**

Look at how `nextId` works in [tasks.service.ts:18-19](src/tasks/tasks.service.ts#L18-L19) alongside `remove` at [tasks.service.ts:50-53](src/tasks/tasks.service.ts#L50-L53). If you delete task `#1`, then create a new task, what ID does the new task get? Could `findOne(1)` ever return the wrong task? Now consider: what if the implementation used `this.tasks.length + 1` as the ID instead of `nextId` — walk through a create/delete/create sequence and show why that would break.

### Con el sistema actual de nextId, si deleteas task #1 y creas una nueva, el nuevo task recibe ID #4 (el siguiente nextId) y findOne(1) nunca retornaría nada después de deletar porque ese ID ya no existe, así que no hay conflicto. Sin embargo, si usaras this.tasks.length + 1 como ID, después de crear task #1, #2, #3, deletear #1 (length = 2) y crear uno nuevo, obtendrías ID = 3 de nuevo causando una colisión con el task #3 existente, demostrando que nextId es correcto porque incrementa sin resetear mientras que length + 1 falla al reutilizar IDs.

---

**Q7 — Module forgotten**

You finish building all five files in the Users module but forget to add `UsersModule` to the `imports` array in [app.module.ts](src/app.module.ts). What happens when you:

a) Start the server — does it crash or start normally?  
b) Send `POST /users` — what is the response status and why?

Now explain: is this a *compile-time* error, a *startup* error, or a *runtime* error in NestJS terms?

### Si olvidas añadir UsersModule a los imports en app.module.ts, el servidor arrancará normalmente sin error, pero POST /users retornará 404 Not Found porque el módulo no está registrado. Este es un error de runtime (en ejecución), no de compilación ni de startup, porque NestJS no se queja al arrancar pues técnicamente todo es válido; solo cuando intentas usar la ruta descubres que no existe.

---

**Q8 — Missing 201**

The stub in [tasks.controller.ts:37-39](src/tasks/tasks.controller.ts#L37-L39) will eventually have a `@Post()` decorator, but there is no `@HttpCode(HttpStatus.CREATED)`. What HTTP status code does a `@Post()` handler return by default in NestJS? Is the absence of `@HttpCode(201)` functionally wrong — could a client break because of it? When does it actually matter?

### El status code por defecto en @Post() es 200 OK, y aunque no es "malo" técnicamente, es semánticamente incorrecto respecto a estándares REST. No es funcionalmente "malo" porque un cliente puede seguir procesando la respuesta, pero algunos clientes sofisticados (navegadores avanzados, herramientas de test) pueden fallar si esperan 201 para creaciones exitosas; la mejor práctica es siempre usar @HttpCode(HttpStatus.CREATED) en métodos @Post() para ser RESTful.

---

**Q9 — Service throws, not returns null**

In [products.service.ts:25-30](src/products/products.service.ts#L25-L30), `findOne` throws `NotFoundException` instead of returning `null`. Rewrite the method signature and the controller's `findOne` method as they would look if the service returned `null` instead. Then explain: which version is better for a growing codebase where `findOne` is called from multiple places (e.g., inside `update` and `remove` as well), and why?

### Si el servicio retornara null en lugar de lanzar excepción, debería retornar Product | null y tanto el controller como los métodos update y remove tendrían que verificar if (!product) y lanzar NotFoundException manualmente en cada lugar. Lanzar excepción en el servicio es mejor porque sigue el principio DRY (Don't Repeat Yourself) evitando repetir el check if (!product) en múltiples métodos, asigna la responsabilidad correctamente al servicio de validar su propia lógica, facilita el mantenimiento porque si cambias el comportamiento de 404 lo haces en un único lugar, y hace el código más legible y limpio.

---

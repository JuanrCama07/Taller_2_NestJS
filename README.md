# Integrantes
## Nombre: Juan David Roncancio Camacho
## Código: 202310897
## Correo: j.roncancioc@uniandes.edu.co

## Respuestas a las preguntas de TASKS.md

# NestJS Workshop – Validation Questions

These questions require you to reason about the code, not just recall definitions. Open the files as you work through them.

---

**Q1 — Dead route diagnosis**

### Obtendrías un error 404 Not Found. Aunque el método findAll() existe en el controlador, sin el decorador @Get() NestJS no sabe que debe mapear la ruta GET /tasks a ese método; el decorador es lo que le indica a NestJS qué ruta HTTP debe ejecutar qué método. Para arreglarlo, debes añadir @Get() encima del método.

---

**Q2 — When `transform: true` is not enough**

### No son lo mismo. transform: true convierte los parámetros después de la validación en el controller, mientras que ParseIntPipe convierte antes y puede validar el tipo en un paso. transform: true convierte string "42" a número 42 pero ya en el controller, mientras que ParseIntPipe valida y convierte en el middleware de parámetros y si no es un número válido lanza error 400 antes de entrar al método. Entonces ParseIntPipe es más estricto y seguro porque rechaza valores inválidos en el nivel de parámetros, no en la transformación de tipos.

---

**Q3 — Silent strip vs hard rejection**

### Con solo whitelist: true, el status sería 201 (success), se crearía el usuario normalmente, y "password" se eliminaría silenciosamente sin llegar al servicio, lo cual es un problema de seguridad porque un cliente podría enviar campos sensibles pensando que se procesan pero se ignoran calladamente; en cambio, con forbidNonWhitelisted: true el status sería 400 Bad Request con un mensaje explícito notificando que hay un problema, lo cual es mucho más seguro que ignorar silenciosamente campos no autorizados.

---

**Q4 — Mutation side-effect**

### Sí, cualquier cambio al objeto retornado por findAll() afectaría directamente los datos del servicio porque findAll() retorna una referencia al array mismo, no una copia (por ejemplo, si hicieras const products = productService.findAll(); products[0].price = 0; modificarías directamente en el servicio). Para arreglarlo, deberías retornar una copia del array: return [...this.products]; o incluso una copia más profunda con return this.products.map(p => ({ ...p }));.

---

**Q5 — The optional field trap**

### Con el primer request {"price": -50}, la validación falla y retorna 400 Bad Request porque -50 viola @IsPositive(), pues @IsOptional() solo significa "si está presente, debe cumplir las otras reglas"; con el segundo request {}, la validación pasa retornando 200 OK y actualiza el producto pero no cambia price, porque el campo está ausente y @IsOptional() lo saltea. La regla exacta de @IsOptional() es: "Si el campo está presente, valida contra todos los otros decoradores. Si está ausente, no valida nada."

---

**Q6 — ID reuse after deletion**

### Con el sistema actual de nextId, si deleteas task #1 y creas una nueva, el nuevo task recibe ID #4 (el siguiente nextId) y findOne(1) nunca retornaría nada después de deletar porque ese ID ya no existe, así que no hay conflicto. Sin embargo, si usaras this.tasks.length + 1 como ID, después de crear task #1, #2, #3, deletear #1 (length = 2) y crear uno nuevo, obtendrías ID = 3 de nuevo causando una colisión con el task #3 existente, demostrando que nextId es correcto porque incrementa sin resetear mientras que length + 1 falla al reutilizar IDs.

---

**Q7 — Module forgotten**

### Si olvidas añadir UsersModule a los imports en app.module.ts, el servidor arrancará normalmente sin error, pero POST /users retornará 404 Not Found porque el módulo no está registrado. Este es un error de runtime (en ejecución), no de compilación ni de startup, porque NestJS no se queja al arrancar pues técnicamente todo es válido; solo cuando intentas usar la ruta descubres que no existe.

---

**Q8 — Missing 201**

### El status code por defecto en @Post() es 200 OK, y aunque no es "malo" técnicamente, es semánticamente incorrecto respecto a estándares REST. No es funcionalmente "malo" porque un cliente puede seguir procesando la respuesta, pero algunos clientes sofisticados (navegadores avanzados, herramientas de test) pueden fallar si esperan 201 para creaciones exitosas; la mejor práctica es siempre usar @HttpCode(HttpStatus.CREATED) en métodos @Post() para ser RESTful.

---

**Q9 — Service throws, not returns null**

### Si el servicio retornara null en lugar de lanzar excepción, debería retornar Product | null y tanto el controller como los métodos update y remove tendrían que verificar if (!product) y lanzar NotFoundException manualmente en cada lugar. Lanzar excepción en el servicio es mejor porque sigue el principio DRY (Don't Repeat Yourself) evitando repetir el check if (!product) en múltiples métodos, asigna la responsabilidad correctamente al servicio de validar su propia lógica, facilita el mantenimiento porque si cambias el comportamiento de 404 lo haces en un único lugar, y hace el código más legible y limpio.

---

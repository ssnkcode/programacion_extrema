# Guía de Ejercicios Prácticos: Análisis de Código, Diagnóstico y Diseño de APIs REST con Express & TypeScript

| | |
|---|---|
| **Materia** | Programación Backend (Tecnicatura Universitaria en Desarrollo Full Stack) |
| **Basado en** | Documento Docente - Unidad 2: Diseño e Implementación de APIs |
| **Objetivo** | Evaluar la comprensión del código, la identificación de errores de arquitectura REST, validación de datos, manejo centralizado de errores, TypeScript y diseño de endpoints. |
| **Profesor** | Narciso Pérez |
| **Autor y creador del trabajo** | Cristian Sasinka |

## Instrucciones generales

- Analiza detenidamente los fragmentos de código TypeScript/Express provistos en cada ejercicio.
- Responde cada una de las preguntas asociadas fundamentando con los conceptos técnicos vistos en la materia (Principios REST, DTO, Manejo de Errores, Inyección de Dependencias, HTTP, etc.).
- En caso de identificar errores o fallas de diseño, propone el fragmento de código corregido o la alternativa técnica recomendada.

---

## Ejercicio 1: Diseño de Rutas y Métodos HTTP (Infracción de Principios REST)

Un desarrollador redactó el siguiente conjunto de rutas en Express para gestionar un recurso de tareas:

```ts
app.get("/api/v1/obtenerTodasLasTareas", (req, res) => { /* ... */ });

app.post("/api/v1/crearNuevaTarea", (req, res) => { /* ... */ });

app.post("/api/v1/modificarTarea/:id", (req, res) => { /* ... */ });

app.get("/api/v1/eliminarTarea/:id", (req, res) => { /* ... */ });
```

### Preguntas

**¿Qué principios de diseño de REST están siendo violados en estas definiciones de rutas?**

- **URIs basadas en acciones (verbos):** En una arquitectura REST las URIs deben representar recursos mediante sustantivos (preferiblemente en plural, como `/tareas`) y no acciones o procesos operativos (como `obtenerTodasLasTareas`, `crearNuevaTarea`, etc.).
- **Uso incorrecto de los métodos HTTP semánticos:**
  - Se utiliza `POST` para modificar un recurso (`modificarTarea/:id`), cuando el estándar REST establece el uso de `PUT` (reemplazo total) o `PATCH` (actualización parcial).
  - Se utiliza `GET` para eliminar un recurso (`eliminarTarea/:id`). Las operaciones que alteran el estado o destruyen datos nunca deben ejecutarse mediante `GET`, ya que este método debe ser seguro y carecer de efectos secundarios. Para eliminar recursos debe usarse estrictamente `DELETE`.

**Reescribe la tabla de rutas aplicando las buenas prácticas de diseño REST orientadas a recursos:**

| Método HTTP | Ruta | Descripción del Recurso / Acción |
|---|---|---|
| `GET` | `/api/v1/tareas` | Obtener la colección de todas las tareas. |
| `POST` | `/api/v1/tareas` | Crear una nueva tarea. |
| `PUT` / `PATCH` | `/api/v1/tareas/:id` | Modificar una tarea específica por su identificador. |
| `DELETE` | `/api/v1/tareas/:id` | Eliminar una tarea específica por su identificador. |

---

## Ejercicio 2: Asunción Falsa de Validación con Tipos de TypeScript

Observa la siguiente implementación de un controlador para la creación de tareas:

```ts
import { Request, Response } from 'express';
import { CrearTareaDto } from './tarea.dto';

export function crearTareaController(req: Request, res: Response) {
    const datos = req.body as CrearTareaDto;

    // El desarrollador asume que 'datos' ya tiene la estructura válida de CrearTareaDto
    const nuevaTarea = tareasService.crear(datos);
    return res.status(201).json({ data: nuevaTarea });
}
```

### Preguntas

**¿Por qué la aserción de tipo `as CrearTareaDto` NO garantiza la validez de los datos recibidos en runtime?**

Las aserciones de tipo en TypeScript (como `as CrearTareaDto`) son únicamente directivas de análisis estático que desaparecen por completo al compilarse a JavaScript. TypeScript no ejecuta ninguna validación, coerción ni comprobación de tipos en tiempo de ejecución; la aserción solo le indica al compilador que ignore las advertencias y confíe en que el objeto tiene la estructura declarada.

**¿Qué vulnerabilidades o fallos en ejecución pueden ocurrir si el cliente envía un JSON malformado o incompleto (ej. `{}` o `{"titulo": 123}`)?**

- **Excepciones por métodos inválidos o propiedades ausentes:** Si la lógica de negocio asume que un campo obligatorio existe y ejecuta operaciones sobre él (como `.trim()`, `.toLowerCase()` o acceso a propiedades anidadas), al recibir `{}` se producirá un `TypeError: Cannot read properties of undefined`. Un valor numérico como `123` en lugar de una cadena fallará al intentar ejecutar métodos de string con `TypeError: datos.titulo.trim is not a function`, lo que puede desestabilizar el servidor o retornar errores 500 no controlados.
- **Violaciones de restricciones en la base de datos:** Al no validar la entrada, los campos obligatorios ausentes o mal tipados llegan directo a la capa de persistencia, provocando que la base rechace la transacción por violar restricciones de nulabilidad (`NOT NULL`) o de tipos, o peor, que almacene datos corruptos e incompletos.
- **Vulnerabilidades de asignación masiva (Mass Assignment):** Al no filtrar los datos entrantes contra un esquema estricto, un cliente malintencionado puede inyectar propiedades adicionales no autorizadas (flags de permisos o identificadores de usuario) que podrían procesarse y guardarse si el modelo de datos las acepta de forma abierta.

---

## Ejercicio 3: Implementación de Función de Validación Manual DTO

El siguiente código valida los datos de entrada para la creación de una tarea:

```ts
function validarCrearTarea(datos: unknown): CrearTareaDto {
    if (typeof datos !== "object" || datos === null) {
        throw new AppError(400, "INVALID_BODY", "El cuerpo debe ser un objeto JSON");
    }
    const objeto = datos as Record<string, unknown>;

    if (typeof objeto.titulo !== "string" || objeto.titulo.trim().length < 3) {
        throw new AppError(422, "INVALID_TITLE", "El título debe tener al menos tres caracteres");
    }

    const prioridades = ["baja", "media", "alta"];
    if (typeof objeto.prioridad !== "string" || !prioridades.includes(objeto.prioridad)) {
        throw new AppError(422, "INVALID_PRIORITY", "La prioridad no es válida");
    }

    return {
        titulo: objeto.titulo.trim(),
        prioridad: objeto.prioridad as CrearTareaDto["prioridad"]
    };
}
```

### Preguntas

**¿Por qué se utiliza el código de estado HTTP 422 Unprocessable Content para el título/prioridad en lugar de un 400 Bad Request?**

El `400 Bad Request` se reserva para errores de sintaxis o estructura general de la petición (como un JSON malformado que el servidor ni siquiera puede parsear). En cambio, el `422 Unprocessable Content` se emplea cuando la petición es sintácticamente correcta y legible, pero los datos semánticos fallan las reglas de validación de negocio (longitudes mínimas de cadenas o valores fuera de los permitidos por un catálogo).

**¿Qué ocurriría si se envía un JSON como `{"titulo": " AB ", "prioridad": "alta"}`? ¿Pasa o falla la validación?**

La validación falla.

- El valor de entrada del título es la cadena con espacios `" AB "`.
- El código ejecuta `objeto.titulo.trim()`, lo que limpia los espacios y deja `"AB"`.
- Se evalúa la longitud: `"AB".length` es igual a `2`.
- Como `2 < 3` es verdadero, se activa el bloque `if` y se lanza el error `AppError` con código `422` y el mensaje "El título debe tener al menos tres caracteres", interrumpiendo la ejecución antes de validar la prioridad.

---

## Ejercicio 4: Semántica de Actualización: PUT vs PATCH

Un servicio expone dos endpoints para modificar recursos existentes:

```ts
// Endpoint A
app.put("/api/v1/tareas/:id", (req, res) => {
    const { titulo, prioridad, completada } = req.body;
    const tareaActualizada = service.reemplazar(Number(req.params.id), { titulo, prioridad, completada });
    res.status(200).json({ data: tareaActualizada });
});

// Endpoint B
app.patch("/api/v1/tareas/:id", (req, res) => {
    const cambios = req.body;
    const tareaModificada = service.actualizarParcial(Number(req.params.id), cambios);
    res.status(200).json({ data: tareaModificada });
});
```

### Preguntas

**Si el cliente envía a `PUT /api/v1/tareas/5` únicamente el cuerpo `{"completada": true}`, ¿cuál es el comportamiento esperado según la especificación HTTP REST y qué problema ocurre con la entidad?**

Según la especificación HTTP REST, `PUT` debe realizar un reemplazo **completo** del recurso. Al enviar únicamente `{"completada": true}`, las propiedades omitidas (`titulo` y `prioridad`) se procesan como `undefined`. Como consecuencia, la entidad almacenada pierde sus valores originales y queda con campos vacíos o nulos, corrompiendo la integridad de los datos. Para actualizaciones parciales donde solo se envían los campos modificados debe utilizarse exclusivamente `PATCH`.

**Explica el concepto de idempotencia en el contexto de las operaciones PUT y POST.**

La idempotencia es la propiedad por la cual realizar una misma petición varias veces produce exactamente el mismo resultado y estado en el servidor que realizarla una sola vez (aunque la respuesta HTTP pueda cambiar).

- **`PUT` es idempotente:** Enviar una petición `PUT` diez veces seguidas con los mismos datos deja el servidor en el mismo estado que tras la primera ejecución, ya que cada llamada sobrescribe el recurso con la misma información exacta.
- **`POST` NO es idempotente:** Cada invocación introduce una acción que genera un nuevo recurso (como registrar una tarea). Ejecutarla diez veces provocará la creación de diez recursos distintos con identificadores diferentes.

---

## Ejercicio 5: Conversión e Interpretación de Parámetros de Ruta

Analiza la extracción del parámetro ID en el siguiente controlador:

```ts
app.get("/api/v1/tareas/:id", (req, res) => {
    const id = Number(req.params.id);

    if (!Number.isInteger(id) || id <= 0) {
        throw new AppError(400, "INVALID_ID", "El identificador no es válido");
    }

    const tarea = service.obtenerPorId(id);
    res.status(200).json({ data: tarea });
});
```

### Preguntas

**¿Qué valor toma la variable `id` y qué respuesta devuelve la API si el cliente realiza una petición a `GET /api/v1/tareas/abc`?**

La variable `id` toma el valor `NaN` (Not a Number) al intentar convertir la cadena `"abc"` mediante `Number()`. Al evaluar la condición, `Number.isInteger(NaN)` retorna `false`, lo que activa inmediatamente el bloque `if` y lanza la excepción. La API devuelve una respuesta HTTP `400 Bad Request` con el código de error `"INVALID_ID"` y el mensaje "El identificador no es válido".

**¿Por qué se valida que el ID sea un entero positivo mayor a cero antes de invocar la capa de servicio?**

- **Principio de fallo rápido (Fail Fast):** Rechaza peticiones inválidas en la frontera de la aplicación (el controlador) antes de consumir recursos computacionales o de red procesando lógica innecesaria.
- **Protección de la base de datos:** Evita que valores como `NaN`, negativos o decimales lleguen a la capa de persistencia, previniendo errores de sintaxis en consultas SQL, ORMs o búsquedas inútiles que podrían saturar o desestabilizar el sistema gestor.
- **Separación de responsabilidades:** La capa de servicio y los repositorios deben asumir que la entrada ya está limpia y validada, y enfocarse solo en ejecutar la lógica de negocio y las operaciones de datos de manera predecible.

---

## Ejercicio 6: Middleware Centralizado de Errores

Considera la siguiente implementación del middleware de manejo global de errores:

```ts
export const errorHandler: ErrorRequestHandler = (error, _request, response, _next) => {
    if (error instanceof AppError) {
        response.status(error.status).json({
            error: { code: error.code, message: error.message, details: error.details }
        });
        return;
    }

    console.error(error);
    response.status(500).json({
        error: { code: "INTERNAL_ERROR", message: "Ocurrió un error interno" }
    });
};
```

### Preguntas

**¿Por qué es fundamental filtrar por `error instanceof AppError` antes de retornar la respuesta?**

Es fundamental para distinguir entre errores **operacionales controlados** (fallos de validación, recursos no encontrados, reglas de negocio) y errores de programación o fallos imprevistos del sistema. Los primeros contienen códigos HTTP personalizados, códigos de error de negocio y mensajes seguros que el cliente necesita conocer; los segundos no forman parte de este flujo controlado y deben derivarse al manejo genérico de fallos internos.

**¿Por qué NO se deben retornar el stack trace ni detalles técnicos de errores no controlados (como un fallo de conexión a BDD) en la respuesta JSON al cliente?**

- **Seguridad e información sensible (Information Disclosure):** Exponer la traza de pila revela la estructura interna del servidor, rutas absolutas de archivos, versiones de dependencias y detalles del motor de base de datos. Esta información puede usarse para ataques dirigidos o para identificar vectores de vulnerabilidad.
- **Experiencia de usuario y robustez:** Los mensajes técnicos o errores crudos de bases de datos son incomprensibles para el cliente y denotan falta de madurez y seguridad en la API, que debe mantener un contrato de respuesta predecible y uniforme ante fallos.

---

## Ejercicio 7: Acoplamiento Directo vs. Inyección de Dependencias

Un estudiante escribió la siguiente clase para su controlador:

```ts
export class TareasController {
    private service: TareasService;

    constructor() {
        const repository = new TareasRepository();
        this.service = new TareasService(repository);
    }

    public obtenerTodas = (_req: Request, res: Response) => {
        const data = this.service.obtenerTodas();
        res.status(200).json({ data });
    }
}
```

### Preguntas

**¿Qué problema de acoplamiento presenta la instanciación interna con `new TareasRepository()` y `new TareasService()` dentro del constructor?**

La instanciación interna genera un acoplamiento fuerte entre el controlador y las clases concretas (`TareasService` y `TareasRepository`). Esto hace que el controlador sea rígido: no se puede reutilizar con otra implementación del servicio, no respeta el Principio de Inversión de Dependencias (la "D" de SOLID) e impide aislar el controlador para pruebas unitarias, ya que al instanciarlo se ejecuta de forma directa e indeseada la lógica de negocio real y las conexiones a la base de datos.

**Refactoriza la clase `TareasController` aplicando Inyección de Dependencias por constructor y explica cómo facilita las pruebas unitarias con mocks.**

```ts
export class TareasController {
    constructor(private readonly service: TareasService) {}

    public obtenerTodas = (_req: Request, res: Response) => {
        const data = this.service.obtenerTodas();
        res.status(200).json({ data });
    }
}
```

**¿Cómo facilita esto las pruebas unitarias con mocks?**

Al recibir la dependencia desde el exterior por constructor, el controlador deja de preocuparse por cómo se crea o de dónde viene el servicio. En las pruebas unitarias esto permite inyectar un mock o stub (un objeto simulado que imita el comportamiento de `TareasService` sin conectar a bases de datos reales). Así es posible:

- Aislar por completo la lógica del controlador de la capa de servicios y persistencia.
- Simular escenarios específicos de forma controlada (por ejemplo, forzar que el servicio devuelva una lista vacía o lance un error controlado) sin afectar el entorno.
- Ejecutar pruebas rápidas, deterministas y sin dependencias externas de infraestructura.

---

## Ejercicio 8: Paginación y Cálculo de Offsets

Analiza el siguiente fragmento del servicio para consultar una colección paginada:

```ts
const page = Math.max(Number(request.query.page) || 1, 1);
const limit = Math.min(
    Math.max(Number(request.query.limit) || 10, 1),
    100
);

const start = (page - 1) * limit;
const end = start + limit;

const tareasPaginadas = todasLasTareas.slice(start, end);
```

### Preguntas

**Si el cliente realiza una petición con los parámetros `?page=3&limit=5`, ¿cuáles serán los valores de `start` y `end` en el método `slice`?**

Para una petición con `?page=3&limit=5`, los valores calculados son `start = 10` y `end = 15`, ya que `(3 - 1) * 5 = 10` y `10 + 5 = 15`.

**¿Qué efecto tienen `Math.max` y `Math.min` si el cliente envía valores negativos o muy elevados como `?page=-2&limit=5000`?**

Estas funciones actúan como un mecanismo de saneamiento y protección de límites de paginación:

- **`Math.max` (protección de mínimos):** Evita páginas o límites negativos o iguales a cero. Si el cliente envía `page=-2`, la expresión pasa por `Math.max(-2, 1)`, que lo corrige y lo fuerza a `1`, evitando desplazamientos negativos inválidos en el arreglo.
- **`Math.min` (protección de máximos/techos):** Evita ataques de denegación de servicio o problemas de rendimiento por consumo excesivo de memoria. Aunque el cliente envíe `limit=5000`, `Math.max(5000, 1)` da `5000`, pero `Math.min(5000, 100)` recorta y restringe el valor máximo a 100 elementos por página.

---

## Ejercicio 9: Violación de la Arquitectura por Capas

Observa la siguiente ruta implementada por un desarrollador:

```ts
router.get("/:id", async (req, res) => {
    const id = Number(req.params.id);
    // Acceso directo a la fuente de datos global/memoria
    const tarea = baseDeDatosMemoria.find(t => t.id === id);

    if (!tarea) {
        return res.status(404).json({ error: "No encontrada" });
    }

    // Regla de negocio ejecutada directamente en la ruta
    tarea.vistas = (tarea.vistas || 0) + 1;

    res.status(200).json({ data: tarea });
});
```

### Preguntas

**Identifica las responsabilidades mezcladas en este bloque y menciona qué capas de la arquitectura modular (Router, Controller, Service, Repository) se están omitiendo o salteando.**

El código mezcla en un solo bloque la gestión de la petición HTTP, el acceso directo a la fuente de datos, la lógica de negocio o modificación de estado (`tarea.vistas = ...`) y el formato de la respuesta. Las capas omitidas por completo son el **Controller**, el **Service** y el **Repository**, colapsando toda la arquitectura modular en una única función anónima dentro del router.

**Indica cuál debería ser la única responsabilidad del Router y del Controller en este flujo.**

- **Responsabilidad del Router:** Su única función es definir el mapeo de URL y método HTTP (`GET /:id`), vinculándolo con el método correspondiente del controlador y aplicando middlewares específicos de ruta (autenticación o validación de formato). No debe contener lógica de negocio, consultas directas ni manipulación de datos.
- **Responsabilidad del Controller:** Actuar como adaptador entre la capa HTTP y la lógica de la aplicación. Debe extraer los datos de la petición (`req.params`, `req.body`), invocar la capa de servicio con parámetros limpios y retornar la respuesta HTTP (o delegar los errores al middleware centralizado), sin ejecutar consultas a bases de datos ni reglas de negocio de forma directa.

---

## Ejercicio 10: Especificación OpenAPI (YAML) y Contratos

Examina el siguiente fragmento de documentación OpenAPI:

```yaml
paths:
  /tareas:
    post:
      summary: Crear una tarea
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              required: [titulo, prioridad]
              properties:
                titulo:
                  type: string
                  minLength: 3
                prioridad:
                  type: string
                  enum: [baja, media, alta]
      responses:
        "201":
          description: Tarea creada
        "422":
          description: Datos inválidos
```

### Preguntas

**Según este contrato OpenAPI, ¿qué respuesta debe retornar la API si un cliente envía `{"titulo": "A", "prioridad": "urgente"}`?**

La API debe retornar un código de estado HTTP **422 Unprocessable Content**. La causa es que los datos incumplen el contrato del esquema en dos puntos: el campo `titulo` tiene una longitud de 1 carácter, violando `minLength: 3`, y el campo `prioridad` con valor `"urgente"` no se encuentra dentro del `enum [baja, media, alta]`.

**¿Qué diferencia fundamental existe entre OpenAPI y Swagger?**

- **OpenAPI** es la especificación o estándar abierto: un formato de descripción de API independiente del lenguaje, basado en JSON/YAML, para documentar servicios web RESTful.
- **Swagger** es el conjunto de herramientas (Swagger UI, Swagger Editor, Swagger Codegen) desarrollado inicialmente por SmartBear para implementar, visualizar y trabajar con la especificación OpenAPI.

---

## Ejercicio 11: Modificación Incompatible y Versionado de APIs

Un equipo tiene desplegada la versión 1 de su API con la siguiente estructura de respuesta en `GET /api/v1/tareas/15`:

```json
{
  "data": {
    "id": 15,
    "titulo": "Preparar TP",
    "prioridad": "alta"
  }
}
```

Para la nueva entrega, deciden renombrar la propiedad `prioridad` a `nivelPrioridad` y mover el `id` dentro de un subobjeto `metadata`.

### Preguntas

**¿Por qué este cambio constituye una "modificación breaking" (incompatible) para las aplicaciones cliente (frontend/mobile) existentes?**

Altera el contrato de la API existente. Al renombrar `prioridad` a `nivelPrioridad` y mover `id` dentro de un subobjeto `metadata`, cualquier código del cliente que acceda directamente a `data.prioridad` o `data.id` obtendrá `undefined`, provocando fallos en tiempo de ejecución, errores de renderizado o ruptura de la lógica de negocio en las aplicaciones ya desplegadas.

**¿Cómo se debe estructurar la URL del nuevo endpoint según las buenas prácticas de versionado de APIs?**

Incrementando el número de versión en el segmento de la ruta: `/api/v2/tareas/15`, para garantizar la coexistencia controlada con la versión anterior y no afectar a los clientes que aún consumen la versión 1.

---

## Ejercicio 12: Implementación de Encabezados y Estructura de Respuesta DTO

Considera la creación exitosa de un recurso en Express:

```ts
router.post("/", (request, response) => {
    const dto = validarCrearTarea(request.body);
    const tarea = service.crear(dto);
    response.location(`/api/v1/tareas/${tarea.id}`).status(201).json({ data: tarea });
});
```

### Preguntas

**¿Cuál es el propósito del encabezado HTTP `Location` asignado en la respuesta a una petición POST exitosa?**

Indica la URI exacta donde el cliente puede encontrar y acceder al recurso recién creado. Se usa junto con el código `201 Created` para que las aplicaciones cliente identifiquen de forma inmediata la ruta del nuevo recurso, facilitando operaciones posteriores como un `GET` para consultar los detalles completos de la tarea registrada.

**¿Por qué se recomienda envolver la entidad devuelta dentro de un objeto con la propiedad `data` (ej. `{ data: tarea }`) en lugar de retornarla en la raíz?**

- **Extensibilidad del contrato:** Permite agregar metadatos en la raíz en el futuro (paginación, enlaces HATEOAS, estados, contadores o advertencias) sin alterar la estructura interna de la entidad ni romper el contrato establecido con los clientes.
- **Consistencia y predictibilidad:** Estandariza la estructura de respuestas de toda la API, asegurando que los consumidores sepan que la carga útil principal siempre está bajo una propiedad homogénea, sin importar el recurso consultado.

---

## Ejercicio 13: Repositorio en Memoria e Inmutabilidad

Analiza el método `obtenerTodas` del siguiente Repositorio en memoria:

```ts
export class TareasRepository {
    private readonly tareas: Tarea[] = [];

    // Opción A
    obtenerTodasA(): Tarea[] {
        return this.tareas;
    }

    // Opción B
    obtenerTodasB(): Tarea[] {
        return [...this.tareas];
    }
}
```

### Preguntas

**¿Qué riesgo de seguridad/encapsulamiento tiene la Opción A si un servicio o controlador modifica el arreglo devuelto (ej. haciendo `repo.obtenerTodasA().pop()`)?**

La Opción A rompe el principio de encapsulamiento al retornar una referencia directa al arreglo privado interno (`this.tareas`). Como en JavaScript/TypeScript los arreglos se pasan por referencia, cualquier componente externo (servicio o controlador) puede modificar el estado interno del repositorio sin control ni validación (vaciar la lista con `.pop()`, agregar con `.push()` o reordenarla), provocando efectos secundarios impredecibles y debilitando la integridad de los datos.

**¿Cómo soluciona la Opción B este problema mediante el operador de propagación (spread operator)?**

La Opción B retorna una copia superficial (shallow copy) del arreglo mediante `[...this.tareas]` en lugar de la referencia original. Al crear un nuevo objeto de arreglo con los mismos elementos, cualquier alteración externa sobre esa copia (eliminar o agregar tareas) afecta únicamente a la respuesta entregada, manteniendo el arreglo interno del repositorio protegido, aislado y seguro.

---

## Ejercicio 14: Pruebas Manuales con cURL

Dado el siguiente comando cURL para probar la API:

```bash
curl -i \
  -X POST \
  -H "Content-Type: application/json" \
  -d '{"titulo":"Estudiar API","prioridad":"alta"}' \
  http://localhost:3000/api/v1/tareas
```

### Preguntas

**¿Qué función cumple la opción `-i` en el comando curl?**

La opción `-i` (o `--include`) le indica a cURL que incluya los encabezados de la respuesta HTTP en la salida estándar, mostrándolos justo antes del cuerpo. Es muy útil para verificar encabezados clave como el código de estado, `Content-Type`, `Location` o cabeceras de caché directamente desde la terminal.

**¿Qué ocurre si se omite el encabezado `-H "Content-Type: application/json"` cuando el backend utiliza el middleware `express.json()`? ¿Qué valor tendrá `req.body`?**

Si se omite ese encabezado, el middleware `express.json()` no identifica el formato del cuerpo y lo ignora por completo. Como consecuencia, `req.body` tendrá el valor `undefined`, lo que hará que el controlador falle al intentar acceder o desestructurar propiedades como `titulo` o `prioridad`.

---

## Ejercicio 15: Generación Autónoma de Identificadores en la Capa de Persistencia

Observa los tipos TypeScript definidos para la aplicación:

```ts
export interface CrearTareaDto {
    titulo: string;
    prioridad: "baja" | "media" | "alta";
}

export interface Tarea {
    id: number;
    titulo: string;
    prioridad: "baja" | "media" | "alta";
    completada: boolean;
    fechaCreacion: string;
}
```

### Preguntas

**¿Por qué el campo `id`, `completada` y `fechaCreacion` están ausentes en la interfaz `CrearTareaDto` pero presentes en la interfaz `Tarea`?**

Están ausentes en `CrearTareaDto` porque este tipo define estrictamente el contrato de entrada (payload) que el cliente puede y debe enviar para solicitar la creación del recurso. En cambio, `Tarea` representa el modelo completo o entidad de dominio persistida, que incluye metadatos y estados internos gestionados de forma autónoma por el servidor (identificador autoincremental, estado inicial por defecto de `completada` y la fecha exacta del sistema).

**¿En qué capa del sistema (Controller, Service o Repository) se deben generar el `id` y la `fechaCreacion` al registrar una nueva tarea, y por qué no deben ser provistos por el cliente HTTP?**

El `id` y la `fechaCreacion` deben generarse en la capa de **Repositorio** (o coordinarse por el **Servicio** utilizando el estado de la persistencia), y nunca deben ser provistos por el cliente, por razones de seguridad e integridad:

- **Prevención de manipulaciones y colisiones:** Si el cliente enviara el `id`, podría provocar colisiones de claves primarias, sobrescribir registros existentes (ataques IDOR o reemplazo no autorizado) o inyectar identificadores arbitrarios.
- **Fiabilidad temporal:** La fecha de creación debe reflejar el tiempo real del servidor (`new Date().toISOString()`). Permitir que el cliente la envíe habilitaría falsificar marcas de tiempo o generar inconsistencias temporales.
- **Encapsulamiento del estado inicial:** El servidor debe garantizar que toda nueva tarea nazca con reglas de negocio predecibles (por ejemplo, `completada: false`), evitando que un cliente malintencionado altere estados internos desde la petición inicial.
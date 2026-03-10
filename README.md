# Informe de pruebas utilizando Continue Dev

La idea de este documento es realizar pruebas con **Continue Dev** utilizado dentro de **Visual Studio Code** midiendo tanto la calidad del código, como el costo por tarea y pruebas varias, entre ellas el uso del plugin como "agente" y no solo autocompletando código.

---

## 1. Configuración del plugin

En Continue.dev, lo más importante está en el archivo `config.json` donde se seleccionan los modelos a utilizar y como realizar la comunicación con servidores remotos que alojan estos LLM.

- Ejemplo:
```
name: Local Config
version: 1.0.0
schema: v1
models:
  - name: Llama 3.3 70B
    provider: openai
    model: llama-3.3-70b-versatile
    apiBase: https://api.groq.com/openai/v1
    apiKey: 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
    roles:
      - chat
      - edit
      - apply

  - name: All MiniLM
    provider: ollama
    model: all-minilm:latest
    apiBase: http://localhost:11434
    roles:
      - embed

  - name: Codestral
    provider: mistral
    model: codestral:latest
    apiBase: https://codestral.mistral.ai/v1
    apiKey: Amj6yDCF9CwwN35VRZltcAd1PSb4jOY9
    roles:
      - autocomplete
```

Aquí configuramos:
- Múltiples modelos con diferentes roles dentro del plugin.
- La fuente del modelo (local con Ollama o remoto vía API).
- El endpoint (apiBase) al cual Continue enviará las requests.
- El tipo de proveedor (openai-compatible, ollama, mistral, etc.).
- Las credenciales (apiKey) necesarias para autenticación en servicios cloud.
- Los roles funcionales que tendrá cada modelo:
  - chat → conversación general y análisis.
  - edit → modificación de código existente.
  - apply → aplicación directa de cambios en archivos.
  - autocomplete → autocompletado inline mientras se escribe.
  - embed → generación de embeddings para búsqueda semántica en el repositorio.

Esta arquitectura permite separar responsabilidades. Por ejemplo:
- Un modelo grande en la nube para chat/edit/apply.
- Un modelo local pequeño para autocomplete.
- Un modelo optimizado para embeddings para búsquedas en el código.

Esta separación es clave en entornos empresariales para optimizar costo, latencia y rendimiento.

---

## Descripción de los diversos modelos

### Modo "Plan"

El modo **Plan** no es simplemente “pensar más”, internamente lo que hace es:
- El modelo analiza tu solicitud.
- Genera una secuencia estructurada de pasos.
- No ejecuta todavía (si está bien configurado).
- Te muestra el plan antes de modificar archivos.
- Es una capa de control antes de la acción.

Se recomienda utilizar este modo para:
- Refactorizaciones grandes.
- Cambios arquitectónicos.
- Tareas multi-archivo.
- Migraciones de versión (ej: actualizar framework).

Permite validar la estrategia antes de permitir que el modelo modifique el código.

### Modo "Chat"

El modo **Chat** es el modo conversacional clásico, permite:
- Permite hacer preguntas sobre el código.
- Explicar archivos o funciones específicas.
- Solicitar generación de código sin aplicarlo automáticamente.
- Pedir revisiones técnicas o análisis de seguridad.

En este modo:
- El modelo no necesariamente modifica archivos.
- Es ideal para exploración, debugging conceptual o aprendizaje.
- Funciona como “pair programming conversacional”.

Se recomienda para:
- Explicaciones de código legacy.
- Análisis de stacktraces.
- Diseño previo antes de implementar.
- Validación de ideas arquitectónicas.

Es el modo más seguro para comenzar a interactuar con un nuevo modelo antes de habilitar capacidades de edición automática.

### Modo "Agente"

El modo **Agente** es el más avanzado y cercano a automatización real, permite:
- Puede planificar y ejecutar pasos.
- Puede leer múltiples archivos del proyecto.
- Puede modificar archivos automáticamente.
- Puede utilizar herramientas habilitadas (filesystem, terminal, etc.).
- Puede encadenar acciones en múltiples pasos.

En este modo el modelo:
- Analiza la tarea.
- Genera un plan (implícito o explícito).
- Ejecuta modificaciones.
- Puede volver a analizar resultados y continuar iterando.

Es ideal para:
- Crear nuevos módulos completos.
- Agregar funcionalidades transversales.
- Generar tests automáticamente.
- Aplicar cambios repetitivos en múltiples archivos.

Requiere:
- Modelos con buena capacidad de razonamiento multi-step.
- Buen manejo de contexto largo.
- Baja tasa de alucinación.

---

#### Ejecución de pruebas

### Primera prueba

Con una estructura de modelos 1) **Llama 3.3 70B** para chat, 2) **Codestral** para codear y 3) **All MiniLM** para el manejo de embedings y tomando como ejemplo el proyecto **Novatium**, se adjuntó documentación de los estándares de integración para BE y FE:

**Paso 1 - Ejecución "Plan"**

Enviamos como prompt:

```
necesito que apliquemos los estandares de calidad para los modulos "core" y "web", basate en este documento para responder @INV001 - Estándares para la integración BE-FE.md
```

A lo que el modelo respondió con un análisis del requerimiento finalizando con un listado de tareas a seguir:

```
Implementación en los Módulos "core" y "web"
Para implementar estos estándares en los módulos "core" y "web", sigue estos pasos:

Revisa y ajusta: Revisa el código existente en ambos módulos y ajusta según los estándares mencionados. Esto incluye la estructura de respuestas, uso de métodos HTTP, seguridad, documentación, y logging.
Aplica consistencia: Asegúrate de que tanto "core" como "web" sigan los mismos estándares. La consistencia es clave para la mantenibilidad y escalabilidad.
Pruebas: Desarrolla pruebas unitarias y de integración para asegurarte de que los cambios implementados funcionen como se espera y no rompan funcionalidades existentes.
Monitoreo y ajuste: Una vez implementados los cambios, monitorea cómo funcionan en un entorno de producción y ajusta según sea necesario.
Recuerda, la implementación de estándares de calidad es un proceso continuo. Mantén una mentalidad de mejora continua y ajusta estos estándares según las necesidades específicas de tu proyecto y equipo.

Si deseas hacer cambios en los archivos existentes, te recomiendo cambiar a modo Agente para tener acceso a herramientas de escritura. ¿Quieres hacerlo?
```

**Comentario:** da una respuesta demasiado amplia, para nada específica, a pesar de eso continuamos con el modo agente.

**Paso 2 - Ejecución "Agente"**

A pesar de no haber específicado nada en cuanto al uso de qué tecnologías se utilizarían para back o front, comenzí a implementar la solución en `typescript` generando un primer intento de estructura de respuesta.

**Comentario:** Este modo "Agente" consume demasiados tokens, por lo visto en el dashboard de **Groq** que nos brinda el modelo de Llama genera más tokens de inputs que output. Supongo que esto se da así por la pregunta y re pregunta que sucede internamente. Consumiendo hasta el momento _~12 tokens_ y _~0,06 USD_ en un chat de 2 mensajes de usuario. Además, todos los archivos son generados por fuera del repositorio, para que edite archivos específicos hay que seleccionar y ejecutar el modo agente, en vez de darle vía libre.

### Segunda prueba

Con una estructura de modelos 1) **Mistral Medium** para chat, 2) **Codestral** para codear y 3) **All MiniLM** para el manejo de embedings y tomando como ejemplo el proyecto **Novatium**, se adjuntó documentación de los estándares de integración para BE y FE:

**Paso 1 - Ejecución "Plan"**

Enviamos como prompt:

```
teniendo en cuenta los siguientes estandares @INV001 - Estándares para la integración BE-FE.md quisiera modificar el módulo "web" específicamente todos los endpoints relacionados a empleados
```

Respondió de manera consisa con un listado de tareas a realizar. Luego de algunos mensajes siguiente me permite "sumarizar" el requerimiento:

```
Comprehensive Summary of Conversation
1. Conversation Overview
The discussion centered on modifying the web module of a Java-based application to align with the INV001 standards for BE-FE integration, specifically focusing on employee-related endpoints in the 
EmpleadoController.java
 file. The conversation initially addressed the coexistence of REST endpoints and MVC (MAV) endpoints in the same controller, requiring standardization. The goal was to ensure compliance with RESTful principles, consistent response structures, and proper error handling.

2. Active Development
The primary focus was on:

Standardizing REST endpoints (e.g., /employees) to follow INV001 guidelines.
Implementing ApiResponse<T> for consistent API responses.
Ensuring proper HTTP methods (GET, POST, PUT, DELETE, PATCH) and status codes.
Handling errors with structured responses (status, message, error, traceId).
Documenting changes to reflect new standards.
3. Technical Stack
Language/Framework: Java, Spring Boot (REST & MVC)
Architecture: RESTful API, MVC (Model-View-Controller)
Response Structure: ApiResponse<T> (with ApiError, PaginationInfo)
Error Handling: Custom exception handlers for structured error responses
Documentation: Swagger/OpenAPI (implied for endpoint documentation)
4. File Operations
Modified File: 
EmpleadoController.java

Purpose: Contains both REST and MAV endpoints for employee management.
Key Changes:
Standardization of REST endpoints (e.g., /employees).
Implementation of ApiResponse<T> for all REST responses.
Error handling with structured responses.
Possible refactoring to separate REST and MAV logic (if needed).
Potential New Files:

ApiResponse.java (if not already present) – Defines the standardized response structure.
ApiError.java – Defines error response structure.
PaginationInfo.java – Handles pagination metadata (if applicable).
5. Solutions & Troubleshooting
Problem: Coexistence of REST and MAV endpoints in the same controller.
Solution: Ensure REST endpoints follow INV001 standards while maintaining MAV endpoints (if necessary).
Problem: Inconsistent response structures.
Solution: Enforce ApiResponse<T> for all REST endpoints.
Problem: Lack of proper error handling.
Solution: Implement structured error responses with traceId and descriptive keys.
6. Outstanding Work
Pending Tasks:

Review and refactor 
EmpleadoController.java
 to align with INV001 standards.
Implement ApiResponse<T> (if not already present).
Ensure proper HTTP methods and status codes for all endpoints.
Add structured error handling with traceId and descriptive keys.
Document changes in Swagger/OpenAPI (if applicable).
Write unit/integration tests to verify compliance.
Next Steps:

Provide the current code of 
EmpleadoController.java
 for review.
Confirm whether ApiResponse<T> and related classes (ApiError, PaginationInfo) exist or need to be created.
Finalize the refactoring approach (e.g., separating REST and MAV logic if necessary).
This summary ensures continuity and provides a clear roadmap for completing the task.
```

**Comentario:** Responde en inglés, habría que ver si hay manera de enviar un contexto que especifique el lenguaje de respuesta.

**Paso 2 - Ejecución "Agente"**

La respuesta fue bastante mejor, el requerimiento era algo más específico, y cumplió con lo esperado:

```
@RestController
@RequestMapping("/api/employees")
public class EmpleadoController {

    @Autowired
    private EmpleadoService empleadoService;

    @GetMapping
    public ResponseEntity<ApiResponse<List<Empleado>>> getAllEmployees() {
        List<Empleado> empleados = empleadoService.getAllEmpleados();
        ApiResponse<List<Empleado>> response = new ApiResponse<>(true, "Employees retrieved successfully", empleados);
        return ResponseEntity.ok(response);
    }

    @GetMapping("/{id}")
    public ResponseEntity<ApiResponse<Empleado>> getEmployeeById(@PathVariable Long id) {
        Empleado empleado = empleadoService.getEmpleadoById(id);
        if (empleado != null) {
            ApiResponse<Empleado> response = new ApiResponse<>(true, "Employee retrieved successfully", empleado);
            return ResponseEntity.ok(response);
        } else {
            ApiResponse<Empleado> response = new ApiResponse<>(false, "Employee not found", null);
            return ResponseEntity.notFound().build();
        }
    }

    @PostMapping
    public ResponseEntity<ApiResponse<Empleado>> createEmployee(@RequestBody Empleado empleado) {
        Empleado newEmpleado = empleadoService.createEmpleado(empleado);
        ApiResponse<Empleado> response = new ApiResponse<>(true, "Employee created successfully", newEmpleado);
        return ResponseEntity.ok(response);
    }

    @PutMapping("/{id}")
    public ResponseEntity<ApiResponse<Empleado>> updateEmployee(@PathVariable Long id, @RequestBody Empleado empleado) {
        Empleado updatedEmpleado = empleadoService.updateEmpleado(id, empleado);
        if (updatedEmpleado != null) {
            ApiResponse<Empleado> response = new ApiResponse<>(true, "Employee updated successfully", updatedEmpleado);
            return ResponseEntity.ok(response);
        } else {
            ApiResponse<Empleado> response = new ApiResponse<>(false, "Employee not found", null);
            return ResponseEntity.notFound().build();
        }
    }

    @DeleteMapping("/{id}")
    public ResponseEntity<ApiResponse<Void>> deleteEmployee(@PathVariable Long id) {
        boolean deleted = empleadoService.deleteEmpleado(id);
        if (deleted) {
            ApiResponse<Void> response = new ApiResponse<>(true, "Employee deleted successfully", null);
            return ResponseEntity.ok(response);
        } else {
            ApiResponse<Void> response = new ApiResponse<>(false, "Employee not found", null);
            return ResponseEntity.notFound().build();
        }
    }
}
```

**Comentario:** Se brindó contexto completo sobre cual archivo específico se quiere modificar, aún así generó uno nuevo y ni siquiera se encontraba dentro del paquete `controllers`. Para analizar el repositorio y entender cómo funcionan los módulos `web` y `core`, así como para compilar y desplegar la aplicación, sigue estos pasos sintetizados:


### Tercer prueba

Con una estructura de modelos 1) **Mistral Medium** para chat, 2) **Codestral** para codear y 3) **All MiniLM** para el manejo de embedings y tomando como ejemplo el proyecto **Novatium**, se busca un poco más de contexto sobre como funciona:

**Paso 1 - Ejecución "Chat"**

Enviamos como prompt:

```
necesito saber como funcionan los modulos web y core haciendo un analisis completo del repositorio. identifica como compilar, que consideraciones hay que tener previamente y como hago para poder desplegar la aplicación en un navegador web cualquiera. responde de manera sintetica, como si estuvieras hablando con alguien que va a comenzar con el bugfixing y no tiene contexto sobre las reglas de negocio.
```

Y nos da una respuesta en donde detalla:

```
### 1. **Estructura del Repositorio**
- **Módulo `core`**: Contiene la lógica principal de la aplicación, reglas de negocio, y funcionalidades centrales.
- **Módulo `web`**: Maneja la interfaz de usuario, rutas, y comunicación con el frontend (HTML, CSS, JavaScript).

### 2. **Compilación**
- **Pre-requisitos**:
  - Asegúrate de tener instalado **Node.js** y **npm** (para el frontend).
  - Instala las dependencias con:
    ```bash
    npm install
    ```
- **Compilar el proyecto**:
  - Para el backend (`core`), usualmente se usa un compilador como `go build` (si es Go) o `python -m pip install -r requirements.txt` (si es Python).
  - Para el frontend (`web`), compila los assets con:
    ```bash
    npm run build
    ```

### 3. **Despliegue en un Navegador**
- **Ejecución local**:
  - Inicia el servidor backend (generalmente con un comando como `npm start` o `python main.py`).
  - Abre el navegador y navega a `http://localhost:3000` (o el puerto configurado).

- **Despliegue en producción**:
  - Configura un servidor web (como Nginx o Apache) para servir los archivos estáticos del frontend.
  - Asegúrate de que el backend esté corriendo en un entorno accesible (como un servidor cloud o local).

### 4. **Consideraciones Importantes**
- **Entorno**: Configura variables de entorno para credenciales y configuraciones sensibles.
- **Dependencias**: Asegúrate de que todas las dependencias estén instaladas y actualizadas.
- **Compatibilidad**: Verifica que el código sea compatible con las versiones de Node.js, Python, o cualquier otro lenguaje/framework que uses.

### 5. **Bugfixing**
- **Debugging**: Usa herramientas como `console.log` (JavaScript) o `print` (Python) para depurar.
- **Pruebas**: Ejecuta pruebas unitarias y de integración para asegurar que los cambios no rompan funcionalidades existentes.

Si necesitas más detalles sobre algún paso en específico, házmelo saber.
```

**Comentario:** No identifica de manera automática que toda la aplicación está desarrollada en Java, creo que es algo fácil de identificar, al igual que el framework. Da una respuesta totalmente errada esperando a que compilemos la aplicación en Python o Node. Luego de otros mensajes en la conversación puedo validar de que nunca analizó bien como para decir "esta aplicación está desarrollada en Spring Boot con Java 7", por ejemplo.

**Comentarios genéricos:** es inevitable que el modo "Agente" vaya describiendo paso por paso. Para que aplique un cambio en un archivo específico desde el side nav con el chat, a veces son entre 2 y 3 mensajes para la solución esperada. A veces la respuesta se "rompe" por el formato .md que se espera, hasta incluso llegó al punto de marcar como "Potencialmente peligroso" una oración de "Si necesitas ayuda...", es confuso.



### Cuarta prueba

Con una estructura de modelos 1) **Gemini 2.0 Flash** para chat y code (consumiendo mediante OpenRouter), 2) **All MiniLM** para el manejo de embedings y tomando como ejemplo el proyecto **Novatium**, se busca un poco más de contexto sobre como funciona:

**Ejecución "Agente"**

Enviamos como prompt:

```
teniendo en cuenta los estandares @Inv - estandares... y los cambios a realizar a nivel base de datos resumidos en este archivo @migrate-postgres.md haz los cambios necesarios a nivel de "repository" dentro del paquete com.midasconsultores.repository
```

**Comentario:** Comenzó a editar los archivos que estaban citados dentro de los .md como ejemplo y tuve que especificar que debía utilizar a modo ejemplo para realizar las modificaciones en todas las clases de tipo repository. Luego de esa aclaración hizo un buen barrido de las clases a modificar, por suerte no eran muchas las modificaciones que debía hacer (específicamente para poder incrementar la versión de Postgres y que siga siendo funcional) y pudo realizarlas con éxito. Lo que sí, me consumió todos los tokens.

### Quinta prueba

Con una estructura de modelos 1) **Step 3.5 Flash** para chat y code (consumiendo mediante OpenRouter), 2) **All MiniLM** para el manejo de embedings y tomando como ejemplo el proyecto **Novatium**, se busca un poco más de contexto sobre como funciona:

**Ejecución Agente**

**Comentario:** se quiso continuar con los pasos realizados por el agente anterior pero este modelo free (Step 3.5 Flash) al parecer no matchea mucho en cuanto a los formatos que utiliza Continue Dev.

# Decisiones técnicas

Cada decisión se documenta con el contexto, la alternativa descartada y la consecuencia. El objetivo no es enumerar tecnología, sino explicar por qué el sistema tiene la forma que tiene.

## 1. MVC adaptado a REST con servicios por feature

**Contexto.** La API tiene dominios distintos: auth, perfil, búsqueda, matching y optimización. Un controlador con lógica de negocio habría mezclado validación, acceso a datos y proveedores externos.

**Decisión.** Rutas y controladores finos, servicios por feature y helpers co-locados. Los controladores traducen HTTP; los servicios contienen las reglas; Prisma queda en la capa de datos.

**Alternativa descartada.** Controladores gruesos con consultas y llamadas a IA dentro de la ruta.

**Consecuencia.** Más archivos y más navegación, pero límites claros, servicios testeables y menor acoplamiento entre dominios.

## 2. Middleware-first para validación y errores

**Contexto.** Las peticiones llegan con body, query y params; los proveedores externos también pueden fallar. Repetir validación y `try/catch` en cada controlador genera respuestas inconsistentes.

**Decisión.** Zod valida antes del controlador. Los servicios lanzan errores tipados y el manejador global decide el código HTTP y la forma de respuesta.

**Alternativa descartada.** Validación inline en cada controlador y respuestas de error ad hoc.

**Consecuencia.** Los contratos son explícitos y las respuestas son homogéneas. El coste es mantener esquemas Zod junto a cada dominio.

## 3. Persistencia por lotes con transacción e idempotencia

**Contexto.** Un ciclo de búsqueda puede devolver muchas ofertas y repetir URLs entre ejecuciones. Guardar una por una genera demasiados viajes a la base de datos y riesgos de duplicados.

**Decisión.** Normalizar y deduplicar en memoria, consultar las ofertas existentes en una sola operación, crear las nuevas con `createMany`, actualizar solo las que cambiaron y vincular las resultados en una transacción.

**Alternativa descartada.** Un `upsert` por oferta dentro de un bucle.

**Consecuencia.** Menos roundtrips y menos duplicados. El coste es una función de persistencia más compleja y la necesidad de mantener un hash de contenido para detectar cambios.

## 4. Matching secuencial con reclamación de estado

**Contexto.** El matching depende de un proveedor de IA con límites de cuota y latencia. Dos ejecuciones simultáneas podrían evaluar la misma oferta o crear la misma candidatura dos veces.

**Decisión.** Cada resultado se reclama con una actualización condicional de `PENDING` a `PROCESSING`. Solo la ejecución que reclama el resultado continúa. Al terminar, una transacción guarda el estado final y crea la candidatura con una restricción única por usuario y oferta.

**Alternativa descartada.** Procesar todos los resultados en paralelo sin reclamación de estado.

**Consecuencia.** El procesamiento es más lento, pero evita dobles evaluaciones y respeta mejor la cuota del proveedor. Los resultados huérfanos se pueden recuperar.

## 5. Umbral de match configurable por usuario

**Contexto.** Un mismo puesto puede encajar de forma distinta según el objetivo, la seniority y la tolerancia al riesgo de cada persona.

**Decisión.** El umbral vive en el perfil y decide cuándo se crea una candidatura. La puntuación se guarda siempre, aunque no supere el umbral.

**Alternativa descartada.** Un umbral fijo global para todos los usuarios.

**Consecuencia.** El producto se adapta mejor al usuario y conserva información para explicar por qué una oferta no pasó el filtro. El coste es más configuración y más estados que probar.

## 6. Proveedor de IA con fallback y salida JSON

**Contexto.** La extracción, el matching y la generación de textos dependen de un servicio externo. Una caída o una respuesta con formato inválido no debe corromper el estado del dominio.

**Decisión.** Groq como proveedor principal, OpenAI como fallback, timeout y reintentos, salida JSON y parseo defensivo. Si la evaluación falla, el resultado vuelve a `PENDING`.

**Alternativa descartada.** Confiar en que el proveedor siempre responde y guardar la respuesta sin validar.

**Consecuencia.** Más resiliencia y errores recuperables. El coste es más código de adaptación y una dependencia explícita de los límites de cada proveedor.

## 7. Privacidad por diseño en el caso de estudio

**Contexto.** El producto es privado y el caso de estudio es público.

**Decisión.** Publicar solo documentación, capturas con datos de ejemplo y fragmentos reducidos. No publicar prompts, pesos, esquema completo, credenciales ni datos de usuarios.

**Alternativa descartada.** Publicar el repositorio completo o dar acceso de solo lectura al repositorio privado.

**Consecuencia.** El caso de estudio demuestra criterio técnico sin exponer propiedad intelectual. La contrapartida es que el lector no puede ejecutar el producto completo desde este repositorio.

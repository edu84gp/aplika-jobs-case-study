# Ruta protegida

## Contexto

Los endpoints de IA consumen cuota y escriben datos del usuario. La autenticación, la autorización, el rate limit y la validación deben ejecutarse antes del controlador.

## Fragmento

Versión reducida y saneada; los nombres de middlewares y el controlador se simplifican.

```ts
router.post(
  "/optimize",
  authenticate,
  authorize("USER", "ADMIN"),
  aiRateLimit,
  validateBody(optimizeRequestSchema),
  optimizeController,
);
```

## Decisión

El orden del middleware es parte del contrato: primero identidad, después permisos, después límite de uso y por último validación. El controlador solo recibe peticiones que ya pasaron esas garantías.

## Trade-off

La cadena de middleware hace que parte del comportamiento no sea visible desde el controlador. A cambio, las reglas transversales se aplican de forma uniforme y no se duplican.

## Aprendizaje

La seguridad de un endpoint no está solo en el controlador. Está en la composición de auth, permisos, límites y validación, y en el orden en que se ejecutan.

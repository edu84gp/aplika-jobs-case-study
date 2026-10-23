# Validación de entrada con Zod

## Contexto

Los endpoints de IA reciben identificadores de ofertas que vienen de un proveedor externo. No son UUID y no se pueden validar como tales. Además, algunos campos nuevos deben ser opcionales para no romper clientes anteriores.

## Fragmento

Versión reducida y saneada; los imports y tipos internos se omiten.

```ts
import { z } from "zod";

export const optimizeRequestSchema = z.object({
  jobOfferId: z.string().min(1),
  forceRegenerate: z.boolean().optional(),
  targetLanguage: z.enum(["es", "en"]).optional(),
});

export const jobOfferParamsSchema = z.object({
  jobOfferId: z.string().min(1),
});

export type OptimizeRequest = z.infer<typeof optimizeRequestSchema>;
```

## Decisión

La validación se ejecuta en un middleware antes del controlador. El esquema distingue entre un identificador externo y un UUID interno. `targetLanguage` y `forceRegenerate` son opcionales para mantener compatibilidad.

## Trade-off

Los esquemas añaden mantenimiento y hay que sincronizarlos con los tipos del dominio. A cambio, el controlador recibe datos ya validados y los errores son consistentes.

## Aprendizaje

Validar en el borde evita que un identificador inválido llegue a Prisma o a un proveedor de IA. También obliga a explicitar qué campos son obligatorios y cuáles pertenecen a una versión nueva del contrato.

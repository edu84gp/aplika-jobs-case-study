# Orquestación de matching

## Contexto

El matching depende de un proveedor de IA con latencia y cuota limitada. Dos ejecuciones simultáneas podrían evaluar la misma oferta o crear la misma candidatura dos veces. Una respuesta inválida no debe dejar el resultado bloqueado.

## Fragmento

Versión reducida y saneada; la llamada al proveedor, los tipos y el manejo de errores se simplifican.

```ts
export async function runMatching(userId: string, threshold: number) {
  const pending = await prisma.searchResult.findMany({
    where: { userId, status: "PENDING" },
    include: { jobOffer: true },
  });

  let processed = 0;
  let matches = 0;

  for (const result of pending) {
    const claimed = await prisma.searchResult.updateMany({
      where: { id: result.id, userId, status: "PENDING" },
      data: { status: "PROCESSING" },
    });

    if (claimed.count === 0) continue;

    const evaluation = await evaluateMatch(result.jobOffer, threshold);

    if (!evaluation.ok) {
      await prisma.searchResult.update({
        where: { id: result.id },
        data: { status: "PENDING" },
      });
      continue;
    }

    await prisma.$transaction(async (tx) => {
      await tx.searchResult.update({
        where: { id: result.id },
        data: {
          status: evaluation.isMatch ? "MATCHED" : "REJECTED",
          score: evaluation.score,
          reason: evaluation.reason,
        },
      });

      if (evaluation.isMatch) {
        await tx.application.upsert({
          where: {
            userId_jobOfferId: {
              userId,
              jobOfferId: result.jobOfferId,
            },
          },
          update: {},
          create: {
            userId,
            jobOfferId: result.jobOfferId,
            status: "TODO",
          },
        });
      }
    });

    processed++;
    if (evaluation.isMatch) matches++;
  }

  return { processed, matches };
}
```

## Decisión

Cada resultado se reclama con una actualización condicional. Solo la ejecución que consigue el cambio de estado continúa. La escritura final y la creación de la candidatura comparten transacción. Un fallo del proveedor devuelve el resultado a `PENDING`.

## Trade-off

El procesamiento secuencial es más lento que un `Promise.all`. A cambio, evita dobles evaluaciones, respeta la cuota del proveedor y permite reintentar sin perder el estado.

## Aprendizaje

En un flujo con IA, el estado del dominio debe ser explícito. El proveedor puede fallar; lo que no puede fallar es la coherencia entre el resultado guardado y la candidatura creada.

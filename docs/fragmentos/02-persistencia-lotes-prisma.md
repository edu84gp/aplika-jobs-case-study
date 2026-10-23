# Persistencia por lotes con Prisma

## Contexto

Un ciclo de búsqueda puede devolver muchas ofertas y repetir URLs entre ejecuciones. Guardar una por una genera demasiados viajes a la base de datos y riesgos de duplicados. Las ofertas editadas manualmente por el usuario tampoco deben sobrescribirse.

## Fragmento

Versión reducida y saneada; los imports, tipos y campos internos se omiten.

```ts
export async function persistOffers(userId: string, offers: RawOffer[]) {
  const unique = dedupeByUrl(offers);

  return prisma.$transaction(async (tx) => {
    const existing = await tx.jobOffer.findMany({
      where: { originalUrl: { in: unique.map((offer) => offer.url) } },
      select: {
        id: true,
        originalUrl: true,
        contentHash: true,
        isManual: true,
      },
    });

    const byUrl = new Map(existing.map((offer) => [offer.originalUrl, offer]));
    const toCreate = unique.filter((offer) => !byUrl.has(offer.url));
    const toRefresh = unique.filter((offer) => {
      const current = byUrl.get(offer.url);
      return Boolean(
        current && !current.isManual && current.contentHash !== offer.contentHash,
      );
    });

    if (toCreate.length > 0) {
      await tx.jobOffer.createMany({
        data: toCreate,
        skipDuplicates: true,
      });
    }

    for (const offer of toRefresh) {
      await tx.jobOffer.update({
        where: { id: byUrl.get(offer.url)!.id },
        data: {
          title: offer.title,
          description: offer.description,
        },
      });
    }

    await tx.searchResult.createMany({
      data: unique.map((offer) => ({ userId, jobOfferId: offer.id })),
      skipDuplicates: true,
    });

    return { created: toCreate.length, refreshed: toRefresh.length };
  });
}
```

## Decisión

La operación completa se ejecuta en una transacción. La deduplicación ocurre antes de tocar la base de datos. Las ofertas existentes se consultan en una sola operación y se comparan por hash de contenido. Las manuales se protegen explícitamente.

## Trade-off

La función es más compleja que un `upsert` por oferta y requiere mantener un hash de contenido. A cambio, reduce roundtrips, evita duplicados y conserva las ediciones del usuario.

## Aprendizaje

La idempotencia no consiste solo en añadir una restricción única. También hay que decidir qué se actualiza, qué se conserva y qué se considera un cambio real.

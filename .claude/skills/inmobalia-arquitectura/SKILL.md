---
name: inmobalia-arquitectura
description: Arquitectura y decisiones del proyecto Inmobalia (agregador diario de anuncios de compra de inmuebles con Symfony + Docker + PostgreSQL). Usar antes de diseñar, implementar o modificar cualquier parte del proyecto: infraestructura Docker, entidades, conectores de portales, comando de importación, criterios de búsqueda, favoritos o interfaz.
---

# Inmobalia — arquitectura y decisiones

## Objetivo

App web local, de un solo usuario, donde se configuran **criterios de compra** de inmuebles.
Una vez al día se importan de varios portales **todos los anuncios existentes** que cumplen esos
criterios. El usuario los revisa en la interfaz y marca los que le interesan como **favoritos**.

## Decisiones tomadas

| Tema | Decisión | Motivo |
|---|---|---|
| Framework | Monolito Symfony (PHP 8.4), Twig + Symfony UX (Live Components), AssetMapper | Sin SPA ni build de JS |
| Servidor | FrankenPHP, basado en la plantilla oficial `dunglas/symfony-docker` | Sin nginx ni php-fpm por separado |
| Base de datos | PostgreSQL 17 con Doctrine ORM; columnas `JSONB` para datos específicos de cada portal | Relacional para criterios y favoritos, JSONB para la flexibilidad que se buscaba en NoSQL. Se descartó MongoDB (ODM peor mantenido) |
| Operación | **Solo compra** (sin alquiler) | Requisito del usuario |
| Portales | Idealista, Fotocasa, Habitaclia, pisos.com, Milanuncios | Un conector independiente por portal |
| Imágenes | **No se almacenan.** Solo se guardan sus URLs y se muestran directamente desde el portal | No hay almacenamiento para ficheros |
| Colas | **Sin Messenger.** Un comando de consola recorre los portales en secuencia | Un usuario, una ejecución diaria |
| Programación | Cron del sistema anfitrión + botón "Importar ahora" en la web | El portátil puede estar apagado a la hora del cron |

## Infraestructura Docker

Solo dos servicios en `compose.yaml`:

- `php`: FrankenPHP + Symfony. Sirve la web y ejecuta `app:import`.
- `database`: `postgres:17-alpine`, con volumen persistente `db_data`.

Opcional, solo si un portal carga los anuncios con JavaScript: un contenedor de Chrome headless para Symfony Panther.

Cron en el anfitrión:

```
0 8 * * * cd ~/Mio/Inmobalia && docker compose exec -T php bin/console app:import
```

## Estructura del código

```
src/
├─ Source/
│  ├─ SourceInterface.php        # search(SearchProfile): iterable<ListingData>
│  ├─ Idealista/IdealistaSource.php
│  ├─ Fotocasa/FotocasaSource.php
│  ├─ Habitaclia/HabitacliaSource.php
│  ├─ PisosCom/PisosComSource.php
│  └─ Milanuncios/MilanunciosSource.php
├─ Import/
│  ├─ ListingData.php            # DTO normalizado, común a todos los portales
│  └─ ImportRunner.php           # criterios × portales; sustituye los anuncios del día
├─ Command/ImportCommand.php     # app:import [portal]
├─ Entity/
│  ├─ SearchProfile.php          # criterios de compra guardados
│  ├─ Listing.php                # anuncio del día (se reescribe en cada importación)
│  ├─ Favorite.php               # copia completa del anuncio + notas; nunca se borra
│  └─ ImportRun.php              # log: portal, fecha, nº de anuncios, errores
├─ Controller/                   # listado, detalle, favoritos, criterios, importar ahora
└─ Twig/Components/              # filtros en vivo con Live Components
```

Los conectores se registran como servicios etiquetados por `SourceInterface`. Añadir un portal consiste en crear una clase nueva.

## Modelo de datos

- **SearchProfile**: nombre, provincia, municipio, zonas/barrios (opcional), precio mín./máx., m² mín./máx., habitaciones mín., baños mín., tipos (piso, ático, dúplex, casa/chalet, estudio), estado (obra nueva, segunda mano, a reformar), extras (ascensor, garaje, terraza, piscina, trastero, exterior), activo sí/no.
- **Listing**: portal, id externo, URL, título, precio, m², habitaciones, baños, tipo, dirección/zona, descripción, URLs de imágenes (array), `raw` JSONB, perfil que lo encontró, `firstSeenAt`, precio anterior, `isNew`.
- **Favorite**: copia completa de los campos de Listing en el momento de marcarlo, notas del usuario y fecha. Es independiente de Listing y sobrevive a las importaciones.
- **ImportRun**: portal, perfil, inicio/fin, nº de anuncios, estado, mensaje de error.

## Flujo de importación (`app:import`)

1. Para cada SearchProfile activo y cada portal, el conector traduce los criterios a la URL de búsqueda del portal.
2. Descarga los resultados con Symfony HttpClient y los parsea con DomCrawler (o Panther si hace falta JavaScript), paginando.
3. Normaliza cada anuncio a `ListingData`. Los filtros que el portal no admita se aplican después, en local.
4. Compara con los anuncios de ayer de ese portal:
   - un id externo que no existía se marca como **"Nuevo hoy"**;
   - si el precio es menor que ayer, se guarda el precio anterior y se marca como **bajada de precio**.
5. En una transacción por portal, borra los anuncios anteriores de ese portal e inserta los nuevos.
6. Si un portal falla, se conservan sus anuncios anteriores y el error queda en ImportRun. El resto de portales continúa.
7. Favorite no se toca nunca.

Deduplicación entre portales (el mismo piso publicado en varios): una huella a partir de dirección, m² y precio. Queda para una fase posterior.

## Scraping: riesgos y normas

- Los portales prohíben el scraping en sus condiciones de uso y tienen protección anti-bots (Idealista usa DataDome). El proyecto es de uso personal y local, con poco volumen.
- Una sola importación al día, con pausas entre peticiones y sin paralelizar contra un mismo portal. Se usa un User-Agent de navegador normal.
- Si un portal cambia su HTML, solo se rompe su conector. Por eso cada conector debe tener tests con HTML de ejemplo guardado en `tests/fixtures/<portal>/`.
- Idealista tiene una API oficial (developers.idealista.com), con acceso bajo solicitud y cuota limitada. Es preferible al scraping si se consigue acceso.

## Orden de implementación

1. Esqueleto: `compose.yaml`, Symfony y PostgreSQL funcionando en local.
2. Entidades, migraciones y pantalla de criterios (CRUD de SearchProfile).
3. Comando `app:import` con el primer conector: **pisos.com**, el más sencillo.
4. Listado con filtros, "Nuevo hoy", bajadas de precio y favoritos.
5. Resto de conectores, uno a uno: Fotocasa, Habitaclia, Milanuncios y, por último, Idealista, que es el que más bloquea.

Actualizar este documento cuando cambie alguna decisión o se complete una fase.

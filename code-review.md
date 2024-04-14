# Code Review

## Introducción

Hola!

Soy Débora Gómez y en este documento os presento algunas mejoras que yo aplicaría si tuviese que llevar a cabo este proyecto.

Me gustaría felicitaros por lo clara que está la prueba técnica, y la verdad es que me ha gustado la experiencia de hacer una code review.

## Arquitectura

El proyecto intenta seguir una arquitectura hexagonal que de hecho se adecua con los casos de usos expuestos en el enunciado pero la organización de las clases no sería la que yo implantaría.

Mi propuesta sería la siguiente:

```
com.idealista
            \_ application
                    \_ rest
                        AdsController.java
                    \_ response
                        PublicAd.java
                        QualityAd.java
            \_ domain
                    \_ service
                        AdService.java
                        AdServiceImpl.java
                    \_ model
                        Ad.java
                        Constants.java
                        Quality.java
                        Typology.java
                        Picture.java
                    \_ repository
                        AdRepository.java
            \_ infrastructure
                    \_ configuration
                        (Aquí irían clases de configuración como por ejemplo la configuración de mongoDB)
                    \_ repository
                            \_memory
                                InMemoryPersistence.java
                                AdVO.java
                                PictureVo.java
                            \_mongo
                                (En  el caso de tener una persistencia en base de datos aquí iria la clase encargada de hablar conl a base de  datos de mongoDB y las `Entity` correspondientes)
            \_ Main.java
```

Esta estructura seguiría las [guidelines establecidas](https://www.baeldung.com/hexagonal-architecture-ddd-spring) por el equipo de Spring boot a la hora de aplicar una arquitectura hexagonal en nuestros microservicios.

Lo que distingue principalemtente a esta arquitectura y a esta propuesta de organización de clases es que nos permitiría mover la capa de `domain` a otro repositorio y solo añádiendola como dependencia el proyecto seguiría funcionando ya que dicha capa es completamente agnóstica a los factores _in_ y _out_ externos.

## Diseño de la REST API

### Documentación

Añadiría la dependencia de Swagger u OpenApi al proyecto para que genere una documentación de API.

Spring boot tiene una dependencia que se llama `Springfox` que nos generaría una interfaz web que otros developers puedan probar dicho API de forma rápida.

Además existen plugins para poder usar este framework siguiendo la práctica _API first_ cuyo objetivo sería definir la API en formato `yml` y a partir de este fichero podríamos generar los DTOs e importarlos en nuestro proyecto.

Si no quisiéramos que esta API fuese pública o accessible se podrían crear diferentes entornos (desarrollo, stable, preprod y prod, por ejemplo) y exponerlo/habilitarlo solo en los que nos interese. Esta creación de entornos se haría a través de diferentes `application.yml` como explico en más detalle en [esta sección](#entornos-y-configuracion).

### Respuesta

En el caso del modelo `QualityAd.java` su campo `irrelevantSince` es de tipo `Date`. Esto es adecuado pero un standard a nivel de REST es que aquellos campos que contengan información de fechas sigan el formato [ISO_8601](https://en.wikipedia.org/wiki/ISO_8601) y una buena práctica sería usar el timezone UTC para que los clientes que lo consuman desde diferentes localizaciones sean los encargados de convertirlo al timezone local del usuario.

De tal modo que, para asegurarnos, lo podríamos conseguir usando por ejemplo la anotación `@JsonFormat`.

### Manejo de errores

Tal y como está codificado el servicio, si hay algún tipo de error devolverá siempre un 500. Para el manejo de errores yo siempre he usado una clase `Error` del dominio que contendría un campo `type` y un campo `message` (al menos) que luego se pudiera utilizar en el controlador y mapear a su correspondiente error HTTP.

Para controlar errores en Spring se podría crear una clase con la anotación `@RestControllerAdvice` que se encargue de mapear dichos errores a respuestas json y códigos HTTP.

## Seguridad

De acuerdo con las especificaciones, los endpoints `/ads/quality` and `/ads/score` deben ser accesibles para ciertas personas con ciertos permisos.

Personalmente delegaría el manejo de la autorización y autenticación a otro componente en la infraestructura, pero si quisiéramos hacerlo a nivel de microservicio estos endpoints deberían estar securizados.

Spring provee el paquete de Spring Security que permitiría que solo ciertas personas/servicios pudieran llamar a esos endpoints. Esto se controlaría añadiendo la anotación `@PreAuthorize("hasRole('ROLE_QUALITY_ADMIN')")` a dichos endpoints, y creaando una clase que implemente `WebSecurityConfigurerAdapter` y que se encargue de la configuración de seguridad de los endpoints. El role asociado a una petición se podría determinar haciendo uso de la clase `SecurityContextHolder`.

## Performance

### Almacenamiento de los datos

La persistencia en memoria no es lo más efectivo y se podría mejorar almacenando dichos datos en una base de datos.

Por la naturaleza de los datos que este microservicio va a manejar, una base de datos no relacional tipo MongoDB nos valdría.

De este modo, la operación de obtener los Ads irrelevantes en el método #InMemoryPersistence#findIrrelevantAds se haría a nivel de base de datos en lugar de volcar todos los ads en memoria y filtrarlos. Otras operaciones como #InMemoryPersistence#findRelevantAds también se verían beneficiadas.

## Clean code principles

### Single Responsability

En la clase `AdServiceImpl.java` los métodos `findPublicAds` & `findQualityAds` no solo ejecutan lógica de negocio sino que también se encargan de mappear las entidades del modelo de negocio al modelo vista.

Este mapeo sería más adecuado hacerlo en la capa del `AdsController` ya que la capa de negocio no debería de conocer el modelo usado por el controlador. Adicionalmente, conseguiríamos que solo tuviese la responsabilidad de ejecutar las reglas del negocio.

### Open/Closed

Tanto `PublicAd.java` como `QualityAd.java` tienen una serie de campos en común que se podrían definir en una clase padre `BaseAd.java` que englobase las características que tienen en común y luego cada una la heredase y añadiese aquello que sea relevante para cada caso de uso.

### Dependencies injection

Las dependencias se están inyectando a nivel de _fields_, sería mejor inyectarlas a través del constructor de la clase ya que esto facilitaría la labor de _testing_ unitario.

La variable es `private` pero no es `final` de modo que no podemos asegurar que su referencia pueda cambiar en algún momento mientras que al inyectar por constructor éstas referencias se mantendrían inmutables.

### Cognitive complexity

El método `AdServiceImpl#calculateScore` viola este principio por varios motivos: longitud del método, número de `if` statements y las diferentes responsabilidades (calcula el _Score_ pero también actualiza la base de datos).

La primera mejora sería refactorizar el método `AdServiceImpl#calculateScores` a algo como:

```
@Override
public void calculateScores() {
    val adsScoreUpdated =
        adRepository
            .findAllAds()
            .stream()
            .map((ad) => updateScore(ad));
    adRepository.saveAll(adsScoreUpdated);
}
```

Donde la función `updateScore` solo se encargaría de actualizar el su score. Este método inlcuso lo podría mover a otra clase (bien estática o un `@Component`) que nos permitiese testear esta lógica a parte y de forma separada.

Además, como decía, la longitud de este método es larga y sería mejor dividirlo en pequeños sub-métodos que ejecutasen las diferentes fases de la lógica haciendo el código más entendible:

```
public static void calculateScore(Ad ad){
    calculateDescriptionLengthScore(ad);
    calculateDescriptionTypologyScore(ad);
    calculateDescriptionHiglightsScore(ad);
    calculateAdCompletionScore(ad);
    ...
}
```

Donde el método `calculateAdCompletionScore` contendría la lógica que actualemnte se encuentra bajo el propio modelo algo que no debería ser así porque si cambiasen las reglas de negocio hab®ia que modificar dicho modelo. Así que sería mejor extraer esa lógica a otra clase.

### Hardcoded Strings

El método `AdServiceImpl#calculateScore` tiene una serie de palabras "hardcodeadas" en español, esto haría imposible la internacionalización del microservicio a otros idiomas (o más que imposible) haría que dicha lógica no funcionase y, por tanto, no se calculase correctamente el _score_.

Se podría hacer uso del `LocaleResolver` que provee Spring boot y tener differentes ficheros de `properties` que contuviesen esas palabras traducidas a su correspondiente idioma, y en el código usar la referencia a esos `properties`.

### Magic numbers

Es cierto que es buena práctica el evitar "hardcodear" números en el código y es mejor convertirlos a constantes pero dándoles un nombre que se abstraiga del valor númerico.

Es decir, la clase `Constants.java` tal y como está ahora, el nombre de las constantes es el valor numérico que contiene. Si se cambiase alguno de los criterios como que la descripción de un chalet fuese de al menos 100 palabras en lugar de 50, tendríamos que cambiar el valor de la constante `FIFTY` y su nombre ya que al ser 100 no reflejaría la realidad.

Yo propondría poner nombres como `CHALET_DESCRIPTION_QUALITY_THRESHOLD` que reflejan el objetivo que tiene dicho número y que si en un futuro hubiese que cambiar el valor, no nos forzaría a renombrar dicha variable también.

### Java Classes nomencalture

Solo mencionar que las _guidelines_ de Java sugieren que las clases sigan una nomenclatura tipo `AdVo.java` en lugar de `AdVO.java`. Configurando SonarQube para este servicio nos avisaría de esto.

## Tests

### Tests unitarios

La cobertura de los tests unitarios es baja ya que solo uno de los casos de negocio es testeado (sin nisiquiera considerar que se produzca algún error, solo se está probando el caso de éxito).

Como mínimo testearía los casos de éxito de `AdsServiceImpl#findQualityAds` y `AdsServiceImpl#findPublicAds` y casos de uso de errores como el considerar qué pasa si al guardar los datos falla.

Lo óptimo sería cubrir con test unitarios `AdsController.java` y `InMemoryPersistence.java` también.

### Tests de integración/e2e

Con el paquete de Spring Starter Web Test se podrían hacer tests de integración de cada uno de los endpoints para comprobar que las respuestas son adecuadas. Además nos permite "mockear" la capa de almacenamiento y así centrarnos en comprobar que todo el flujo del microservicio es el adecuado.

### Tests de e2e

Seguramente esto está fuera del _scope_ de esta prueba asi que simplemente mencionar que personalmente me gusta tener los e2e tests de toda la arquitectura en un repositorio independiente ya que facilita en la mayoría de los casos tener _pipelines_ que hagan _fetch_ de un solo repositorio y ejecuten los e2e tests que se deseen.

Pero también he trabajado en equipos en los que este tipo de tests formaban parte del repositorio del microservicio.

## Entornos y configuracion

No sé si estaría dentro del _scope_ de esta code review pero como desarrolladora echo en falta disponer de differentes configuraciones para diferentes entornos.

Es decir, en Spring boot podemos especificar diferentes `application.yml` que permitan al servicio consumir de bases de datos en local, o en remoto (por ejemplo).

Para facilitar labores de desarrollo, se podría tener un `docker-compose.yml` con el servicio de spring boot y una base de datos con los anuncios que permitiese a cualquier desarrollador poder replicar el entorno de forma fácil y rápida.

En los ficheros de configuración las URL/URIs a la base de datos cambiarían dependiendo del entorno en el que quisiéramos ejecutar el servicio.

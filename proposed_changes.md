## Actualización de la tecnología
- La versión de spring boot está fuera de soporte. Actualizaría a la última versión (3.2.5)
- La versión de java está fuera de soporte. Como vamos a actualizar a Spring Boot 3, la versión mínima es Java 17.

## Documentación, despliegue y herramientas de desarrollo

- En el README.md falta información de como levantar el proyecto.
- Sugeriría un Dockerfile para facilitar la creación del contenedor de la aplicación, asi como un docker-compose para facilitar el despliegue en local de todos los elementos necesarios.
- Si quisiéramos evitar la necesidad de instalar maven en los ordenadores de los desarrolladores, necesitaríamos generar un mvnw. Para ello un desarrollador que tenga maven tiene que ejecutar este código ```mvn wrapper:wrapper```.
- No hay logs ni estructura para crearlos.

## Testeo
- El único test es unitario y no testea prácticamente nada del método asociado, más allá de que busca en el repositorio y que guarda. Si nos basamos en el nombre del método, lo nuclear del método testeado AdsServiceImpl#calculateScores debería ser calcular la valoración, y no es comprobado que la calcule bien. Varios métodos a lo largo de la API tienen errores que de tener test relevantes y bien construidos serían reconocidos fácilmente sin necesitar una revisión profunda de código.
- Faltan test de integración para garantizar el correcto lanzamiento de la aplicación, asi como la correcta publicación de los endpoints.

## Autogeneración de código
- Sugeriría un acercamiento API First, generando así un contrato fácil de compartir y permitiendo autogenerar los controladores, reduciendo el posible error humano. Una vez tengamos un yaml para definir la API, autogeneraría los controladores con el plugin de org.openapitools:openapi-generator-maven-plugin.
- Utilización de Lombok para reducir el uso de *Boilerplate code*, como constructores, getters y setters. Esto se puede ver en multitud de ficheros, como en PublicAd o en AdVO. Si lo autogeneramos, evitamos el posible error humano.
- Utilización de algún plugin para hacer automáticamente mapeos entre los distintos objetos, reduciendo la cantidad de código sujeto a errores humanos. Recomendaría utilizar mapstruct. Uno de los mapeos sujeto a ser reemplazado puede verse en AdsServiceImpl#25. Otro ejemplo serían los mapeos en InMemoryPersistence, por ejemplo InMemoryPersistence#mapToDomain.

## Estructura y delegación de tareas

- No hay un controlador dedicado a excepciones que pudieran ocurrir en el código, delegando la gestión de los códigos de error al controlador principal, lo que favorece una posible repetición innecesaria de la gestión de las respuestas de error, asi como cargando de forma innecesaria el controlador. De igual forma, no existe esa gestión de códigos de error.
- AdRepository debería estar en el paquete infrastructure.persistence.
- Los mapeos, por ejemplo en AdsServiceImpl#25, deberían ser hechos en la capa de infraestructura, ya que utilizan objetos de la capa de infraestructura.
- Para un entorno de producción real, sugeriría la utilización de una base de datos real, no confiando en la base de datos en memoria, asi como la utilización de un ORM y JPA, para facilitar el desarrollo. Viendo que al menos los dos objetos están relacionados, sugeriría una base de datos relacional.

## Código limpio y uso de nombres adecuados y descriptivos

- Cambiaria el nombre de PublicAd por RelevantAd y el de QualityAd por IrrelevantAd y haría que extendieran de un objeto común que tuviera los atributos comunes de los dos. De la misma forma, cambiaría los métodos y urls para representar ese cambio. Un ejemplo son los métodos AdsController#qualityListing y AdsService#findQualityAds.
- Eliminar comentarios innecesarios y redundantes con el código. Por ejemplo AdsServiceImpl#70, AdsServiceImpl#72. Además, están en español, lo cual es una mala práctica en equipos internacionales.
- Hay código que no pasa el linter. Por ejemplo AdsController#3 debería estar después de los otros imports.
- Todos los endpoints de AdsController tienen una ruta común, asi que definiría "/ads" como *base path* para el controlador.
- AdsServiceImpl#93 cambiar "wds" por "nOfWords" para mayor claridad.
- AdsServiceImpl#110, AdsServiceImpl#111, AdsServiceImpl#112, AdsServiceImpl#113, AdsServiceImpl#114 se usan literales.Lo cambiaría por constantes. Por claridad, añadiría corchetes a los ifs y los pondría siguiendo el mismo estilo que el resto.

## Arquitectura  

- Según el volumen esperado, calcular la puntuación de todos los anuncios de una sola vez puede resultar costoso. Si por operativa fuera correcto que siempre se recalculasen los valores, intentaría hacerlo en otro servicio, para asegurar la disponibilidad. Si no se pudiera hacer, debería llegar a un acuerdo para gestionarlo a través de un proceso cron en las horas de baja demanda. En cualquier caso, parece una mejor solución que la puntuación se calcule al subir un anuncio o al modificarlo. En caso de que fuera obligatorio poder recalcular bajo demanda, intentaría reducir el número de anuncios a recalcular mediante algún acuerdo con el equipo de calidad. Si ninguno de estos casos se pudiera cubrir, ese endpoint no debería de ser un GET sino un POST. AdsController#27.
- No hay ningún tipo de perfilado para asegurar que solo los encargados de calidad puedan acceder a endpoints sensibles, como sería el de anuncios irrelevantes o el de calcular puntuación.

## Errores de código y optimización
- El endpoint de "/ads/public" devuelve los anuncios en orden creciente de puntuación, cuando el requisito es que sea de mejor a peor. Además, no comprueba que las puntuaciones hayan sido generadas, lo que puede provocar un error en InMemoryPersistence#66. Para evitar esto se podría o bien filtrar previamente los no calculados, o bien forzar a recalcular siempre o bien calcular los no calculados en el momento de hacer el find.
- El endpoint de "/ads/quality" tampoco comprueba si la puntuación ha sido calculada, asi que también tiene el mismo problema en InMemoryPersistence#75.
- AdsServiceImpl#21 Si usásemos una base de datos real, dejaría que el ordenamiento lo hiciera el motor de la base de datos, en lugar de con código.
- AdsServiceImpl#71 no comprueba si "ad.getPictures()" es nulo.
- AdsServiceImpl#97 se podría hacer alguna optimización de código al poner un else después del if, evitando que compruebe de forma innecesaria el segundo if.
- AdsServiceImpl#119 sobrescribe la puntación, en lugar de incrementarse como debería.
- De AdsServiceImpl#122 a AdsServiceImpl#136 hay una optimización posible
```java
int  score = Math.min(Math.max(score, Constants.ZERO), Constants.ONE_HUNDRED);
ad.setScore(score);
ad.setIrrelevantSince(score < Constants.FORTY  ?  new  Date() :  null);
```
- AdsServiceImpl#133 el cálculo de la de la fecha se hace basado en el tiempo de la máquina en la que se despliegue el servicio. Para evitar problemas asociados, podría configurar el servicio y la base de datos para gestionar siempre las fechas en UTC.
- AdsServiceImpl#calculateScores si utilizásemos una base de datos real y un ORM, probablemente tenga un uso más eficiente si hacemos un saveAll en lugar de un ```foreach(x -> save(x))```, por la gestión de la sesión.
- Ya que el anuncio contiene lógica de negocio Ad#isComplete, para clarificar como funciona se podría aplicar el patrón estrategia.
```java
public  class  Ad {
// ...
	private  static  final  Map<Typology, AdCompletionStrategy> strategies =
		Map.of(Typology.GARAGE, new  GarageCompletionStrategy(),
				Typology.FLAT, new  FlatCompletionStrategy(),
				Typology.CHALET, new  ChaletCompletionStrategy());

	public  boolean  isComplete() {
		if (pictures.isEmpty()) {
			return  false;
		}
		AdCompletionStrategy  strategy = strategies.getOrDefault(typology, new DefaultCompletionStrategy());
		return  strategy.isComplete(this);
	}
// ...
}

public  interface  AdCompletionStrategy {
	boolean  isComplete(Ad  ad);
}

public  class  GarageCompletionStrategy  implements  AdCompletionStrategy {
	@Override
	public  boolean  isComplete(Ad  ad) {
		return !ad.getPictures().isEmpty();
	}
}

public  class  FlatCompletionStrategy  implements  AdCompletionStrategy {
	@Override
	public  boolean  isComplete(Ad  ad) {
		return !ad.getPictures().isEmpty() &&
			ad.getDescription() != null && !ad.getDescription().isEmpty() &&
			ad.getHouseSize() != null;
	}
}

public  class  ChaletCompletionStrategy  implements  AdCompletionStrategy {
	@Override
	public  boolean  isComplete(Ad  ad) {
		return !ad.getPictures().isEmpty() &&
			ad.getDescription() != null && !ad.getDescription().isEmpty() &&
			ad.getHouseSize() != null && ad.getGardenSize() != null;
	}
}

public  class  DefaultCompletionStrategy  implements  AdCompletionStrategy {
	@Override
	public  boolean  isComplete(Ad  ad) {
		return  false;
	}
}
```

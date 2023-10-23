# Async-JS

## Tipos de asincronía en Javascript 
- **Events**: nos permiten responder a un suceso de determinado tipo. Podemos trabajar con eventos nativos del DOM, o bien lanzar y escuchar nuestros propios eventos: <br>
```js client 
button.addEventListener("click", (ev) => {
	const itemId = ev.target.dataset.itemid;
	document.dispatchEvent(new CustomEvent("itemAdded", { detail: { item: itemId } }));
});
```
 
- **Callbacks**: Son funciones que pasamos como parámetro a otra función. Esto permite a esta otra función cuando será el momento adecuado para ejecutar el callback: <br>
```js client
app.get("/", (req, res) => {
	res.send("Hello from a callback!");
});
```
En el caso de los callbacks tienen como desventaja que la anidación de estos produce un codigo ilegible y difícil de leer, para evitar este riesgo se utilizan las promesas en su lugar. 
- **Promises**: nos permiten actuar cuando un proceso asíncrono termina, sea con éxito o no. También nos aportan más legibilidad: 
```js client 
fetch(`https://gutendex.com/books?search=${query}`)
  .then((response) => response.json())
```

## Como evitar callbacks usando promesas

Para evitar los denominados "Callback hell" el prototipo promise nos permite convertir cualquier función en promesa usando el constructor **Promise** 

```js client
function writeFilePromisified(path, data) {
	return new Promise((resolve) => {
		writeFile(path, data, () => {
			resolve();
		});
	});
}
```

De esta forma hemos convertido la función writeFile que recibia un path un data y una función de callback en una promesa, esto nos permitirá encadenar diferentes acciones que segun el ejemplo nos permitirian manejar las acciones asincronas referentes a la escritura de un fichero, como se muestra a continuación:

```js client
writeFilePromisified(file, "")
	.then(getUsersFromApi)
	.then((users) => {
		return appendFile(file, JSON.stringify(users));
	})
	.then(() => {
		return copyFile(file, "../dest.json");
	})
	.then(() => {
		return unlink(file);
	})
	.then(() => console.log("Process finished!"));
```

## Mejora la legibilidad: de promises de a async/await
Async/await es otra forma de trabajar con promesas. En lugar de encadenar comportamientos usando .then(), podemos marcar una función como asíncrona con async, y esperar la resolución de la promesa con await para luego seguir con nuestro código de forma habitual.  

Con este planteamiento el proceso anterior quedaría de la siguiente forma, transformando nuestro código en un estilo más imperativo: 
```js client 
await writeFilePromisified(file, "");
const users = await getUsersFromApi();
await appendFile(file, JSON.stringify(users));
await copyFile(file, "../dest.json");
await unlink(file);
console.log("Process finished!");
```

## Ciclo de vida de una promesa 
### Estados de una promesa: 
- **pending**: estado inicial, no se ha cumplido ni se ha rechazado.
- **fulfilled**: la operación se ha completado con éxito.
- **rejected**: la operación ha fallado.
- **settled**: sería el estado que engloba fulfilled y rejected, es decir, cuando la promesa ya no está pending.
### Métodos que dependen de los cambios de estado 
- then(): se ejecuta cuando la promesa se resuelve, es decir, cuando pasa de pending a fulfilled.
- catch(): se ejecuta cuando algo sale mal y la promesa se rechaza, es decir, cuando pasa de pending a rejected.
- finally(): Se ejecuta cuando la promesa se resuelve o se rechaza, es decir, cuando pasa de pending a settled.
### Controlar errores usando reject 
Este método nos permite rechazar la promesa "manualmente" acorde a la ejecución de esta, es una alternativa al método catch, que nos permite pasar la promesa al estado rejected por nuestra cuenta. 

En el siguiente ejemplo se rechaza la promesa en caso de que el número de libros obtenidos sea 0. 
``` js client 
export function getBooksFromApi(search) {
	return new Promise((resolve, reject) => {
		fetch(`https://gutendex.com/books?search=${search}`)
			.then((response) => response.json())
			.then((data) => {
				if (data.count === 0) {
					reject();
				}
				resolve(data.results);
			});
	});
}
```

## Encadenar promesas sin perder legilibilidad 
Se aconseja usar catch y finally al final de la cadena de promesas, ya que usarlo en lugares intermedios permite controlar mejor el ciclo de vida de las promesas pero empeora mucho la legilibilidad del código, además debemos tener en cuenta que finally y catch también devuelven una promesa. 


## Event loop 
El event loop es un bucle que queda a la espera de tareas a realizar continuamente y se basa en una política FIFO, no obstante, las tareas a realizar se guardan en pilas diferentes, Call Stack, Microtasks y Macrotasks. 

El orden de prioridad de ejecución sería el siguiente: <br>
**Task Queue -> MicroTask > Render > Macrotask** 

Esto se define así ya que la prioridad máxima se le da a las acciones del usuario y no a las tareas que se ejecutan en segundo plano. 

**¿Como se comportan las promesas en este entorno?**

Para entender esto se puede ejecutar el siguiente código y estudiar la salida: 
``` js client 
function a() {
  console.log("1: soy el primer console log")

  setTimeout(() => {
    console.log("2: soy el console log del timeout")
  }, 0)

  promise().then(() => {
    console.log("3: soy el console log de la promise")
  })
}

function promise() {
  return Promise.resolve()
}
```

Curiosamente la ejecución de este código produce el siguiente orden de salida: 
```
1: soy el primer console log
3: soy el console log de la promise
2: soy el console log del timeout
```

Durante esta ejecución se ha comenzado ejecutando directamente el console log , a continuación se ha encolado en macrotasks el timeout ya que contiene un callback y la promesa se inserta en la lista de microtasks, esta lista conlleva una prioridad urgente por lo que se ejecuta antes que el timeout. Una vez el event loop ve que la unica tarea pendiente es la que está en macrotasks y procede a ejecutarla en este caso en último lugar. 

## Métodos de las promesas
### Promise.all - promesas en paralelo

Este método nos permite realizar varias llamadas "paralelamente", lo que nos permite realmente es lanzar varias promesas a la vez y esperar por ambas. 

Esto sería un ejemplo de llamadas de forma secuencial: 
``` js client 
const userInfo = await getUserInfo();
const enrolledCourses = await getUserEnrolledCourses();
```
A continuación se ve como podríamos realizar estas llamadas de forma paralela:
```js client 
Promise.all([getUserInfo(),getUserEnrolledCourses()]).then(([userInfo, enrolledCourses]) => {
	// ...
});
```
Este método tiene como inconveniente que si una de las promesas falla, todas fallarían, a continuación vemos otro método que contempla esto.
### Promise.allSettled
En el ejemplo anterior veíamos que en el Promise.all solo pasamos al then cuando todas las promesas del array se resuelven con éxito. Si no queremos que todo falle si solo falla una promesa, podemos usar Promise.allSettled y tratar el resultado según el status:

```js client 
Promise.allSettled([getUserInfo(), getUserEnrolledCourses()]).then((promisesResults) => {
	const [userInfoPromiseResult, enrolledCoursesPromiseResult] = promisesResults;
	
	if (userInfoPromiseResult.status === "fulfilled") {
		// tratamos caso de éxito de userInfo
	}

	if (enrolledCoursesPromiseResult.status === "fulfilled") {
		// tratamos caso de éxito de enrolledCourses
	}

	promisesResults
		.filter((r) => r.status === "rejected")
		.forEach((r) => {
			// tratamos los errores de forma genérica
		});
});

```

### Promise.any 
Si tenemos un array de promises y queremos quedarnos sólo con la primera que resuelva con éxito, podemos usar Promise.any. Por ejemplo, con este ejemplo podemos determinar la zona más rápida de AWS:
```js client 
const fastestRegion = await Promise.any([
	ping(AWSRegions[0]),
	ping(AWSRegions[1]),
	ping(AWSRegions[2]),
	ping(AWSRegions[3]),
]);
```

### Promise.race 
Similar al ejemplo anterior del Promise.any, pero en este caso se resolverá con la primera promesa que resuelva independientemente de si se resuelve con éxito o error. Por ejemplo, podemos forzar un timeout si una llamada tarda demasiado:

```js client 
const timeout = process.argv[2] || 5000;

Promise.race([getUserList(), timeoutCheck(timeout)])
	.then(() => {
		console.log("🚀 Users found");
	})
	.catch(() => {
		console.log("⌛ Timeout error in getUserList");
	});
```
## Async / await

Anteriormente en el curso hemos visto como async/await puede ayudar a mejorar la legibilidad del código cuando trabajamos con promesas. Pero no siempre es así, por ejemplo en este caso se hace bastante complicado de leer:

```js client 
const books = await (await fetch(
  `https://gutendx.com/books?s=${req.query.search}`
)).json();
```

Lo cierto es que ambas sintaxis pueden convivir:
```js client 
const books = await fetch(
  `https://gutendx.com/books?s=${req.query.search}`
).then(response => response.json());
```

Aplicando buenas prácticas, refactorizando nuestro código en métodos con semántica, podemos mejorar la legibilidad de nuestro código y usar .then o async/await según nos convenga:

```js client 
const books = await searchBookFromApi(req.query.search)

function searchBookFromApi(query) {
  return fetch(`https://gutendx.com/books?s=${query}`)
    .then((response) => response.json());
}
```

### Manejar el reject de las promesas 
Si una promesa falla y no hemos usado catch, tendremos un error de JavaScript. Es algo especialmente preocupante en un entorno de Node.js, ya que desde la versión 15 provocan que el proceso se cierre.

En general es buena práctica tratar las promesas correctamente, pero también tenemos listeners globales que nos permiten capturar cualquier error.

En el buscador:
```js client
window.addEventListener("unhandledrejection", (event) => {
	console.error("Unexpected error");
});
```

No obstante la recomendación en la mayoría de casos sería usar el **catch**

Para esto se recomienda usar un lintern: 
La regla de ESLint no-floating-promises nos puede ayudar a detectar casos en los que no estemos tratando las promesas con catch ni devolviéndolas.

### Ejemplos aplicados de promesas

Como cancelar una promesa de fetch :
Para cancelar esto se usa el objeto ** Abort Controller **, en primer lugar habría que crear una instancia del objeto, y utilizarla como parámetro del fetch, una vez realizado esto la clave es que en el evento que produce la llamada se compruebe si exite ya un controller, y si es así se aborta, se muestra ejemplificado en el siguiente código que construye una barra de busqueda dinámica con peticiones: 

```js client 
function searchBooks() {
	const controller = new AbortController();

	const query = (search) =>
		fetch(`https://gutendex.com/books?search=${search}`, { signal: controller.signal }).then(
			(response) => response.json()
		);

	return {
		controller,
		query,
	};
}

const { query, controller } = searchBooks();

const input = document.getElementById("search");
const books = document.getElementById("books");

let ctr;

input.addEventListener("input", (event) => {
	if (ctr) {
		console.log("abort", event.target.value);
		ctr.abort();
	}

	const { query, controller } = searchBooks();
	ctr = controller;

	query(event.target.value)
		.then((response) => {
			books.innerHTML = "";
			response.results.forEach((book) => {
				books.innerHTML += `<li>${book.title}</li>`;
			});
		})
		.catch(() => {
			console.log("fetch cancelado");
		});
});
```

### Bucles de promesas
Cuando debemos realizar diferentes promesas, podemos optar por diferentes formas, el menos comun es el secuencial con un for, que tiene como inconveniente que cada promesa tarda su tiempo mas el tiempo de las anteriores, por otra parte tenemos el promise.all que nos permite hacerlas en paralelo pero no nos va a permitir mostrar datos hasta que esten todas las promesas resueltas, y en ultimo lugar tenemos el foreach, que en algunos casos puede ser de mucha utilidad ya que nos permite mostrar los datos de las promesas a medida que se van calculando: <br>
**for**
```js client 
/*
 With a for loop the promises are resolved sequentially
*/
for (let index = 0; index < metricQueries.length; index++) {
	const [metricQuery, selector] = metricQueries[index];
	const result = await metricQuery();
	printCustomerMetric(selector, result);
}
```
**for each**
```js client
/*
 With a forEach loop the promises are resolved in parallel
*/
metricQueries.forEach(async (metricQuery) => {
	const [query, selector] = metricQuery;
	const result = await query();
	printCustomerMetric(selector, result);
});
```
**Promise.all**
```js client 
Promise.all([searchTotalCustomers(), searchMonthlyRevenue(), searchOpenIncidents()]).then(
	([customers, revenue, incidents]) => {
		printCustomerMetric("customers", customers);
		printCustomerMetric("revenue", revenue);
		printCustomerMetric("incidents", incidents);
	}
);
```

### Typescript gestión del error 
Para gestionar los errores en nuestras promesas podemos usar try/catch o el método catch del prototipo de Promise.


Si a la premisa anterior unimos TypeScript vemos que hay diferencias interesantes en cómo se comporta el lenguaje en cada uno de los casos.


En el caso del catch, TypeScript (version 5.1) establece por defecto el tipo del error a any dejando de mano de los desarrolladores el garantizar el tipado.

```js client
getUsers()
  .then((response) => {
    console.log(response);
  })
  .catch((error) => {
    // Error is not type safe because is any
    console.error(error.toStrin());
  });

```
Sin embargo, usando try/catch y con el flag useUnknownInCatchVariables del tsconfig, el tipado por defecto de los errores se establece a unknown obligando a los desarrolladores a hacer narrowing antes de acceder a las propiedades del error.


Esta alternativa es más segura a nivel de tipos aunque requiera más pasos para gestionar los errores.

```js client 
  try {
    const users = await getUsers();
  } catch (error) {
    // error is unkown so it is type safe and we need to narrow it before usage
    //error.toStrin()

    if (error instanceof UsersNotFoundError) {
      console.error(error.errorMessage());
    }
  }


```

## Otras formas de asincronía 

### Web workers 
Aunque JavaScript es single thread es posible ejecutar procesos en paralelo dentro del browser gracias a los web workers.

Ejemplo de uso: 
```js client 
const myWorker = new Worker("worker.js", { type: "module" });
```

Una vez tenemos registrado un worker, podemos comunicarnos con este proceso de la siguiente manera.
```js client 
myWorker.postMessage(nPrime);
myWorker.onmessage = (e) => {
	printResult("nPrimeWorkerResult", nPrime, e.data);
};
```
[Librería de workers que nos permite usarlos como promesas y mas cosas.](https://link-url-here.org)

### Service workers 

Los service workers son una versión mejorada de los web workers que vimos en el anterior vídeo pero con funcionalidades de red permiten cachear tráfico, hacer de proxy y cualquier funcionalidad relacionada con la intercepción de llamadas fetch de nuestra aplicación.

#### Ejemplo de uso - Evitar imagenes rotas en la web

Para evitar esto vamos a usar un service worker que actue como proxy de nuestras peticiones fetch: 

- Registramos el service worker: 
```js client 
navigator.serviceWorker
	.register(new URL("sw.js", import.meta.url))
	.then(() => {
		// Registration was successful
		console.log("ServiceWorker registered");
	})
	.catch((err) => {
		// registration failed :(
		console.log("ServiceWorker registration failed: ", err);
	});
```
- Programamos el service worker que acabamos de registrar para interceptar el tráfico y cachear los recursos que queramos.
```js client 
self.addEventListener("fetch", (e) => {
	e.respondWith(
		fetch(e.request)
			.then((response) => {
				if (!response.ok && isImage(e.request) && response.type !== "opaque") {
					console.log("Serving broken image");

					return caches.match(brokenImage);
				}

				return response;
			})
			.catch((err) => {
				if (isImage(e.request)) {
					return caches.match(brokenImage);
				}
				throw err;
			})
	);
});
```

 ### Websockets
 [Básicamente mirar esto](
 https://developer.mozilla.org/en-US/docs/Web/API/WebSockets_API)


 ### RXJS - Buscador con fetch cancelable

 ```js client 
 import { debounceTime, distinctUntilChanged, filter, fromEvent, map, switchMap, tap } from "rxjs";
import { fromFetch } from "rxjs/fetch";

const input = document.getElementById("search");
const books = document.getElementById("books");

const bookRequest = (search) => {
	return fromFetch(`https://gutendex.com/books?search=${search}`).pipe(switchMap((r) => r.json()));
};

fromEvent(input, "keyup")
	.pipe(
		debounceTime(500),
		map((e) => e.target.value),
		filter((e) => !!e),
		tap(console.log),
		distinctUntilChanged(),
		switchMap(bookRequest),
		map((resp) => resp.results.map((book) => book.title)),
		tap((titles) => {
			books.innerHTML = "";
			titles.forEach((title) => {
				books.innerHTML += `<li>${title}</li>`;
			});
		})
	)
	.subscribe();
```

**fromEvent()** -> registra el evento key up <br>
**distinctUntilChanged()** -> no emite el evento hasta que no <br> cambie el valor 
**switchmap()** -> Si hay otro observable ejecutandose lo cancela 

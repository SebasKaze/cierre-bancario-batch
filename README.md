# Cierre bancario con Spring Batch

**Autor:** Hilario Sebastian Espinoza Garcia

## Cómo correrlo

    docker compose up -d --wait
    ./correr.sh 2026-09-30 prueba
    ./ver-batch.sh

## Día 1 · Mi primer Job

### Boleto de salida

1. ¿Qué diferencia hay entre un proceso batch y la API REST de la Semana 3? Da dos.

- La API REST tiene que estar arriba siempre que se quiera recibir informacion, batch no, corre una unica vez por ciclo
- La API REST trabaja bajo llamadas individuales de manera rapida, batch procesa cantidades masivas de informacion de manera mas lenta 

2. ¿Qué es un Job, qué es un Step y qué es un Tasklet?

- Job: Es un contenedor lógico que agrupa la configuración general y define el flujo de ejecución.
- Step: Es una fase dentro de un Job que encapsula la lógica de negocio real. Un Job está compuesto por uno o más Steps. Un Step puede procesar datos de forma iterativa o puede ejecutar una tarea única.
- Tasklet: Es una interfaz simple utilizada dentro de un Step para ejecutar una tarea puntual e indivisible, en lugar de procesar grandes volúmenes de datos iterativamente.

3. Con tus tablas: ¿qué diferencia hay entre una **JobInstance** y una **JobExecution**?

JobInstance parece que solo se crea una vez para una combinación específica de parámetros. Y JobExecution se crea cada vez que se intenta ejecutar un JobInstance

4. ¿Por qué Spring Batch no deja correr dos veces el cierre del 28?

Spring Batch utiliza este mecanismo para evitar el procesamiento duplicado. Una JobInstance representa un trabajo  único que se identifica por el nombre del Job más sus parámetros de ejecución. Si ese trabajo  finaliza con éxito (estado COMPLETED), bloquea cualquier intento de volver a correrla con los mismos parámetros. Si la instancia falla, sí permite reiniciarla (creando un nuevo JobExecution para la misma JobInstance), pero nunca si ya fue exitosa.

5. (MP-4, paso 6) Si mañana llega el archivo del 25 y corres otra vez el cierre del 25, ¿será otra instancia u otra ejecución de la misma? ¿Por qué lo crees?

Será otra instancia. El JobInstance se define por la combinación del nombre del Job y sus JobParameters identificadores. Al enviarle parámetros diferentes o valores nuevos el identificador deberia cambiar.




## Día 2 · El primer chunk

### Boleto de salida

1. ¿Qué diferencia hay entre un step de tipo Tasklet y uno de tipo chunk?

Un Tasklet se ejecuta una sola vez de como una única transacción. Un Chunk procesa grandes volumenes de datos y los divide en bloques 

2. ¿Qué hace cada una de las tres piezas de un chunk? ¿Cuál es opcional?

- ItemReader: Lee los datos secuencialmente de un origen, devolviendo un elemento a la vez o devolviendo null cuando se terminan los datos.
- ItemProcessor: Recibe el elemento leído, aplica la lógica de negocio y devuelve un elemento modificado. Si el elemento no cumple una condición y no debe ser escrito, el procesador puede devolver null para descartarlo de ese lote. ** Este es opcional ** 
- ItemWriter: Recibe el chunk de los elementos ya procesados y los guarda físicamente en el destino

3. Con 45 movimientos y chunks de 10, ¿cuántos commits habría? ¿Y con chunks de 50?

Con 45 movimientos haria 5 commits y con 50 unicamente 1

4. ¿Por qué el Escritor recibe el chunk completo y no un movimiento a la vez?

Por una cuestion de optimización, generaría una saturación masiva de la red y de la base de datos debido a las múltiples transacciones abiertas y cerradas.

5. Mi predicción de la MP-3, paso 1: ¿qué habría pasado sin el Procesador?

Se guardarian los datos tal como vienen de los bloques, esto puede generar inconsistencia o agrupaciones erroneas

## Día 3 · Parámetros, fallas y reinicio

### Boleto de salida

1. ¿Qué diferencia hay entre una JobInstance y una JobExecution? Usa como ejemplo el cierre del 25.

- JobInstance: Es aquello que se va a ejecutar que involucra la logica del negocio
- JobExecution: Es el proceso o intento de ejecutar el plan o JobInstance

2. ¿En qué caso Spring Batch se niega a correr un cierre, y en qué caso lo reinicia?

Cuando un job ya esta marcado como completado, lo que si puede reiniciar es aquello marcado como failedo stopped

3. En el reinicio del día 5, ¿por qué el step de carga leyó 10 movimientos y no 20?

Porque ya habia procesado algunos, y empieza desde donde hubo el error para no cargar los anteriores

4. ¿Qué diferencia hay entre un movimiento **filtrado** y uno **omitido**?

- Filtrado: Es un descarte intencional y esperado basado en la lógica de negocio
- Omitido: Es un descarte causado por una falla o anomalía técnica, pero que el framework decide "dejar pasar" para no arruinar todo el lote

5. ¿Por qué importa el código de salida, si el estado ya queda en las tablas?

Para saber el porque de ciertas cosas como los fallos, ya que ahi podemos ver a detalle que es lo que fallo



## Día 4 · De MySQL a MongoDB

### Boleto de salida

1. ¿Qué hace cada uno de los tres steps de tu Job, y de qué tipo es cada uno?

Son 3 steps.
- verificarArchivoStep: Verifica que los movimientos en base a las fechas existan, si no arroja una excepcion y detiene la ejecucion.
- cargarMovimientosStep: Procesa los datos, escribe los chunks en la BD 
- publicarSaldosStep: Lee la informacion de la BD con una consulta y manda estos datos a Mongo

2. ¿Por qué el cierre del 9 no duplicó los saldos, y el del 10 (sin `@Id`) sí?

Al quitar el Id no reconoce la cuenta, entonces la crea, esto crea duplicados aunque con informacion diferente

3. Al reiniciar el cierre del 11, ¿por qué no se cargó otra vez el archivo?

Porque ya habia un registro de donde se quedo, solo el step 3 y se cargo la informacion

4. ¿Qué diferencia hay entre `spring-boot-starter-data-mongodb` y «Spring Batch MongoDB» (`batch-data-mongodb`)?

La principal diferencia que existe es en su proposito, spring-boot-starter-data-mongodb tiene como proposito el CRUD y provee la conexion a base de datos.

Por otro lado, batch-data-mongodb se encarga del tratamiento de informacion masiva 


## Lo que aprendí esta semana

(Con tus palabras, en 5 a 10 renglones: qué es un proceso batch, qué piezas tiene un Job y qué hace Spring
Batch cuando algo falla.)


- Un proceso batch es aquel que procesa un gran volumen de datos en una sola instancia, sin que interactue nadie con el, por lo general estan programados para suceder en una hora en concreto.
- Un job es el proceso de inicio a fin, y esta compuesto por:
    - Step: una fase del job la cual puede realizar una activdad solamente (Tasklet) o por chunks, y pueden ser leer movimientos, calcular, exportar, entre otros.
    - Chunk: es el tamaño del bloque en el cual se va a divir la informacion para ser procesada. Si el chunk es 100, se lee y procesa 100 items, los escribe juntos y hace commit de una transacción. Luego repite.
    - JobParameters: son los parámetros con los que se lanza el Job, por ejemplo, una fecha.
    - JobInstance: es la ejecución lógica, identificada por el nombre del Job más sus parámetros. “Cierre del 2026-10-08”.
    - JobExecution: es cada intento concreto de correr esa instancia o ese step. Una JobInstance puede tener varias ejecuciones (la primera falló, la segunda terminó).
    - JobRepository: guarda todos los metadatos (estado, contadores, errores, posición del reader) en tablas BATCH_*.
    - JobLauncher: lo que lanza el Job.
- Spring Batch cuando algo falla tiene varias maneras de sortear un fallo o problema, estan las configurables que en caso de detectar algo no deseado como un nombre mal formateado puede skipearlo. En caso de que haya fallado un chunk hace Rollback desde el chunk donde encontro el problema, sin volver a cargar los anteriores. Tambien permite reiniciar si se relanza el Job con los mismos parámetros, Spring crea una nueva JobExecution de la misma JobInstance. Los steps que ya terminaron se saltan, y el step que falló continúa desde el último chunk confirmado
    


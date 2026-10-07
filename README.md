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
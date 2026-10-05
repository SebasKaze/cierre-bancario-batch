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

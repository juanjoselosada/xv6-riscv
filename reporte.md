Informe - planificación de procesos en XV6

Samuel Sepulveda 2435333
Juan Jose Losada 2439399

1. Preparar el repositorio y el entorno



i. La versión del linux donde se clonó es:
Ubuntu 26.04.1 LTS

ii. El identificador del commit:
fa2ddcb7ca87a77af579380a6eafd5aff163b44f



i. La versión de las herramientas que nos permiten compilar el sistema operativo so las siguientes:
g++ 15.2.0
nano 8.7.1
qemu 10.2.1
GCC + Binutils
GNU Make 4.4.1
GDB 17.1



ii. Imagen de las herramientas y su versión

Evidenciamos la versión de nano funcionando

2. Compilar y ejecutar XV6
i. Evidencia de que esta operativo el XV6. Captura de ls y cat, u otras pruebas.

Captura XV6

Captura ls

Captura Cat
(imagenes) https://docs.google.com/document/d/13NaJKhjNVwsPThQ-U8Hi31RdOgirXvllXr5JmFlVwU4/edit?pli=1&tab=t.0

Configuren la ejecución del sistema operativo XV6 para que utilice una sola CPU virtual en qemu
i. Consulta de cómo se hace esto
ii. En la primera ejecucion del scrip para que funcione el sistema operativo en mononucleo, salió error exec fail. Lo compilamos dentro del sistema. La solucion, ejecutarlo por fuera.

3. Investigar el planificador original

Mediante la lectura del código, la documentación y el diálogo con la IA ( OpenCode puede ayudarles a procesar archivos de su computador con un modelo de IA), respondan en sus propias palabras:
i. Round Robins
Pruebas: Intentamos ejecutar por todos los medios un codigo que nos permitiera visualizar la forma en que trabajan los procesos en el SO XV6, pero todas las pruebas nos dieron error.
ii. La funcion es scheduler(), y se encuentra en el kernel
iii. El proceso que estaba ahí sigue hasta terminar su quantum (o hasta que termine o se bloquee), y el proceso que entra se coloca al final de la cola de listos
iv. El código del planificador está en kernel/proc.c, en scheduler(), yield() y sched(). Al ejecutar make, el Makefile compila ese archivo a kernel/proc.o con el gcc de RISC-V, junto con el resto del kernel.

4. Describir MLFQ y explorar su incorporación
Lo que habría que tocar está casi todo en el planificador. En struct proc (kernel/proc.h) hace falta guardar el nivel de cola, el tiempo ya usado en ese nivel y cuánto lleva el proceso sin correr. userinit y fork en kernel/proc.c tendrían que dejar al proceso nuevo en la cola más alta, solo lo marcan RUNNABLE. La selección cambiaría en scheduler(): en vez del primer RUNNABLE del arreglo, se elige el de la cola más alta que tenga listos. El quantum y la degradación irían en yield() y en el timer de kernel/trap.c (which\_dev == 2). El boost, para no dejar un proceso abajo para siempre, se podría revisar en clockintr() o al entrar otra vez a scheduler().
Los campos en struct proc hacen falta porque no hay nivel ni quantum, sin eso no se degrada ni se detecta inanición. Inicializar en userinit y fork atiende la regla de que el proceso nuevo entra arriba, si no, nace sin prioridad MLFQ. Cambiar scheduler() es lo central: si no se toca, el kernel sigue eligiendo como round robin y lo demás no sirve. El timer y yield() sirven para bajar al que se come todo el quantum, y para no tratar igual al de E/S que se duerme en sleep() antes de gastar su tiempo. El boost cubre la inanición: subir al proceso solo cuando las colas de arriba están vacías no sirve si justo hay gente arriba. Por eso quedamos en una espera por proceso, con un tiempo S.
Cuántas colas usar (hablamos de Q8 a Q0, pero en XV6 quizá basten menos). Si el quantum es igual en todos los niveles o más largo abajo. El valor de S en ticks de clockintr(). La regla exacta en sleep()/wakeup() a mitad de quantum. Si dentro de una cola hay FIFO de verdad o se sigue el orden de proc\[]. Un caso que no cerramos: A casi cumple S en cola baja, B y C acaban de entrar arriba, y se cumple S de A mientras B aún tiene quantum: no acordamos si A saca a B ya o espera al próximo yield(). Con CPUS=1 no miramos SMP.
Nuestra versión. Varias colas, mayor número = más prioridad, el nuevo entra arriba, si gasta el quantum baja. E/S que suelta pronto no se degrada igual. El pesado no sube solo porque arriba esté vacío, cada uno tiene su espera S y, si se cumple, vuelve a la cola máxima. Si llega alguien de más prioridad, ese corre primero.

Al inicio escribimos que “los más importantes son los que ingresan a la CPU”. Eso era ambiguo: no es el que está corriendo, es el nuevo. También mezclamos degradar con volver a subir. Samuel describió bien el síntoma (inanición), Juan Jose acertó el disparador (S sin ejecutarse).

En XV6 la selección está en scheduler(), el corte de quantum en el timer y yield(), y el planificador llega al binario porque proc.o se enlaza en kernel/kernel. En esta entrega no se implementa MLFQ.

Lo que nos corrigió la IA. Preguntó en qué cola nace un proceso, dijimos “arriba, porque así es MLFQ” y no le bastó. Con A abajo y B nuevo, dijimos que corre B y que A, al cumplir s, volvía a lo más alto. Ahí nos marcó la contradicción. Luego dijimos subir a A si no había nadie más importante, el tutor preguntó qué se gana si, con las colas altas vacías, A ya podría correr desde abajo. Entre las dos respuestas, se quedó con la de Juan José. La espera por proceso para inanición sí la dio por registrada.



Nuevo vs. ya degradado. A calcula y ya debería estar abajo, después se lanza B. Se comprueba que el nuevo entra arriba y toma la CPU. A se queda en su cola baja, no vuelve solo a la máxima.

Degradación. Un bucle largo. Cada vez que gasta el quantum de su cola, debe bajar de nivel, no quedarse siempre arriba.

E/S vs. CPU. Uno que lee/escribe y se duerme, otro que no suelta la CPU. El de E/S debería quedarse más arriba, el pesado, bajar.

Inanición. Varios procesos cortos renovándose arriba y uno pesado abajo. Al cumplirse S, el de abajo vuelve a la cola máxima y en algún momento corre, si S no se cumple, sigue abajo.

Misma cola. Dos procesos que entran casi juntos arriba. Deben alternarse mientras no aparezca alguien de más prioridad, y no “ganar” siempre el mismo por el orden fijo de proc\[], salvo que eso sea lo que definamos.


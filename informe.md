# Informe Sistemas Operativos - Laboratorio 3: Planificador de Procesos

Grupo: - DTML -
Integrantes: Lisandro Allio, Mauricio Pintos, Dianela Fernandez, Tomás Romanutti.

___

## Primera Parte

1. **¿Qué política de planificación utiliza xv6-riscv para elegir el próximo proceso a ejecutarse?**

Utiliza la politica Round-Robin, donde se permite la ejecucion de procesos en simultaneo, 
aquellos que esten disponibles para ejecutar (marcados con RUNNABLE), con un "tiempo" llamado **quantum**.
La funcion scheduler recorre todos los procesos del sistema (recorriendo el array proc) hasta encontrar el 
primero en RUNNABLE y lo ejecuta, pasandolo a estado RUNNING  y realizando el context switch.

2. **¿Cuánto dura un quantum en xv6-riscv?**

El quantum dura aproximadamente 1/10 segundos (100 milisegundos) 

3. **¿Cuánto dura un cambio de contexto en xv6-riscv?**

El cambio de tiempo se puede calcular usando la siguiente fórmula:

`13 * (sd) + 13 * (ld) + 26 * (fetch)`

Donde sd es lo que dura realizar una instrucción sd, ld la duración de una instrucción ld, y fetch, cuanto tarda la computadora en actualizar el PC.
Notar que el tiempo de cambio de contexto depende de las especificaciones de la computadora, en específico la velocidad del reloj.
Generalmente, las instrucciones "sd" y "ld" tardan 2 ciclos de reloj, mientras que actualizar el PC tarda un solo ciclo. Entonces la fórmula queda de la siguiente manera:

`13 * (2 ciclos) + 13 * (2 ciclos) + 26 * (1 ciclo) = 75 ciclos de reloj`

Dependiendo de la frecuencia, estos 75 ciclos tardan **entre 46 y 375 nanosegundos.**

4. **¿El cambio de contexto consume tiempo de un quantum?**

NO, quantum es tiempo que se asigna por proceso en user_space. EL context_switch se 
desempeña por el S.O en kernel space

5. **¿Hay alguna forma de que a un proceso se le asigne menos tiempo?**

Si, modificando la variable interval.

6. **¿Cúales son los estados en los que un proceso pueden permanecer en xv6-**
**riscv y que los hace cambiar de estado?**

Un proceso puede permanecer en los estados:
UNUSED, USED, SLEEPING, RUNNABLE, RUNNING, ZOMBIE

Las siguientes funciones hacen cambiar de estado:

 * **UNUSED:** freeproc(), procinit()

 * **USED:** allocproc()

 * **SLEEPING:** sleep()

 * **RUNNABLE:** userinit(), fork(), yield(), wakeup(), kill()

 * **RUNNING:** scheduler()

 * **ZOMBIE:** exit()

___

## Mediciones del planificador

Todas las mediciones se hicieron de la misma manera, con solo una CPU, en la misma máquina cuyas especificaciones se muestran debajo.

**Hardware:** Intel (R) Core(TM) i3-4100M CPU @ 2,50GHz

**Software:** Qemu versión 6,2,0 

En absolutamente todos los casos, OPW/100T y OPR/100T daban igual.


### Round Robin

#### Quantum Original

| Caso                            | Promedio OPW/100T / OPR/100T          | Promedio MFLOP100T | Promedio MFLOP100T (2° cpubench) | Cant. selec. IO | Cant. selec. CPU | Cant. Selec. CPU 2 | Descripción |
|:------------------------------- |:-------------------------------------:|:------------------:|:--------------------------------:|:---------------:|:----------------:|:------------------:|:----------- |
| iobench                         | 4682,4                                | -                  | -                                | 290132          | -                | -                  | La alta cantidad de selecciones se debe a que el programa se bloquea por su cuenta. |
| cpubench                        | -                                     | 461                | -                                | -               | 2103             | -                  | Funcionamiento normal. |
| iobench &; cpubench             | 42,6                                  | 456,2              | -                                | 2226            | 2140             | -                  | La cantidad de selecciones del IO no se disparó debido a que cada vez que termina de leer o escribir el archivo, tiene que esperar a que cpubench termine. |
| cpubench &; cpubench            | -                                     | 558                | 556,7                            | -               | 1062             | 1063               | Funcionamiento normal. Agregar otro cpubench hace que cada uno reciba la mitad de selecciones.|
| cpubench &; cpubench &; iobench | 20,3                                  | 557,4              | 553,7                            | 1228            | 1051             | 1066               | Agregar un IO no divide a la mitad la cantidad de seleciones porque este se bloquea por su cuenta, aumentando las selecciones. |

#### Quantum Reducido

| Caso                            | Promedio OPW/100T / OPR/100T          | Promedio MFLOP100T | Promedio MFLOP100T (2° cpubench) | Cant. selec. IO | Cant. selec. CPU | Cant. Selec. CPU 2 | Descripción |
|:------------------------------- |:-------------------------------------:|:------------------:|:--------------------------------:|:---------------:|:----------------:|:------------------:|:----------- |
| iobench                         | 4667,2                                | -                  | -                                | 304136          | -                | -                  | El aumento fue leve debido a que IO pocas veces usa el quantum completo sin bloquearse, sea reducido o no. |
| cpubench                        | -                                     | 458,35             | -                                | -               | 21023            | -                  | La cantidad de selecciones es cerca de 10 veces más.|
| iobench &; cpubench             | 335                                   | 431,5              | -                                | 20902           | 21142            | -                  | Excluyendo la mayor cantidad de selecciones, no hay cambios. |
| cpubench &; cpubench            | -                                     | 553,4              | 502                              | -               | 10565            | 10646              | El promedio es levemente menor que con el quantum original, debido al tiempo que se pierde con las interrupciones. |
| cpubench &; cpubench &; iobench | 169,28                                | 539                | 500,9                            | 10495           | 10451            | 10649              | IObench tiene más promedio ya que se lo selecciona más veces, tiene que esperar menos a que los cpubench terminen.|

### Multi-Level Feedback Queue

#### Quantum original

| Caso                            | Promedio OPW/100T / OPR/100T          | Promedio MFLOP100T | Promedio MFLOP100T (2° cpubench) | Cant. selec. IO | Cant. selec. CPU | Cant. Selec. CPU 2 | Descripción |
|:------------------------------- |:-------------------------------------:|:------------------:|:--------------------------------:|:---------------:|:----------------:|:------------------:|:----------- |
| iobench                         | 4313,8                                | -                  | -                                | 269207          | -                | -                  | Funcionamiento parecido a RR. El menor promedio se puede deber a cambios externos en el sistema. |
| cpubench                        | -                                     | 459,69             | -                                | -               | 2126             | -                  | Funcionamiento normal. |
| iobench &; cpubench             | 45,25                                 | 467,7              | -                                | 2234            | 2127             | -                  | Como solo existen dos procesos, estos no tienen que pelear por prioridad: uno corre, el otro duerme. Resultados parecidos a RR.|
| cpubench &; cpubench            | -                                     | 570,57             | 573,19                           | -               | 1062             | 1068               | Mayor promedio se puede deber a factores externos.|
| cpubench &; cpubench &; iobench | 36,87                                 | 558,18             | 555,13                           | 2220            | 1065             | 1074               | IObench duplica la cantidad de selecciones, ya que cualquier CPUbench termine su tiempo, IObench siempre es elegido. IObench tiene cantidad de selecciones parecidas al caso IO & CPU, y ambos CPUbench tienen números parecidos al caso CPU & CPU.|

#### Quantum Reducido

| Caso                            | Promedio OPW/100T / OPR/100T          | Promedio MFLOP100T | Promedio MFLOP100T (2° cpubench) | Cant. selec. IO | Cant. selec. CPU | Cant. Selec. CPU 2 | Descripción |
|:------------------------------- |:-------------------------------------:|:------------------:|:--------------------------------:|:---------------:|:----------------:|:------------------:|:----------- |
| iobench                         | 4365,5                                | -                  | -                                | 284569          | -                | -                  | Funcionamiento parecido a RR. |
| cpubench                        | -                                     | 466,23             | -                                | -               | 21257            | -                  | Funcionamiento parecido a RR. |
| iobench &; cpubench             | 334,67                                | 432,58             | -                                | 20966           | 20949            | -                  | Funcionamiento parecido a RR. |
| cpubench &; cpubench            | -                                     | 528,13             | 528,79                           | -               | 10642            | 10661              | Funcionamiento parecido a RR. |
| cpubench &; cpubench &; iobench | 190,5                                 | 531,2              | 530,67                           | 12351           | 10466            | 10529              | IO está favorecido, pero en menor porcentaje. El incremento de selecciones es solo de 20%. Esto podría indicar que la prioridad de MLFQ hacia las IO decrece cuando decrece el quantum. |

## Cuarta Parte
3. **Para análisis responda: ¿Se puede producir starvation en el nuevo planificador? Justifique su respuesta.**

Con este nuevo planificador SI puede ocurrir starvation. 
Cuando un proceso se bloquea asciende de prioridad. Si tenemos multiples proceso I/O bound que pasan al estado SLEEPING pero por muy poco tiempo, 
con el unico objetivo de bloquearse para ceder la CPU antes del quantum, monopolizaran la CPU. 
Estaran siempre en la prioridad mas alta donde luego seran planificados via RR. Procesos CPU bound que estaran en las prioridades mas bajas no tendran acceso suficiente a la CPU para ejecutarse.

## Test Manuales
A la hora de realizar las mediciones se debieron comprobar los siguientes test:

* **En un workload/escenario de cpubench todos los procesos deberían recibir la misma cantidad de quantums y tener el mismo número de MFLOPs totales. Esto debería ser cierto para RR y MLFQ**

Esto se verifica en las mediciones tanto para RR como para MLFQ.

-En el planificador RR es claro que a todos los procesos se les asigna un mismo quamtum en el cual pueden usar la CPU. Cuando se les acaba, el planificador 
 recorre el arreglo de procesos buscando al proximo a ejecutar, de forma que todos los procesos tienen el tiempo de computo.
 
-En el planificador MLFQ esto tambien se cumple ya que al ser procesos CPU bound, agotaran su quantum, llamaran a yield y descenderán de prioridad. 
 Luego todos los cpubench estarán en la prioridad mas baja y se planificarán via RR

* **En un escenario con dos(2) cpubench y un(1) IObench: MLFQ debería favorecer al proceso que hace IO en comparación con RR. O sea se debería observar más cambios de contexto y más IOPs totales**

Esto se observa en nuestras mediciones. El planificador MLFQ implementado beneficia a procesos I/O bound ya que al ocurrir un bloque de proceso (via sleep()) el proceso asciende de prioridad.
Por lo tanto los proceso IObench estaran siempre en las prioridades mas altas y cuando estos pasen de SPLEEPING a RUNNABLE seran los primeros en planificarse. Por lo que tendrán más cambios de contexto 
y más IOPs totales que los cpubench

* **Con RR, un quantum menor debería favorecer a los procesos que hacen IO (en comparación al mismo workload/escenario con quantum mas grande)**

En el caso de IObench junto a otros CPUbench esto puede observarse en nuestras metricas. Mientras que el promedio de op. en los cpubench se mantiene parecido, en los IO aumenta bastante. Esto creemos 
que se debe a que al ser el quantum mas chico, los CPUbench cederán la cpu antes, por lo que habra mas cambios de contexto. Al haber mas cambios de contextos el IObench se planifica mas veces y relizas mas
operaciones durante 100 TICKS, por lo que su promedio de op aumentara.

En el caso de solo IObench el test no se observa tan claro en nuestras mediciones. Si bien hay un pequeño aumento en la cantidad de selecciones al programa, el promedio de operaciones es el mismo. 
Podemos observar en el quantum original que la cantidad de selecciones ya es muy grande. Esto se debe a que el programa se bloquea rapidamente antes de acabar su quantum.
Luego al disminiur su quantum, creemos que el aumento fue leve por esta misma razon: Debido a que IObench pocas veces usa el quantum completo ya que se bloquea antes. Con quantum reducido o no la 
cantidad de cambios de contexto sera parecida. 

* **En escenarios de todos cpubench: al achicar el quantum el output total de trabajo debería ser menor o igual.**

Esto puede verse en nuestra mediciones.  Al tener un quantum mas chico, se realizan mas cambios de contexto. Luego el promedio es levemente menor que con el quantum original, debido al tiempo que se 
pierde con las interrupciones (Saltar de userspace al kernell via trap; llamar a yield; volver a la funcion scheduler; elegir el proximos proceso llamando a swtch y saltar de nuevo a userspace).

* **Correr exactamente el mismo escenario 5 veces debería dar números muy similares**

Comprobamos este test corriendo el Caso 3: "1 iobench con 1 cpubench", tanto en RR como MLFQ:

**RR**

| Intentos     | Promedio OPW/100T / OPR/100T   | Promedio MFLOP100T | Cant. selec. IO | Cant. selec. CPU | 
|:------------:|:------------------------------:|:------------------:|:---------------:|:----------------:|
| 1            | 42,6                           | 467,7              | 2226            | 2103             | 
| 2            | 42,7                           | 483.1              | 2234            | 2106             |  
| 3            | 42,7                           | 487,2              | 2226            | 2116             | 
| 4            | 42,7                           | 483.4              | 2225            | 2106             |
| 5            | 42,6                           | 474,1              | 2226            | 2117             | 


**MLFQ**

| Intentos     | Promedio OPW/100T / OPR/100T   | Promedio MFLOP100T | Cant. selec. IO | Cant. selec. CPU | 
|:------------:|:------------------------------:|:------------------:|:---------------:|:----------------:|
| 1            | 45,25                          | 456,2              | 2234            | 2127             | 
| 2            | 42,6                           | 487.2              | 2234            | 2106             |  
| 3            | 42,6                           | 478,6              | 2233            | 2130             | 
| 4            | 42,7                           | 483.4              | 2225            | 2106             |
| 5            | 42,7                           | 467,7              | 2226            | 2116             | 


## CONCLUSIÓN (RR VS MLFQ)

Al comparar las veces en las que los procesos fueron elegidos con los schedulers Round Robin y MLFQ, llegamos a la conclusión de que si bien los procesos cpubench
no cambian tanto entre los distintos schedulers, los procesos iobench, cuando corren al mismo tiempo que los cpubenchs, tiene un mejor desempeño con la implementación de MLFQ.
Esto se debe a que en MLFQ los procesos iobench tienen mayor prioridad debido a que se bloquean más seguido, haciendo que sean elegidos por el scheduler una mayor cantidad de veces.
Los cpubench, en cambio, usan su quantum completo haciendo cálculos y cuando son interrumpidos debido al timer su prioridad decrece.

Concuimos que el scheduler MLFQ tiene un mejor rendimiento para procesos "cortos" I/O bound y beneficia la interactividad, a la vez que realiza progresos en los procesos largos que 
requieren mucho computo, eligiendolos y asignandoles un quantum de manera "fair" (via RR). No obstante, este implimentacion no esta exenta de problemas de "starvation" o "engaños al 
scheduler" al no tener implementados un "priority boost" y otras caracteristicas.
+++
title = "El creador de horarios"
date = 2021-02-23
draft = true
+++

Hace dos años, me dí a la tarea de resolver uno de los problemas que asedian a
todo estudiante cada inicio de semestre: crear un horario que incluya todas las
clases que necesito tomar en el semestre, con una distribución de horas de descanso 
razonable y sobrevivir en el intento. No sabía
exactamente cómo iba a resolver el problema, pero sabía que tenía la capacidad
de intentarlo, y por el momento eso me bastó.

El problema es sencillo de entender:  

Supongamos la universidad provee de una lista de materias ofertadas durante el
semestre:

* Español
* Inglés
* Historia
* Matemáticas
* Computación 

Además, provee de una lista de profesores que imparten la asignatura junto
con el horario en el que la imparten de la siguiente manera:

| Materia     | Profesor(a)   | Horario                               |
| ----------- | ------------- | ------------------------------------- |
| Español     | Isabel A.     | lunes y miércoles de 8:00 a 10:00     |
| Español     | Gabriel G.    | martes y jueves de 7:00 a 9:00        |
| Inglés      | William S.    | martes y viernes de 7:00 a 9:00       |
| Inglés      | Mary S.       | lunes y viernes de 7:00 a 9:00        |
| Historia    | Bernal D.     | jueves 9:00 a 13:00                   |
| Matemáticas | Isaac N.      | martes y jueves de 17:00 a 19:00      |
| Matemáticas | Alberto E.    | miércoles y viernes de 15:00 a 17:00  |
| Computación | Alan T.       | martes y jueves de 17:00 a 19:00      |
| Computación | Grace H.      | lunes y miércoles de 17:00 a 19:00    |

A partir de esta información pueden ocurrir los siguientes escenarios:

## Conozco exactamente qué materias y grupos deseo inscribir

En este caso, la función del programa es muy sencilla. Su única labor es
proveer de una representación visual del horario y confirmar que no existe
ningun empalme entre los grupos.


## Conozco las materias que deseo cursar pero desconozco los horarios que se pueden formar con los grupos disponibles.

En este caso, el programa generará todas las combinaciones posibles (y
válidas) de los grupos existentes para las materias seleccionadas.

## Hay un número fijo de materias que me gustaría cursar este semestre de entre un número mayor de materias.

En este caso, supongamos que quiero inscribir tres materias este semestre.
Estas tres materias pueden ser cualquier combinación válida de los
siguientes candidatos:

* Español
* Inglés
* Matemáticas
* Computación

Por lo que puede ser cualquier configuración de horario válida para las siguientes
combinaciones de materias: 

1. Español, Inglés, Matemáticas
2. Español, Inglés, Computación
3. Español, Matemáticas, Computación
4. Inglés, Matemáticas, Computación



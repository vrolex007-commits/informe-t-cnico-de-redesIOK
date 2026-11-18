# ECMP y su comportamiento al alterar la topología

## Respuesta a la consigna sobre Equal-Cost Multi-Path

**Institución:** [Completar]
**Materia:** Redes
**Alumno:** [Completar]
**Profesor:** [Completar]
**Fecha:** [Completar]

---

## 1. Introducción

La consigna plantea tres cosas que están relacionadas entre sí: qué es el ECMP, cómo se manifiesta en la topología propuesta (con R1 llegando a la red de R4 por dos caminos independientes), y qué le pasa al balanceo de carga cuando se modifica la red agregando un router intermedio en uno de los vínculos. En este trabajo se responde cada uno de esos puntos, primero explicando las condiciones que hacen posible el ECMP y después analizando qué cambia cuando la simetría de la topología se rompe, tanto con rutas estáticas como con el protocolo RIPv2.

---

## 2. ¿Qué es el ECMP?

ECMP (Equal-Cost Multi-Path, o Caminos Múltiples de Costo Igual) es una estrategia de reenvío que le permite a un router usar dos o más rutas paralelas hacia un mismo destino al mismo tiempo, repartiendo el tráfico entre ellas.

La idea detrás del ECMP es simple: si una red tiene dos caminos igual de buenos para llegar a un destino, no tiene sentido usar solo uno y dejar el otro quieto esperando una falla. Con ECMP, los dos enlaces se aprovechan a la vez, se gana ancho de banda útil y, de paso, se obtiene redundancia: si un camino se corta, el otro sigue disponible.

Un punto que conviene tener claro es que el reparto del tráfico no significa que el router mande un paquete por un camino y el siguiente por el otro de manera alternada. El mecanismo exacto depende del dispositivo y de su configuración (en los routers de Cisco lo habitual es el balanceo por destino, donde cada flujo viaja siempre por el mismo camino y los distintos flujos se distribuyen entre los caminos disponibles). Lo que sí se puede verificar siempre en la tabla de enrutamiento es la condición previa e indispensable: que convivan dos rutas hacia el mismo destino con el mismo costo.

---

## 3. Cómo se manifiesta ECMP en la topología propuesta

En el escenario práctico de la consigna, el ECMP aparece de forma natural. El router de origen, R1, tiene dos caminos independientes para llegar a la red de destino que está detrás de R4:

- **Camino superior:** R1 → R2 → R4
- **Camino inferior:** R1 → R3 → R4

Como los dos caminos tienen exactamente el mismo costo —en este caso, la misma cantidad de saltos—, R1 determina que ambos son igual de buenos y en lugar de descartar uno y dejarlo como simple respaldo, reparte el flujo de datos entre los dos enlaces. De esa forma aprovecha el ancho de banda disponible de ambas ramas de la red simultáneamente.

En la tabla de enrutamiento de R1 esto se ve como dos rutas hacia la red de destino: una con próximo salto hacia R2 y otra con próximo salto hacia R3, ambas con la misma distancia administrativa y la misma métrica.

> **[Espacio para captura 1]** — Topología en Cisco Packet Tracer con los dos caminos R1 → R2 → R4 y R1 → R3 → R4.

> **[Espacio para captura 2]** — Tabla de enrutamiento de R1 (`show ip route`) mostrando las dos rutas hacia la red de destino.

---

## 4. Las dos condiciones para que funcione el ECMP

Para que el ECMP funcione de forma natural y efectiva, las rutas deben cumplir estrictamente con dos condiciones:

1. **Tener la misma Distancia Administrativa (DA).** La DA es un valor que indica la confiabilidad de la fuente de una ruta (0 para una red conectada, 1 para una ruta estática, 110 para OSPF, 120 para RIP). Si dos rutas hacia el mismo destino provienen de fuentes con distinta DA, el router elige la de menor valor y no hay balanceo posible, porque una ruta se considera más confiable que la otra.
2. **Tener la misma métrica.** La métrica es el costo numérico del camino, calculado por el protocolo o definido por el administrador. Solo cuando los costos finales coinciden el router considera a los caminos equivalentes e instala los dos.

En otras palabras: primero hay que empatar en confiabilidad de la fuente, y recién ahí se comparan los costos. Si ambos empates se dan, hay ECMP. Si falta uno de los dos, el router se queda con un solo camino.

---

## 5. ¿Qué ocurre con el ECMP al alterar la topología?

### 5.1. La modificación: se agrega R5

La alteración propuesta consiste en interponer un router intermedio nuevo, R5, en uno de los vínculos. Tomando el ejemplo de la consigna, R5 se coloca entre R1 y R2, y el camino superior queda así:

- **Camino superior:** R1 → R5 → R2 → R4
- **Camino inferior:** R1 → R3 → R4 (sin cambios)

Al agregar R5, la simetría de la red se rompe. El camino modificado ahora requiere un salto más, o dicho en términos de RIP, tiene un costo mayor: el superior queda en 3 saltos y el inferior se mantiene en 2. Los dos caminos dejaron de ser iguales, y como vimos en la sección anterior, sin igualdad de costos no hay ECMP que valga.

> **[Espacio para captura 3]** — Topología modificada en Cisco Packet Tracer, con R5 intercalado entre R1 y R2.

### 5.2. Comportamiento con RIPv2

Con un protocolo dinámico como RIP, la respuesta es automática. RIP calcula su métrica por conteo de saltos, así que al converger descubre que el camino superior está a 3 saltos y el inferior a 2. Como el protocolo siempre elige el camino de menor costo, en la tabla de R1 termina instalada una sola ruta hacia la red de destino: la del camino inferior, con `[120/2]` (120 es la distancia administrativa de RIP y 2 la cantidad de saltos).

El ECMP, entonces, **se desactiva automáticamente**: dejó de haber dos rutas con igual costo que balancear. Pero el camino largo no se pierde. RIP sigue recibiendo los anuncios de esa red por el lado de R5 y R2, simplemente los descarta por ser menos convenientes. Así, el camino superior queda convertido en un **respaldo pasivo** (failover): si el camino principal falla —por ejemplo, si se cae el enlace R1-R3 o R3-R4—, RIP recalcula e instala la ruta por el camino superior, y la comunicación entre las dos extremidades de la red se mantiene. El costo de esa recuperación es el tiempo que tarda el protocolo en converger, que no es instantáneo porque RIP trabaja con actualizaciones y temporizadores periódicos.

> **[Espacio para captura 4]** — Tabla de enrutamiento de R1 (`show ip route`) luego de la convergencia de RIP con la topología modificada.

Completar con la transcripción real:

```
[Completar: transcribir aquí la tabla de enrutamiento de R1]
```

### 5.3. Comportamiento con rutas estáticas (aclaración)

La consigna resume el caso de las rutas estáticas diciendo que "pierden la capacidad de balancear el tráfico de forma equitativa". El resultado es correcto en la práctica, pero la razón es distinta a la de RIP y conviene aclararlo para no mezclar los dos mecanismos.

Una ruta estática no calcula costos: no hay protocolo midiendo saltos ni recalculando nada. Por eso, agregar R5 no modifica por sí mismo la métrica de una ruta estática (que se muestra siempre como 0). Lo que sí ocurre es que las rutas estáticas que apuntaban al segmento viejo R1-R2 quedan **inválidas**, porque su próximo salto ya no es alcanzable desde R1, y el router las saca de la tabla. En ese estado, efectivamente, el balanceo se pierde... pero porque la configuración quedó vieja, no porque el router haya comparado costos.

Para que haya balanceo de nuevo hay que reescribir las rutas a mano (una vía R5 y otra vía R3), y acá aparece la diferencia clave con RIP: si ambas rutas se cargan con igual distancia administrativa y métrica, **el ECMP vuelve a funcionar aunque los caminos tengan distinta cantidad de saltos**, porque el router hace exactamente lo que se le indica y no mide la longitud del camino. Y si en lugar de balancear se quiere priorizar un camino, basta con configurar el otro como ruta flotante (con una DA mayor) para que quede como respaldo.

En resumen: con rutas estáticas no existe un ECMP "natural" ni automático. El comportamiento final —balancear, priorizar un camino o dejar uno de reserva— lo define el administrador con la configuración, mientras que con RIP lo decide el protocolo según los costos que calcula.

---

## 6. Verificación en el simulador

Para comprobar todo lo anterior en Cisco Packet Tracer se sugieren las siguientes pruebas, dejando constancia de los resultados realmente obtenidos:

| Prueba | Qué permite verificar |
|---|---|
| `show ip route` en R1 (topología simétrica) | Presencia de dos rutas con igual DA y métrica (ECMP activo). |
| `show ip route` en R1 (con R5 y RIP) | Una sola ruta con `[120/2]` por el camino inferior; el superior ausente como ruta principal. |
| `ping` de PC_Origen a PC_Destino | Conectividad de punta a punta en cada escenario. |
| `traceroute` desde la PC de origen | Por qué camino viaja realmente el tráfico. |
| Caída simulada de un enlace (`shutdown`) | Entrada en acción del camino de respaldo y reconvergencia. |

Resultados obtenidos (completar):

```
[Completar: resultados de las pruebas realizadas en Packet Tracer]
```

---

## 7. Conclusión

El ECMP no es una función que se encienda o se apague con un comando: es la consecuencia de que dos caminos tengan el mismo costo. Mientras la topología mantiene esa igualdad —como en el escenario original, donde los dos caminos tenían 2 saltos—, el router reparte el tráfico entre los dos enlaces y aprovecha el ancho de banda de ambos. En cuanto se rompe la simetría, como al agregar R5, el balanceo desaparece: RIP deja el camino largo como respaldo y activa solo el camino corto, y con rutas estáticas el resultado depende de cómo se reconfiguren. Lo que más se aprendió trabajando este tema es que la distancia administrativa y la métrica son las dos variables que gobiernan cada decisión del router, y que tener un camino de respaldo no sirve de nada si no hay un mecanismo —manual o automático— que lo use cuando la red lo necesita.

---

## Bibliografía

1. *ECMP (Equal-Cost Multi-Path o Caminos Múltiples de Costo Igual).* Nota teórica de la cátedra (archivo de texto).
2. *RIP_v2 — Configuración y funcionamiento de RIPv2 en la topología del trabajo práctico.* Material complementario de la cátedra (PDF).
3. Cisco Systems. *IP Routing: RIP Configuration Guide* (documentación oficial de Cisco IOS). Disponible en: cisco.com.

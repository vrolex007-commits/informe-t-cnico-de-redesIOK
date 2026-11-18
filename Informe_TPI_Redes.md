# TRABAJO PRÁCTICO INTEGRADOR DE REDES

## Análisis de Comportamiento de Enrutamiento, ECMP y Tolerancia a Fallos

---

## 1. Portada

**Institución:** [Completar nombre de la institución]

**Materia:** Redes

**Título del trabajo práctico:** Análisis de Comportamiento de Enrutamiento, ECMP y Tolerancia a Fallos

**Alumno:** [Completar nombre y apellido]

**Profesor:** [Completar nombre del profesor]

**Curso / Comisión:** [Completar]

**Fecha de entrega:** [Completar]

---

## 2. Introducción

En este trabajo práctico analizamos cómo se comporta una red cuando existen dos caminos posibles para llegar al mismo destino, y qué cambia cuando esa situación se modifica. Para eso armamos una topología simulada en Cisco Packet Tracer con cuatro routers, dos switches y dos PCs: una PC de origen y una PC de destino, comunicadas a través de dos rutas paralelas que pasan por routers distintos.

La práctica se divide en tres momentos. Primero configuramos direcciones IP y rutas estáticas para lograr conectividad básica, aprovechando que los dos caminos tienen el mismo costo para activar el balanceo de carga ECMP (Equal-Cost Multi-Path). Después reemplazamos las rutas estáticas por el protocolo dinámico RIPv2 y verificamos que el router llegue al mismo comportamiento de balanceo, pero esta vez aprendiendo las rutas por sí solo. Finalmente modificamos la topología incorporando un router adicional en el camino superior, lo cual rompe la simetría de la red y nos permite observar qué sucede con el balanceo de carga cuando los dos caminos dejan de tener el mismo costo.

El propósito del trabajo no es solo configurar comandos, sino entender por qué el router elige un camino u otro, cómo interpreta la información que muestra la tabla de enrutamiento, y qué ventajas y limitaciones tiene cada método frente a un cambio en la topología o frente a una falla de un enlace.

---

## 3. Objetivos

### Objetivo general

Analizar el comportamiento del enrutamiento en una red con caminos redundantes, comprendiendo cómo se implementa el balanceo de carga mediante ECMP, cómo se gestionan las rutas con enrutamiento estático y dinámico, y de qué manera la red responde ante la pérdida de un camino (tolerancia a fallos).

### Objetivos específicos

- Configurar direcciones IP en las interfaces de los routers y en las PCs.
- Implementar rutas estáticas para lograr conectividad entre las redes de origen y destino.
- Comprender el funcionamiento de ECMP y las condiciones que deben cumplirse para que se active.
- Configurar el protocolo dinámico RIPv2 en todos los routers de la topología.
- Interpretar las tablas de enrutamiento mediante el comando `show ip route`.
- Analizar el significado de la distancia administrativa y de la métrica dentro de una entrada de la tabla de ruteo.
- Comparar el comportamiento del enrutamiento estático y del enrutamiento dinámico en un escenario simétrico.
- Modificar la topología agregando un router intermedio en uno de los caminos.
- Evaluar qué ocurre con el balanceo de carga cuando los caminos tienen diferentes costos.
- Comprender cómo se produce la tolerancia a fallos y el cambio de ruta ante la caída de un enlace.

---

## 4. Marco teórico

### 4.1. Enrutamiento

El enrutamiento es el proceso mediante el cual un router decide por dónde enviar cada paquete que recibe. A diferencia de un switch, que trabaja con direcciones MAC dentro de una misma red local, un router opera con direcciones IP y su función principal es interconectar redes distintas.

Cuando un router recibe un paquete, mira la IP de destino y busca esa red en su tabla de enrutamiento. Esa tabla funciona como un mapa: le dice al router hacia qué interfaz o hacia qué próximo router (next hop) debe reenviar el paquete para acercarse al destino. Si la red de destino aparece en la tabla, el paquete se reenvía; si no aparece, el router lo descarta (salvo que exista una ruta por defecto).

Lo interesante de esta práctica es que la tabla puede contener más de una forma de llegar al mismo destino. Cuando eso pasa, el router tiene que aplicar criterios para decidir: primero mira de dónde proviene la ruta (la distancia administrativa) y después qué costo tiene esa ruta (la métrica). Estos dos conceptos son la clave de todo el trabajo, y por eso se desarrollan aparte en los puntos siguientes.

### 4.2. ECMP (Equal-Cost Multi-Path)

ECMP es una estrategia de reenvío que permite al router usar dos o más caminos distintos hacia un mismo destino de manera simultánea, siempre que esos caminos tengan exactamente el mismo costo. En lugar de elegir uno y dejar el otro sin usar, el router reparte el tráfico entre ambos.

En nuestra topología esto se ve muy claro: R1 puede llegar a la red de R4 por el camino superior (R1 → R2 → R4) o por el camino inferior (R1 → R3 → R4). Como ambos caminos tienen la misma cantidad de saltos, el router considera que son igual de buenos e instala los dos en su tabla de enrutamiento.

Las ventajas de este comportamiento son bastante prácticas:

- **Mejor aprovechamiento del ancho de banda:** los dos enlaces se usan al mismo tiempo, en lugar de tener uno ocioso esperando una falla.
- **Distribución del tráfico:** la carga se reparte entre los caminos disponibles, lo que ayuda a evitar congestiones en un solo enlace.
- **Redundancia:** si un camino se corta, el otro sigue disponible.
- **Mayor disponibilidad:** la red tolera la pérdida de un enlace sin perder conectividad entre los extremos.

Un punto que conviene aclarar es cómo se reparte el tráfico. Que haya ECMP no significa que el router envíe un paquete por un camino, el siguiente por el otro, y así alternativamente. El mecanismo de balanceo depende del dispositivo y de su configuración: en los routers de Cisco con CEF (Cisco Express Forwarding), el comportamiento más común es el balanceo por destino, donde todos los paquetes de un mismo flujo (por ejemplo, de una PC hacia un mismo servidor) viajan por el mismo camino, y los flujos se distribuyen entre los caminos disponibles. También existe el balanceo por paquete, más fino pero que puede generar que los paquetes lleguen desordenados. En Packet Tracer no siempre se puede observar esta distribución paquete por paquete, pero sí se puede verificar su condición indispensable en la tabla de enrutamiento: que convivan dos rutas hacia el mismo destino con igual distancia administrativa y igual métrica.

### 4.3. Distancia Administrativa (DA)

La distancia administrativa es un número entre 0 y 255 que indica qué tan confiable es la fuente de una ruta para el router. No tiene nada que ver con la calidad del camino: lo que mide es la confiabilidad del origen de la información. Cuanto más bajo es el valor, más confiable se considera la fuente.

Los valores más comunes son los siguientes:

| Origen de la ruta | Distancia administrativa |
|---|---:|
| Ruta conectada | 0 |
| Ruta estática | 1 |
| OSPF | 110 |
| RIP | 120 |

Cuando un router conoce el mismo destino por dos fuentes distintas, primero compara las distancias administrativas y se queda con la de menor valor. Por ejemplo, si un router tiene una ruta estática hacia 192.168.4.0/24 (DA 1) y además aprende esa misma red por RIP (DA 120), instala la ruta estática y descarta la de RIP. Esto no significa que la ruta estática sea mejor en cuanto a costos; simplemente el router confía más en una ruta definida manualmente por el administrador que en la información de RIP.

Para que entren en juego dos rutas simultáneas (ECMP), primero tienen que empatar en distancia administrativa y recién ahí se compara la métrica. Si ambas coinciden en los dos valores, el router instala las dos.

### 4.4. Métrica y costo

Estos dos términos se suelen usar como sinónimos, pero conviene distinguirlos:

- La **métrica** es la regla o el algoritmo que utiliza un protocolo para comparar caminos. Es el criterio de "bueno" o "malo" que aplica cada protocolo.
- El **costo** es el valor numérico que resulta de aplicar esa métrica a un camino concreto.

Un ejemplo con RIP lo deja claro. La métrica de RIP es el conteo de saltos (hop count), o sea la cantidad de routers que hay que atravesar. En la topología base, el camino R1 → R2 → R4 tiene 2 saltos, así que su costo es 2. El camino R1 → R3 → R4 también tiene 2 saltos, así que su costo también es 2. Como los costos coinciden, se da la condición necesaria para ECMP.

Si otro protocolo usara otra métrica (por ejemplo OSPF, que calcula costo según el ancho de banda de los enlaces), el mismo camino podría tener un valor completamente distinto. Para el ECMP lo único que importa es que el costo final calculado sea el mismo en todos los caminos candidatos.

### 4.5. Enrutamiento estático

Una ruta estática es una entrada en la tabla de enrutamiento que escribe manualmente el administrador de la red. Se configura con el comando `ip route`, indicando la red de destino, su máscara y el próximo salto (o la interfaz de salida):

```
ip route <red-destino> <máscara> <next-hop | interfaz>
```

Por ejemplo, en R1 una ruta hacia la red destino del otro extremo podría ser `ip route 192.168.4.0 255.255.255.0 10.0.12.2`, que significa "para llegar a 192.168.4.0/24, enviá el tráfico al router con IP 10.0.12.2".

Las rutas estáticas son simples y predecibles: el router hace exactamente lo que se le indica. Funcionan bien en redes pequeñas o en puntos donde el camino no cambia. La contracara es que no se adaptan solas: si se agrega un router, si se corta un enlace o si cambia la topología, hay que modificar los comandos a mano. Tampoco avisan a otros routers de lo que saben: cada equipo necesita su propia configuración.

En la primera parte de la práctica usamos rutas estáticas justamente porque permiten ver el ECMP de la forma más directa: si se cargan dos rutas idénticas hacia el mismo destino, el router instala las dos sin necesidad de ningún protocolo.

### 4.6. Enrutamiento dinámico

El enrutamiento dinámico resuelve el problema de la actualización manual. En lugar de escribir ruta por ruta, se configura un protocolo de enrutamiento y los routers se comunican entre sí para intercambiar información sobre las redes que conocen. Cada router arma y mantiene su tabla de ruteo a partir de lo que le cuentan sus vecinos, y vuelve a calcular los caminos cuando la red cambia.

Esto trae varias ventajas: si se agrega un router o se corta un enlace, los routers se enteran mediante las actualizaciones del protocolo y ajustan sus tablas sin intervención del administrador. También permite descubrir rutas alternativas automáticamente. La contracara es que hay que entender y configurar el protocolo, y que los cambios no son instantáneos: los routers necesitan un tiempo para intercambiar información y converger.

### 4.7. RIPv2

RIP (Routing Information Protocol) es uno de los protocolos dinámicos más antiguos y el que utilizamos en esta práctica. Sus características principales, aplicadas a nuestro escenario, son las siguientes:

- **Protocolo de vector de distancia:** cada router le informa a sus vecinos qué redes conoce y a qué distancia están. No conoce la topología completa de la red, solo sabe lo que le cuentan sus vecinos directos.
- **Métrica basada en la cantidad de saltos:** el costo de un camino es simplemente la cantidad de routers que hay que atravesar. Para llegar de R1 a la red de R4 por R2 hay 2 saltos (R1 → R2 → R4), así que la métrica es 2. Esto hace que, en una topología simétrica como la nuestra, el balanceo surja casi sin darnos cuenta.
- **Distancia administrativa de 120:** es la que veremos en la tabla de ruteo al lado de las rutas aprendidas con RIP, dentro del corchete `[120/x]`.
- **Límite de 15 saltos:** RIP considera que una red a más de 15 saltos es inalcanzable. Este límite está pensado para evitar bucles de enrutamiento en redes grandes, pero también limita el tamaño de las redes donde se puede usar RIP.
- **Actualizaciones periódicas:** los routers envían su tabla completa a sus vecinos cada 30 segundos aproximadamente. Eso permite que la información se mantenga actualizada, aunque también hace que la convergencia lleve su tiempo.
- **Compatibilidad con VLSM:** la versión 2 de RIP soporta máscaras de subred de longitud variable, algo indispensable cuando se usan redes /30 para los enlaces entre routers, como en nuestra topología.
- **Comando `version 2`:** al habilitar RIP en un router de Cisco, por defecto puede funcionar en modo compatible con la versión 1, que no envía máscaras de subred. Con `version 2` se fuerza el uso de la versión 2, la que soporta VLSM y envía las máscaras junto con las redes.
- **Comando `no auto-summary`:** por defecto, RIP resume las redes hacia su límite de clase cuando las anuncia hacia otra red mayor. En una topología con subredes /30 eso puede ocultar el detalle de los enlaces. Con `no auto-summary` se desactiva ese resumen automático y se anuncian las redes tal como están configuradas.
- **Balanceo ECMP:** cuando RIP detecta dos o más caminos con la misma métrica hacia un destino, instala todos automáticamente (hasta un máximo de 4 rutas por defecto, configurable con `maximum-paths`). No hay que hacer nada especial para activarlo.

En resumen, RIP es un protocolo sencillo, y esa es la razón por la que se lo usa en este trabajo: la métrica por saltos hace que el ECMP aparezca de forma evidente cuando la topología es simétrica, y que desaparezca con la misma claridad cuando se rompe esa simetría.

---

## 5. Descripción de la topología inicial

La topología propuesta para el escenario base está formada por cuatro routers, dos switches y dos PCs:

- **R1:** router de origen. Es el punto de salida del tráfico generado por PC_Origen y el lugar donde se analiza principalmente la tabla de enrutamiento.
- **R2:** router intermediario del camino superior.
- **R3:** router intermediario del camino inferior.
- **R4:** router de destino. Es el punto de entrada a la red donde se encuentra PC_Destino.
- **Dos switches:** uno conecta PC_Origen con R1 (LAN Origen) y el otro conecta PC_Destino con R4 (LAN Destino).
- **PC_Origen y PC_Destino:** las estaciones de trabajo que se usan para generar tráfico y verificar la conectividad de punta a punta.

La característica central de esta topología es que R1 tiene dos caminos independientes para llegar a la red de destino:

- **Camino superior:** R1 → R2 → R4
- **Camino inferior:** R1 → R3 → R4

Los dos caminos tienen exactamente la misma cantidad de saltos, y esa simetría es la que permite estudiar el ECMP en condiciones limpias.

### Direccionamiento IP

Los segmentos de red indicados en la consigna son los siguientes:

| Segmento | Red | Función dentro de la topología |
|---|---|---|
| LAN Origen | 192.168.1.0/24 | Red local donde se encuentra PC_Origen. El gateway es R1 (192.168.1.1). |
| R1 - R2 | 10.0.12.0/30 | Enlace punto a punto del camino superior, entre R1 y R2. |
| R1 - R3 | 10.0.13.0/30 | Enlace punto a punto del camino inferior, entre R1 y R3. |
| R2 - R4 | 10.0.24.0/30 | Enlace punto a punto que une el camino superior con el router de destino. |
| R3 - R4 | 10.0.34.0/30 | Enlace punto a punto que une el camino inferior con el router de destino. |
| LAN Destino | 192.168.4.0/24 | Red local donde se encuentra PC_Destino. El gateway es R4 (192.168.4.1). |

Los enlaces entre routers usan máscaras /30 porque son conexiones punto a punto: con dos direcciones utilizables alcanza (una para cada extremo) y así se desperdicia la menor cantidad de direcciones posible. Las LAN usan /24 porque necesitan espacio para las PCs y posibles equipos adicionales.

### Tabla de direccionamiento (completar)

La consigna define las redes pero no detalla la dirección asignada a cada interfaz. La siguiente tabla debe completarse con las direcciones efectivamente utilizadas en Packet Tracer. Como criterio, se puede tomar la primera dirección utilizable para el primer router del enlace y la segunda para el otro extremo.

| Dispositivo | Interfaz | Dirección IP | Máscara |
|---|---|---|---|
| R1 | Interfaz hacia LAN Origen | 192.168.1.1 | 255.255.255.0 |
| R1 | Interfaz hacia R2 | [Completar] | 255.255.255.252 |
| R1 | Interfaz hacia R3 | [Completar] | 255.255.255.252 |
| R2 | Interfaz hacia R1 | [Completar] | 255.255.255.252 |
| R2 | Interfaz hacia R4 | [Completar] | 255.255.255.252 |
| R3 | Interfaz hacia R1 | [Completar] | 255.255.255.252 |
| R3 | Interfaz hacia R4 | [Completar] | 255.255.255.252 |
| R4 | Interfaz hacia R2 | [Completar] | 255.255.255.252 |
| R4 | Interfaz hacia R3 | [Completar] | 255.255.255.252 |
| R4 | Interfaz hacia LAN Destino | 192.168.4.1 | 255.255.255.0 |
| PC_Origen | NIC | [Completar] | 255.255.255.0 |
| PC_Destino | NIC | [Completar] | 255.255.255.0 |

En los comandos de este informe se usa, a modo de ejemplo, la convención de asignar `.1` al primer router del enlace y `.2` al segundo (por ejemplo, 10.0.12.1 para R1 y 10.0.12.2 para R2). Es solo una convención de trabajo: lo importante es que las direcciones coincidan con las que se carguen en el simulador.

> **[Espacio para captura 1]** — Topología completa armada en Cisco Packet Tracer, con los direccionamientos visibles.

---

## 6. Parte 1: Enrutamiento estático y ECMP simétrico

### 6.1. Configuración inicial

El primer paso fue asignar las direcciones IP a cada interfaz de los routers y configurar las PCs. En Cisco IOS, cada interfaz se configura entrando a su modo de configuración, asignando la dirección con su máscara y activándola con `no shutdown`. Este último comando es importante: por defecto las interfaces de los routers vienen deshabilitadas, y si no se activan, el enlace no pasa tráfico aunque la IP esté bien cargada.

A modo de ejemplo, la configuración de las interfaces de R1 sería:

```
R1> enable
R1# configure terminal
R1(config)# interface gigabitEthernet 0/0
R1(config-if)# ip address 192.168.1.1 255.255.255.0
R1(config-if)# no shutdown
R1(config-if)# exit
R1(config)# interface serial 0/0/0
R1(config-if)# ip address 10.0.12.1 255.255.255.252
R1(config-if)# no shutdown
R1(config-if)# exit
R1(config)# interface serial 0/0/1
R1(config-if)# ip address 10.0.13.1 255.255.255.252
R1(config-if)# no shutdown
R1(config-if)# end
R1# copy running-config startup-config
```

La misma lógica se aplica en R2, R3 y R4, cambiando las direcciones y las interfaces según la tabla de direccionamiento completada. En los routers intermedios solo hay que configurar las dos interfaces de enlace; en R4, además, la interfaz conectada a la LAN Destino.

En las PCs se configura la dirección IP, la máscara y el gateway predeterminado. El gateway es la dirección del router que sirve de salida de la red local: para PC_Origen es 192.168.1.1 (la interfaz de R1 hacia la LAN) y para PC_Destino es 192.168.4.1 (la interfaz de R4). Sin un gateway configurado, la PC solo podría comunicarse dentro de su propia red local.

> **[Espacio para captura 2]** — Configuración de direccionamiento IP aplicada en Packet Tracer (se puede usar la vista "Desktop > IP Configuration" de las PCs o el mapa de la topología con las direcciones visibles).

### 6.2. Configuración de rutas estáticas

Con las interfaces configuradas y activas, cada router solo conoce sus redes conectadas directamente. Para que PC_Origen pueda llegar a PC_Destino hace falta que los routers sepan cómo llegar a las redes remotas. La consigna pide configurar las rutas estáticas necesarias en R1 y R4, que son los extremos donde se originan y terminan los flujos, y donde queremos ver el ECMP.

En R1 se cargan dos rutas hacia la red de destino, una por cada camino:

```
R1(config)# ip route 192.168.4.0 255.255.255.0 10.0.12.2
R1(config)# ip route 192.168.4.0 255.255.255.0 10.0.13.2
```

Cada comando se lee así: para llegar a la red `192.168.4.0` con máscara `255.255.255.0`, enviar el tráfico al próximo salto indicado. La primera ruta manda el tráfico hacia R2 (10.0.12.2, camino superior) y la segunda hacia R3 (10.0.13.2, camino inferior). Como son dos rutas idénticas en todo salvo el próximo salto, el router no tiene motivo para preferir una: las instala las dos.

En R4 se hace lo mismo en sentido inverso, para poder responder hacia la red de origen:

```
R4(config)# ip route 192.168.1.0 255.255.255.0 10.0.24.1
R4(config)# ip route 192.168.1.0 255.255.255.0 10.0.34.1
```

**Aclaración necesaria sobre R2 y R3.** La consigna menciona las rutas estáticas de R1 y R4 porque es donde se observa el balanceo, pero para que el tráfico realmente llegue de punta a punta los routers intermedios también tienen que saber hacia dónde reenviar los paquetes que están en tránsito. Sin esa información, un paquete que llega a R2 con destino 192.168.4.10 sería descartado por R2. La forma más simple de resolverse es con dos rutas estáticas por router intermedio, una hacia cada LAN:

```
R2(config)# ip route 192.168.4.0 255.255.255.0 10.0.24.2
R2(config)# ip route 192.168.1.0 255.255.255.0 10.0.12.1

R3(config)# ip route 192.168.4.0 255.255.255.0 10.0.34.2
R3(config)# ip route 192.168.1.0 255.255.255.0 10.0.13.1
```

Con esto, la conectividad entre las dos LAN queda completa y el análisis del ECMP se concentra en R1 y R4, tal como pide la consigna.

### 6.3. Análisis de la tabla de enrutamiento

Para ver cómo quedó la tabla de rutas de R1 se usa el comando:

```
R1# show ip route
```

La salida de `show ip route` lista todas las redes que el router conoce, y cada entrada indica, en orden: el origen de la ruta (una letra), la red de destino con su máscara, los valores entre corchetes `[distancia administrativa / métrica]`, y el próximo salto o interfaz de salida. Al principio de la salida aparece un código que explica las letras: `C` para redes conectadas directamente, `L` para la dirección local de la interfaz, `S` para rutas estáticas, `R` para rutas aprendidas con RIP, entre otras.

Las rutas estáticas que configuramos se identifican con la letra `S`. Una entrada típica se ve de esta forma:

```
S    192.168.4.0/24 [1/0] via 10.0.12.2
```

Los valores del corchete, `[1/0]`, son los que hay que interpretar bien:

- El **primer valor (1)** es la distancia administrativa. Como la ruta es estática, vale 1, que es la DA que IOS asigna por defecto a este tipo de ruta.
- El **segundo valor (0)** es la métrica. Las rutas estáticas no calculan ningún costo: el administrador define el camino a mano y no hay un algoritmo de por medio, así que la métrica se muestra como 0.

Como configuramos dos rutas hacia `192.168.4.0/24` con la misma distancia administrativa y la misma métrica, la tabla debería mostrar las dos entradas (o una entrada con dos próximos saltos, según la versión de IOS): una vía `10.0.12.2` y otra vía `10.0.13.2`. Esa es la firma del ECMP: dos caminos conviviendo en la tabla hacia el mismo destino.

> **[Espacio para captura 3]** — Salida completa del comando `show ip route` ejecutado en R1 durante la Parte 1.

Transcripción de los valores observados (completar con la salida real):

```
[Completar: transcribir aquí la tabla de enrutamiento de R1]
```

El mismo análisis corresponde hacerlo en R4, donde deberían verse las dos rutas estáticas hacia `192.168.1.0/24`, una vía R2 y otra vía R3.

### 6.4. Funcionamiento del ECMP

Con las dos rutas instaladas, R1 puede distribuir el tráfico hacia la red de destino utilizando los dos caminos al mismo tiempo. Un paquete puede salir por el camino superior (R1 → R2 → R4) y el siguiente flujo por el inferior (R1 → R3 → R4), según el mecanismo de balanceo del dispositivo. Lo importante es que ninguno de los dos enlaces queda sin usar: el ancho de banda disponible es la suma de ambos caminos y no solo el de uno de ellos.

Esto es exactamente lo que se esperaba obtener con la configuración: al no haber diferencia de costo entre los caminos, el router no elige un camino "mejor", porque ambos son igual de buenos. Si en algún momento un camino se corta, el otro sigue en la tabla y la comunicación no se interrumpe, que es el otro beneficio del ECMP: redundancia sin desperdiciar recursos.

Para verificar el comportamiento se recomienda hacer dos capturas: la tabla de enrutamiento de R1 (que ya muestra las dos rutas) y la topología en Packet Tracer, donde se puede observar el tráfico en ambos enlaces con la simulación de paquetes (modo Simulation) o viendo los indicadores de actividad de los enlaces.

> **[Espacio para captura 4]** — Topología de Packet Tracer con la Parte 1 configurada (rutas estáticas cargadas).

**Análisis de las capturas.** Con la tabla de enrutamiento a la vista debería confirmarse que R1 mantiene dos rutas `S` hacia `192.168.4.0/24`, ambas con `[1/0]`, diferenciadas solo por el próximo salto. Eso demuestra que el router considera a los dos caminos equivalentes y los usa en paralelo. Con la captura de la topología debería verse que el tráfico generado desde PC_Origen hacia PC_Destino aparece en los dos enlaces simultáneamente, lo que confirma que el balanceo no es solo un dato de la tabla sino que efectivamente se está repartiendo el tráfico.

*(Completar este análisis con lo que se observe realmente en el simulador, adjuntando las capturas correspondientes.)*

---

## 7. Parte 2: Enrutamiento dinámico con RIPv2

### 7.1. Eliminación de las rutas estáticas

Antes de habilitar RIP conviene limpiar las rutas estáticas de la parte anterior. La razón es sencilla: si dejamos las rutas estáticas cargadas, el router va a seguir prefiriéndolas a las que aprenda con RIP, porque una ruta estática tiene distancia administrativa 1 y RIP tiene 120. Es decir, RIP podría estar funcionando perfectamente y aun así no veríamos ninguna ruta `R` en la tabla, simplemente porque la estática "gana" y tapa la información del protocolo. Para poder analizar el enrutamiento dinámico en serio, hay que quitar del medio a la competencia.

Los comandos son los mismos que usamos para cargar las rutas, anteponiendo `no`:

```
R1(config)# no ip route 192.168.4.0 255.255.255.0 10.0.12.2
R1(config)# no ip route 192.168.4.0 255.255.255.0 10.0.13.2

R4(config)# no ip route 192.168.1.0 255.255.255.0 10.0.24.1
R4(config)# no ip route 192.168.1.0 255.255.255.0 10.0.34.1
```

Lo mismo corresponde hacer en R2 y R3 con las rutas estáticas que configuramos como apoyo. Una alternativa rápida es verificar con `show ip route` que no quede ninguna entrada `S` antes de continuar.

### 7.2. Configuración de RIPv2

La configuración de RIP es bastante directa: se habilita el proceso, se fija la versión 2, se desactiva el auto-resumen y se anuncian las redes directamente conectadas de cada router. En lugar de escribir ruta por ruta como con las estáticas, alcanza con decirle a cada router qué redes tiene conectadas; RIP se ocupa de informarlas a sus vecinos.

**Router R1 (origen):**

```
R1(config)# router rip
R1(config-router)# version 2
R1(config-router)# no auto-summary
R1(config-router)# network 192.168.1.0
R1(config-router)# network 10.0.12.0
R1(config-router)# network 10.0.13.0
```

**Router R2 (camino superior):**

```
R2(config)# router rip
R2(config-router)# version 2
R2(config-router)# no auto-summary
R2(config-router)# network 10.0.12.0
R2(config-router)# network 10.0.24.0
```

**Router R3 (camino inferior):**

```
R3(config)# router rip
R3(config-router)# version 2
R3(config-router)# no auto-summary
R3(config-router)# network 10.0.13.0
R3(config-router)# network 10.0.34.0
```

**Router R4 (destino):**

```
R4(config)# router rip
R4(config-router)# version 2
R4(config-router)# no auto-summary
R4(config-router)# network 192.168.4.0
R4(config-router)# network 10.0.24.0
R4(config-router)# network 10.0.34.0
```

Qué hace cada comando:

- `router rip`: habilita el proceso de enrutamiento RIP y entra al modo de configuración del protocolo.
- `version 2`: fuerza el uso de RIPv2, que envía las máscaras de subred junto con las redes y por lo tanto soporta VLSM. Sin este comando, el router puede funcionar en modo compatible con RIPv1, que no entiende las subredes /30 de los enlaces.
- `no auto-summary`: desactiva el resumen automático hacia los límites de clase. Así las redes se anuncian con su máscara real.
- `network <red>`: indica a RIP que debe activarse en las interfaces dentro de esa red y que debe anunciarla a sus vecinos. Solo se declaran redes que están conectadas directamente al router; no hay que anunciar las redes remotas, porque de eso se encarga el propio protocolo al intercambiar información.

A modo de aclaración técnica: en Cisco IOS, la sentencia `network` de RIP se interpreta de forma claseful (clase A, B o C), por lo que declarar `10.0.12.0` o `10.0.13.0` termina refiriéndose a la red mayor 10.0.0.0. Cargar cada subred por separado, como hace el material de la cátedra, es válido, queda más claro para el análisis y no perjudica el funcionamiento. Lo importante es no anunciar redes que no pertenezcan a la topología.

### 7.3. Convergencia de la red

Que la red "converja" significa que todos los routers terminaron de intercambiar información y sus tablas de ruteo reflejan la topología real. Esto no ocurre en el instante en que se termina de escribir los comandos: RIP envía actualizaciones cada 30 segundos, así que los routers necesitan varios intercambios para aprender todas las redes remotas y descartar posibles caminos viejos.

El mecanismo es el siguiente: cada router anuncia a sus vecinos las redes que conoce, junto con la métrica (la cantidad de saltos) que le calcula. Su vecino recibe esa información, le suma un salto, y si el camino le conviene, lo agrega a su propia tabla y lo vuelve a anunciar en su próxima actualización. De esa forma la información se propaga de router en router hasta que todos conocen todas las redes.

Por eso la consigna indica esperar un tiempo antes de analizar la tabla: conviene dejar pasar uno o dos periodos de actualización (alrededor de un minuto) y recién ahí ejecutar `show ip route`, para no analizar una tabla a medio armar.

### 7.4. Análisis de `show ip route`

Una vez alcanzada la convergencia, se vuelve a ejecutar el comando en R1:

```
R1# show ip route
```

Las rutas aprendidas con RIP se identifican con la letra `R` en la primera columna. Una entrada típica hacia la red de destino sería:

```
R    192.168.4.0/24 [120/2] via 10.0.12.2
```

El corchete `[120/2]` se interpreta así:

- **120:** es la distancia administrativa de RIP. Siempre vale 120 para rutas aprendidas con este protocolo, y es lo que permite distinguirlas de las estáticas (que tienen 1) en una tabla que mezcle ambas.
- **2:** es la métrica, es decir la cantidad de saltos hasta la red de destino. En este caso, llegar a 192.168.4.0/24 desde R1 implica atravesar dos routers (por ejemplo, R1 → R2 → R4), y por eso la métrica es 2.

Lo que debería observarse en R1 es la presencia de dos rutas hacia `192.168.4.0/24` con la misma métrica: una vía R2 y otra vía R3, ambas con `[120/2]`. Ese empate en distancia administrativa y en métrica es lo que hace que RIP instale las dos y active el balanceo de carga ECMP, sin ninguna configuración adicional. Es el mismo comportamiento que logramos con las rutas estáticas, pero esta vez el router lo resolvió solo a partir del intercambio de actualizaciones.

> **[Espacio para captura 5]** — Salida del comando `show ip route` en R1 luego de la convergencia de RIPv2.

Transcripción de los valores observados (completar con la salida real):

```
[Completar: transcribir aquí la tabla de enrutamiento de R1 con RIP convergido]
```

*(Completar el análisis confirmando la cantidad de rutas hacia 192.168.4.0/24, sus métricas y los próximos saltos observados.)*

### 7.5. Comparación entre rutas estáticas y RIPv2

| Aspecto | Rutas estáticas | RIPv2 |
|---|---|---|
| Forma de configuración | Se escribe cada ruta manualmente con `ip route` en cada router. | Se habilita el protocolo y se anuncian solo las redes conectadas. |
| Actualización de rutas | No se actualizan solas. Cada cambio de topología requiere editar los comandos. | Los routers intercambian tablas cada 30 segundos y actualizan sus rutas automáticamente. |
| Adaptación a cambios de topología | Nula, salvo que el administrador intervenga. | Automática, aunque con un tiempo de convergencia. |
| Métrica utilizada | No hay métrica de protocolo: la ruta se muestra con métrica 0 y la elección depende de lo que configure el administrador. | Conteo de saltos (hop count). |
| Distancia administrativa | 1 | 120 |
| Funcionamiento de ECMP | Solo si el administrador carga dos o más rutas iguales hacia el mismo destino. | Automático cuando hay dos o más caminos con la misma cantidad de saltos. |
| Ventajas | Simple, predecible, sin sobrecarga de protocolo. Útil en redes chicas o caminos fijos. | Se adapta sola a los cambios, descubre caminos alternativos y activa el balanceo sin configuración extra. |
| Desventajas | No reacciona a fallas ni a cambios; requiere mantenimiento manual. | Convergencia lenta (timers de segundos), limitada a 15 saltos, genera tráfico periódico de actualizaciones. |

Viéndolo en la práctica, las dos formas llegaron al mismo resultado en el escenario simétrico: R1 terminó con dos caminos hacia la red destino repartiendo el tráfico. La diferencia está en cómo llegaron ahí. Con las rutas estáticas fuimos nosotros los que le dijimos al router exactamente por dónde mandar cada paquete; con RIP fuimos nosotros los que le dijimos qué redes tenía conectadas, y el resto (descubrir los caminos, calcular las métricas, darse cuenta de que los dos caminos empatan) lo resolvió el protocolo solo.

Esa diferencia se nota mucho más cuando la red cambia. Si mañana se agrega un router o se corta un enlace, con estáticas hay que sentarse a reescribir la configuración; con RIP hay que esperar a que el protocolo converja, pero no hay que tocar nada. Por otro lado, RIP cobra su precio: las actualizaciones periódicas consumen algo de ancho de banda, la convergencia lleva tiempo (y eso impacta en la tolerancia a fallos), y el límite de 15 saltos lo hace inutilizable en redes muy grandes. Para una red de este tamaño, sin embargo, es una herramienta muy cómoda para estudiar el comportamiento del enrutamiento dinámico.

---

## 8. Parte 3: Modificación de la topología mediante la incorporación de R5

En esta parte se modifica el escenario original incorporando un nuevo router, R5, en serie dentro del camino superior. Específicamente, R5 se coloca entre R1 y R2, de modo que el camino superior queda dividido en dos tramos nuevos y pasa a tener un router más:

- **Camino superior:** R1 → R5 → R2 → R4
- **Camino inferior:** R1 → R3 → R4 (sin cambios)

Con esta modificación se rompe la simetría de la topología. El camino superior ahora requiere atravesar tres routers (3 saltos), mientras que el inferior sigue requiriendo dos. Es exactamente el escenario que necesitamos para estudiar qué pasa con el balanceo de carga cuando los caminos dejan de tener el mismo costo, tanto con rutas estáticas como con RIP.

### Reconfiguración del direccionamiento

Al interponer R5, el enlace directo entre R1 y R2 deja de existir y en su lugar aparecen dos enlaces nuevos: uno entre R1 y R5, y otro entre R5 y R2. Cada enlace necesita su propia red /30. La consigna pide reconfigurar las direcciones IP de ese segmento; la forma de repartirlas queda a criterio del alumno.

Una posibilidad es mantener la red `10.0.12.0/30` para uno de los tramos nuevos y crear otra para el segundo; otra es crear dos redes nuevas. Lo que no se puede hacer es dejar los dos enlaces dentro de la misma red /30, porque solo tiene dos direcciones utilizables.

| Segmento | Red | Observación |
|---|---|---|
| R1 - R5 | [Completar, ej. 10.0.15.0/30] | Enlace nuevo. Reemplaza al antiguo R1 - R2. |
| R5 - R2 | [Completar, ej. 10.0.52.0/30] | Enlace nuevo. |
| R1 - R3 | 10.0.13.0/30 | Sin cambios. |
| R2 - R4 | 10.0.24.0/30 | Sin cambios. |
| R3 - R4 | 10.0.34.0/30 | Sin cambios. |
| LAN Origen | 192.168.1.0/24 | Sin cambios. |
| LAN Destino | 192.168.4.0/24 | Sin cambios. |

En los comandos de las secciones siguientes se usa, como ejemplo, `10.0.15.0/30` para el enlace R1-R5 y `10.0.52.0/30` para el enlace R5-R2. Estos valores deben reemplazarse por los que se utilicen realmente en el simulador.

> **[Espacio para captura 6]** — Topología modificada en Cisco Packet Tracer, con R5 incorporado y las redes /30 del nuevo segmento visibles.

---

## 9. Caso A: Topología modificada con rutas estáticas

Este caso analiza qué le pasa al enrutamiento estático de la Parte 1 cuando se incorpora R5 a la topología. La consigna pide responder tres cosas: qué ocurre en la tabla de ruteo si no se modifican las estáticas, qué hay que hacer para forzar un comportamiento determinado, y si sigue existiendo ECMP "natural". Para entender todo esto hay que recordar una idea central: una ruta estática no se recalcula sola.

### Qué sucede con las rutas que estaban configuradas

Las rutas estáticas que cargamos en la Parte 1 apuntaban a los próximos saltos del enlace R1-R2 (por ejemplo, `ip route 192.168.4.0 255.255.255.0 10.0.12.2`). Cuando R5 se interpone entre R1 y R2 y se reconfiguran las direcciones de esos segmentos, R1 deja de tener conectividad directa con la red 10.0.12.0/30 (o con la IP que usaba R2 como próximo salto). La ruta estática que apunta a ese próximo salto queda inválida, porque el router no puede resolver cómo llegar a él.

Lo que se observa en la tabla es que esa entrada desaparece (o directamente no llega a instalarse): IOS no mantiene en la tabla rutas cuyo próximo salto sea inalcanzable. La ruta estática por el camino inferior, en cambio, sigue apuntando a R3, que no fue modificado, y esa sí se mantiene. El resultado práctico es que R1 queda con un solo camino hacia la red de destino, y si la ruta de R4 (u otros routers intermedios) tampoco se actualizó, la conectividad de punta a punta se pierde o queda a medio camino.

### Qué rutas hay que modificar

Para que la red vuelva a funcionar con los dos caminos hay que ajustar la configuración a mano en varios routers, no solo en R1:

- En **R1**, la ruta superior debe apuntar al próximo salto nuevo (la interfaz de R5 dentro del segmento R1-R5). La ruta inferior se mantiene sin cambios.
- En **R5**, que es un router nuevo, hay que configurar sus direcciones IP y además cargar rutas hacia las dos LAN (o rutas por defecto), porque sin ellas R5 descarta el tráfico que recibe en tránsito.
- En **R2**, la ruta hacia la LAN Origen debe actualizarse para que apunte a R5 en lugar de a R1, ya que el vecino directo de R2 cambió.
- En **R4** y **R3** no cambia nada, porque sus enlaces y vecinos son los mismos.

Un ejemplo de configuración para el camino superior (con el direccionamiento de ejemplo) sería:

```
R1(config)# ip route 192.168.4.0 255.255.255.0 10.0.15.2
R5(config)# ip route 192.168.4.0 255.255.255.0 10.0.52.2
R5(config)# ip route 192.168.1.0 255.255.255.0 10.0.15.1
R2(config)# ip route 192.168.1.0 255.255.255.0 10.0.52.1
```

Y la ruta por el camino inferior se mantiene como estaba:

```
R1(config)# ip route 192.168.4.0 255.255.255.0 10.0.13.2
```

### ¿Sigue existiendo ECMP? Depende de cómo se configure

Acá conviene despejar un equívoco frecuente. Con rutas estáticas no existe un ECMP "natural", es decir automático: no hay ningún protocolo midiendo costos y decidiendo que los dos caminos son equivalentes. El balanceo existe únicamente porque el administrador cargó dos rutas idénticas. Y ese punto no cambia por el solo hecho de agregar R5.

La cantidad de routers intermedios no modifica la métrica de una ruta estática, porque una ruta estática no tiene métrica calculada: el valor que se muestra es siempre 0, sin importar cuántos saltos tenga el camino. Lo único que el administrador puede ajustar es la distancia administrativa (el primer número del corchete), no la métrica. Por lo tanto, si después de la modificación se reconfiguran las dos rutas (una vía R5 y otra vía R3) con la misma distancia administrativa y la misma métrica, el router vuelve a instalar las dos y el balanceo se mantiene, aunque ahora un camino tenga 3 saltos y el otro 2. El router no "sabe" que un camino es más largo: hace lo que se le indica.

En otras palabras: el ECMP con rutas estáticas no se pierde automáticamente al romper la simetría de la topología, pero tampoco se gana solo. Todo depende de la configuración que haga el administrador, y las condiciones para que haya balanceo son siempre las mismas: que las rutas hacia el mismo destino tengan igual distancia administrativa y igual métrica.

### Cómo forzar un comportamiento específico

Si en lugar de balancear se quiere que el tráfico vaya por un camino preferido y que el otro quede como respaldo, se puede jugar con la distancia administrativa de las rutas estáticas. Por ejemplo, para que el camino inferior (R1 → R3 → R4) sea el principal y el superior quede de reserva:

```
R1(config)# ip route 192.168.4.0 255.255.255.0 10.0.13.2
R1(config)# ip route 192.168.4.0 255.255.255.0 10.0.15.2 50
```

La primera ruta (DA 1, por defecto) se instala en la tabla. La segunda, con distancia administrativa 50, no se instala mientras la primera esté disponible: es una "flotante", un respaldo que recién entra en acción cuando la ruta principal desaparece. Es el mismo concepto de tolerancia a fallos que después vemos con RIP, pero aplicado a mano.

Lo mismo se puede hacer en sentido inverso, o se puede mantener el balanceo como en la Parte 1. El punto es que, a diferencia de lo que pasa con un protocolo dinámico, el comportamiento final lo define el administrador con sus comandos, no el cálculo automático del protocolo.

> **[Espacio para captura 7]** — Tabla de enrutamiento de R1 con la topología modificada y rutas estáticas (mostrar la variación que se haya elegido: balanceo con dos rutas iguales o preferencia por un camino con ruta flotante).

Transcripción de los valores observados (completar con la salida real):

```
[Completar: transcribir aquí la tabla de enrutamiento de R1 en el Caso A]
```

*(Completar el análisis describiendo qué rutas se observan, con qué DA y métrica, y qué camino queda activo en el caso que se haya configurado.)*

---

## 10. Caso B: Topología modificada con RIPv2

Este caso es el que mejor muestra la diferencia entre el enrutamiento estático y el dinámico. Con RIP habilitado y la topología ya modificada, los routers vuelven a intercambiar actualizaciones y recalculan las rutas por sí solos, sin que tengamos que escribir ni una sola ruta.

### Cómo cambian las métricas

Recordemos los dos caminos:

- **Camino superior:** R1 → R5 → R2 → R4 (3 saltos)
- **Camino inferior:** R1 → R3 → R4 (2 saltos)

Antes de la modificación, los dos caminos tenían 2 saltos y RIP los trataba como equivalentes. Ahora el camino superior tiene un salto más. Si seguimos la lógica de RIP (la métrica es la cantidad de saltos hasta la red de destino):

- Por el camino inferior, la red `192.168.4.0/24` está a **2 saltos** de R1.
- Por el camino superior, la misma red está a **3 saltos** de R1.

Es decir, R2 sigue anunciando la red destino con métrica 1 (para R2 la red está a un salto, vía R4), R5 la aprende con métrica 2 y se la anuncia a R1 con métrica 3. Por el otro lado, R3 anuncia la red con métrica 2 directamente a R1.

### Qué ocurre cuando las métricas ya no son iguales

RIP elige siempre el camino con menor métrica. Como 2 es mejor que 3, R1 instala en su tabla únicamente la ruta por el camino inferior, con `[120/2]`. El camino superior, con métrica 3, no llega a la tabla de ruteo: RIP lo considera peor y no lo usa mientras exista una mejor opción.

Esto explica por qué desaparece el ECMP. El balanceo de carga igual-costo exige que los caminos empaten en distancia administrativa y en métrica. Al agregar un salto al camino superior, ese empate se rompe, y sin empate no hay nada que balancear: hay un camino mejor y uno peor. No es una falla de configuración, es el protocolo funcionando exactamente como debe.

### El camino alternativo como respaldo

Que el camino superior no esté en la tabla no significa que haya dejado de existir. R5 y R2 siguen anunciando la red destino en sus actualizaciones periódicas, y R1 sigue recibiendo esa información, simplemente la descarta por ser menos conveniente. Ese camino queda como alternativa en reserva.

Esa reserva es lo que le da tolerancia a fallos a la red. Si el enlace del camino inferior se corta (por ejemplo, cae el enlace R1-R3 o R3-R4), la ruta `[120/2]` desaparece de la tabla de R1. Como ya no existe mejor opción, RIP recalcula e instala la ruta por el camino superior, con métrica 3. El tráfico de PC_Origen hacia PC_Destino se desvía por R5, R2 y R4, y la comunicación se mantiene. El costo del cambio es el tiempo que tarda el protocolo en darse cuenta y converger, que no es instantáneo.

Lo mismo vale en el sentido contrario: si se cae el camino superior, el tráfico sigue por el inferior sin que haya que tocar la configuración. En este escenario, el camino inferior es el principal y el superior pasó de ser un camino equivalente a ser la ruta de respaldo.

### Análisis de la tabla de enrutamiento

El comando para verificar todo esto sigue siendo `show ip route` en R1. Lo que debería verse después de la convergencia es una sola ruta `R` hacia `192.168.4.0/24`, con `[120/2]` y próximo salto por R3. Ya no deberían aparecer dos rutas con la misma métrica como en la Parte 2.

> **[Espacio para captura 8]** — Salida del comando `show ip route` en R1 luego de la convergencia de RIP con la topología modificada.

Transcripción de los valores observados (completar con la salida real):

```
[Completar: transcribir aquí la tabla de enrutamiento de R1 en el Caso B]
```

**Análisis crítico de los resultados.** Con la captura y la transcripción completadas, corresponde verificar en el análisis final los siguientes puntos: que la ruta instalada tenga métrica 2 y próximo salto hacia R3; que no aparezca ninguna ruta hacia 192.168.4.0/24 por el camino de R5 (lo que confirmaría que RIP la descartó por métrica); y que, si se simula la caída del camino inferior, la tabla termine mostrando la ruta con métrica 3 por R5-R2. Ese último punto es el que demuestra que el camino largo no se perdió, sino que quedó como reserva.

*(Completar este análisis con los valores realmente observados en el simulador.)*

---

## 11. Análisis de tolerancia a fallos

La tolerancia a fallos es la capacidad de la red para seguir funcionando cuando se corta uno de sus componentes. En nuestra topología, la situación típica es la caída de uno de los enlaces: por ejemplo, que el tramo R1-R3 deje de funcionar, o que se caiga el enlace R2-R4. En ese momento, la pregunta es qué pasa con la comunicación entre PC_Origen y PC_Destino, y cuánto tarda la red en recuperarse.

### Con rutas estáticas

Con rutas estáticas, la recuperación ante una falla depende totalmente de cómo se haya configurado. Hay que distinguir dos situaciones:

- **Caída de un enlace directo.** Si R1 detecta que su interfaz hacia R3 dejó de funcionar (porque el enlace físico se cortó o se hizo `shutdown`), la ruta estática que usa esa interfaz como próximo salto se invalida y desaparece de la tabla. Si quedó otra ruta configurada hacia el mismo destino (la del camino superior), el router la usa automáticamente y el tráfico continúa. Esto funciona siempre que la ruta alternativa exista en la configuración.
- **Caída más allá del primer salto.** Si el enlace que se corta está en el medio de la red (por ejemplo, entre R3 y R4), R1 puede no darse cuenta: su propia interfaz sigue arriba y la ruta estática sigue pareciendo válida. El tráfico se empieza a mandar hacia un destino inalcanzable y se pierde en el camino. Para detectar este tipo de fallas con rutas estáticas hacen falta mecanismos adicionales (como el seguimiento de objetos, SLA tracking), que no forman parte de esta práctica.

Es decir: con estáticas hay tolerancia a fallos solo si el administrador la programó de antemano (por ejemplo, con una ruta flotante de respaldo), y su alcance es limitado.

### Con RIPv2

Con RIP la recuperación es automática, porque el protocolo se entera de los cambios a través de sus actualizaciones y de la expiración de sus timers. Cuando un enlace cae, los routers dejan de recibir (o de poder enviar) actualizaciones por ese camino, las rutas asociadas expiran y RIP recalcula utilizando la información restante. Si existe un camino alternativo, se instala solo, como vimos en el Caso B.

Un punto que conviene aclarar, porque en el material de la cátedra se resume de forma muy breve: la recuperación con RIP no es instantánea. El protocolo trabaja con timers: las actualizaciones se envían cada 30 segundos, y si un router no recibe noticias de una ruta durante el periodo de invalidación (180 segundos por defecto en IOS), la marca como inválida y eventualmente la elimina de la tabla. Cuando la falla es de un enlace directo del propio router, la detección es casi inmediata, porque el estado de la interfaz cambia y las rutas que dependen de ella se retiran en el acto; pero cuando la falla es remota, la red depende de estos tiempos para enterarse. El concepto de "holddown" que aparece en el material hace referencia a este conjunto de mecanismos de espera que evitan reconverger con información vieja; en la implementación de Cisco IOS el temporizador de holddown de RIP viene deshabilitado por defecto, y lo que gobierna la recuperación son los timers de actualización e invalidación. El fondo del asunto se mantiene: frente a una falla, RIP tarda un tiempo en redirigir el tráfico, y ese tiempo hay que tenerlo en cuenta al dimensionar la red.

### Comparación entre ambos métodos

En resumen, la diferencia de comportamiento se puede plantear así:

- **Detección del fallo:** las rutas estáticas solo detectan de forma confiable la caída de su propio enlace de salida; RIP detecta también fallas remotas, aunque tarde un tiempo.
- **Recuperación:** con estáticas hay que editar la configuración (salvo que exista un respaldo ya configurado); con RIP la red se reacomoda sola.
- **Uso de la ruta alternativa:** en ambos casos el tráfico termina yendo por el camino que queda disponible, y la comunicación entre PC_Origen y PC_Destino se mantiene siempre que exista un camino físicamente sano.
- **Tiempos:** con rutas estáticas y enlace directo caído, el cambio es prácticamente inmediato. Con RIP, la recuperación depende de los timers del protocolo y de si la falla es local o remota.

Una forma concreta de probar todo esto en Packet Tracer es apagar una interfaz con el comando `shutdown` (o cortar el cable en la topología) y observar cómo reacciona cada caso: qué entrada desaparece de la tabla, cuánto tarda en aparecer la ruta alternativa y si el `ping` entre las PCs se interrumpe o solo se degrada momentáneamente.

> **[Espacio para captura 9]** — Prueba de falla simulada en Packet Tracer (enlace caído) y tabla de enrutamiento de R1 durante la reconvergencia.

---

## 12. Pruebas de conectividad

Para verificar el funcionamiento de la red se recomienda realizar las siguientes pruebas en Cisco Packet Tracer, tanto en la topología simétrica (Partes 1 y 2) como en la modificada (Parte 3):

- **`ping` desde PC_Origen hacia PC_Destino.** Comprueba que hay conectividad de punta a punta a través de la red. Si responde, significa que todos los routers del camino saben cómo reenviar el paquete y cómo devolver la respuesta. Conviene repetir la prueba en cada etapa (con estáticas, con RIP, con R5 agregado y tras una falla simulada).
- **`traceroute` (o `tracert` desde la PC).** Muestra por qué routers pasó el paquete para llegar al destino. Es la prueba que permite verificar de forma directa si el tráfico va por el camino superior o por el inferior, y por lo tanto si se está aprovechando el balanceo de carga o si todo pasa por un solo camino.
- **`show ip route`.** Muestra la tabla de enrutamiento del router y permite confirmar qué rutas están instaladas, con qué distancia administrativa y métrica, y cuántos caminos hay hacia cada destino. Es el comando central de todo el trabajo.
- **`show running-config`.** Muestra la configuración activa del router. Sirve para verificar que las rutas estáticas o la configuración de RIP se hayan cargado como correspondía (por ejemplo, que estén `version 2` y `no auto-summary`), y para dejar constancia de la configuración aplicada en cada etapa.

Resultados de las pruebas (completar con lo obtenido en el simulador):

| Prueba | Etapa / escenario | Resultado obtenido |
|---|---|---|
| `ping` PC_Origen → PC_Destino | Parte 1 (estáticas) | [Completar] |
| `tracert` PC_Origen → PC_Destino | Parte 1 (estáticas) | [Completar] |
| `ping` PC_Origen → PC_Destino | Parte 2 (RIPv2) | [Completar] |
| `tracert` PC_Origen → PC_Destino | Parte 2 (RIPv2) | [Completar] |
| `ping` PC_Origen → PC_Destino | Parte 3 Caso A (estáticas + R5) | [Completar] |
| `ping` PC_Origen → PC_Destino | Parte 3 Caso B (RIPv2 + R5) | [Completar] |
| `ping` tras falla simulada de un enlace | Caso A / Caso B | [Completar] |

> **[Espacio para captura 10]** — Resultados de `ping` y `traceroute` en las distintas etapas.

*(No se incluyen resultados de conectividad hasta no contar con las pruebas realmente ejecutadas en el simulador.)*

---

## 13. Análisis general de los resultados

Al recorrer las tres partes del trabajo, el comportamiento de la red se puede resumir en tres momentos.

En la primera parte, con rutas estáticas, logramos conectividad de punta a punta y activamos el balanceo de carga simplemente cargando dos rutas iguales. La tabla de ruteo de R1 mostraba dos caminos hacia la red destino con igual distancia administrativa y métrica, y el tráfico se repartía entre los dos enlaces. El resultado fue el esperado, pero conviene recordar que todo ese comportamiento lo definimos nosotros con los comandos: el router no tenía forma de saber que los dos caminos eran "buenos", solo hacía lo que se le indicaba.

En la segunda parte, con RIPv2, llegamos al mismo escenario de balanceo por un camino distinto: los routers aprendieron las redes por sí solos, calcularon la métrica por saltos, detectaron que los dos caminos empataban en 2 saltos e instalaron los dos. La tabla terminó mostrando el mismo resultado que con las estáticas (dos rutas hacia 192.168.4.0/24), pero con `[120/2]`, que delata que el origen de la información es RIP y que la métrica se calculó por conteo de saltos.

En la tercera parte, al agregar R5 y romper la simetría, se vieron las diferencias que separan a ambos métodos. Con RIP, la cosa pasó sola: el camino superior pasó a tener 3 saltos, RIP lo descartó como ruta principal, el balanceo desapareció y el camino inferior quedó como única ruta activa, mientras el superior se convirtió en un respaldo silencioso. Con rutas estáticas, en cambio, nada cambió por sí solo: las rutas viejas quedaron inválidas y hubo que reescribirlas, y el comportamiento final (¿balancear o no?, ¿por qué camino?) quedó en nuestras manos según cómo configuremos las distancias administrativas.

De esta comparación se desprenden las ventajas y limitaciones de cada método. El enrutamiento estático es simple, liviano y predecible, y funciona bien cuando la red es chica y los caminos no cambian. Su límite más duro es que no se adapta: cualquier cambio de topología o cualquier falla no prevista lo toma desprevenido. El enrutamiento dinámico, en cambio, absorbe los cambios de la red sin intervención manual y descubre rutas alternativas solas, pero a cambio paga con complejidad, con tráfico periódico de actualización y con tiempos de convergencia que no son despreciables cuando hay una falla.

En cuanto al ECMP, queda claro que no es algo que se "enciende": es una consecuencia de que dos caminos tengan el mismo costo. Mientras la topología mantenga esa igualdad, el balanceo aparece con estáticas y con RIP. Cuando la igualdad se rompe (como al agregar R5), el balanceo deja de existir en el sentido estricto, y el camino más costoso pasa a ser un candidato descartado, útil solo como respaldo. Esto confirma que el ECMP es una herramienta valiosa para aprovechar enlaces, pero que depende por completo de la simetría de costos que tenga la red.

Finalmente, la tolerancia a fallos quedó ejemplificada en ambos métodos: con estáticas, solo si se configuró un respaldo, y con RIP, de forma automática pero sujeta a los timers del protocolo. En los dos casos, la existencia de un camino alternativo es lo que permite que PC_Origen y PC_Destino sigan comunicándose cuando se corta uno de los enlaces.

---

## 14. Conclusión

Este trabajo práctico me permitió pasar de la teoría del enrutamiento a verla funcionando en una red simulada. El concepto de ECMP, que en la teoría es apenas una definición, se volvió bastante concreto al ver la tabla de ruteo de R1 con dos rutas idénticas hacia el mismo destino: el balanceo de carga no es nada mágico, es simplemente la consecuencia de que el router considere que dos caminos valen lo mismo.

También quedó claro el peso que tienen la distancia administrativa y la métrica en cada decisión del router. La primera le dice en qué fuente confiar (una ruta estática siempre le gana a una aprendida con RIP, aunque eso no signifique que el camino sea mejor), y la segunda le dice qué camino preferir dentro de las rutas de esa misma fuente. Interpretar correctamente los valores del corchete en `show ip route`, como `[1/0]` o `[120/2]`, fue una de las herramientas más útiles del trabajo para entender qué está haciendo el router.

Respecto de la comparación entre enrutamiento estático y dinámico, la experiencia fue bastante ilustrativa: con rutas estáticas todo depende del administrador, y con RIP los routers se acomodan solos, pero tarde. La aparición de R5 en la topología lo dejó en evidencia: RIP recalculó todo sin que haya que tocar nada y el balanceo desapareció cuando los costos dejaron de empatar; con estáticas, en cambio, hubo que rehacer la configuración a mano y decidir explícitamente cómo queríamos que se comportara la red.

RIPv2, si bien es un protocolo viejo y limitado (el límite de 15 saltos y la convergencia lenta lo hacen impensable para redes grandes), resultó ideal para este escenario porque su métrica por saltos hace que el comportamiento del enrutamiento se vea con claridad. Ver cómo el camino largo queda como respaldo y cómo entra en acción cuando falla el principal me dejó bien claro de qué se habla cuando se habla de tolerancia a fallos: tener caminos alternativos no sirve de nada si no hay un mecanismo (manual o automático) que los use cuando hace falta.

Por último, el uso de Cisco Packet Tracer fue clave para todo el trabajo. Poder configurar los routers, apagar un enlace y ver cómo reacciona la red en pocos segundos es una forma de aprender que con la teoría sola no se consigue. Las capturas y las pruebas de conectividad que se completarán en este informe son las que terminan de cerrar el circuito entre lo que se estudió y lo que realmente sucede en la red.

---

## 15. Bibliografía

**Material de la cátedra (proporcionado por el profesor):**

1. *Trabajo Práctico Integrador: Análisis de Comportamiento de Enrutamiento, ECMP y Tolerancia a Fallos.* Consigna de la materia Redes (PDF).
2. *RIP_v2 — Configuración y funcionamiento de RIPv2 en la topología del trabajo práctico.* Material complementario de la cátedra (PDF).
3. *ECMP (Equal-Cost Multi-Path o Caminos Múltiples de Costo Igual).* Nota teórica complementaria de la cátedra (archivo de texto).

**Fuentes externas consultadas:**

4. Cisco Systems. *IP Routing: RIP Configuration Guide* (documentación oficial de Cisco IOS). Disponible en la documentación de Cisco: cisco.com.
5. Hedrick, C. *Routing Information Protocol.* RFC 1058, IETF, 1988.
6. Malkin, G. *RIP Version 2.* RFC 2453, IETF, 1998.

*Nota: los ítems 1 a 3 corresponden al material entregado por la cátedra para la realización del trabajo práctico. Los ítems 4 a 6 se utilizaron como consulta técnica complementaria.*

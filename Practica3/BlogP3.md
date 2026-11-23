# Práctica 3: Controlador de máquina expendedora
 
## 1. Introducción y objetivo
 
Esta práctica consiste en diseñar y programar el controlador de una máquina expendedora automática con una placa Arduino UNO y los sensores y actuadores del kit de laboratorio. La máquina atiende al usuario con una pantalla LCD, un joystick y un botón externo, enciende LEDs indicadores y, a la vez, mide temperatura, humedad y distancia sin quedarse bloqueada.
 
Un sistema empotrado es un sistema de computación diseñado para realizar una o pocas funciones de control, con recursos limitados. Nuestra máquina, además, interacciona repetidamente con el entorno físico y debe reaccionar a eventos en un tiempo razonable: en la teoría, en un sistema de tiempo real no basta con que las acciones sean correctas, tienen que ejecutarse dentro de un intervalo de tiempo determinado. Como un retraso ocasional degradaría el servicio pero no tendría consecuencias graves, la clasificamos como un sistema de tiempo real suave (los plazos son importantes pero no estrictamente críticos).
 
El enunciado pide tres modos de funcionamiento (Arranque, Servicio y Administración) y recomienda usar threads, interrupciones hardware y watchdog, evitando los retardos bloqueantes. Este documento explica cómo se ha montado el circuito, cómo funciona cada modo, qué recursos se han usado y por qué, qué pruebas se han hecho y qué problemas han surgido.
 
## 2. Hardware y montaje
 
### 2.1 Componentes
 
- **Arduino UNO:** placa con el microcontrolador ATmega328P de 8 bits, con 32 KB de flash y 2 KB de SRAM, que incluye interrupciones, watchdog, conversor analógico/digital (ADC) y comunicación serie.
- **Pantalla LCD 16x2:** muestra los mensajes al usuario. Se maneja con la librería oficial LiquidCrystal.
- **Joystick:** los ejes X e Y analógicos sirven para navegar y su pulsador integrado para confirmar.
- **Sensor DHT11:** mide temperatura y humedad y entrega una señal digital calibrada.
- **Sensor de ultrasonidos HC-SR04:** mide distancias como un sonar: emite un sonido de unos 40 kHz y mide el tiempo de vuelo del eco.
- **Botón externo:** sirve para cancelar (pulsación de 2 a 3 s) y para entrar y salir del modo Admin (5 s o más).
- **LED1 y LED2:** indicadores. LED2 está en un pin PWM para poder variar su brillo.
### 2.2 Tabla de conexiones
 
| Elemento | Pin | Detalle |
|---|---|---|
| Botón externo | 2 (INT0) | INPUT_PULLUP: el otro extremo va a GND, pulsado = LOW |
| DHT11 (DATA) | 3 | Además de alimentación y GND |
| HC-SR04 TRIG | 4 | Salida: dispara el pulso |
| HC-SR04 ECHO | 5 | Entrada: se mide con pulseIn |
| LED2 | 6 (PWM) | Con resistencia en serie |
| LED1 | 13 | Con resistencia en serie (el LED integrado de la placa también se enciende porque está en ese pin) |
| Joystick X / Y | A0 / A1 | Entradas analógicas |
| Joystick pulsador | A2 | INPUT_PULLUP, pulsado = LOW |
| LCD | 12, 11, 10, 9, 8, 7 | RS=12, E=11, D4=10, D5=9, D6=8, D7=7 |
| A5 | Sin conectar | Su ruido se usa como semilla de random() |
 
[Indica aquí el valor de las resistencias que has usado y cómo has alimentado el LCD y ajustado su contraste.]
 
### 2.3 Por qué estos pines
 
- **Botón en el pin 2:** el Arduino UNO solo tiene dos pines con interrupción hardware: INT0 (pin 2) e INT1 (pin 3). El botón externo necesita interrupción, así que usa el pin 2.
- **LED2 en el pin 6:** los pines PWM del UNO son 3, 5, 6, 9, 10 y 11, y los pines 5 y 6 usan el Timer0 a 980 Hz. Ese timer es el mismo que usa millis(), pero la teoría indica que es seguro usarlo para PWM junto con millis() mientras no se cambie su frecuencia de trabajo, y aquí no se cambia.
- **INPUT_PULLUP en el botón externo y en el pulsador del joystick:** una entrada digital sin nada conectado está en estado flotante y puede leer valores inesperados. La resistencia pull-up interna del ATmega fija el nivel HIGH hasta que se pulsa (LOW) sin añadir componentes.
- **Joystick en A0 y A1:** el ADC del UNO tiene 10 bits y devuelve valores de 0 a 1023. En reposo el joystick marca aproximadamente 512, y se consideran movimientos los valores por debajo de 300 o por encima de 700.
- **Sin TimerOne:** al no usar esta librería no hay conflicto de timers con el PWM.
### 2.4 Fotos del montaje
 
FOTO 1: vista general del montaje completo, con todos los componentes visibles  
<img width="5709" height="3094" alt="circuitoEntero1" src="https://github.com/user-attachments/assets/008d99b2-6fe5-4d65-868c-15f028abe0d6" />  

<img width="5571" height="2718" alt="circuitoEntero2" src="https://github.com/user-attachments/assets/5d29ca75-27b5-4cd0-ab94-66674d8d47a8" />  

  
FOTO 2: detalle del Arduino UNO y del cableado  
<img width="4032" height="3024" alt="ArduinoUno" src="https://github.com/user-attachments/assets/cde4b468-9b8f-4d31-bbdb-c759d3c6c04f" />  
  
FOTO 3: detalle de la pantalla LCD y del joystick  
<img width="3024" height="4032" alt="LCD1" src="https://github.com/user-attachments/assets/e6fc7e22-1ffe-4841-a5c1-c16f318533a2" />  

<img width="4032" height="3024" alt="LCD2" src="https://github.com/user-attachments/assets/c44b19a5-f05b-4a55-b9fc-b6392bd5521b" />  

<img width="3024" height="4032" alt="Joystick" src="https://github.com/user-attachments/assets/a8896703-b564-4211-89de-bbde300c1827" />  
  
FOTO 4: detalle del DHT11 y del sensor de ultrasonidos HC-SR04  
 <img width="3024" height="4032" alt="DHT11_1" src="https://github.com/user-attachments/assets/0c66d12c-b2bb-4ffb-b68b-4cae1cfdd73c" />  

 <img width="3024" height="4032" alt="DHT11_2" src="https://github.com/user-attachments/assets/9ebbf543-471b-49e2-aaae-141d6e87b59b" />  

 <img width="3024" height="4032" alt="Ultrasonidos1" src="https://github.com/user-attachments/assets/c61447c6-f45f-48d3-b287-5ff3140876ae" />  

 <img width="3024" height="4032" alt="Ultrasonidos2" src="https://github.com/user-attachments/assets/fcfb6bfd-81d1-4cf5-bb8c-c47f0c8642f4" />  
   
   
FOTO 5: detalle del botón externo y de los dos LEDs (LED1: verde y LED2: rojo)  
<img width="3024" height="4032" alt="BotonLeds1" src="https://github.com/user-attachments/assets/406d80a6-53b2-4a9e-83a1-5d51270cbadc" />  

<img width="3024" height="4032" alt="BotonLeds2" src="https://github.com/user-attachments/assets/27a762bf-8a95-461f-8b01-10b3c88b4012" />  
  

### 2.5 Esquemático del circuito
 
El esquemático se ha realizado con Fritzing.
 
<img width="1209" height="536" alt="imagen" src="https://github.com/user-attachments/assets/97652994-9867-4933-bc09-e84d5aa75d71" />  
  
 
## 3. Funcionamiento del sistema
 
### 3.1 Resumen de modos y pantallas
 
[INSERTAR CAPTURAS DEL LCD: una por cada fila de la tabla (fotogramas de los vídeos o fotos): CARGANDO, ESPERANDO CLIENTE, temperatura y humedad, lista, Preparando, RETIRE BEBIDA y menú Admin]
 
### 3.2 Modo Arranque
 
Al energizar o reiniciar la placa, `setup()` configura los pines, el LCD, el sensor, la interrupción y los threads. LED1 se enciende y el LCD muestra `CARGANDO ...`. Un thread cambia el estado de LED1 cada 500 ms (un parpadeo completo dura 1 s). Tras 3 parpadeos apaga LED1, se deshabilita a sí mismo y pasa a Servicio. Durante el arranque las entradas del usuario se ignoran.
 
| Modo / estado | Qué ocurre | Mensaje en el LCD | LEDs |
|---|---|---|---|
| Arranque | LED1 parpadea 3 veces y pasa a Servicio | CARGANDO ... | LED1 parpadeando, LED2 apagado |
| Servicio: espera | Nadie cerca (distancia de 1 m o más, o sin eco) | ESPERANDO / CLIENTE (en dos líneas porque no cabe en 16 columnas) | Apagados |
| Servicio: ambiente | Se acerca alguien (menos de 1 m): temperatura y humedad durante 5 s | Temp: XX C / Hum: YY % | Apagados |
| Servicio: lista | Lista de productos con precio | Dos productos a la vez, cursor parpadeante | Apagados |
| Servicio: preparando | Entre 4 y 8 s aleatorios | Preparando / nombre del producto... | LED2 sube de brillo con PWM |
| Servicio: retirar | 3 s y vuelta a la espera | RETIRE BEBIDA | Apagados |
| Admin | Menú de supervisión y ajuste | Ver temperatura, Ver dist. sensor, Ver contador, Modif. precios | LED1 y LED2 encendidos |
 
### 3.3 Modo Servicio
 
1. **Presencia:** si la distancia medida es menor de 100 cm se pasa a mostrar la temperatura y la humedad durante 5 s; si es mayor o no hay eco, se muestra ESPERANDO CLIENTE.
2. **Lista de productos:** pasados los 5 s se muestra la lista (Café Solo 1.00, Café Cortado 1.10, Café Doble 1.25, Café Premium 1.50 y Chocolate 2.00 euros). Con el joystick arriba o abajo se mueve el cursor y la ventana de dos filas se desplaza. El pulsador del joystick confirma el producto.
3. **Preparación:** se genera una duración pseudoaleatoria entre 4 y 8 s. Mientras dura, el LCD muestra Preparando y el nombre del producto, y LED2 sube de brillo de forma proporcional al porcentaje de progreso.
4. **Retirada:** al terminar se apaga LED2, se muestra RETIRE BEBIDA durante 3 s y la máquina vuelve al estado de espera.
5. **Cancelación:** en cualquier momento del Servicio, mantener el botón externo y soltarlo entre 2 y 3 s devuelve la máquina al inicio del Servicio, apagando los LEDs y sin reiniciar la placa.  
### 3.4 Modo Administración
 
- **Entrar y salir:** se mantiene el botón externo 5 s o más. Con el botón aún pulsado, al llegar a los 5 s se cambia de modo. Mientras se está en Admin, LED1 y LED2 permanecen encendidos.
- **Menú:** cuatro opciones que se recorren con el joystick arriba y abajo. El pulsador del joystick entra en la opción y el movimiento a la izquierda vuelve al menú.
- **Ver temperatura:** muestra temperatura y humedad del DHT11, refrescadas cada 500 ms.
- **Ver distancia sensor:** muestra la distancia en centímetros, refrescada cada 500 ms (si no hay eco muestra sin eco).
- **Ver contador:** muestra los segundos transcurridos desde el encendido, calculados con `millis()`.
- **Modificar precios:** se elige un producto de la lista y con el joystick arriba o abajo su precio sube o baja de 0,05 en 0,05 euros (entre 0,05 y 9,95). El pulsador confirma y la izquierda descarta el cambio. Los precios se guardan en la memoria de datos, así que los nuevos importes se ven en la lista del Servicio; no se guardan en EEPROM, como permite el enunciado.
### 3.5 Parámetros del sistema
 
| Parámetro | Valor |
|---|---|
| Distancia de detección de cliente | menos de 100 cm |
| Temperatura y humedad al acercarse | 5 s |
| Preparación del producto | aleatoria, de 4 a 8 s |
| Mensaje RETIRE BEBIDA | 3 s |
| Cancelación | botón externo mantenido y soltado entre 2 y 3 s |
| Entrar o salir de Admin | botón externo mantenido 5 s o más |
| Antirrebote del botón externo | 30 ms |
| Joystick | reposo cerca de 512; inclinado si baja de 300 o sube de 700 |
| Paso de precio en Admin | 0,05 euros |
 
## 4. Solución software
 
### 4.1 Recursos utilizados y por qué
 
| Recurso | Dónde se usa | Por qué |
|---|---|---|
| Threads (ArduinoThread) | Cuatro tareas periódicas | Hay varias tareas con periodos distintos y no se pueden usar delay() |
| Interrupción hardware (INT0, pin 2) | Botón externo | Es un evento externo impredecible, y no conviene sondearlo |
| Watchdog (2 s) | setup() y loop() | Si un thread se bloquea, se bloquea todo el programa |
| PWM (analogWrite) | Brillo de LED2 | Permite intensidades intermedias en un pin digital |
| millis() | Tiempos de los estados | Mide tiempo sin parar la CPU, a diferencia de delay()|
 
### 4.2 Threads con ArduinoThread
 
Según la teoría, un único flujo de ejecución con bucle infinito y sondeo de entradas es sencillo pero poco flexible y difícil de ampliar. Un sketch crece en complejidad cuando hay muchas tareas con periodicidades distintas, y la implementación manual con millis() que se muestra en las transparencias se presenta como una solución incómoda cuando las tareas aumentan. Además, delay() no libera la CPU del microcontrolador. Por eso se usa la librería ArduinoThread con un ThreadController: cada tarea se define con un periodo (setInterval) y una función (onRun), y el bucle principal solo llama al controlador.
 
```cpp
tControl.setInterval(50);    tControl.onRun(controlFSM);
tDistancia.setInterval(200); tDistancia.onRun(leerDistancia);
tDHT.setInterval(2000);      tDHT.onRun(leerDHT);
tParpadeo.setInterval(500);  tParpadeo.onRun(cbParpadeo);
controller.add(&tControl);
controller.add(&tDistancia);
controller.add(&tDHT);
controller.add(&tParpadeo);
```
 
| Thread | Periodo | Función |
|---|---|---|
| tControl | 50 ms | controlFSM: lee joystick y botón, ejecuta la máquina de estados y repinta el LCD |
| tDistancia | 200 ms | leerDistancia: mide con el HC-SR04 |
| tDHT | 2000 ms | leerDHT: lee temperatura y humedad |
| tParpadeo | 500 ms | cbParpadeo: parpadeo de LED1 en el Arranque; se deshabilita al terminar |
 
El periodo de cada thread depende de lo deprisa que cambia lo que atiende: el control del usuario (50 ms) es lo que más rápido debe responder, la presencia se comprueba cuatro veces por segundo y la temperatura y la humedad, que varían despacio, se leen cada 2 s. Las transparencias plantean un reparto parecido, con periodos distintos para cada tarea (un LED cada 500 ms, un pulsador cada 170 ms, un sensor de temperatura cada segundo).
 
En términos de planificación, son tareas periódicas con un periodo de activación. ArduinoThread no ejecuta los threads en paralelo: hay un único procesador y cada tarea se ejecuta completa antes de pasar a la siguiente, de forma parecida al modelo de planificación cíclica de los apuntes. Por eso cada tarea debe ser corta, con un tiempo de cómputo menor que su periodo. El peor caso es el thread de distancia, que espera el eco con pulseIn limitado a 15 ms, muy por debajo de sus 200 ms de periodo. La consecuencia de este modelo es que si un thread se queda bloqueado, todo el programa se queda bloqueado, y de ahí la necesidad del watchdog.
 
### 4.3 Interrupción hardware para el botón externo
 
El botón externo se pulsa en cualquier momento, y la teoría clasifica este tipo de evento como tarea esporádica: se activa en instantes aleatorios y depende de un evento externo; su ejemplo es precisamente la interrupción causada por un interruptor. Con sondeo (polling) el microcontrolador gasta ciclos preguntando cuando casi nunca hay datos, se pueden perder eventos, no hay prioridad y se consume más energía. Por eso se usa la interrupción hardware INT0 del pin 2 con el modo CHANGE, que salta en los dos flancos y permite saber cuándo se pulsa y cuándo se suelta:
 
```cpp
attachInterrupt(digitalPinToInterrupt(PIN_BOTON), isrBoton, CHANGE);
```
 
La rutina de servicio (ISR) solo apunta cuándo se pulsa y se suelta; la lógica se hace fuera:
 
```cpp
void isrBoton() {
  unsigned long ahora = millis();
  if (ahora - ultimoFlanco < REBOTE_MS) return;   // antirrebote
  ultimoFlanco = ahora;
  if (digitalRead(PIN_BOTON) == LOW) {            // flanco de bajada: pulsado
    botonPulsado = true;
    inicioPulsacion = ahora;
  } else if (botonPulsado) {                      // flanco de subida: soltado
    botonPulsado = false;
    duracionPulsacion = ahora - inicioPulsacion;
    botonSoltado = true;
  }
}
```
 
Esta ISR sigue las normas de la teoría:
 
- **Dura lo mínimo:** solo modifica variables, sin cálculos complejos, sin comunicación y sin cambiar salidas digitales.
- **Variables volatile:** las variables que cambia la ISR y se leen en otras funciones (botonPulsado, botonSoltado, inicioPulsacion, duracionPulsacion y ultimoFlanco) se declaran volatile, para que se carguen siempre de memoria.
- **millis() dentro de la ISR:** solo se lee su valor. La teoría avisa de que durante una ISR millis() no se actualiza, así que se mantiene la rutina muy corta.
- **Antirrebote:** al pulsar un interruptor hay rebotes hasta que la señal se estabiliza, y se pueden resolver con condensadores o con código. Aquí se resuelve por código con un umbral de tiempo de 30 ms, la misma técnica que usa la teoría en su ejemplo de ISR con millis().
La duración de la pulsación se clasifica fuera de la ISR, en `leerBoton()`, que se ejecuta dentro del thread tControl. Esta función copia los valores de la ISR desactivando un instante las interrupciones con `noInterrupts()` e `interrupts()`, para no leer valores a medias mientras la ISR los cambia. Después aplica las reglas del enunciado: si el botón lleva pulsado 5 s o más se genera el evento de Admin (sin esperar a soltar), y si se suelta con una duración de 2 a 3 s se genera el evento de cancelación. Una pulsación de entre 3 y 5 s no hace nada, consecuencia directa de los intervalos que fija el enunciado.
 
### 4.4 Watchdog
 
Un watchdog es un mecanismo de seguridad que provoca un reset del sistema si este se ha bloqueado o no responde como se espera. Es un timer dentro del microcontrolador, independiente del resto de timers y del sketch, con un reloj interno de 128 kHz, y aparece en la lista de recursos auxiliares de un microcontrolador. Aquí se usa por dos razones: ArduinoThread no es paralelo y un thread bloqueado detiene todo el programa, y un sistema empotrado debe seguir funcionando y recuperarse ante fallos.
 
Se configura con las funciones de la librería avr/wdt.h que se ven en la teoría: `wdt_disable()` y `wdt_enable()` al final de `setup()`, y `wdt_reset()` al final de cada vuelta de `loop()`.
 
```cpp
wdt_disable();
wdt_enable(WDTO_2S);   // en setup(), una vez configurado todo
 
void loop() {
  // PRUEBA WDT (descomentar solo para probar):
  // if (digitalRead(PIN_JOY_SW) == LOW) while (1);
  controller.run();    // ejecuta los threads que toquen
  wdt_reset();         // sigo vivo: si un thread se bloquea, no se llega aquí
}
```
 
Si `loop()` tarda más de 2 s en llegar a `wdt_reset()`, la placa se reinicia y vuelve al modo Arranque. Se eligió 2 s porque es mucho mayor que cualquier espera del programa (la más larga es el límite de 15 ms de pulseIn) y suficientemente corto para que la recuperación sea rápida.
 
**Cómo se probó:** se provoca un bloqueo a propósito con un bucle infinito, `while (1);`, la misma construcción que usan las transparencias para ilustrar que la CPU no se libera. La línea de prueba está comentada en `loop()`: al descomentarla, pulsar el joystick cuelga el programa y, a los 2 s aproximadamente, el watchdog reinicia la placa y vuelve a aparecer `CARGANDO ...`. Hay que pulsar y soltar enseguida: si el botón se mantiene pulsado mientras la placa arranca, el bloqueo vuelve a producirse tras cada reinicio.
 
[INSERTAR VÍDEO V4: prueba del watchdog (congelado, reinicio y vuelta a CARGANDO ...)]
 
### 4.5 PWM, sensores, joystick y pantalla
 
- **PWM en LED2:** analogWrite genera una señal con un ciclo de trabajo (duty cycle) entre 0 y 255, es decir, de 0 % a 100 %. Durante la preparación se aplica el porcentaje de progreso, así el brillo del LED refleja el avance de la bebida.
```cpp
analogWrite(PIN_LED2, (t * 255UL) / prepDur);   // brillo = % de progreso (0-255)
```
 
- **Ultrasonidos HC-SR04:** se dispara un pulso de 10 microsegundos en TRIG con delayMicroseconds, que no usa timer sino ciclos de procesador y es exacto entre 3 y 16383 microsegundos. Con pulseIn en ECHO se mide el tiempo de vuelo del eco en microsegundos. Como el sonido viaja a 343 m/s y el eco hace ida y vuelta, la distancia es el tiempo multiplicado por 0,01715 cm/us. Se limita la espera a 15 ms (unos 2,5 m de alcance, más que suficiente para detectar menos de 1 m); si no hay eco se interpreta que no hay nadie delante, y así el thread nunca se queda esperando.
- **DHT11:** se usa la librería DHT de Adafruit, que hay que instalar desde el IDE. Se lee cada 2 s y se guardan los últimos valores válidos; si aún no hay lectura, el LCD muestra `--`.
- **Joystick:** se leen los ejes con analogRead (0 a 1023) y se transforman en tres zonas (reposo, bajo y alto). El evento de arriba, abajo o izquierda se genera solo al empezar a inclinar el joystick, así que un movimiento equivale a un único paso. El pulsador se detecta por su flanco de pulsación.
- **LCD:** solo se repinta cuando hace falta (bandera redibujar). La excepción son las pantallas con datos que cambian solos (temperatura, distancia y contador), que se refrescan cada 500 ms.
- **Aleatoriedad:** el tiempo de preparación se obtiene con random(4000, 8001), un valor pseudoaleatorio entre 4 y 8 s. La semilla se toma de la lectura del pin A5, que está sin conectar, para que la secuencia sea distinta en cada arranque.
### 4.6 Máquina de estados
 
El enunciado define tres modos con muchas pantallas y transiciones, así que se implementó como una máquina de estados: una variable estado con los valores ARRANQUE, ESPERA, AMBIENTE, LISTA, PREPARANDO, RETIRE (Servicio) y ADMIN_MENU, ADMIN_TEMP, ADMIN_DIST, ADMIN_CONT, ADMIN_PRECIOS, ADMIN_EDITAR (Admin). Todas las transiciones se deciden en `controlFSM()`:
 
- ARRANQUE a ESPERA: al terminar los 3 parpadeos.
- ESPERA a AMBIENTE: distancia menor de 100 cm.
- AMBIENTE a LISTA: pasados 5 s.
- LISTA a PREPARANDO: pulsador del joystick.
- PREPARANDO a RETIRE: al cumplirse la duración aleatoria.
- RETIRE a ESPERA: pasados 3 s.
- Cualquier estado de Servicio a ESPERA: cancelación de 2 a 3 s.
- Servicio a Admin y Admin a Servicio: botón externo 5 s o más.
- Dentro de Admin: el menú entra en cada opción con el pulsador y todas vuelven con la izquierda; en Modificar precios, el pulsador abre la edición del precio.
[INSERTAR DIAGRAMA (opcional): diagrama de estados dibujado a mano o con una herramienta]
 
### 4.7 Recursos que no se han usado
 
- **TimerOne (interrupciones por software):** las ISR deben ser mínimas, sin cálculos complejos ni comunicaciones. Las tareas periódicas de esta práctica escriben en el LCD o leen sensores con esperas (DHT11 y ultrasonidos), que no son adecuadas para una ISR. Además, ThreadController ya aporta la periodicidad sin ISR. TimerOne usa el Timer1, que comparte recursos con el PWM de los pines 9, 10 y 11 y puede dar comportamientos no deseados.
- **RTOS:** un RTOS es un sistema operativo multitarea con planificador determinista y prioridades. Para cuatro tareas periódicas sencillas, que el enunciado propone resolver con ArduinoThread y recursos de Arduino, no ha sido necesario.
## 5. Pruebas y resultados
 
Todas las pruebas se han grabado en vídeo con el montaje real. La tabla resume qué se probó, qué se observó y en qué vídeo se ve.
 
[INSERTAR VÍDEO V1: arranque, servicio completo (presencia, lista, preparación con LED2, cancelación) y opciones de Admin. Pon el enlace o el vídeo incrustado]
 
[INSERTAR VÍDEO V2: entrada en Admin con LED1 y LED2 encendidos y salida con 5 s]
 
| Prueba | Qué se hizo | Resultado observado | Vídeo |
|---|---|---|---|
| Arranque | Conectar o reiniciar la placa | El LCD muestra CARGANDO ..., LED1 parpadea y después aparece ESPERANDO CLIENTE | V1 |
| Presencia | Acercarse al sensor de ultrasonidos | Pasa a temperatura y humedad (unos 24 C y 37 % en las pruebas) y a los 5 s aparece la lista | V1 |
| Lista y selección | Mover el joystick arriba y abajo y pulsar | El cursor y la ventana se desplazan hasta Chocolate y el pulsador confirma el producto | V1 |
| Preparación | Seleccionar un producto | Aparece Preparando y el nombre del producto, LED2 sube de brillo (unos 5 s en la prueba grabada), después RETIRE BEBIDA 3 s y vuelta a ESPERANDO CLIENTE | V1, V3 |
| Cancelación | Mantener el botón externo y soltarlo entre 2 y 3 s | Vuelve a ESPERANDO CLIENTE sin reiniciar la placa | V1 |
| Entrar en Admin | Mantener el botón externo 5 s o más | Se encienden LED1 y LED2 y aparece el menú | V2 |
| Opciones de Admin | Ver distancia, contador y modificar precios | La distancia cambia al acercar la mano (de 14 a 5 cm), el contador avanza (de 88 a 91 s) y un precio sube de 1.25 a 1.30 | V1 |
| Precio visible en Servicio | Salir de Admin y ver la lista | La lista muestra el nuevo precio, 1.30 | V1 |
| Salir de Admin | Mantener el botón externo 5 s o más | Los LEDs se apagan y se vuelve al Servicio | V2 |
| Watchdog | Colgar el programa a propósito | Pantalla congelada y reinicio a los 2 s aproximadamente, con CARGANDO ... | V4 |
 
[INSERTAR VÍDEO V3 (opcional): varias preparaciones seguidas, para mostrar que cada una dura un tiempo distinto]
 
[INSERTAR CAPTURAS DEL LCD de Admin: Ver temperatura, Distancia, Contador, lista de precios y edición de un precio]
 
## 6. Problemas encontrados y soluciones
 
**6.1 LED2 no se encendía.** En las primeras pruebas LED1 funcionaba, pero LED2 no se encendía nunca: ni en el Arranque, ni en Admin ni durante la preparación. En Admin el código escribe HIGH directamente en el pin 6, así que el fallo no podía estar en el PWM sino en la conexión del LED. Se revisaron el pin al que iba el cable, la polaridad del LED (ánodo hacia el pin y cátodo hacia GND) y la conexión a tierra. [Explica aquí qué estaba mal y cómo lo corregiste.]
 
**6.2 El brillo de LED2 parecía no ser progresivo.** Con LED2 ya funcionando se revisó el vídeo fotograma a fotograma: el brillo sube de forma progresiva durante unos 3 s hasta quedar en su máximo, y se mantiene así hasta que termina la preparación (unos 5 s en esa prueba). El código aplica un ciclo de trabajo lineal de 0 a 255, es decir, de 0 % a 100 %, y no se modificó. Con preparaciones más largas (hasta 8 s) la subida se aprecia mejor.
 
**6.3 El LED1 parpadea nada más conectar la placa.** No es un fallo. El Modo Arranque exige que LED1 parpadee 3 veces al energizar o reiniciar. Además, al arrancar la placa se ejecuta primero el bootloader durante un tiempo y el sketch empieza con un retraso de unos segundos. Lo mismo ocurre cada vez que la placa se reinicia: al conectar el USB, al subir un sketch o cuando salta el watchdog.
 
**6.4 Pulsaciones del botón entre 3 y 5 s.** Con los intervalos del enunciado (cancelación entre 2 y 3 s, Admin a partir de 5 s), una pulsación de 3 a 5 s no activa ninguna acción. Se ha mantenido así para respetar los intervalos exactos que pide el enunciado.
 
**6.5 Prueba del watchdog.** Para probarlo hay que colgar el programa a propósito, y la línea de prueba solo se debe descomentar durante la prueba: si se deja activa, cada pulsación del joystick bloquea la máquina. Por eso se deja comentada en `loop()`.
 
## 7. Conclusiones
 
- La máquina cumple los tres modos del enunciado: Arranque, Servicio y Administración.
- **Threads:** ArduinoThread permite repartir las tareas con periodos distintos sin delay(), aunque no son paralelos y un thread bloqueado detiene todo.
- **Interrupción:** el botón externo se atiende por interrupción con una ISR mínima, y la clasificación de la pulsación se hace fuera de ella.
- **Watchdog:** completa el diseño porque permite recuperar la placa si el programa se bloquea.
- **Mejora posible:** guardar los precios modificados en la EEPROM, que el Arduino UNO tiene (1 KB), para que no se pierdan al apagar. El enunciado no lo exige.


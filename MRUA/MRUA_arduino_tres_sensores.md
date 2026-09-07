# Guía práctica: Medición de aceleración en MRUA con Arduino y tres sensores

## Objetivo de aprendizaje
Que el estudiante determine experimentalmente la aceleración de un objeto en Movimiento Rectilíneo Uniformemente Acelerado (MRUA), midiendo dos velocidades promedio en tramos consecutivos y calculando:

**a = (v₂ − v₁) / (t₂ − t₁)**

donde v₁ y v₂ son velocidades medias en dos tramos sucesivos del recorrido, y t₁, t₂ son los instantes (tiempos medios de cada tramo) en que ocurren esas velocidades.

---

## 1. Fundamento físico

A diferencia del MRU, en el MRUA la velocidad cambia de manera constante en el tiempo. Con **dos sensores** solo se obtiene una velocidad promedio del recorrido completo; para detectar el cambio de velocidad —y por tanto la aceleración— se necesita **al menos un tercer punto de medición**, de modo que se puedan calcular dos velocidades independientes en tramos consecutivos.

Ecuaciones de referencia que se comprobarán con los datos obtenidos:

| Ecuación | Uso |
|---|---|
| v = v₀ + a·t | velocidad final en función del tiempo |
| x = x₀ + v₀·t + ½·a·t² | posición en función del tiempo |
| v² = v₀² + 2·a·Δx | velocidad sin depender del tiempo |

Con tres sensores se generan dos velocidades promedio (v₁ entre sensor 1 y 2; v₂ entre sensor 2 y 3), y con la diferencia de esas velocidades entre los tiempos en que "ocurren" (tiempo medio de cada tramo) se estima la aceleración.

> **Nota conceptual para el estudiante:** cada v calculada con dos sensores es una velocidad *media* del tramo, no instantánea. Cuanto más corto sea el tramo, más se aproxima a la velocidad instantánea en el punto medio de ese tramo — esta es una buena discusión de límites y derivadas si el curso lo permite.

---

## 2. Cómo generar el MRUA

Para que el objeto acelere de forma constante y controlada, cualquiera de estas opciones funciona bien en el aula:

- **Plano inclinado (más recomendado):** un riel o canaleta inclinada un ángulo pequeño (5°–10°), dejando caer un carrito o esfera por gravedad. La aceleración teórica es a = g·sen(θ), útil para comparar con el valor medido.
- **Sistema de poleas (máquina de Atwood simplificada):** un carrito sobre riel horizontal, halado por un hilo que pasa por una polea y sostiene una masa colgante.
- **Carrito motorizado con aceleración programada:** un carrito con motor DC controlado por PWM creciente (más complejo, opcional para cursos con enfoque en control).

El plano inclinado es el montaje más simple de reproducir con el material típico de laboratorio y es el que se documenta a continuación.

---

## 3. Materiales

| Cantidad | Componente |
|---|---|
| 1 | Arduino Uno (o compatible) |
| 3 | Sensores infrarrojos de barrera (tipo FC-51 / módulo IR obstáculo) — recomendados para esta práctica por su disparo más limpio ante objetos en movimiento rápido |
| 1 | Protoboard |
| — | Cables jumper |
| 1 | Riel o canaleta recta, con posibilidad de inclinarse |
| 1 | Base o soporte para fijar el ángulo de inclinación (libros, soporte de laboratorio, o base graduada) |
| 1 | Transportador o inclinómetro (para medir el ángulo θ) |
| 1 | Cinta métrica o regla |
| 1 | Carrito o esfera de baja fricción |
| 1 | Cable USB / fuente de alimentación |
| Opcional | Nivel de burbuja, para asegurar que el riel esté recto lateralmente |

---

## 4. Montaje físico

1. Instala el riel con un extremo elevado, formando un ángulo θ pequeño y constante respecto a la horizontal. Mide θ con el transportador.
2. Coloca los tres sensores en línea sobre el riel, en el orden **Sensor 1 (arriba, cerca del punto de salida), Sensor 2 (intermedio), Sensor 3 (abajo)**.
3. Mide y registra las distancias:
   - **d₁₂**: distancia entre Sensor 1 y Sensor 2.
   - **d₂₃**: distancia entre Sensor 2 y Sensor 3.
   - Se recomienda que **d₁₂ = d₂₃** para simplificar los cálculos, aunque el código funciona igual si son distintas.
4. Verifica que el objeto pase exactamente por el haz de los tres sensores sin desviarse.

```
 (salida)                                                  
    ●                                                       
     \                                                      
      \        [Sensor 1] --d12-- [Sensor 2] --d23-- [Sensor 3]
       \___________________________________________________
                            riel inclinado (ángulo θ)
```

---

## 5. Conexiones eléctricas

| Sensor IR | Señal (OUT) | Arduino |
|---|---|---|
| Sensor 1 | Digital | Pin 2 (interrupción) |
| Sensor 2 | Digital | Pin 3 (interrupción) |
| Sensor 3 | Digital | Pin 4 (lectura por sondeo) |

Todos los sensores comparten **VCC → 5V** y **GND → GND** del Arduino.

> El Arduino Uno solo tiene dos pines con interrupción externa (2 y 3). El tercer sensor se lee por sondeo (`digitalRead` en el `loop()`), lo cual es suficientemente rápido para velocidades típicas de este experimento. Si tu placa tiene más pines de interrupción (ej. Mega, Leonardo), puedes usar interrupciones en los tres.

---

## 6. Código Arduino

```cpp
// Medición de aceleración en MRUA con 3 sensores IR de barrera
// v1 = d12 / (t2 - t1)      -> velocidad media en el primer tramo
// v2 = d23 / (t3 - t2)      -> velocidad media en el segundo tramo
// a  = (v2 - v1) / (tm2 - tm1)  -> usando tiempos medios de cada tramo

const byte pinSensor1 = 2; // interrupción
const byte pinSensor2 = 3; // interrupción
const byte pinSensor3 = 4; // sondeo (digitalRead)

// Distancias reales entre sensores (AJUSTAR a tu montaje)
const float d12 = 0.30; // metros
const float d23 = 0.30; // metros

volatile unsigned long t1 = 0, t2 = 0;
volatile bool marco1 = false, marco2 = false;
unsigned long t3 = 0;
bool marco3 = false;

void ISR_sensor1() {
  if (!marco1) { t1 = micros(); marco1 = true; }
}

void ISR_sensor2() {
  if (marco1 && !marco2) { t2 = micros(); marco2 = true; }
}

void setup() {
  Serial.begin(9600);
  pinMode(pinSensor1, INPUT);
  pinMode(pinSensor2, INPUT);
  pinMode(pinSensor3, INPUT);
  attachInterrupt(digitalPinToInterrupt(pinSensor1), ISR_sensor1, FALLING);
  attachInterrupt(digitalPinToInterrupt(pinSensor2), ISR_sensor2, FALLING);
  Serial.println("Sistema listo. Suelta el objeto desde la parte alta del riel.");
}

void loop() {
  // Sondeo del tercer sensor, solo válido después de pasar por el 2do
  if (marco2 && !marco3) {
    if (digitalRead(pinSensor3) == LOW) { // ajustar según polaridad del módulo (LOW = objeto detectado en la mayoría de FC-51)
      t3 = micros();
      marco3 = true;
    }
  }

  if (marco1 && marco2 && marco3) {
    float t12 = (t2 - t1) / 1000000.0; // s
    float t23 = (t3 - t2) / 1000000.0; // s

    float v1 = d12 / t12; // velocidad media tramo 1
    float v2 = d23 / t23; // velocidad media tramo 2

    // tiempos medios de cada tramo, referidos al instante t1 = 0
    float tm1 = t12 / 2.0;
    float tm2 = t12 + (t23 / 2.0);

    float aceleracion = (v2 - v1) / (tm2 - tm1);

    Serial.println("---------------------------------------");
    Serial.print("t12: "); Serial.print(t12, 4); Serial.println(" s");
    Serial.print("t23: "); Serial.print(t23, 4); Serial.println(" s");
    Serial.print("v1 (tramo 1): "); Serial.print(v1, 3); Serial.println(" m/s");
    Serial.print("v2 (tramo 2): "); Serial.print(v2, 3); Serial.println(" m/s");
    Serial.print("Aceleracion: "); Serial.print(aceleracion, 3); Serial.println(" m/s^2");
    Serial.println("---------------------------------------");

    delay(2000);
    marco1 = false;
    marco2 = false;
    marco3 = false;
  }
}
```

---

## 7. Procedimiento de laboratorio

1. Mide el ángulo θ del riel y las distancias d₁₂ y d₂₃. Regístralas.
2. Carga el código y abre el Monitor Serie a 9600 baudios.
3. Suelta el objeto **sin impulso inicial** desde el punto de partida, justo antes del Sensor 1.
4. Repite el experimento **al menos 6 veces** con el mismo ángulo.
5. Registra t₁₂, t₂₃, v₁, v₂ y la aceleración calculada por el Arduino en cada repetición.
6. Cambia el ángulo θ (increméntalo) y repite el proceso completo, para comparar cómo cambia la aceleración medida.
7. Calcula la aceleración teórica esperada: **a_teórica = g · sen(θ)**, con g = 9.81 m/s², y compárala contra el promedio medido.

### Tabla de registro sugerida (por cada ángulo θ)

| Intento | t₁₂ (s) | t₂₃ (s) | v₁ (m/s) | v₂ (m/s) | a medida (m/s²) |
|---|---|---|---|---|---|
| 1 | | | | | |
| 2 | | | | | |
| 3 | | | | | |
| ... | | | | | |
| Promedio | — | — | — | — | |
| a teórica (g·senθ) | | | | | |
| % de error | | | | | |

---

## 8. Preguntas de análisis para el estudiante

1. ¿Por qué con solo dos sensores no es posible calcular la aceleración, únicamente una velocidad media?
2. Compara la aceleración medida con la teórica (g·senθ). ¿Qué factores explican la diferencia (fricción, alineación, resistencia del aire, error de medición angular)?
3. Si duplicas el ángulo θ, ¿la aceleración medida se duplica también? Verifícalo con datos y explica si la relación es lineal.
4. ¿Qué pasaría con la precisión del experimento si los tramos d₁₂ y d₂₃ fueran muy cortos? ¿Y si fueran muy largos?
5. Usando v₁, v₂ y la aceleración medida, calcula la velocidad teórica esperada en el Sensor 3 con v = v₀ + a·t y compárala contra v₂. ¿Coinciden?
6. Grafica manualmente (o en hoja de cálculo) v vs. t con los dos puntos (tm1, v1) y (tm2, v2). ¿Qué representa la pendiente de esa recta?

---

## 9. Extensiones posibles

- Agregar un cuarto sensor para obtener tres velocidades y verificar que la aceleración se mantiene aproximadamente constante entre todos los tramos (confirmando que el movimiento es MRUA y no otro tipo).
- Registrar los datos en tarjeta microSD y graficar posición vs. tiempo (debe verse una parábola) y velocidad vs. tiempo (debe verse una línea recta).
- Repetir el experimento con distintas masas sobre el carrito para discutir si la aceleración depende de la masa en un plano inclinado sin fricción apreciable (debería ser prácticamente independiente, reforzando la segunda ley de Newton en este caso particular).
- Combinar con la práctica de MRU previa: comparar cualitativamente las gráficas x-t y v-t de ambos movimientos.

---

## 10. Fuentes de error a discutir en clase

- **Medición del ángulo θ**: pequeños errores angulares generan errores considerables en a_teórica, ya que a = g·sen(θ) es sensible cerca de ángulos pequeños.
- **Fricción del riel y del objeto**: reduce la aceleración medida respecto a la teórica; es normal que a_medida < a_teórica.
- **Sondeo del tercer sensor**: al no usar interrupción, existe un retardo mínimo asociado a la velocidad del `loop()`; discutir por qué sigue siendo válido para las velocidades típicas de este experimento.
- **Impulso inicial accidental**: si el objeto se suelta con un pequeño empujón involuntario, v₀ ya no es cero y esto afecta el análisis si se compara con x = ½at² (que asume v₀ = 0).

# Guía práctica: Medición de velocidad en MRU con Arduino y dos sensores

## Objetivo de aprendizaje
Que el estudiante determine experimentalmente la velocidad de un objeto en Movimiento Rectilíneo Uniforme (MRU) midiendo el tiempo que tarda en recorrer una distancia conocida entre dos sensores, aplicando la relación:

**v = d / t**

donde *d* es la distancia fija entre los sensores (en metros) y *t* es el intervalo de tiempo entre las detecciones (en segundos).

---

## 1. Fundamento físico

En un MRU la velocidad es constante, por lo que basta con medir el tiempo que tarda un objeto en recorrer un tramo de longitud conocida para obtener su rapidez promedio en ese tramo:

- Si el objeto realmente se mueve a velocidad constante, la velocidad calculada será representativa de todo el trayecto.
- Si existe aceleración, lo que se obtiene es la **velocidad media** entre los dos puntos, útil para introducir la diferencia entre velocidad instantánea y media.

Este montaje también permite discutir fuentes de error: tiempo de respuesta del sensor, ancho del objeto, alineación de los sensores, y resolución temporal del microcontrolador.

---

## 2. Materiales

| Cantidad | Componente |
|---|---|
| 1 | Arduino Uno (o compatible) |
| 2 | Sensores ultrasónicos HC-SR04 **o** 2 sensores infrarrojos de barrera (tipo FC-51 / módulo IR obstáculo) |
| 1 | Protoboard |
| — | Cables jumper macho-macho y macho-hembra |
| 1 | Riel o pista recta (guía para el objeto, ej. canaleta de PVC o riel de aluminio) |
| 1 | Cinta métrica o regla |
| 1 | Objeto móvil (carrito, esfera, deslizador) |
| 1 | Cable USB / fuente de alimentación |
| Opcional | Módulo LCD I2C o monitor serie del PC para lectura de datos |

> **Nota sobre elección de sensor:** el HC-SR04 es más preciso para medir presencia por reflexión de onda (ideal si el objeto es sólido y perpendicular al haz), mientras que los sensores IR de barrera son más simples y confiables cuando el objeto interrumpe físicamente un haz de luz (menos sensible al ángulo del objeto). Para principiantes, **los sensores IR de barrera son más fáciles de calibrar en este experimento**.

---

## 3. Montaje físico

1. Coloca el riel o pista en una superficie horizontal y nivelada.
2. Fija el **Sensor 1** en un extremo del riel y el **Sensor 2** a una distancia *d* conocida (por ejemplo, 50 cm). Mide esta distancia con la mayor precisión posible; es la fuente de error más importante del experimento.
3. Asegúrate de que ambos sensores estén a la misma altura y orientados de forma idéntica respecto al riel.
4. El objeto móvil debe pasar frente a ambos sensores en línea recta, sin desviarse.

```
   [Sensor 1] -------- distancia d -------- [Sensor 2]
        |                                        |
   (detecta entrada)                    (detecta salida)
        └──────────── trayectoria del objeto ─────────►
```

---

## 4. Conexiones eléctricas

### Opción A — Sensores ultrasónicos HC-SR04

| HC-SR04 #1 | Arduino | HC-SR04 #2 | Arduino |
|---|---|---|---|
| VCC | 5V | VCC | 5V |
| GND | GND | GND | GND |
| Trig | Pin 9 | Trig | Pin 11 |
| Echo | Pin 10 | Echo | Pin 12 |

### Opción B — Sensores infrarrojos de barrera (salida digital)

| IR #1 | Arduino | IR #2 | Arduino |
|---|---|---|---|
| VCC | 5V | VCC | 5V |
| GND | GND | GND | GND |
| OUT | Pin 2 (interrupción) | OUT | Pin 3 (interrupción) |

---

## 5. Código Arduino

### Opción A — Con sensores ultrasónicos HC-SR04

```cpp
// Medición de velocidad en MRU con 2 sensores HC-SR04
// v = d / t

const int trigPin1 = 9,  echoPin1 = 10;
const int trigPin2 = 11, echoPin2 = 12;

const float distanciaSensores = 0.50; // metros (AJUSTAR a tu montaje)
const float umbralDeteccion = 15.0;   // cm: distancia por debajo de la cual se considera "objeto presente"

unsigned long tiempoSensor1 = 0;
unsigned long tiempoSensor2 = 0;
bool detectado1 = false;
bool detectado2 = false;

float medirDistanciaCM(int trigPin, int echoPin) {
  digitalWrite(trigPin, LOW);
  delayMicroseconds(2);
  digitalWrite(trigPin, HIGH);
  delayMicroseconds(10);
  digitalWrite(trigPin, LOW);
  long duracion = pulseIn(echoPin, HIGH, 30000); // timeout 30 ms
  if (duracion == 0) return 999; // sin eco = "vacío"
  return duracion * 0.0343 / 2.0; // cm
}

void setup() {
  Serial.begin(9600);
  pinMode(trigPin1, OUTPUT); pinMode(echoPin1, INPUT);
  pinMode(trigPin2, OUTPUT); pinMode(echoPin2, INPUT);
  Serial.println("Sistema listo. Distancia entre sensores: " + String(distanciaSensores) + " m");
}

void loop() {
  float d1 = medirDistanciaCM(trigPin1, echoPin1);
  float d2 = medirDistanciaCM(trigPin2, echoPin2);

  // Detección en sensor 1
  if (d1 < umbralDeteccion && !detectado1) {
    detectado1 = true;
    tiempoSensor1 = micros();
    Serial.println("Objeto detectado en Sensor 1");
  }

  // Detección en sensor 2 (solo cuenta si ya pasó por el 1)
  if (d2 < umbralDeteccion && detectado1 && !detectado2) {
    detectado2 = true;
    tiempoSensor2 = micros();

    float deltaT = (tiempoSensor2 - tiempoSensor1) / 1000000.0; // segundos
    float velocidad = distanciaSensores / deltaT;

    Serial.println("Objeto detectado en Sensor 2");
    Serial.print("Tiempo transcurrido: "); Serial.print(deltaT, 4); Serial.println(" s");
    Serial.print("Velocidad calculada: "); Serial.print(velocidad, 3); Serial.println(" m/s");
    Serial.println("--------------------------------------");

    // Reiniciar para la siguiente medición
    delay(1500);
    detectado1 = false;
    detectado2 = false;
  }

  delay(20); // pequeña pausa entre lecturas
}
```

### Opción B — Con sensores IR de barrera (más precisa, usa interrupciones)

```cpp
// Medición de velocidad en MRU con 2 sensores IR de barrera
// Usa interrupciones para mayor precisión temporal

const byte pinSensor1 = 2; // debe ser pin de interrupción
const byte pinSensor2 = 3; // debe ser pin de interrupción
const float distanciaSensores = 0.50; // metros (AJUSTAR)

volatile unsigned long t1 = 0;
volatile unsigned long t2 = 0;
volatile bool marco1 = false;
volatile bool marco2 = false;

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
  attachInterrupt(digitalPinToInterrupt(pinSensor1), ISR_sensor1, FALLING);
  attachInterrupt(digitalPinToInterrupt(pinSensor2), ISR_sensor2, FALLING);
  Serial.println("Sistema listo (modo interrupciones).");
}

void loop() {
  if (marco1 && marco2) {
    float deltaT = (t2 - t1) / 1000000.0;
    float velocidad = distanciaSensores / deltaT;

    Serial.print("Delta t: "); Serial.print(deltaT, 4); Serial.println(" s");
    Serial.print("Velocidad: "); Serial.print(velocidad, 3); Serial.println(" m/s");
    Serial.println("--------------------------------------");

    delay(1500);
    marco1 = false;
    marco2 = false;
  }
}
```

> El modo con interrupciones (Opción B) es preferible cuando se busca mayor precisión, ya que elimina el retardo introducido por el `delay(20)` del bucle principal en la Opción A.

---

## 6. Procedimiento de laboratorio

1. Mide y registra la distancia *d* entre los dos sensores (repite la medición 3 veces y promedia).
2. Carga el código correspondiente y abre el Monitor Serie a 9600 baudios.
3. Desliza el objeto por el riel manteniendo, en lo posible, una velocidad constante.
4. Repite el experimento **al menos 8 veces**, variando la velocidad de empuje inicial en algunos intentos.
5. Registra en una tabla el tiempo Δt y la velocidad v que muestra el monitor serie para cada repetición.
6. Calcula el promedio y la desviación estándar de las velocidades obtenidas.

### Tabla de registro sugerida

| Intento | Δt (s) | v = d/t (m/s) |
|---|---|---|
| 1 | | |
| 2 | | |
| 3 | | |
| ... | | |
| Promedio | — | |
| Desv. estándar | — | |

---

## 7. Preguntas de análisis para el estudiante

1. ¿Qué porcentaje de error introduce una imprecisión de ±0.5 cm en la medición de *d* si la distancia real es 50 cm?
2. Si el ancho del objeto es de 3 cm y el umbral de detección no es puntual, ¿cómo afecta esto al tiempo medido?
3. Compara los resultados obtenidos con sensores ultrasónicos y con sensores IR (si se probaron ambos). ¿Cuál dio menor variabilidad y por qué?
4. ¿Qué pasaría con los cálculos de v si el objeto **acelera** entre los dos sensores? ¿Qué tipo de velocidad se estaría midiendo entonces?
5. Propón una modificación al montaje para medir **aceleración** en lugar de velocidad constante (pista: usar 3 sensores).

---

## 8. Extensiones posibles

- Agregar una pantalla LCD I2C para mostrar la velocidad sin necesidad de una computadora conectada.
- Registrar los datos en una tarjeta microSD para análisis posterior en hoja de cálculo.
- Ampliar a 3 sensores para calcular aceleración (MRUA) comparando dos velocidades consecutivas.
- Graficar posición vs. tiempo y velocidad vs. tiempo usando los datos exportados al monitor serie.

---

## 9. Fuentes de error a discutir en clase

- **Alineación de los sensores**: si no están perfectamente perpendiculares a la trayectoria, la distancia efectiva cambia.
- **Umbral de detección**: un umbral mal calibrado detecta el objeto antes o después del punto real.
- **Fricción y resistencia del aire**: pueden hacer que el movimiento no sea perfectamente uniforme, introduciendo diferencias entre repeticiones.
- **Resolución temporal**: `micros()` en Arduino Uno tiene una resolución de 4 µs, insignificante frente a los tiempos típicos del experimento (decenas a cientos de milisegundos).

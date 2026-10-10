# Plan de Implementación: Erradicación del Ruido de Interferencia al Acelerar a Fondo

## 1. Diagnóstico Técnico Exhaustivo del Problema

Tras analizar la respuesta del usuario (*"Al acelerar a fondo y subir de revoluciones"* y *"Conservar el sonido que tiene ahora"*), el análisis físico y de procesamiento de señales en Web Audio API determinó el origen exacto del ruido que suena a **interferencia de radio / estática**:

### Causas Fundamentales Identificadas:
1. **Desbordamiento de Rango Dinámico (Gain Staging Desequilibrado)**:
   - Al acelerar a fondo (`throttle = 1.0`, `rpmNorm \to 1.0`), los 4 osciladores del motor (`oscBase`, `oscHarmonic1`, `oscHarmonic2`, `oscIntake`) suman sus ganancias hasta superar un valor pico combinado de **4.25**.
   - Con la pre-ganancia del `WaveShaper` en `0.24`, las amplitudes superan el rango unitario $[-1.0, 1.0]$. En Web Audio API, cualquier muestra que exceda $\pm 1.0$ sufre un **recorte abrupto (hard-clipping)** en los bordes de la tabla de ondas.
2. **Cascada de Ganancia Resonante Excesiva (+27 dB)**:
   - Tras el WaveShaper, la señal pasa por un post-gain de `3.6x`, seguido de dos filtros resonantes en cascada (`exhaustResonance` subiendo hasta $+11\text{ dB}$ y `exhaustBassPunch` subiendo hasta $+13\text{ dB}$).
   - La amplificación total acumulada supera los **75x**, lo que envía una señal de más de $+35\text{ dBFS}$ al compresor y al nodo de destino de audio (`ctx.destination`).
3. **Intermodulación y Bombeo en el Compresor Dinámico**:
   - El `DynamicsCompressor` configurado con umbral en $-20\text{ dB}$ y ataque rápido de $12\text{ ms}$ entra en saturación forzada de más de $-40\text{ dB}$ de reducción. A frecuencias de motor altas (150–300 Hz), el compresor modula la amplitud de forma discontinua en cada ciclo, creando **bandas laterales de modulación de amplitud (AM)** que son indistinguibles de una interferencia de radiofrecuencia VHF/AM.
4. **Filtro de Admisión (*Airbox Intake Sawtooth*)**:
   - `oscIntake` inyecta una onda de diente de sierra cruda que escala drásticamente con `throttleSq`. Los armónicos infinitos de alta frecuencia de la sierra sin antialiasing, combinados en el saturador con los otros 3 osciladores, generan frecuencias fantasma de intermodulación (*heterodyne beats*).

---

## 2. Solución Propuesta (Manteniendo el 100% del Carácter y Timbre del Motor)

El objetivo es preservar el timbre y presencia visceral que el usuario aprecia, eliminando estrictamente el artefacto de interferencia.

### Acciones Técnicas:
1. **Calibración del Gain Staging Pre-Distorsión**:
   - Escalar las amplitudes relativas de entrada para que la suma máxima en aceleración a fondo permanezca estrictamente en $[-0.85, 0.85]$, evitando el choque contra los límites del `WaveShaper`.
2. **Función de Transferencia Monótona y Suave en el `WaveShaper`**:
   - Reemplazar la componente senoidal secundaria por una saturación asintótica suave tipo tanh normalizada con continuidad $C^1$ en los extremos, eliminando cualquier discontinuidad o *aliasing* armónico.
3. **Rebalanceo de Ganancias en la Cadena Resonante**:
   - Atenuar el post-gain del distorsionador a un nivel calibrado y moderar el Q de los picos resonantes para que el nivel pico que ingresa al compresor esté entre $-6\text{ dB}$ y $0\text{ dBFS}$.
4. **Optimización del Compresor Dinámico**:
   - Elevar el umbral a $-12\text{ dB}$, ajustar el ataque a $24\text{ ms}$ (evitando modulación cíclica intra-onda a frecuencias de audio) y un *release* transparente de $140\text{ ms}$.
5. **Filtrado Suave de Armónicos de Admisión (`oscIntake`)**:
   - Limitar la ganancia máxima de la sierra de admisión y añadir un corte paso-bajo suave a 1.200 Hz para evitar que armónicos ultrasónicos generen ruido de radio al saturar.
6. **Protección Anti-Clipping en Master**:
   - Asegurar que el `masterGain` opere en una ventana segura sin recorte a nivel de hardware/DAC.

---

## 3. Plan de Pruebas y Validación

1. **Prueba de aceleración estática en punto muerto**:
   - Acelerar de 1.000 a 9.500 RPM sin carga; verificar que no haya chisporroteo ni ruido blanco de fondo.
2. **Prueba de aceleración en pista a fondo (WOT - Wide Open Throttle)**:
   - Conducir en 1ª, 2ª, 3ª y 4ª marcha con acelerador al 100%; verificar que el rugido del motor se mantenga limpio, potente y continuo, sin ningún ruido similar a estática o radio.
3. **Verificación de corte de inyección (*Rev Limiter*)**:
   - Alcanzar el límite de revoluciones para confirmar que el corte staccato funcione con precisión sin saturar el bus principal.
4. **Compilación y verificación**:
   - Ejecutar `compile_applet` y validar que no existan regresiones de tipado ni errores en tiempo de ejecución.

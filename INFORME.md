Alumno: Sotelo Benjamin

# Informe — Sincronización de relojes en sistemas distribuidos

**Asignatura:** Sistemas Operativos / Sistemas Distribuidos  
**Tecnologías:** C#, ASP.NET Core Web API, .NET 10, Docker Compose  
**Alcance:** los relojes de Cristian y Berkeley se corrigen **solo en la aplicación**. No se modifica la hora del sistema operativo ni del contenedor.


## Cómo ejecutar

Docker Desktop debe estar iniciado. Los ejercicios **Berkeley** y **Lamport** no deben correr al mismo tiempo: ambos usan los puertos `8081`, `8082` y `8083`.

```powershell
# Ejercicio 1
cd "C:\Users\Benja\Desktop\caso practico sincronizacion\CristianSync"
docker compose up --build

# Ejercicio 2 (en otra terminal, con el 1 detenido si hay conflicto de CPU)
cd "C:\Users\Benja\Desktop\caso practico sincronizacion\BerkeleyLab"
docker compose up -d --build
docker compose logs -f coordinator
Invoke-RestMethod http://localhost:8081/api/clock
Invoke-RestMethod http://localhost:8082/api/clock
Invoke-RestMethod http://localhost:8083/api/clock

# Ejercicio 3
cd "C:\Users\Benja\Desktop\caso practico sincronizacion\LamportPedidos"
docker compose up -d --build
$body = @{ orderId = "ORD-001"; productId = "P001"; amount = 1500 } | ConvertTo-Json
Invoke-RestMethod -Method Post -Uri "http://localhost:8081/api/orders" -ContentType "application/json" -Body $body
docker compose logs --timestamps orders-service inventory-service payments-service
```

Para detener: `docker compose down` en la carpeta del ejercicio.

---

## Preguntas teóricas del PDF

### 1. ¿Cuál es la diferencia entre un reloj físico y un reloj lógico?

Un **reloj físico** mide tiempo real (segundos, UTC). En esta práctica se simula con `DateTimeOffset.UtcNow` más un offset. Un **reloj lógico** es un contador de eventos (Lamport): no representa hora del día, solo orden relativo. Cristian y Berkeley ajustan relojes físicos simulados; Lamport no sincroniza hora, ordena eventos.

### 2. ¿Qué es la deriva de un reloj y cómo afecta a los sistemas distribuidos?

La **deriva** (*clock drift*) es el desvío de un oscilador respecto de una referencia. Aunque dos relojes se sincronicen, vuelven a apartarse. En un sistema distribuido eso produce marcas de tiempo inconsistentes, timeouts erróneos y un orden cronológico incorrecto de operaciones.

### 3. ¿Qué es el Tiempo Atómico Internacional (TAI) y cuál es su relación con UTC?

**TAI** es una escala continua basada en relojes atómicos. **UTC** se deriva de TAI e inserta *leap seconds* para seguir la rotación terrestre. Los sistemas informáticos usan UTC (o offsets sobre UTC). Esta práctica trabaja con UTC para evitar husos horarios.

### 4. ¿Por qué resulta difícil establecer un orden global de eventos usando solo relojes físicos?

No hay un reloj global. Cada nodo tiene su propio reloj, con offset y deriva. La latencia de red impide observar el instante exacto de un evento remoto. Dos marcas físicas no son comparables con precisión milimétrica; eventos concurrentes pueden aparecer invertidos.

### 5. ¿Cómo funciona el algoritmo de Cristian? Ventajas y limitaciones.

El cliente envía una petición al servidor de tiempo, mide RTT (`t1 - t0`) y estima la hora actual como `Ts + RTT/2`. Luego corrige su reloj.  
**Ventajas:** simple, un único servidor de referencia, adecuado si hay una fuente de tiempo confiable.  
**Limitaciones:** asume ida ≈ vuelta; un RTT alto o asimétrico degrada la precisión; el servidor es un punto único de falla; no promedia varios nodos.

### 6. ¿Cómo funciona Berkeley? ¿Qué función cumple el coordinador?

El **coordinador** consulta la hora de cada participante, estima desfases (con RTT/2), calcula un **promedio** (incluye su propio reloj, diferencia 0) y envía a cada nodo la corrección `promedio - desfase_i`. No hay un servidor de tiempo “verdadero”: se busca un consenso interno. El coordinador orquesta la ronda; sin él no hay nuevas correcciones.

### 7. ¿Qué diferencias existen entre Cristian y Berkeley?

| | Cristian | Berkeley |
|---|---|---|
| Referencia | Servidor de tiempo externo | Promedio del conjunto |
| Topología | Cliente–servidor | Coordinador + participantes |
| Uso típico | Hay una fuente de hora confiable | Intranet sin UTC externo |
| Falla del centro | Los clientes no se sincronizan | Los nodos quedan con el último offset |

### 8. ¿Qué es la relación happens-before de Lamport?

`a → b` (*a sucede antes que b*) si: (1) ambos ocurren en el mismo proceso y `a` precede a `b`; (2) `a` es el envío de un mensaje y `b` su recepción; o (3) hay transitividad. Si `a → b` entonces `L(a) < L(b)`.

### 9. ¿Cómo funcionan los relojes lógicos de Lamport?

Cada proceso mantiene `L`:

1. Evento interno: `L = L + 1`
2. Envío: `L = L + 1` y se incluye `L` en el mensaje (`X-Lamport-Time`)
3. Recepción con marca `T`: `L = max(L, T) + 1`

### 10. ¿Qué significa que dos eventos sean concurrentes?

No hay camino causal entre ellos: ni `a → b` ni `b → a`. Pueden ocurrir en paralelo en procesos distintos. Lamport puede asignarles marcas distintas; eso **no** implica causalidad.

### 11. ¿Qué diferencias existen entre sincronizar relojes físicos y establecer un orden lógico?

La sincronización física acerca las horas reales (Cristian, Berkeley, NTP). El orden lógico (Lamport, vectores) no corrige UTC: garantiza que si hay causalidad, las marcas crecen. Sirve para logs, replicación y consistencia, no para “qué hora es”.

### 12. ¿Cómo funciona NTP y qué problemas permite resolver?

NTP consulta varios servidores, filtra muestras de alta dispersión, estima offset y delay, y ajusta el reloj de forma gradual. Resuelve deriva, elección de fuentes y, en parte, asimetría. Esta práctica es un prototipo didáctico: una muestra, sin filtros NTP ni slewing.

---

## Ejercicio 1 — Algoritmo de Cristian

**Caso:** tres servidores de registro (A, B, C) y un servidor de tiempo. Offsets simulados: A = 0 s, B = +8 s, C = −5 s (equivalente al enunciado 10:00:00, 10:00:08, 09:59:55 respecto de una base común).

**Fórmula:** `RTT = t1 − t0`, `T_estimado = Ts + RTT/2`, `corrección = T_estimado − reloj_local_simulado`.

### Resultados esperados (tras la primera muestra válida)

| Cliente | Desfase inicial (s) | Corrección aproximada | Después de sincronizar |
|---|---:|---|---|
| A (`client-a`) | 0 | ~0 ms (solo RTT/2 y jitter) | Cerca de la hora del servidor |
| B (`client-b`) | +8 | ≈ −8000 ms | Cerca de la hora del servidor |
| C (`client-c`) | −5 | ≈ +5000 ms | Cerca de las muestras siguientes el residual es del orden del RTT |

Las muestras 2 a 5 deben mostrar correcciones pequeñas (milisegundos), no segundos. Completar la tabla con `docker compose logs client-a client-b client-c`.

### Preguntas del ejercicio 1 (PDF + guía)

**¿Cómo influye la latencia de red en la precisión?**  
El error máximo de Cristian es del orden de RTT/2. Más latencia ⇒ más incertidumbre. Si la ida y la vuelta no son iguales, `RTT/2` sesga la estimación.

**¿Qué sucede si el RTT es muy elevado?**  
La corrección puede estar varios cientos de milisegundos (o más) fuera. Un sistema real descartaría esas muestras (NTP usa umbrales de delay/dispersión).

**¿Qué ocurre si el servidor deja de responder?**  
El cliente captura la excepción, registra el error y no ajusta el reloj. El offset simulado se queda en el último valor válido.

**Otras de la guía**

1. Hora física = UTC del host/contenedor. Reloj simulado = UTC + offset de aplicación.  
2. Se usa RTT/2 porque no se observa por separado ida y vuelta.  
3. Latencia alta empeora la cota de error.  
4. RTT elevado → muestra de mala calidad.  
5. Servidor caído → no hay sync; el cliente sigue con su offset.  
6. `Stopwatch` mide intervalo monotónico; no se ve afectado si se “moviera” el reloj de pared.  
7. Asimetría: el error puede acercarse a todo el RTT.  
8. Cambiar el reloj del SO afecta procesos, logs y certificados; además requiere privilegios.  
9. NTP usa varias fuentes, filtros, compensación gradual y autenticación.  
10. Descartar RTT por encima de un umbral o quedarse con el mínimo RTT de N muestras.

---

## Ejercicio 2 — Algoritmo de Berkeley

Offsets iniciales: coordinador 0 ms, nodo1 +5000 ms, nodo2 +10000 ms, nodo3 −20000 ms.

Promedio de diferencias (incluyendo coordinador = 0):

`(0 + 5000 + 10000 + (−20000)) / 4 = −1250 ms`

Correcciones:

| Nodo | Offset inicial (ms) | Diferencia vs coordinador | Corrección = promedio − diferencia | Offset final teórico |
|---|---:|---:|---:|---:|
| Coordinador | 0 | 0 | −1250 | −1250 |
| Nodo 1 | 5000 | ~5000 | −6250 | −1250 |
| Nodo 2 | 10000 | ~10000 | −11250 | −1250 |
| Nodo 3 | −20000 | ~−20000 | +18750 | −1250 |

Tras una ronda, los cuatro relojes simulados convergen cerca de −1250 ms respecto de UTC del host (pequeño error por RTT). Verificar con `Invoke-RestMethod` y logs del coordinador.

### Pruebas

- **Nodo atípico (5 min = 300000 ms):** con `OUTLIER_THRESHOLD_MS=60000` el nodo 3 se descarta. El promedio usa coordinador + nodo1 + nodo2: `(0+5000+10000)/3 ≈ 5000`. El atípico no arrastra al resto.  
- **Falla de un participante:** el coordinador registra “no disponible” y promedia los restantes.  
- **Falla del coordinador:** los nodos conservan el último offset; no hay nuevas rondas. Tolerancia: elección de líder (Bully, Raft) + coordinador de respaldo.

### Preguntas del ejercicio 2

Berkeley no usa un tiempo “verdadero”; Cristian sí. Un reloj extremo distorsiona el promedio salvo que se filtre. Sin coordinador no hay algoritmo. Mejoras reales: autenticación de `/adjust`, estimación de incertidumbre, líder electo, más de una ronda de medición.

---

## Ejercicio 3 — Relojes lógicos de Lamport

Flujo: pedido → inventario → pago → confirmación. Marca en encabezado `X-Lamport-Time`.

### Tabla de eventos (un pedido en frío, relojes en 0)

| Proceso | Evento | Operación | Lamport |
|---|---|---|---:|
| Orders | Pedido recibido | `Tick()` | 1 |
| Orders | Consulta inventario enviada | `Tick()` | 2 |
| Inventory | Consulta recibida | `Receive(2)` | 3 |
| Inventory | Inventario verificado | `Tick()` | 4 |
| Inventory | Respuesta enviada | `Tick()` | 5 |
| Orders | Respuesta inventario recibida | `Receive(5)` | 6 |
| Orders | Solicitud de pago enviada | `Tick()` | 7 |
| Payments | Pago recibido | `Receive(7)` | 8 |
| Payments | Pago procesado | `Tick()` | 9 |
| Payments | Respuesta enviada | `Tick()` | 10 |
| Orders | Respuesta pago recibida | `Receive(10)` | 11 |
| Orders | Pedido confirmado | `Tick()` | 12 |

Respuesta HTTP esperada: `"status": "Confirmed", "lamport": 12`. Un segundo pedido (`ORD-002`) **no** arranca en 1: los contadores siguen (Orders ~13, etc.).

### Experimentos (guía)

- **A — Segundo pedido:** los relojes no vuelven a cero porque viven en memoria del proceso.  
- **D — Falla de Payments:** el pedido se corta tras inventario; Orders puede devolver 502/500. Al restaurar, Payments arranca en 0 si el contenedor se recreó, o continúa si solo se pausó.  
- **Reinicio de Inventory:** su `L` vuelve a 0; Orders y Payments no. Las marcas dejan de ser comparables entre ejecuciones.

### Preguntas del ejercicio 3 (PDF)

**¿Por qué Lamport no representa la hora real?** Es un contador de eventos, no un oscilador.  
**¿Qué ocurre si llega un T mayor?** `L = max(L, T) + 1`, el receptor se adelanta al emisor.  
**¿Cómo mantiene el orden causal?** Toda cadena happens-before incrementa marcas a lo largo del camino de mensajes.  
**¿Se puede saber concurrencia solo con Lamport?** No de forma completa: `L(a) < L(b)` no implica `a → b`. Los relojes vectoriales sí distinguen concurrencia.

---

## Ejercicio 4 — Comparación

| Característica | Cristian | Berkeley | Lamport |
|---|---|---|---|
| Tipo de reloj | Físico simulado | Físico simulado | Lógico (contador) |
| Objetivo principal | Ajustar al servidor de tiempo | Acordar una hora media del grupo | Orden causal de eventos |
| Requiere servidor central | Sí (time server) | Sí (coordinador) | No |
| Utiliza mensajes | Request/response de hora | Consulta + ajuste | Cada HTTP lleva `X-Lamport-Time` |
| Sincroniza hora física | Sí (aproximada) | Sí (promedio interno) | No |
| Establece orden causal | No | No | Sí (`a → b` ⇒ `L(a) < L(b)`) |
| Comportamiento ante fallas | Sin servidor no hay sync | Sin coordinador no hay rondas; un nodo caído se omite | Un servicio caído corta el flujo; el reloj de los demás sigue |

### Respuestas del ejercicio 4

1. **Varios servidores de una organización, sin UTC externo:** Berkeley. **Con fuente de hora confiable (o NTP):** Cristian / NTP.  
2. **Orden causal entre microservicios:** Lamport (o vectores si hace falta detectar concurrencia).  
3. **Física vs lógica:** física alinea instantes reales; lógica ordena causa-efecto sin coincidir en UTC.  
4. **¿Combinar?** Sí. NTP/Berkeley para logs y certificados; Lamport/vectores para el protocolo de negocio (pedidos, replicación). Son problemas distintos y complementarios.

---

## Conclusión

Cristian demuestra estimación por RTT frente a un servidor de tiempo. Berkeley muestra consenso interno y el rol (y fragilidad) del coordinador. Lamport muestra que el orden de un pedido distribuido se puede preservar aunque las horas físicas impresas no coincidan. Ningún prototipo sustituye NTP ni un bus de mensajes de producción; sí permiten observar, con Docker, los tres mecanismos pedidos en el caso práctico.

## Bibliografía

- Tanenbaum y Van Steen, *Distributed Systems: Principles and Paradigms*, 2.ª ed.  
- Coulouris et al., *Distributed Systems: Concepts and Design*.  
- Documentación ASP.NET Core y Docker Compose.

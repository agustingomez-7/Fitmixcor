# Revisión rápida de tu firmware (ESP32 + HX711 + relé)

Tu arquitectura general está bien (máquina de estados, endpoints HTTP y tests por Serial), pero los dos síntomas que describes son clásicos:

1. **Primer pesaje exagerado (ej. 600 g al primer ciclo):**
   - suele venir de `tare()` demasiado pronto, sin calentamiento del HX711,
   - lectura inicial sin descartar muestras “sucias”,
   - ruido EMI por motor/relé/servos justo antes de pesar.

2. **Ruido fuerte al prender/apagar relé:**
   - si el relé está conmutando en parpadeo o múltiples cambios de estado, mete interferencia.
   - solución: usarlo **solo en evento** (verde fijo 8 s al tocar FDC2), sin blink continuo.

---

## Cambios recomendados (directo al código)

### 1) Arranque robusto del HX711 (evita primer lectura loca)

En `setup()`, justo después de `balanza.begin(DT, SCK)`:

```cpp
// Warm-up HX711
unsigned long t0 = millis();
while (!balanza.is_ready() && (millis() - t0) < 3000) {
  delay(10);
}

// Descartar lecturas iniciales inestables
for (int i = 0; i < 12; i++) {
  if (balanza.is_ready()) (void)balanza.get_units(1);
  delay(80);
}

balanza.set_scale(factor_esc);
balanza.tare(20);   // mejor que tare() simple
```

> Extra recomendado: al comenzar un nuevo pedido, hacer una mini-retara (`tare(10)`) con tolva quieta.

---

### 2) Lectura filtrada (mediana + promedio)

Sustituye bloque de pesado por función robusta:

```cpp
float leerPesoFiltrado() {
  const int N = 9;
  float v[N];
  int n = 0;

  for (int i = 0; i < N; i++) {
    if (!balanza.is_ready()) continue;
    v[n++] = balanza.get_units(1);
    delay(35);
  }

  if (n < 5) return NAN;

  // sort simple
  for (int i = 0; i < n - 1; i++)
    for (int j = i + 1; j < n; j++)
      if (v[j] < v[i]) { float t = v[i]; v[i] = v[j]; v[j] = t; }

  // promedio central (recorta extremos)
  int ini = n / 4;
  int fin = n - ini;
  float suma = 0;
  int c = 0;
  for (int i = ini; i < fin; i++) { suma += v[i]; c++; }

  return (c > 0) ? (suma / c) : NAN;
}
```

Uso en estado `PESANDO`:

```cpp
float p = leerPesoFiltrado();
if (!isnan(p)) pesoActual = p;
```

---

### 3) Congelar actuadores mientras pesa

Cuando entres a `ESPERANDO_ESTABILIZACION` / `PESANDO`:
- motor detenido,
- servos en reposo,
- sin conmutación del relé.

Esto baja mucho el ruido eléctrico y vibración mecánica.

---

### 4) Política del relé (tu requisito: solo verde 8 s al tocar FDC2)

Tu lógica actual de `LED_PARPADEO` conmuta seguido. Si querés evitar ruido, deja el relé en rojo fijo y solo verde 8 s al detectar FDC2.

#### Simplificación sugerida

- Elimina/paraliza `LED_PARPADEO`.
- En `loop()`:

```cpp
// default: rojo fijo
relaySet(false);

// evento one-shot: verde 8s
if (entregaOkActiva) {
  relaySet(true);
  if (millis() - tInicioEntregaOk >= 8000) {
    entregaOkActiva = false;
    relaySet(false);
  }
}
```

- En `ENTREGANDO`, al detectar `FDC_2 == LOW`:

```cpp
entregaOkActiva = true;
tInicioEntregaOk = millis();
```

Con eso, el relé solo conmuta en dos momentos: al entrar/salir de verde.

---

### 5) Evitar bloqueos largos (`while` + `delay`) en entrega

Tus `while` con `delay(50)` durante 20 s funcionan, pero te meten latencia y pueden mezclar tareas sensibles. Conviene migrarlo a subestados no bloqueantes con `millis()` (más estable para web + balanza + IO).

---

## Checklist de hardware (muy importante)

- HX711 y celda de carga con cable corto, trenzado y lejos de motor/relé.
- **Masa en estrella** (separar retornos de potencia y señal).
- Fuente separada o bien filtrada para motor/servos vs lógica/HX711.
- Diodo flyback correcto en bobina de relé (si no es módulo integrado).
- Capacitores cerca de ESP32 y HX711 (100 nF + 10 µF típicos).

---

## Prioridad práctica (qué haría primero)

1. Warm-up + descarte + `tare(20)` del HX711.
2. Quitar parpadeo del relé y dejar one-shot verde 8 s en FDC2.
3. Filtro de lectura robusto (mediana/recorte).
4. Separación eléctrica (fuentes/masas/cableado).

Con esos 4 pasos normalmente desaparece el “primer pesaje loco” y cae muchísimo el ruido por relé.

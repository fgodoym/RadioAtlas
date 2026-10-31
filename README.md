# RadioAtlas

Dispositivo de creación de un poema visual impreso utilizando transmisión de tonos FSK entre dos Raspberry Pi separadas físicamente en la sala. En el receptor el texto se transforma visualmente usando el **Método Berenguer** —inspirado en el libro *Composición de Lugar* de la poeta uruguaya Carmen Berenguer.

---

## Versiones

| Versión | Protocolo | Salida | Contexto | Estado |
|---------|-----------|--------|----------|--------|
| `v1-sstv` | SSTV — imágenes por audio | — | — | Abandonado: canal acústico degrada imagen hasta ilegibilidad |
| `v2-fsk` | FSK 50 baud — texto por audio | Impresora térmica de rollo continuo | **RadioAtlas: la cosmonauta Berenguer · Festival de la Imagen · Manizales · 2026** | Exhibido |
| `v2-fsk + display` | FSK 50 baud — texto por audio | Impresora térmica + pantallas pygame (emisor + receptor) | **RadioAtlas: cartografía y supervivencia · CAC Cerrillos · Santiago · 2026** | En montaje (3 meses) — receptor corregido el 31 Oct 2026 |

---

## Arquitectura

```
PI-A · radioatlasemisor (transmisor)       PI-B · radioatlasreceptor (receptor)
────────────────────────────────           ────────────────────────────────────
frases.txt  (versos del poema)             Allen & Heath ZEDi8 (USB, hw:2,0, 48 kHz)
transmisor_fsk_v2.py                       receptor_fsk_v2.py
→ modulación FSK 1200/2200 Hz             → Goertzel vectorizado (numpy)
→ 50 baud · preámbulo 2s                  → detección de preámbulo por energía
→ aplay → parlante                        → niveles por preámbulo + resta de cola
display_emisor.py                          → búsqueda de deriva + seguimiento de sincronía + CRC
→ telemetría en verde                      → metodo_berenguer_v2.py
→ ASCII art typewriter                     display_receptor.py
                                           → bloque Berenguer + ASCII art

        ──── canal acústico ───►
```

En Manizales la entrada del receptor era una consola Behringer con micrófono (44.1 kHz, 1 canal).

---

## Versión actual — CAC Cerrillos (Santiago)

**RadioAtlas: cartografía y supervivencia**  
Centro de Arte Contemporáneo Cerrillos · Santiago · 2026 · 3 meses

El texto del poema sigue viajando por audio FSK entre las dos Pis. Se mantiene la impresión en papel térmico y se suma una pantalla conectada a cada Pi: el emisor muestra telemetría en tiempo real y el receptor visualiza el Método Berenguer.

**Código instalado, informe técnico y checklist de montaje:** carpeta [`santiago_2026/`](santiago_2026/) —
[informe técnico del 31 Oct 2026](santiago_2026/docs/INFORME_TECNICO_20261031.md) ·
[checklist de montaje](santiago_2026/docs/CHECKLIST_MONTAJE.md).

### Corrección del receptor (31 Oct 2026)

Con el ZEDi8 el CRC fallaba en el 100 % de los paquetes. Medido: no era el ZEDi8 (las bandas de ±25 Hz ya están en el WAV del emisor), sino el canal acústico y el reloj del emisor:

- 2200 Hz llega ~10 dB más fuerte que 1200 Hz;
- cada tono deja una cola de −6 a −10 dB durante ≥40 ms;
- el reloj del emisor corre ~+400 ppm y su deriva no es constante (hasta 8 ms dentro de un paquete).

El receptor ahora empareja niveles con el preámbulo, resta la cola, prueba varias derivas y, si hace falta, sigue la sincronía tramo a tramo. Resultado en vivo: **19 de 19 paquetes con CRC OK** (antes 0 de 7). Detalle en el informe técnico.

### Emisor (Pi-A · radioatlasemisor)

- `transmisor_fsk_v2.py` — FSK 50 baud, deriva + corte de frases, SILENCIO=180s entre envíos
- `display_emisor.py` — pantalla pygame: telemetría verde en tiempo real + ASCII art typewriter al entrar en silencio
- `start_display.sh` — lanzador del display (xinit, xrandr portrait, pygame)
- Servicios systemd: `radioatlas-display.service` + `transmisor.service` (auto-start al encender)
- Audio: `SDL_AUDIODRIVER=dummy` libera ALSA para `aplay`

### Timers display

| Parámetro | Valor | Descripción |
|-----------|-------|-------------|
| `LOG_IDLE_SECS` | 20s | Silencio antes de mostrar imagen |
| `ASCII_BUILD_SECS` | 120s | Tiempo en construir imagen (typewriter) |
| `ASCII_HOLD_SECS` | 180s (emisor) · 60s (receptor) | Imagen completa visible antes de ciclar |

### Receptor (Pi-B · radioatlasreceptor)

- `receptor_fsk_v2.py` — demodulador FSK (entrada ZEDi8 vía `arecord`, S32_LE, 2 canales, canal 0 = micrófono)
- `metodo_berenguer_v2.py` — 10 transformaciones tipográficas
- `display_receptor.py` — pantalla: log en vivo, bloque Berenguer y ASCII art
- Impresora térmica de rollo continuo (papel cae al suelo)
- Servicios systemd: `radioatlas-receptor.service` + `radioatlas-display-receptor.service`
- Registro permanente `~/registro_recepciones.tsv` y audio de fallos en `~/fallos_fsk/`
- `/etc/cron.d/radioatlas-recortar-log` — mantiene `/tmp/receptor.log` (en RAM) por debajo de 1 MB

---

## Método Berenguer

Diez transformaciones tipográficas aplicadas secuencialmente al texto recibido. El índice **nunca se resetea** — la secuencia no se repite aunque el pool de frases dé una vuelta completa.

| idx | transformación | descripción |
|-----|---------------|-------------|
| 0 | `tension` | palabras en línea, una destacada en bold |
| 1 | `fragmentacion` | frase dividida en 2–3 fragmentos a distintos tamaños |
| 2 | `vertical` | letras en columna vertical |
| 3 | `escalada` | palabras que crecen de pequeño a grande |
| 4 | `colapso` | palabras que menguan de grande a pequeño |
| 5 | `pliegue` | primera mitad normal, segunda girada 180° |
| 6 | `cascada_diagonal` | palabras en diagonal por el plano |
| 7 | `palabras_dispersas` | 4 palabras dispersas en espacio blanco |
| 8 | `mesostico` | mesostico tipo John Cage con palabra espina |
| 9 | `escalera_letras` | letras individuales en escalera diagonal |

---

## Archivos

| Archivo | Descripción | Pi |
|---------|-------------|-----|
| `transmisor_fsk_v2.py` | Modulador FSK + reproductor | emisor |
| `display_emisor.py` | Pantalla emisor (telemetría + ASCII art) | emisor |
| `start_display.sh` | Lanzador pygame (xinit + xrandr) | emisor |
| `receptor_fsk_v2.py` | Demodulador FSK | receptor |
| `metodo_berenguer_v2.py` | Transformaciones tipográficas | receptor |
| `display_receptor.py` | Pantalla receptor (Berenguer + ASCII art) | receptor |
| `frases.txt` | Versos del poema fuente (el mismo archivo en las dos Pi) | ambas |

La versión instalada en Santiago, con servicios systemd, versiones anteriores, scripts de medición y `banco_imagenes/`, está en [`santiago_2026/`](santiago_2026/).

**No versionados:**
- `estado_poema.json` — estado de runtime del receptor
- `fsk_output_berenguer/*.wav` — audio generado
- `sstv_recibido/` — imágenes recibidas

---

## Dependencias

```bash
# emisor (radioatlasemisor)
pip install numpy scipy
sudo apt install python3-pygame

# receptor (radioatlasreceptor · entorno_arte)
pip install numpy scipy sounddevice Pillow python-escpos
```

---

## Pendientes antes de la exposición

Lista completa en [`santiago_2026/docs/CHECKLIST_MONTAJE.md`](santiago_2026/docs/CHECKLIST_MONTAJE.md). Lo principal:

- [ ] Texto nuevo: el mismo `frases.txt` en las dos Pi (cada frase ≤ 80 bytes UTF-8)
- [ ] Funcionar sin internet: módulo RTC DS3231 en cada Pi, apagado diario por horas desde el arranque, enchufe con timer
- [ ] Apagado seguro en sala (pulsador GPIO3) — **nunca desenchufar sin apagar**
- [ ] Wifi de la galería cargado en las dos Pi; vencimiento de clave de Tailscale desactivado
- [ ] En la galería: ubicar micrófono, ajustar ganancia y medir el ambiente de las otras obras
- [ ] Horario de la exhibición (hoy el emisor transmite 0:00–23:59)
- [ ] Conectar la impresora térmica
- [ ] Prueba de reinicio completo de las dos Pi
- [x] `display_receptor.py` desarrollado (parpadeo corregido el 31 Oct 2026)
- [x] `banco_imagenes/` en las dos Pi (respaldado en `santiago_2026/banco_imagenes/`)
- [x] Archivos del receptor sincronizados al repo (`santiago_2026/receptor/`)

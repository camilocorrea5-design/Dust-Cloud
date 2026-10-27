# Polvo

Convierte una imagen en una nube de 262.144 puntos en 3D. Todo pasa en el navegador: no hay servidor, no hay librerías, no hay build.

![Polvo](docs/cover.png)

**[Ver en vivo →](https://camilocorrea5-design.github.io/polvo/)** · un solo archivo HTML, 59 KB

---

## Qué hace

Soltás hasta cuatro imágenes. Polvo estima la profundidad de cada una, recorta el fondo y las dibuja como partículas que flotan, estallan y se reúnen. Espacio pasa de una a otra con un morph punto por punto.

El cursor empuja las partículas: donde lo pongas se abre un hueco y el borde se enciende.

| | |
|---|---|
| ![Papel](docs/papel.png) | ![Negativo](docs/negativo.png) |
| Modo papel con estelas | Modo negativo con el eje de reunión activo |

## Controles

| Tecla | Qué hace |
|---|---|
| `Espacio` | Siguiente imagen |
| `L` | Cambia la luz |
| `H` | Muestra u oculta la interfaz |
| `R` | Vuelve a la vista inicial |
| `F` | Pantalla completa |
| `C` | Copia el cuadro al portapapeles |
| `1` – `4` | Salta a una imagen |
| `?` | Lista de atajos |

Arrastrar gira la cámara. La rueda acerca. En móvil: un dedo gira, dos hacen zoom.

### Sliders

| Control | Qué mueve |
|---|---|
| **Deriva** | Cuánto se mueven las partículas en el campo de ruido |
| **Estallido** | Empuje radial desde el centro, pulsante |
| **Reunir** | Atrae todo hacia el eje que definís con el widget |
| **Profundidad** | Escala del eje Z |
| **Recorte** | Cuánto fondo se descarta |
| **Pincel** | Fuerza y radio del empuje del cursor. En 0 se apaga |

### Modos

- **Dibujo** — trazo (testigos), enlaces, barrido, estelas, marcas. Se combinan entre sí.
- **Movimiento** — viaje, enjambre, fundir.
- **Color** — papel (tinta sobre blanco), negativo, real.
- **Luz** — siete tintes, incluido un prisma que da color por partícula.

---

## Cómo funciona

### 1. Muestreo

La imagen se dibuja en un canvas de 512×512 (256×256 en móvil). Cada píxel es una partícula: 262.144 en total. La posición X/Y sale de la grilla, corregida por la proporción de la imagen.

### 2. Profundidad

No hay modelo de ML. La profundidad sale de cuatro señales que se mezclan:

```
z = 0.50·luminancia + 0.16·saturación + 0.22·nitidez + 0.12·centro
```

- **Luminancia** — lo claro tiende a estar adelante.
- **Saturación** — el color vivo suele ser el sujeto.
- **Nitidez** — magnitud del gradiente (Sobel) sobre la luminancia, difuminada con un box blur. Lo enfocado va adelante, lo borroso atrás.
- **Centro** — sesgo radial suave hacia el medio del encuadre.

Después se suaviza y se normaliza entre los percentiles 3 y 97, así una imagen con un reflejo quemado no aplasta todo el rango.

Es una aproximación, no una medición. Funciona bien con fotos de producto, retratos y logos; se confunde con escenas planas o muy ruidosas.

### 3. Recorte del fondo

Polvo promedia el color del marco exterior de la imagen y mide qué tan lejos está cada píxel de ese color. Esa distancia se guarda en el canal `w` de la textura de posiciones, y el shader descarta lo que quede por debajo del umbral del slider **Recorte**.

Por eso un logo sobre fondo plano queda flotando en el aire en vez de pegado a una pared de puntos. Y por eso el umbral se mueve en vivo: el cálculo pesado ya está hecho, el shader solo compara.

### 4. Render

Cada imagen vive en dos texturas:

- `RGBA32F` — posición XYZ más la distancia al fondo.
- `RGBA8` — color.

El vertex shader usa `gl_VertexID` para sacar su texel con `texelFetch`. No hay buffers de atributos ni VBOs: un solo `drawArrays` con 262.144 vértices.

El desplazamiento es analítico, sin simulación ni ping-pong de FBOs. Una función de ruido barata (sumas de senos) alimenta deriva, estallido, reunión y los tres modos de movimiento. Es determinista, así que las **estelas** se dibujan evaluando la misma función en `t` y en `t − 0.05` y uniendo los dos puntos con una línea.

### 5. El pincel

El empuje se calcula en espacio de pantalla, no en 3D. El punto se proyecta, mide su distancia al cursor y se aparta en esa dirección antes de salir del shader:

```glsl
vec2 scr = (clip.xy / clip.w) * vec2(uAspect, 1.0);
float dd  = length(scr - uBrush);
float f   = 1.0 - smoothstep(0.0, uBrushR, dd);
clip.xy  += dir * f * f * uBrushK * (0.6 + 0.8 * h) * clip.w * vec2(1.0/uAspect, 1.0);
```

Cada partícula tiene su propio factor `h` entre 0.6 y 1.4, así el borde del hueco queda irregular y no un círculo perfecto.

### 6. Morph

Las dos imágenes se mezclan en el shader con un `t` escalonado por partícula, más un empuje radial que llega al máximo a mitad de camino. Por eso la nube se abre y se vuelve a cerrar en vez de deslizarse.

### 7. Testigos

Los brackets con coordenadas no son decoración: el desplazamiento está escrito dos veces, en GLSL y en JavaScript, y las etiquetas siguen la posición real de esa partícula. También obedecen al pincel.

---

## Rendimiento

El bucle mide los fps cada 120 cuadros y baja solo:

1. Por debajo de 36 fps, corta el DPR a 1.
2. Si sigue por debajo de 30, reconstruye todo a 256×256 — 65.536 puntos — y avisa.

En móvil arranca directamente en 256×256. Respeta `prefers-reduced-motion`: sin rotación automática y con deriva baja.

## Usarlo en tu sitio

Es un archivo suelto. Copialo y listo.

```html
<iframe src="polvo/index.html" style="width:100%;height:100svh;border:0"></iframe>
```

En Framer: **Embed → HTML** y pegás el contenido, o apuntás a la URL donde lo subas.

Dos cosas a saber si lo embebés:

- La tecla `C` copia el cuadro al portapapeles. Chrome y Edge sí; Safari a veces no.
- Para descargar PNG en vez de copiar, cambiá `copyFrame()` por un `<a download>` con `out.toDataURL()`. Funciona fuera de un iframe con sandbox.

## Accesibilidad

- Los sliders son `<input type="range">` de verdad: Tab y flechas funcionan.
- Los toggles son botones con `aria-pressed`.
- El widget de eje se mueve con las flechas del teclado.
- Foco visible en todo.
- Interfaz en español e inglés, con un botón para cambiar.

## Estructura

```
polvo/
├── index.html        la app entera — HTML, CSS, GLSL y JS
├── docs/             capturas
└── README.md
```

## Navegadores

Necesita **WebGL 2**: Chrome, Edge, Firefox, Safari 15+. Si no está, muestra un mensaje claro en vez de una pantalla negra.

## Crédito

Escrito desde cero por [Camilo Correa](https://github.com/camilocorrea5-design), diseñador visual y web en Medellín.

El género —instrumentos de nube de puntos con interfaz tipo HUD— está inspirado en [Gather 聚](https://gather.be-of-dao.com/) de DAO. Polvo no comparte código con esa app: el motor, el estimador de profundidad, los shaders y la interfaz son propios.

## Licencia

MIT. Ver [LICENSE](LICENSE).

---

## English

**Polvo** turns an image into a 262,144-point 3D cloud, entirely in the browser. No server, no libraries, no build step — one 59 KB HTML file.

Drop up to four pictures. Polvo estimates depth from luminance, saturation, edge sharpness and a centre bias, cuts the background by sampling the frame colour, and draws the result as particles you can orbit, scatter and gather. Your cursor pushes them aside.

Everything runs as a single `drawArrays` call over a float position texture, with analytic displacement in the vertex shader — no simulation passes. The UI is bilingual: hit the `EN` button top right.

Controls: `Space` next · `L` light · `H` hide UI · `R` reset · `F` fullscreen · `C` copy frame · `1`–`4` jump · `?` shortcuts.

MIT licensed.

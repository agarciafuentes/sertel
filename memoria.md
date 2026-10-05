# Memoria de Prácticas: Desarrollo Web (HTML5 y CSS3)

Práctica P1 – Smart Room · Servicios Telemáticos · Universidad de Alcalá

---

## Apartado 2: Validación de la temperatura

El enunciado pide comprobar que la temperatura máxima sea mayor o igual que la mínima combinando restricciones HTML con una comprobación en JavaScript. La solución tiene dos niveles que se complementan.

**1. Restricciones HTML (nivel declarativo).** Los dos campos de alerta son `<input type="number">` con los atributos `min="-20"`, `max="45"` y `step="1"`. Con ellos, el navegador comprueba por sí solo, al enviar el formulario, que el valor sea numérico, esté entre −20 °C y 45 °C y sea un número entero. Estas restricciones no pueden comparar un campo con otro, y por eso hace falta el segundo nivel.

**2. Comprobación en JavaScript (nivel dinámico).** Se ha implementado un script que escucha el evento `input` de ambos campos. En cada cambio limpia el mensaje de validez anterior con `setCustomValidity("")`, lee los dos valores y actúa según el caso:

- **Falta alguno de los dos valores:** el resultado se vacía.
- **Alguno está fuera del rango de −20 °C a 45 °C:** el elemento `<output>` muestra el mensaje «Error: las temperaturas deben estar entre -20 y 45 grados C.». Además, el navegador marca el campo como inválido por los atributos `min` y `max`, y la hoja de estilos lo pinta en rojo.
- **El máximo es inferior al mínimo:** se ejecuta `inputMax.setCustomValidity("La Tº maxima debe ser mayor o igual a la minima.")` y el `<output>` muestra «Error: la temperatura maxima debe ser mayor o igual que la minima.». Con un mensaje de validez personalizado, el campo pasa a ser inválido: el navegador bloquea el envío del formulario y muestra ese texto como aviso nativo, y la hoja de estilos marca el campo en rojo.
- **Los valores son correctos:** el `<output>` muestra la diferencia entre ambas temperaturas (por ejemplo, «15 grados C»).

De este modo, el usuario ve un mensaje comprensible en pantalla en cuanto introduce un valor no válido, sin esperar a pulsar el botón de enviar.

*Nota:* se trata de una validación en el lado del cliente, orientada a mejorar la experiencia de usuario. Por seguridad, siempre debe complementarse con una validación en el servidor.

---

## Apartado 4: Control de caché (cabeceras HTTP)

### 4.1 Diferencia entre `no-cache` y `no-store`

- **`no-cache`**: el navegador **puede almacenar** el recurso en su caché local, pero está obligado a validarlo con el servidor antes de usarlo (mediante validadores como `ETag` o `Last-Modified`). Si el archivo no ha cambiado, el servidor responde `304 Not Modified` sin enviar el contenido y el navegador usa su copia; si ha cambiado, responde `200` con el contenido nuevo.
- **`no-store`**: es la directiva más restrictiva. Prohíbe al navegador y a cualquier caché intermedia **guardar** el recurso en ningún momento. Es útil para páginas que manejan datos confidenciales (por ejemplo, información bancaria).

En resumen: `no-cache` significa «guárdalo, pero pregunta antes de usarlo», y `no-store` significa «no lo guardes».

### 4.2 Política elegida para el documento HTML principal

Para `index.html` se elige `Cache-Control: no-cache`. Es el punto de entrada de la aplicación y su contenido cambia cada vez que se actualiza la web, además de ser el documento que enlaza la hoja de estilos y los scripts. Si se almacenara durante mucho tiempo, los usuarios seguirían viendo una versión antigua. Con `no-cache` el navegador comprueba en cada visita si ha cambiado y, si no es así, recibe una respuesta `304` muy ligera. No es necesario `no-store`, porque la página no contiene información confidencial.

### 4.3 Configuración en Apache

> **PENDIENTE:** aplicar esta configuración en el servidor del Apartado 7 y comprobar que funciona antes de dar este apartado por cerrado.

Se activa el módulo de cabeceras con `sudo a2enmod headers` y se añade lo siguiente al fichero del host virtual:

```apache
<Directory /var/www/iroom>
    Require all granted
    <FilesMatch "\.html$">
        Header set Cache-Control "no-cache"
    </FilesMatch>
</Directory>
```

Después se recarga la configuración con `sudo service apache2 reload`.

### 4.4 Cabeceras observadas y comparación antes y después

> **PENDIENTE:** rellenar con lo observado en la pestaña **Red** de las herramientas de desarrollo del navegador, antes y después de aplicar la configuración. Incluir las capturas en la carpeta `capturas`.

| Situación | Cabeceras de respuesta observadas | Código HTTP | Comportamiento observado |
| :--- | :--- | :--- | :--- |
| Antes · recarga normal | *(completar)* | *(completar)* | *(completar)* |
| Antes · recarga forzada | *(completar)* | *(completar)* | *(completar)* |
| Después · recarga normal | *(completar)* | *(completar)* | *(completar)* |
| Después · recarga forzada | *(completar)* | *(completar)* | *(completar)* |

### 4.5 Recursos estáticos versionados

Para archivos como hojas de estilo (`.css`), scripts o imágenes, usar `no-cache` es menos eficiente: obliga a consultar al servidor en cada uso, aunque casi nunca cambien. Lo ideal es darles una caducidad larga con `max-age` y, cuando el archivo se actualiza, **cambiarle el nombre** (por ejemplo, `iroom.v2.css`), de modo que el navegador lo trate como un recurso nuevo. Esto se conoce como versionado de recursos y permite una política como `Cache-Control: public, max-age=31536000, immutable`, que evita preguntar al servidor.

En esta práctica los archivos no llevan versión en el nombre (`iroom.css`), por lo que una caducidad muy larga haría que los visitantes siguieran viendo estilos antiguos tras una actualización. Mientras no se versionen, conviene una caducidad corta o usar `no-cache`.

---

## Apartado 6: Actualización mediante eventos enviados por el servidor (SSE) — Opcional

Para que los datos de los sensores se actualicen sin que el usuario recargue la página, se propone una arquitectura basada en **Server-Sent Events (SSE)**: el navegador abre una única conexión HTTP con el servidor y este va enviando por ella los datos nuevos. La comunicación es unidireccional (del servidor al cliente), que es justo lo que necesita un panel de sensores. Frente al *polling*, evita peticiones repetidas que a menudo no traen nada nuevo, y frente a WebSocket es más sencillo porque funciona sobre HTTP normal y el navegador se reconecta solo.

**Arquitectura.** El servidor expone un recurso (por ejemplo, `/api/sensores/stream`), atendido por un script del servidor (PHP, Python, Node...), porque un servidor de archivos estáticos no puede mantener la conexión abierta ni generar eventos. El script responde con las cabeceras:

```
Content-Type: text/event-stream
Cache-Control: no-cache
Connection: keep-alive
```

**Formato de los mensajes `text/event-stream`.** Es texto plano. Cada mensaje está formado por líneas del tipo `campo: valor` y termina con una **línea en blanco**. Los campos habituales son `data` (el contenido, obligatorio), `event` (nombre del evento; si no se indica, es `message`), `id` (identificador, que el navegador reenvía al reconectar) y `retry` (milisegundos de espera para reconectar). Las líneas que empiezan por `:` son comentarios.

```
event: sensores
id: 42
data: {"temperatura": 22.5, "humedad": 41}

```

**Conexión desde el navegador con `EventSource`:**

```javascript
const fuente = new EventSource("/api/sensores/stream");

fuente.addEventListener("sensores", (evento) => {
    const datos = JSON.parse(evento.data);
    document.getElementById("valor-temperatura").textContent = datos.temperatura + "ºC";
});

fuente.onerror = () => {
    console.log("Conexión perdida; el navegador reintentará automáticamente.");
};
```

**Procedimiento de actualización de la interfaz.** Cada vez que el servidor detecta una medida nueva, escribe un mensaje en la conexión abierta. El navegador lo recibe, dispara el manejador del evento y el código JavaScript modifica el DOM: en este caso, el contenido de la celda de la tabla de sensores, a la que se le asignaría un `id` (por ejemplo, `valor-temperatura`). Si la conexión se cae, `EventSource` reintenta de forma automática y envía la cabecera `Last-Event-ID` para que el servidor pueda retomar desde el último evento recibido.

---

## Apartado 9: Validación y calidad del código

Se han utilizado las herramientas oficiales del W3C para validar el HTML (W3C Nu Html Checker, en `validator.w3.org`) y la hoja de estilos (W3C CSS Validation Service, en `jigsaw.w3.org/css-validator`), y la consola de las herramientas de desarrollo del navegador para comprobar que no aparecen errores. La siguiente tabla recoge las herramientas utilizadas, los problemas identificados y las correcciones aplicadas.

| Herramienta | Archivo | Resultado / problema identificado | Corrección aplicada |
| :--- | :--- | :--- | :--- |
| **W3C Nu Html Checker** | index.html (versión original) | Dos avisos: *Warning: Section lacks heading* en `<section id="habitacion">` (línea 18) y en `<section id="sensores">` (línea 26). El estándar recomienda identificar con un encabezado `h2`–`h6` el contenido de cada `<section>`. | Se añadió un `<h2>` en las secciones «Datos de la habitación» y «Datos de los sensores». |
| **W3C Nu Html Checker** | index.html (versión final) | Sin errores ni avisos. | — |
| **W3C Nu Html Checker** | config.html (versión final) | Sin errores ni avisos. | — |
| **W3C Nu Html Checker** | minombre.html (versión final) | Sin errores ni avisos. | — |
| **W3C CSS Validation Service** | css/iroom.css (versión final) | Sin errores (CSS nivel 3 + SVG). | — |
| **Consola de Microsoft Edge (DevTools)** | index.html | Al abrir la página como archivo local aparecía el mensaje *«Unsafe attempt to load URL file:///... from frame with URL file:///...»*. | El mensaje desaparece al abrir la página en una ventana InPrivate, sin extensiones, por lo que no procede del código de la práctica. Con ese método, las consolas de las tres páginas aparecen sin errores («No issues»). |

### Capturas de validación

**index.html (avisos originales):**
![Avisos en index.html](capturas/val_index_warning.jpg)

**index.html (validación final):**
![Validación de index.html](capturas/validacion_index_html.png)

**config.html (validación final):**
![Validación de config.html](capturas/validacion_config_html.png)

**minombre.html (validación final):**
![Validación de minombre.html](capturas/validacion_minombre_html.png)

**iroom.css (validación final):**
![Validación de iroom.css](capturas/validacion_css.png)

### Capturas de la consola del navegador

**index.html:**
![Consola de index.html](capturas/consola_index.png)

**config.html:**
![Consola de config.html](capturas/consola_config.png)

**minombre.html:**
![Consola de minombre.html](capturas/consola_minombre.png)

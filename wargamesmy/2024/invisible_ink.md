# Invisible_Ink

**CTF:** Wargames.MY 2024  
**Category:** misc  
**Original writeup:** <https://k3sero.github.io/posts/Invisible-Ink-WarGamesCTF2024/> — by **k3sero**. All credit goes to the original author; this is an archived copy.

---

Invisible Ink - WarGamesCTF2024

Autor del reto: Yes

Dificultad: Fácil

Enunciado

“The flag is hidden somewhere in this GIF. You can’t see it? Must be written in transparent ink.”

Archivos

En reto, solo nos dan el siguiente archivo.

challenge.gif : Contiene un archivo .gif.

Archivos utilizados aquí.

Analizando el código

En este reto básicamente tenemos que extraer la información del gif aportado. Si abrimos dicho archivo, nos encontraremos con un mensaje sin mayor relevancia.

Solución

En este caso, teníamos que utilizar stegsolve para obtener todos los frames que contaba dicho gif, ya que si queremos extraerlos con herramientas como Pillow, nos da error debido a la gran cantidad de píxeles que contiene el gif. (Pill sólo obtenía los frames 0, 1, 2 y 3 pero nos decía que los frames resultantes superaba el máximo de píxeles permitidos)

Para ello teníamos que obtener los frames resultantes que es donde realmente está la información oculta. Para ello utilicé la herramienta mencionada anteriormente para la extracción de estos dos últimos frames.

Finalmente para obtener la flag tenemos que combinar los frames anteriores. En Gimp simplemente tenemos que añadir cada imagen a una capa diferente, no sin antes aplicar un color distinto a cada frame, para que el resultado sea mucho más visible que en blanco y negro.

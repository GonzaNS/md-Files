# Implementación Real: Interfaz Moderna con Jetpack Compose

En la sección anterior vimos cómo *se haría* esta pantalla en XML clásico. Ahora veremos cómo **realmente está hecha** en nuestro proyecto "Reproductor1".

Hemos decidido usar **Jetpack Compose**, el kit de herramientas moderno recomendado por Google.

---

## 💡 ¿Por qué Compose y no XML?

Para este proyecto, Compose ofrece ventajas decisivas sobre el enfoque clásico de la Semana 3:

| Enfoque Clásico (XML) | Enfoque Moderno (Compose) |
| :--- | :--- |
| **Imperativo**: "Busca el TextView y cambia su texto". | **Declarativo**: "La UI es un reflejo del estado actual". |
| Mucho código repetitivo (`findViewById`, Adapters). | Código conciso y 100% Kotlin. |
| Separación física (Layout en XML, Lógica en Java/Kotlin). | Todo está en un solo lugar, fácil de trazar. |
| Sistema de Views antiguo/legado. | Rendimiento optimizado y animaciones sencillas. |

---

## 🏗️ La Estructura Real (Column vs ConstraintLayout)

En XML usamos `ConstraintLayout` para atar elementos. En Compose, pensamos en **Contenedores**.

Para nuestro reproductor, la estructura es vertical, así que usamos una **`Column`** (Columna):

```kotlin
@Composable
fun MusicPlayerScreen(viewModel: MusicPlayerViewModel) {
    // Surface = El lienzo (fondo)
    Surface(modifier = Modifier.fillMaxSize()) {
        
        // Column = Pila vertical de elementos
        Column(
            horizontalAlignment = Alignment.CenterHorizontally,
            verticalArrangement = Arrangement.SpaceEvenly // Distribuye el espacio
        ) {
            // 1. Carátula
            SongCover(currentSong)
            
            // 2. Título y Artista
            SongInfo(currentSong)
            
            // 3. Barra de Progreso
            TrackProgress(...)
            
            // 4. Controles (Fila horizontal)
            PlayerControls(...)
        }
    }
}
```

Es mucho más limpio: no hay constraints complejos, solo una lista de elementos uno debajo del otro.

---

## 🖼️ 1. La Carátula (Image con Modifiers)

En XML configuramos `scaleType`. En Compose usamos `ContentScale` y `Clip`.

```kotlin
@Composable
fun SongCover(song: Song) {
    Image(
        bitmap = song.albumArt.asImageBitmap(),
        contentDescription = "Carátula",
        modifier = Modifier
            .size(300.dp) // Tamaño fijo
            .clip(RoundedCornerShape(16.dp)) // ¡Bordes redondeados fáciles!
            .shadow(8.dp), // Sombra elegante
        contentScale = ContentScale.Crop
    )
}
```
*Nota: Hacer bordes redondeados y sombras en XML requería crear archivos `drawable` extra. En Compose es una sola línea.*

---

## 🎵 2. Título y Artista (Text)

Simple y directo.

```kotlin
@Composable
fun SongInfo(song: Song) {
    Column(horizontalAlignment = Alignment.CenterHorizontally) {
        // Título
        Text(
            text = song.title,
            style = MaterialTheme.typography.headlineMedium,
            maxLines = 1,
            overflow = TextOverflow.Ellipsis
        )
        
        // Artista
        Text(
            text = song.artist,
            style = MaterialTheme.typography.bodyLarge,
            color = Color.Gray
        )
    }
}
```

---

## 🎚️ 3. Barra de Progreso (Slider)

En lugar de `SeekBar` (XML), usamos **`Slider`**. La magia aquí es que el valor del slider está "viamculado" al estado.

```kotlin
@Composable
fun TrackProgress(
    progress: Float, // Valor actual (0.0 a 1.0)
    onProgressChanged: (Float) -> Unit, // Qué hacer al moverlo
    duration: String
) {
    Column(modifier = Modifier.padding(horizontal = 24.dp)) {
        Slider(
            value = progress,
            onValueChange = onProgressChanged,
            colors = SliderDefaults.colors(
                thumbColor = MaterialTheme.colorScheme.primary
            )
        )
        
        // Tiempos (Fila horizontal)
        Row(
            modifier = Modifier.fillMaxWidth(),
            horizontalArrangement = Arrangement.SpaceBetween
        ) {
            Text(text = formatTime(progress)) // Actual
            Text(text = duration) // Total
        }
    }
}
```

---

## ⏯️ 4. Controles (Row)

Para poner los botones uno al lado del otro, usamos una **`Row`** (Fila). Esto reemplaza a las "Chains" horizontales de XML.

```kotlin
@Composable
fun PlayerControls(
    isPlaying: Boolean,
    onPlayPause: () -> Unit,
    onNext: () -> Unit,
    onPrev: () -> Unit
) {
    Row(
        verticalAlignment = Alignment.CenterVertically,
        horizontalArrangement = Arrangement.spacedBy(16.dp)
    ) {
        // Anterior
        IconButton(onClick = onPrev) {
            Icon(Icons.Filled.SkipPrevious, null, modifier = Modifier.size(48.dp))
        }

        // Play/Pause (Círculo grande)
        IconButton(
            onClick = onPlayPause,
            modifier = Modifier
                .size(72.dp)
                .background(MaterialTheme.colorScheme.primary, CircleShape)
        ) {
            Icon(
                imageVector = if (isPlaying) Icons.Filled.Pause else Icons.Filled.PlayArrow,
                contentDescription = null,
                tint = Color.White,
                modifier = Modifier.size(48.dp)
            )
        }

        // Siguiente
        IconButton(onClick = onNext) {
            Icon(Icons.Filled.SkipNext, null, modifier = Modifier.size(48.dp))
        }
    }
}
```

---

## 🎓 Conclusión

Aunque aprendimos los fundamentos con XML y ConstraintLayout (importante para entender la historia de Android), nuestro **Reproductor1** utiliza la tecnología del futuro.

**Compose nos permite:**
1.  Escribir menos código.
2.  Reutilizar componentes (`SongCover` se puede usar en otras pantallas).
3.  Cambiar el diseño rápidamente (cambiar de `Column` a `Row` es cambiar una palabra).

¡Así es como se construye una UI profesional en 2025! 🚀

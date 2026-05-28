# Documentación del Proyecto: Reproductor de Música (GonZound) - Parte 1

Este documento cubre la primera fase del desarrollo (Semanas 1-6) del proyecto "Reproductor1", una aplicación de música moderna construida con **Jetpack Compose** y **Kotlin**.

---

## SEMANA 1: Introducción y Configuración

### 1. Conceptos Clave
El proyecto utiliza la estructura moderna de Android con **Gradle Kotlin DSL** y **Jetpack Compose** para la interfaz de usuario, eliminando el uso de XML para layouts.
- **Gradle Kotlin DSL (.kts)**: Sistema de construcción moderno que usa Kotlin para mayor legibilidad y seguridad de tipos.
- **Jetpack Compose**: Kit de herramientas declarativo para construir UI nativa.
- **Permisos**: Declaraciones esenciales para que el sistema operativo permita a la app acceder a hardware o datos sensibles.

### 2. Implementación

**Archivo: `build.gradle.kts` (Módulo: app)**
```kotlin
plugins {
    // Plugins oficiales para construir apps Android con Kotlin
    alias(libs.plugins.android.application)
    alias(libs.plugins.kotlin.android)
    // Plugin para integrar servicios de Google (necesario para Firebase)
    id("com.google.gms.google-services")
}

android {
    namespace = "com.example.reproductor1"
    compileSdk = 35

    buildFeatures {
        // Habilita Jetpack Compose en el proyecto, permitiendo usar código UI en Kotlin
        compose = true
    }
}

dependencies {
    // BOM (Bill of Materials) asegura que todas las versiones de Compose sean compatibles
    implementation(platform(libs.androidx.compose.bom))
    // Componentes de Material Design 3 (botones, tarjetas, temas)
    implementation(libs.androidx.material3)
    // Librerías de Firebase para autenticación y base de datos
    implementation("com.google.firebase:firebase-auth")
    implementation("com.google.firebase:firebase-firestore")
}
```
**Análisis del Código:**
En el bloque `plugins`, se aplican las herramientas necesarias para compilar código Android y Kotlin. El plugin `google-services` es crucial porque procesa el archivo `google-services.json` para conectar la app con la consola de Firebase.
Dentro del bloque `android`, la línea `compose = true` es el interruptor maestro que habilita el compilador de Compose; sin esto, no podríamos escribir funciones `@Composable`.
En `dependencies`, usamos el **BOM** de Compose. Esto es una buena práctica porque nos permite declarar una sola versión para la plataforma y no tener que gestionar versiones individuales para cada librería de UI (texto, gráficos, material), evitando conflictos de versiones.

**Archivo: `AndroidManifest.xml`**
```xml
<manifest xmlns:android="http://schemas.android.com/apk/res/android" ...>

    <!-- Permite leer archivos de audio del almacenamiento externo -->
    <uses-permission android:name="android.permission.READ_MEDIA_AUDIO" />

    <!-- Permite que el servicio de música funcione en primer plano -->
    <uses-permission android:name="android.permission.FOREGROUND_SERVICE" />

    <!-- Permite acceder al GPS de alta precisión -->
    <uses-permission android:name="android.permission.ACCESS_FINE_LOCATION" />

    <application ...>
        <!-- Registro del servicio que reproducirá la música -->
        <service
            android:name=".service.MusicPlayerService"
            android:foregroundServiceType="mediaPlayback"
            android:exported="false" />
    </application>
</manifest>
```
**Análisis del Código:**
Aquí definimos los "contratos" con el sistema operativo.
- `READ_MEDIA_AUDIO`: En Android 13+ (API 33), ya no pedimos acceso a *todo* el almacenamiento, sino específicamente a archivos de audio, mejorando la privacidad.
- `FOREGROUND_SERVICE`: Es obligatorio para aplicaciones que realizan tareas largas visibles para el usuario (como reproducir música) mientras la app no está en pantalla.
- La etiqueta `<service>` registra nuestro `MusicPlayerService`. El atributo `foregroundServiceType="mediaPlayback"` es un requisito reciente de Android 14 para justificar por qué el servicio necesita ejecutarse en primer plano.

### 3. Resultado Visual
La configuración inicial no tiene interfaz visible, pero se establece el tema base `Reproductor1Theme` en `ui/theme` que define los colores y tipografía de la app.

[IMAGEN SUGERIDA: Captura de Android Studio mostrando la estructura del proyecto y el archivo build.gradle.kts abierto]

---

## SEMANA 2: Arquitectura y Ciclo de Vida

### 1. Conceptos Clave
La aplicación sigue el patrón de arquitectura **MVVM (Model-View-ViewModel)**.
- **ViewModel**: Gestiona el estado de la UI y sobrevive a cambios de configuración (como rotar la pantalla).
- **ComponentActivity**: Clase base moderna para actividades que usan Compose.
- **State Hoisting**: Patrón donde el estado se mueve "hacia arriba" para que los componentes sean "stateless" (sin estado).

### 2. Implementación

**Archivo: `MainActivity.kt`**
```kotlin
class MainActivity : ComponentActivity() {
    // Inyectamos el ViewModel. 'by viewModels()' crea una instancia retenida
    private val viewModel: MusicPlayerViewModel by viewModels()

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        // Habilita el diseño de borde a borde (pantalla completa real)
        enableEdgeToEdge()
        
        // Define el contenido de la UI usando Compose en lugar de XML
        setContent {
            Reproductor1Theme {
                // 'remember' guarda el estado en memoria durante recomposiciones
                val drawerState = rememberDrawerState(initialValue = DrawerValue.Closed)
                
                // 'mutableStateOf' crea un estado observable. Si cambia, la UI se redibuja.
                var currentScreen by remember { mutableStateOf(AppScreen.Home) }
                
                // Estructura principal con menú lateral (Drawer) y contenido (Scaffold)
                ModalNavigationDrawer(
                    drawerState = drawerState,
                    drawerContent = { ... }
                ) {
                    Scaffold(modifier = Modifier.fillMaxSize()) { innerPadding ->
                        // Lógica de navegación simple basada en el estado 'currentScreen'
                        when (currentScreen) {
                            AppScreen.Home -> MusicPlayerScreen(...)
                            AppScreen.MyMusic -> MyMusicScreen(...)
                            // ... otros casos
                        }
                    }
                }
            }
        }
    }
}
```
**Análisis del Código:**
Este código reemplaza completamente la antigua forma de inflar layouts con `setContentView(R.layout.activity_main)`.
- `setContent { }`: Es el puente entre la Activity (mundo Android clásico) y Compose.
- `Reproductor1Theme`: Es una función composable que envuelve toda la app, proveyendo colores y tipografías consistentes a todos los hijos.
- `var currentScreen by remember { ... }`: Esta línea es el corazón de nuestra navegación manual. `remember` asegura que la variable no se resetee cada vez que la función se redibuja (recomposición). Cuando asignamos un nuevo valor a `currentScreen`, Compose detecta el cambio y vuelve a ejecutar el bloque `when`, mostrando la nueva pantalla automáticamente.

### 3. Resultado Visual
Al iniciar, la app ejecuta lógica para decidir qué pantalla mostrar. Si es la primera vez, muestra `WelcomeScreens`; de lo contrario, carga la pantalla principal `Home`.

[IMAGEN SUGERIDA: Diagrama simple de la arquitectura MVVM: Activity <-> ViewModel <-> Data, mostrando cómo fluyen los datos]

---

## SEMANA 3: Layouts y Controles (Jetpack Compose)

### 1. Conceptos Clave
En Jetpack Compose, la UI se construye componiendo funciones.
- **Column/Row**: Reemplazan a `LinearLayout`. Organizan elementos en vertical u horizontal.
- **Surface**: Provee un fondo y maneja el contenido según el sistema de diseño Material.
- **Modificadores**: Permiten ajustar el layout, añadir padding, clics, etc.

### 2. Implementación

**Archivo: `MainActivity.kt` (Función MusicPlayerScreen)**
```kotlin
@Composable
fun MusicPlayerScreen(viewModel: MusicPlayerViewModel, ...) {
    // Surface provee el fondo y manejo de colores del tema
    Surface(modifier = modifier.fillMaxSize()) {
        // Column organiza los elementos verticalmente, uno debajo del otro
        Column(
            horizontalAlignment = Alignment.CenterHorizontally, // Centra horizontalmente
            verticalArrangement = Arrangement.spacedBy(12.dp)   // Espacio uniforme entre hijos
        ) {
            // Componente personalizado para la carátula
            SongCover(song = currentSong)
            
            // Componente para info de texto (Título, Artista)
            SongInfo(song = currentSong, viewModel = viewModel)
            
            // Slider para la barra de progreso
            TrackProgress(
                progress = currentPosition,
                onProgressChanged = viewModel::updatePosition, // Referencia a función del VM
                duration = song.duration
            )
            
            // Fila de botones de control (Play, Pause, Next)
            PlayerControls(
                isPlaying = isPlaying,
                onPlayPause = { viewModel.togglePlayPause() }, // Lambda para manejar el evento
                onPrevious = { viewModel.previousSong() },
                onNext = { viewModel.nextSong() }
            )
        }
    }
}
```
**Análisis del Código:**
Aquí vemos la potencia de la composición. `MusicPlayerScreen` no sabe *cómo* se dibuja una carátula o un botón, solo coordina componentes más pequeños (`SongCover`, `PlayerControls`).
- `Arrangement.spacedBy(12.dp)`: Elimina la necesidad de poner márgenes manuales en cada elemento hijo; el contenedor se encarga del espaciado.
- **Paso de Funciones**: Observa cómo pasamos `viewModel::updatePosition` o lambdas `{ viewModel.togglePlayPause() }`. Esto significa que la UI es "tonta": solo notifica "el usuario hizo clic aquí", y el ViewModel decide qué hacer. La UI no modifica los datos directamente.

### 3. Resultado Visual
Una pantalla limpia y moderna. La carátula del álbum domina el centro. Debajo, el título de la canción se desplaza si es muy largo (efecto Marquee). La barra de progreso permite al usuario arrastrar para cambiar el tiempo.

[IMAGEN SUGERIDA: Pantalla de reproducción en ejecución mostrando carátula, slider y controles]

---

## SEMANA 4: Eventos y Listas (LazyColumn)

### 1. Conceptos Clave
- **LazyColumn**: Es el componente para listas largas y dinámicas. Solo renderiza los elementos visibles en pantalla, reciclándolos al hacer scroll. Es el equivalente moderno de `RecyclerView`.
- **Items**: Función DSL para definir cómo se renderiza cada elemento de la lista de datos.

### 2. Implementación

**Archivo: `MainActivity.kt` (Función MyMusicScreen)**
```kotlin
LazyColumn(
    modifier = Modifier.fillMaxWidth(),
    contentPadding = PaddingValues(bottom = 72.dp) // Espacio extra al final
) {
    // Itera sobre la lista de canciones de manera eficiente
    itemsIndexed(songs) { index, song ->
        Card(
            modifier = Modifier
                .fillMaxWidth()
                // Detecta el clic y llama al ViewModel con el índice exacto
                .clickable { viewModel.playSongAtIndex(index) }, 
            colors = CardDefaults.cardColors(containerColor = Color(0xFFF7F7F7))
        ) {
            // Row organiza imagen y texto horizontalmente
            Row(
                verticalAlignment = Alignment.CenterVertically,
                modifier = Modifier.padding(12.dp)
            ) {
                // Imagen de la canción
                Image(bitmap = song.albumArt, ...)
                
                Spacer(modifier = Modifier.width(12.dp))
                
                // Columna de texto (Título sobre Artista)
                Column {
                    Text(text = song.title, fontWeight = FontWeight.Bold)
                    Text(text = song.artist, color = Color.Gray)
                }
            }
        }
    }
}
```
**Análisis del Código:**
- `LazyColumn` es inteligente: si tienes 1000 canciones, solo crea las ~10 vistas que caben en la pantalla. A medida que deslizas, reutiliza las vistas que salen para mostrar las que entran.
- `contentPadding`: Añadimos 72dp al final de la lista. ¿Por qué? Para que el último elemento no quede oculto detrás de la barra de navegación o controles flotantes si los hubiera.
- `.clickable { ... }`: Este modificador hace que toda la tarjeta sea interactiva. Al tocarla, ejecutamos `viewModel.playSongAtIndex(index)`, lo que inicia la reproducción instantáneamente.

### 3. Resultado Visual
Una lista vertical fluida. Cada elemento es una tarjeta (`Card`) que reacciona visualmente al tacto (efecto ripple). Muestra la miniatura del álbum a la izquierda y los detalles a la derecha.

[IMAGEN SUGERIDA: Pantalla "Mi Música" con la lista de canciones scrolleable]

---

## SEMANA 5: Navegación y Notificaciones

### 1. Conceptos Clave
- **ModalNavigationDrawer**: Componente estándar de Material Design para menús laterales ("Hamburguesa").
- **NotificationChannel**: Requisito de Android 8.0+ para agrupar notificaciones.
- **MediaStyle**: Estilo de notificación específico para reproductores que muestra controles expandidos y carátula en la pantalla de bloqueo.

### 2. Implementación

**Archivo: `MainActivity.kt` (Configuración del Drawer)**
```kotlin
NavigationDrawerItem(
    label = { Text("Inicio") },
    selected = currentScreen == AppScreen.Home, // Resalta si es la pantalla activa
    onClick = {
        currentScreen = AppScreen.Home // Actualiza el estado de navegación
        scope.launch { drawerState.close() } // Cierra el menú con animación asíncrona
    },
    icon = { Icon(Icons.Filled.PlayArrow, null) }
)
```
**Análisis del Código:**
El `NavigationDrawerItem` es un componente prefabricado que maneja el estilo de selección (color de fondo, icono tintado).
Usamos `scope.launch` porque `drawerState.close()` es una función suspendida (animación) y debe llamarse dentro de una corrutina para no bloquear el hilo principal de la UI.

**Archivo: `service/MusicPlayerService.kt` (Notificación)**
```kotlin
val notification = NotificationCompat.Builder(this, CHANNEL_ID)
    .setContentTitle(song.title)
    .setContentText(song.artist)
    .setSmallIcon(R.drawable.ic_launcher_foreground)
    // 'setOngoing(true)' impide que el usuario borre la notificación deslizando
    .setOngoing(isPlaying) 
    // Define la prioridad para que aparezca arriba en la lista
    .setPriority(NotificationCompat.PRIORITY_LOW)
    .build()

// Inicia el servicio en primer plano, vinculando la notificación
startForeground(NOTIFICATION_ID, notification)
```
**Análisis del Código:**
La notificación no es solo visual; es lo que mantiene viva la aplicación. Al llamar a `startForeground`, le decimos al sistema Android: "Esta app está haciendo algo importante (reproducir música), por favor no mates el proceso para liberar memoria".
`setOngoing(isPlaying)` es un detalle de UX importante: si la música suena, la notificación es fija. Si se pausa, el usuario puede deslizarla para cerrarla (y detener el servicio).

### 3. Resultado Visual
- **Menú**: Un panel lateral que se desliza suavemente sobre el contenido principal.
- **Notificación**: Aparece en la barra de estado. Muestra qué canción suena y permite volver a la app rápidamente.

[IMAGEN SUGERIDA: Menú lateral abierto y captura de la notificación en la barra de estado]

---

## SEMANA 6: Listas Avanzadas (Material Design Cards)

### 1. Conceptos Clave
- **Material Design 3**: Sistema de diseño que usa elevación tonal y esquinas redondeadas.
- **Card**: Contenedor semántico para contenido agrupado.
- **Clip & Shape**: Herramientas gráficas para recortar contenido (imágenes) con formas específicas.

### 2. Implementación

**Archivo: `MainActivity.kt` (Estilo de Tarjeta)**
```kotlin
Card(
    modifier = Modifier.fillMaxWidth(),
    // Personalización de colores y elevación
    colors = CardDefaults.cardColors(containerColor = Color(0xFFF7F7F7)),
    // Sombra suave para dar profundidad
    elevation = CardDefaults.cardElevation(defaultElevation = 2.dp)
) {
    Row(modifier = Modifier.padding(12.dp)) {
        // Imagen con recorte redondeado
        Image(
            modifier = Modifier
                .size(48.dp)
                .clip(RoundedCornerShape(8.dp)), // Recorta la imagen cuadrada con bordes suaves
            contentScale = ContentScale.Crop // Recorta la imagen para llenar el cuadro sin deformar
        )
        // ...
    }
}
```
**Análisis del Código:**
Aquí refinamos la estética.
- `CardDefaults.cardColors`: En lugar del blanco por defecto, usamos un gris muy claro (`0xFFF7F7F7`) para diferenciar las tarjetas del fondo blanco de la pantalla.
- `RoundedCornerShape(8.dp)`: Aplicamos esto a la imagen (`Image`). Si no lo hiciéramos, la imagen sería un cuadrado perfecto y duro, lo cual desentonaría con las esquinas redondeadas de la tarjeta contenedora.
- `ContentScale.Crop`: Es vital para imágenes de álbumes que pueden tener diferentes relaciones de aspecto. Asegura que la imagen llene el espacio de 48x48dp sin estirarse ni dejar bordes vacíos.

### 3. Resultado Visual
La lista se siente pulida y profesional. Cada canción es una entidad física separada visualmente. La consistencia en los bordes redondeados (tanto en la tarjeta como en la imagen) crea una armonía visual agradable.

[IMAGEN SUGERIDA: Detalle en primer plano de una tarjeta de canción en la lista, destacando las sombras y bordes redondeados]

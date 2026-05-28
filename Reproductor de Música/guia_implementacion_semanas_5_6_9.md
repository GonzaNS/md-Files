# Guía de Implementación - Reproductor de Música KDM
## Semanas 5, 6 y 9

Esta guía documenta la implementación de los temas cubiertos en las semanas 5, 6 y 9 del curso de Aplicaciones Móviles.

---

## 📱 SEMANA 5: Fragment y Context

### 1. Fragment

**Descripción:**
Un Fragment representa una porción reutilizable de la interfaz de usuario dentro de una Activity. Los Fragments tienen su propio ciclo de vida y pueden ser añadidos, removidos o reemplazados dinámicamente. En Jetpack Compose, el concepto equivalente son los **Composables** que se pueden mostrar/ocultar condicionalmente.

**Implementación en el Reproductor:**

**Figura 18:** Composables como equivalente a Fragments

```kotlin
// MainActivity.kt - Líneas 140-280
// En lugar de Fragments tradicionales, usamos Composables que se muestran condicionalmente
setContent {
    Reproductor1Theme {
        val isFirstTime = sharedPreferences.getBoolean("is_first_time", true)
        var showWelcome by remember { mutableStateOf(isFirstTime) }
        var currentScreen by remember { mutableStateOf(AppScreen.Home) }
        
        // Equivalente a Fragment: mostrar diferentes pantallas según el estado
        if (showWelcome) {
            // "Fragment" de bienvenida
            WelcomeScreens(
                onComplete = {
                    showWelcome = false
                    sharedPreferences.edit().putBoolean("is_first_time", false).apply()
                }
            )
        } else {
            // "Fragment" principal con navegación
            ModalNavigationDrawer(
                drawerState = drawerState,
                drawerContent = { /* Menú lateral */ }
            ) {
                // Cambiar entre diferentes "Fragments" según la pantalla actual
                when (currentScreen) {
                    AppScreen.Home -> MusicPlayerScreen(
                        viewModel = viewModel,
                        onRequestPermission = { requestStoragePermission() }
                    )
                    AppScreen.MyMusic -> MyMusicScreen(viewModel = viewModel)
                    AppScreen.Locations -> LocationsScreen(
                        viewModel = viewModel,
                        onRequestLocationPermission = { requestLocationPermission() }
                    )
                    AppScreen.Settings -> SettingsScreen(viewModel = viewModel)
                    AppScreen.Cloud -> CloudScreen(viewModel = viewModel)
                }
            }
        }
    }
}
```

**Sugerencia de Interfaz:**
> 💡 **Navegación entre Pantallas** - El menú lateral (drawer) permite cambiar entre diferentes "fragments": Inicio, Mi música, Ubicaciones, Configuración y Mi Nube.
> 💡 **Pantallas de Bienvenida** - Las tres pantallas de bienvenida funcionan como fragments que se muestran solo la primera vez.

---

### 2. Ciclo de Vida de un Fragment

**Descripción:**
El ciclo de vida de un Fragment incluye métodos como `onCreate()`, `onCreateView()`, `onStart()`, `onResume()`, `onPause()`, `onStop()`, y `onDestroy()`. En Compose, estos conceptos se manejan con **efectos secundarios** como `LaunchedEffect`, `DisposableEffect`, y `remember`.

**Implementación en el Reproductor:**

**Figura 19:** Gestión del ciclo de vida con remember y LaunchedEffect

```kotlin
// WelcomeScreens.kt - Líneas 23-70
@Composable
fun WelcomeScreens(
    onComplete: () -> Unit,
    modifier: Modifier = Modifier
) {
    // remember: mantiene el estado entre recomposiciones (equivalente a onSaveInstanceState)
    var currentPage by remember { mutableStateOf(0) }
    
    Column(
        modifier = modifier
            .fillMaxSize()
            .background(Color(0xFF1976D2)),
        horizontalAlignment = Alignment.CenterHorizontally
    ) {
        // Indicador de páginas
        Row(
            modifier = Modifier.padding(top = 60.dp),
            horizontalArrangement = Arrangement.spacedBy(8.dp)
        ) {
            repeat(3) { index ->
                Box(
                    modifier = Modifier
                        .size(if (index == currentPage) 12.dp else 8.dp)
                        .clip(RoundedCornerShape(4.dp))
                        .background(
                            if (index == currentPage) Color.White else Color.White.copy(alpha = 0.5f)
                        )
                )
            }
        }
        
        // Cambiar entre páginas (equivalente a FragmentTransaction)
        when (currentPage) {
            0 -> WelcomePage1(onNext = { currentPage = 1 })
            1 -> WelcomePage2(
                onNext = { currentPage = 2 },
                onPrevious = { currentPage = 0 }
            )
            2 -> WelcomePage3(
                onComplete = onComplete,
                onPrevious = { currentPage = 1 }
            )
        }
    }
}
```

**Figura 20:** LaunchedEffect y BackHandler para ciclo de vida

```kotlin
// MainActivity.kt - Líneas 1071-1079 (LaunchedEffect - equivalente a onResume)
LaunchedEffect(showSearch) {
    if (showSearch) focusRequester.requestFocus()
}

// BackHandler - equivalente a onBackPressed en Fragment
BackHandler(enabled = showSearch) {
    showSearch = false
    searchText = ""
    viewModel.clearSearch()
}
```

**Sugerencia de Interfaz:**
> 💡 **Indicador de Páginas en Bienvenida** - Los puntos que cambian de tamaño y opacidad según la página actual, demostrando el ciclo de vida del fragment.

---

### 3. Context Activity

**Descripción:**
Context es una interfaz que proporciona acceso a recursos de la aplicación, servicios del sistema, y operaciones específicas de la aplicación. Se utiliza para acceder a SharedPreferences, servicios del sistema (LocationManager, SensorManager), y recursos.

**Implementación en el Reproductor:**

**Figura 21:** Uso de Context para SharedPreferences

```kotlin
// MainActivity.kt - Líneas 134-138
override fun onCreate(savedInstanceState: Bundle?) {
    super.onCreate(savedInstanceState)
    enableEdgeToEdge()
    
    // Context usado para acceder a SharedPreferences
    val sharedPreferences = getSharedPreferences("kdm_prefs", MODE_PRIVATE)
    
    setContent {
        // ...
    }
}
```

**Figura 22:** Context para acceder a servicios del sistema

```kotlin
// LocationManager.kt - Líneas 20-26
class LocationManager(private val context: Context) {
    
    private val androidLocationManager: AndroidLocationManager =
        context.getSystemService(Context.LOCATION_SERVICE) as AndroidLocationManager
    
    // ExecutorService para manejar operaciones de GPS en hilos separados
    private val executorService = Executors.newSingleThreadExecutor()
}

// SensorManager.kt - Líneas 15-18
class SensorManager(private val context: Context) {
    
    private val sensorManager: AndroidSensorManager =
        context.getSystemService(Context.SENSOR_SERVICE) as AndroidSensorManager
}

// MusicScanner.kt - Líneas 15-22
class MusicScanner(private val context: Context) {
    
    suspend fun scanDeviceMusic(): List<Song> = withContext(Dispatchers.IO) {
        val songs = mutableListOf<Song>()
        
        try {
            val contentResolver: ContentResolver = context.contentResolver
            // Usar ContentResolver para escanear archivos de música
        }
    }
}
```

**Figura 23:** Context para verificar permisos

```kotlin
// LocationManager.kt - Líneas 29-38 (Uso de Context para verificar permisos)
fun hasLocationPermission(): Boolean {
    return ContextCompat.checkSelfPermission(
        context,
        Manifest.permission.ACCESS_FINE_LOCATION
    ) == PackageManager.PERMISSION_GRANTED ||
    ContextCompat.checkSelfPermission(
        context,
        Manifest.permission.ACCESS_COARSE_LOCATION
    ) == PackageManager.PERMISSION_GRANTED
}
```

**Sugerencia de Interfaz:**
> 💡 **Toda la Aplicación** - El Context se usa en segundo plano en todas las pantallas para acceder a servicios del sistema (GPS, sensores, almacenamiento).

---

### 4. Mensajes: Toast y SnackBar

**Descripción:**
Toast y SnackBar son componentes para mostrar mensajes breves al usuario. Toast es un mensaje flotante temporal, mientras que SnackBar aparece en la parte inferior y puede incluir acciones. En Jetpack Compose, se pueden usar `Snackbar` con `SnackbarHost` o mostrar mensajes con `AlertDialog`.

**Implementación en el Reproductor:**

**Figura 24:** Mensaje de error (equivalente a SnackBar)

```kotlin
// MainActivity.kt - Líneas 1797-1811 (Mensaje de error - equivalente a SnackBar)
// Mostrar error si existe
authError?.let { error ->
    Spacer(modifier = Modifier.height(16.dp))
    Card(
        modifier = Modifier.fillMaxWidth(),
        colors = CardDefaults.cardColors(containerColor = Color(0xFFFFEBEE))
    ) {
        Text(
            text = error,
            modifier = Modifier.padding(16.dp),
            color = Color(0xFFC62828),
            fontSize = 14.sp
        )
    }
}
```

**Figura 25:** Mensaje de sugerencia con acción (equivalente a SnackBar)

```kotlin
// MainActivity.kt - Líneas 1273-1303 (Mensaje de sugerencia - equivalente a SnackBar con acción)
suggestedPlaylist?.let { playlist ->
    detectedLocation?.let { location ->
        Card(
            modifier = Modifier.fillMaxWidth(),
            colors = CardDefaults.cardColors(containerColor = Color(0xFFE3F2FD))
        ) {
            Column(
                modifier = Modifier.padding(16.dp),
                verticalArrangement = Arrangement.spacedBy(8.dp)
            ) {
                Text(
                    text = "📍 Ubicación detectada: ${location.name}",
                    fontWeight = FontWeight.Bold,
                    fontSize = 16.sp
                )
                Text(
                    text = "Lista sugerida: ${playlist.name}",
                    fontSize = 14.sp,
                    color = Color.Gray
                )
                Button(
                    onClick = { viewModel.playSuggestedPlaylist() },
                    colors = ButtonDefaults.buttonColors(
                        containerColor = Color(0xFF1976D2)
                    )
                ) {
                    Text("Reproducir ${playlist.name}")
                }
            }
        }
    }
}
```

**Figura 26:** Mensaje de permisos (equivalente a SnackBar)

```kotlin
// MainActivity.kt - Líneas 1307-1334 (Mensaje de permisos - equivalente a SnackBar)
if (!hasLocationPermission) {
    Card(
        modifier = Modifier.fillMaxWidth(),
        colors = CardDefaults.cardColors(containerColor = Color(0xFFFFF3E0))
    ) {
        Column(
            modifier = Modifier.padding(16.dp),
            verticalArrangement = Arrangement.spacedBy(8.dp)
        ) {
            Text(
                text = "Permisos de ubicación necesarios",
                fontWeight = FontWeight.Bold
            )
            Text(
                text = "Necesitas conceder permisos de ubicación para usar esta funcionalidad",
                fontSize = 14.sp,
                color = Color.Gray
            )
            Button(
                onClick = onRequestLocationPermission,
                colors = ButtonDefaults.buttonColors(
                    containerColor = Color(0xFF1976D2)
                )
            ) {
                Text("Conceder permisos")
            }
        }
    }
}
```

**Sugerencia de Interfaz:**
> 💡 **Pantalla de Autenticación** - Card roja que muestra mensajes de error cuando falla el login.
> 💡 **Pantalla de Ubicaciones** - Card azul que sugiere reproducir una playlist cuando se detecta una ubicación guardada.
> 💡 **Pantalla de Ubicaciones** - Card naranja que solicita permisos de ubicación.

---

## 🎨 SEMANA 6: Material Design

### 1. RecyclerView (LazyColumn en Compose)

**Descripción:**
RecyclerView es un componente avanzado para mostrar listas grandes de datos de manera eficiente, reciclando vistas que salen de la pantalla. En Jetpack Compose, **LazyColumn** y **LazyRow** proporcionan la misma funcionalidad con mejor rendimiento y sintaxis más simple.

**Implementación en el Reproductor:**

**Figura 27:** LazyColumn para lista de canciones en MyMusicScreen

```kotlin
// MainActivity.kt - Líneas 897-1089 (MyMusicScreen - Lista de canciones)
@Composable
fun MyMusicScreen(
    viewModel: MusicPlayerViewModel,
    modifier: Modifier = Modifier
) {
    val allSongs by viewModel.allSongs.collectAsState()
    val currentSong by viewModel.currentSong.collectAsState()
    val filteredSongs by viewModel.filteredSongs.collectAsState()
    
    var showSearch by remember { mutableStateOf(false) }
    var searchText by remember { mutableStateOf("") }
    
    Surface(modifier = modifier.fillMaxSize()) {
        Column(modifier = Modifier.fillMaxSize()) {
            Text(
                text = "Mi Música",
                fontSize = 24.sp,
                fontWeight = FontWeight.Bold,
                modifier = Modifier.padding(16.dp)
            )
            
            if (allSongs.isEmpty()) {
                Box(
                    modifier = Modifier.fillMaxSize(),
                    contentAlignment = Alignment.Center
                ) {
                    Column(
                        horizontalAlignment = Alignment.CenterHorizontally,
                        verticalArrangement = Arrangement.spacedBy(16.dp)
                    ) {
                        CircularProgressIndicator()
                        Text("Escaneando música...", color = Color.Gray)
                    }
                }
            } else {
                // LazyColumn: equivalente a RecyclerView
                LazyColumn(
                    modifier = Modifier.weight(1f)
                ) {
                    itemsIndexed(filteredSongs) { index, song ->
                        val isCurrentSong = currentSong?.id == song.id
                        
                        // Card: cada item de la lista (equivalente a ViewHolder)
                        Card(
                            modifier = Modifier
                                .fillMaxWidth()
                                .clickable { viewModel.playSong(song) },
                            colors = CardDefaults.cardColors(
                                containerColor = if (isCurrentSong) 
                                    Color(0xFFE3F2FD) 
                                else 
                                    Color(0xFFF7F7F7)
                            )
                        ) {
                            Row(
                                modifier = Modifier
                                    .fillMaxWidth()
                                    .padding(12.dp),
                                verticalAlignment = Alignment.CenterVertically,
                                horizontalArrangement = Arrangement.spacedBy(12.dp)
                            ) {
                                // Carátula del álbum
                                Box(
                                    modifier = Modifier
                                        .size(56.dp)
                                        .clip(RoundedCornerShape(8.dp))
                                        .background(Color(0xFF1976D2))
                                ) {
                                    Icon(
                                        imageVector = Icons.Filled.MusicNote,
                                        contentDescription = null,
                                        tint = Color.White,
                                        modifier = Modifier
                                            .align(Alignment.Center)
                                            .size(32.dp)
                                    )
                                }
                                
                                // Información de la canción
                                Column(modifier = Modifier.weight(1f)) {
                                    Text(
                                        text = song.title,
                                        fontWeight = FontWeight.Bold,
                                        fontSize = 16.sp,
                                        maxLines = 1
                                    )
                                    Text(
                                        text = song.artist,
                                        fontSize = 14.sp,
                                        color = Color.Gray,
                                        maxLines = 1
                                    )
                                    Text(
                                        text = song.album,
                                        fontSize = 12.sp,
                                        color = Color.Gray,
                                        maxLines = 1
                                    )
                                }
                                
                                // Duración
                                Text(
                                    text = formatTime(song.duration),
                                    fontSize = 12.sp,
                                    color = Color.Gray
                                )
                            }
                        }
                    }
                }
            }
        }
    }
}
```

**Figura 28:** LazyColumn para lista de ubicaciones

```kotlin
// MainActivity.kt - Líneas 1384-1483 (LocationsScreen - Lista de ubicaciones)
LazyColumn(
    verticalArrangement = Arrangement.spacedBy(12.dp)
) {
    items(savedLocations.size) { index ->
        val location = savedLocations[index]
        val associatedPlaylist = playlists.find { it.locationId == location.id }
        
        Card(
            modifier = Modifier.fillMaxWidth(),
            colors = CardDefaults.cardColors(containerColor = Color(0xFFF7F7F7))
        ) {
            Column(
                modifier = Modifier.padding(16.dp),
                verticalArrangement = Arrangement.spacedBy(8.dp)
            ) {
                Row(
                    modifier = Modifier.fillMaxWidth(),
                    horizontalArrangement = Arrangement.SpaceBetween,
                    verticalAlignment = Alignment.CenterVertically
                ) {
                    Column(modifier = Modifier.weight(1f)) {
                        Text(
                            text = "${location.category.icon} ${location.name}",
                            fontWeight = FontWeight.Bold,
                            fontSize = 16.sp
                        )
                        Text(
                            text = location.category.displayName,
                            fontSize = 12.sp,
                            color = Color.Gray
                        )
                    }
                }
            }
        }
    }
}
```

**Figura 29:** LazyColumn con checkboxes para selección múltiple

```kotlin
// MainActivity.kt - Líneas 1625-1662 (AddPlaylistDialog - Lista de canciones con checkboxes)
LazyColumn(
    modifier = Modifier.weight(1f)
) {
    items(allSongs.size) { index ->
        val song = allSongs[index]
        val isSelected = selectedSongs.contains(song.id)
        
        Row(
            modifier = Modifier
                .fillMaxWidth()
                .clickable {
                    selectedSongs = if (isSelected) {
                        selectedSongs - song.id
                    } else {
                        selectedSongs + song.id
                    }
                }
                .padding(8.dp),
            verticalAlignment = Alignment.CenterVertically
        ) {
            Checkbox(
                checked = isSelected,
                onCheckedChange = {
                    selectedSongs = if (it) {
                        selectedSongs + song.id
                    } else {
                        selectedSongs - song.id
                    }
                }
            )
            Spacer(modifier = Modifier.width(8.dp))
            Column(modifier = Modifier.weight(1f)) {
                Text(song.title, fontWeight = FontWeight.Medium)
                Text("${song.artist} • ${song.album}", fontSize = 12.sp, color = Color.Gray)
            }
        }
    }
}
```

**Sugerencia de Interfaz:**
> 💡 **Pantalla "Mi Música"** - Lista desplazable de todas las canciones con carátula, título, artista, álbum y duración.
> 💡 **Pantalla "Ubicaciones"** - Lista de ubicaciones GPS guardadas con sus listas de reproducción asociadas.
> 💡 **Diálogo "Crear Playlist"** - Lista de canciones con checkboxes para seleccionar.

---

### 2. Material Design

**Descripción:**
Material Design es el sistema de diseño de Google que proporciona principios, componentes y herramientas para crear interfaces consistentes y atractivas. Incluye colores, tipografía, elevación, sombras, y animaciones.

**Implementación en el Reproductor:**

**Figura 30:** Principios de Material Design aplicados

```kotlin
// MainActivity.kt - Principios de Material Design aplicados

// 1. Elevación y Sombras con Cards
Card(
    modifier = Modifier.fillMaxWidth(),
    colors = CardDefaults.cardColors(containerColor = Color(0xFFF7F7F7))
) {
    // Contenido
}

// 2. Colores del tema Material
colors = ButtonDefaults.buttonColors(
    containerColor = Color(0xFF1976D2), // Material Blue 700
    contentColor = Color.White
)

// 3. Tipografía Material
Text(
    text = "Título",
    fontSize = 24.sp,
    fontWeight = FontWeight.Bold
)

// 4. Iconografía Material
Icon(
    imageVector = Icons.Filled.MusicNote,
    contentDescription = "Música",
    tint = Color(0xFF1976D2)
)

// 5. Formas redondeadas (Material Shape)
shape = RoundedCornerShape(28.dp)

// 6. Espaciado consistente
Arrangement.spacedBy(16.dp)
Modifier.padding(16.dp)

// 7. Feedback visual (Ripple effect en clickable)
Modifier.clickable { /* acción */ }
```

**Figura 31:** Switches Material en SettingsScreen

```kotlin
// MainActivity.kt - Líneas 1194-1233 (SettingsScreen - Switches Material)
Row(
    modifier = Modifier.fillMaxWidth(),
    horizontalArrangement = Arrangement.SpaceBetween,
    verticalAlignment = Alignment.CenterVertically
) {
    Text("Tema oscuro")
    Switch(
        checked = darkTheme,
        onCheckedChange = { viewModel.setDarkTheme(it) }
    )
}
```

**Sugerencia de Interfaz:**
> 💡 **Toda la Aplicación** - Uso consistente de colores Material (azul #1976D2), Cards con elevación, iconos Material, y espaciado uniforme.
> 💡 **Pantalla de Configuración** - Switches Material para preferencias.

---

### 3. CardView (Card en Compose)

**Descripción:**
CardView es un contenedor con bordes redondeados y sombra que sigue los principios de Material Design. En Compose, se usa el componente `Card` que proporciona elevación, forma, y colores personalizables.

**Implementación en el Reproductor:**

**Figura 32:** Card para items de lista

```kotlin
// MainActivity.kt - Card para items de lista (Mi Música)
Card(
    modifier = Modifier
        .fillMaxWidth()
        .clickable { viewModel.playSong(song) },
    colors = CardDefaults.cardColors(
        containerColor = if (isCurrentSong) 
            Color(0xFFE3F2FD) 
        else 
            Color(0xFFF7F7F7)
    )
) {
    Row(
        modifier = Modifier
            .fillMaxWidth()
            .padding(12.dp),
        verticalAlignment = Alignment.CenterVertically
    ) {
        // Contenido de la canción
    }
}
```

**Figura 33:** Card para mensajes y notificaciones

```kotlin
// Card para mensajes (Ubicaciones detectadas)
Card(
    modifier = Modifier.fillMaxWidth(),
    colors = CardDefaults.cardColors(containerColor = Color(0xFFE3F2FD))
) {
    Column(
        modifier = Modifier.padding(16.dp),
        verticalArrangement = Arrangement.spacedBy(8.dp)
    ) {
        Text(
            text = "📍 Ubicación detectada: ${location.name}",
            fontWeight = FontWeight.Bold,
            fontSize = 16.sp
        )
        Button(onClick = { /* acción */ }) {
            Text("Reproducir ${playlist.name}")
        }
    }
}
```

**Figura 34:** Card para información y configuración

```kotlin
// Card para información (Configuración)
Card(
    modifier = Modifier.fillMaxWidth(),
    colors = CardDefaults.cardColors(containerColor = Color(0xFFF7F7F7))
) {
    Column(
        modifier = Modifier.padding(16.dp),
        verticalArrangement = Arrangement.spacedBy(12.dp)
    ) {
        Text(
            text = "Funcionalidades",
            fontSize = 18.sp,
            fontWeight = FontWeight.Bold
        )
        Text(
            text = "• Escaneo automático de música del dispositivo",
            fontSize = 14.sp
        )
    }
}
```

**Figura 35:** Card para errores

```kotlin
// Card para errores (Autenticación)
Card(
    modifier = Modifier.fillMaxWidth(),
    colors = CardDefaults.cardColors(containerColor = Color(0xFFFFEBEE))
) {
    Text(
        text = error,
        modifier = Modifier.padding(16.dp),
        color = Color(0xFFC62828),
        fontSize = 14.sp
    )
}
```

**Figura 36:** Card para perfil de usuario

```kotlin
// Card para perfil de usuario (Mi Nube)
Card(
    modifier = Modifier.fillMaxWidth(),
    colors = CardDefaults.cardColors(containerColor = Color(0xFFF7F7F7))
) {
    Row(
        modifier = Modifier
            .fillMaxWidth()
            .padding(12.dp),
        verticalAlignment = Alignment.CenterVertically,
        horizontalArrangement = Arrangement.spacedBy(12.dp)
    ) {
        Icon(
            imageVector = Icons.Filled.Person,
            contentDescription = "Usuario",
            modifier = Modifier.size(48.dp),
            tint = Color(0xFF1976D2)
        )
        Column(modifier = Modifier.weight(1f)) {
            Text(
                text = userDisplayName ?: "Sin nombre",
                fontSize = 18.sp,
                fontWeight = FontWeight.Bold
            )
            Text(
                text = userEmail ?: "Sin email",
                fontSize = 14.sp,
                color = Color.Gray
            )
        }
    }
}
```

**Sugerencia de Interfaz:**
> 💡 **Pantalla "Mi Música"** - Cards grises para cada canción, con fondo azul claro para la canción actual.
> 💡 **Pantalla "Ubicaciones"** - Cards para ubicaciones guardadas y sugerencias de playlist.
> 💡 **Pantalla "Configuración"** - Cards para agrupar información y preferencias.
> 💡 **Pantalla "Mi Nube"** - Card para perfil de usuario y favoritos.

---

## 🛰️ SEMANA 9: Hardware - GPS y Sensores

### 1. Hardware

**Descripción:**
El hardware del dispositivo incluye componentes físicos como GPS, acelerómetro, giroscopio, cámara, etc. Android proporciona APIs para acceder a estos sensores y servicios de hardware a través del Context.

**Implementación en el Reproductor:**

**Figura 37:** Declaración de hardware en AndroidManifest

```xml
<!-- AndroidManifest.xml - Declaración de hardware -->
<!-- Permisos para GPS y ubicación -->
<uses-permission android:name="android.permission.ACCESS_FINE_LOCATION" />
<uses-permission android:name="android.permission.ACCESS_COARSE_LOCATION" />
<uses-permission android:name="android.permission.ACCESS_BACKGROUND_LOCATION" />

<!-- Permisos para sensores (hardware) - Acelerómetro y giroscopio -->
<uses-feature android:name="android.hardware.sensor.accelerometer" android:required="false" />
<uses-feature android:name="android.hardware.sensor.gyroscope" android:required="false" />
```

**Figura 38:** Verificación de disponibilidad de hardware

```kotlin
// SensorManager.kt - Líneas 20-28 (Verificar disponibilidad de hardware)
// Verificar si el dispositivo tiene acelerómetro
fun hasAccelerometer(): Boolean {
    return sensorManager.getDefaultSensor(Sensor.TYPE_ACCELEROMETER) != null
}

// Verificar si el dispositivo tiene giroscopio
fun hasGyroscope(): Boolean {
    return sensorManager.getDefaultSensor(Sensor.TYPE_GYROSCOPE) != null
}
```

**Sugerencia de Interfaz:**
> 💡 **Pantalla "Ubicaciones"** - Usa el hardware GPS para detectar la ubicación actual del usuario.

---

### 2. Sonido (Reproducción de Audio)

**Descripción:**
Android proporciona MediaPlayer y ExoPlayer para reproducir audio. El reproductor implementa un servicio en segundo plano para continuar la reproducción incluso cuando la app está minimizada.

**Implementación en el Reproductor:**

**Figura 39:** Servicio de reproducción de música

```xml
<!-- AndroidManifest.xml - Servicio de reproducción -->
<service
    android:name=".service.MusicPlayerService"
    android:enabled="true"
    android:exported="false"
    android:foregroundServiceType="mediaPlayback" />
```

**Figura 40:** Controles de reproducción

```kotlin
// MainActivity.kt - Líneas 823-888 (Controles de reproducción)
@Composable
fun PlayerControls(
    isPlaying: Boolean,
    onPlayPause: () -> Unit,
    onPrevious: () -> Unit,
    onNext: () -> Unit
) {
    Row(
        modifier = Modifier.fillMaxWidth(),
        horizontalArrangement = Arrangement.SpaceEvenly,
        verticalAlignment = Alignment.CenterVertically
    ) {
        // Botón anterior
        IconButton(
            onClick = onPrevious,
            modifier = Modifier.size(64.dp)
        ) {
            Icon(
                imageVector = Icons.Filled.SkipPrevious,
                contentDescription = "Anterior",
                modifier = Modifier.size(48.dp)
            )
        }
        
        // Botón play/pause
        IconButton(
            onClick = onPlayPause,
            modifier = Modifier.size(80.dp)
        ) {
            Icon(
                imageVector = if (isPlaying) Icons.Filled.Pause else Icons.Filled.PlayArrow,
                contentDescription = if (isPlaying) "Pausar" else "Reproducir",
                modifier = Modifier.size(64.dp)
            )
        }
        
        // Botón siguiente
        IconButton(
            onClick = onNext,
            modifier = Modifier.size(64.dp)
        ) {
            Icon(
                imageVector = Icons.Filled.SkipNext,
                contentDescription = "Siguiente",
                modifier = Modifier.size(48.dp)
            )
        }
    }
}
```

**Sugerencia de Interfaz:**
> 💡 **Pantalla Principal del Reproductor** - Controles de reproducción (anterior, play/pause, siguiente) que controlan el servicio de audio.

---

### 3. Manejo de GPS

**Descripción:**
El GPS (Global Positioning System) permite obtener la ubicación geográfica del dispositivo. Android proporciona LocationManager para acceder a proveedores de ubicación (GPS, Network). El reproductor usa GPS para detectar ubicaciones guardadas y sugerir playlists automáticamente.

**Implementación en el Reproductor:**

**Figura 41:** Gestión de permisos y estado de GPS

```kotlin
// LocationManager.kt - Líneas 17-68 (Gestión de GPS)
class LocationManager(private val context: Context) {
    
    private val androidLocationManager: AndroidLocationManager =
        context.getSystemService(Context.LOCATION_SERVICE) as AndroidLocationManager
    
    // ExecutorService para manejar operaciones de GPS en hilos separados
    private val executorService = Executors.newSingleThreadExecutor()
    
    // Verificar si tiene permisos de ubicación
    fun hasLocationPermission(): Boolean {
        return ContextCompat.checkSelfPermission(
            context,
            Manifest.permission.ACCESS_FINE_LOCATION
        ) == PackageManager.PERMISSION_GRANTED ||
        ContextCompat.checkSelfPermission(
            context,
            Manifest.permission.ACCESS_COARSE_LOCATION
        ) == PackageManager.PERMISSION_GRANTED
    }
    
    // Verificar si el GPS está habilitado
    fun isGpsEnabled(): Boolean {
        return androidLocationManager.isProviderEnabled(AndroidLocationManager.GPS_PROVIDER) ||
               androidLocationManager.isProviderEnabled(AndroidLocationManager.NETWORK_PROVIDER)
    }
    
    // Obtener ubicación actual (una sola vez) usando hilos explícitos
    fun getCurrentLocation(callback: (Location?) -> Unit) {
        if (!hasLocationPermission()) {
            callback(null)
            return
        }
        
        executorService.execute {
            try {
                val location = if (androidLocationManager.isProviderEnabled(AndroidLocationManager.GPS_PROVIDER)) {
                    androidLocationManager.getLastKnownLocation(AndroidLocationManager.GPS_PROVIDER)
                } else if (androidLocationManager.isProviderEnabled(AndroidLocationManager.NETWORK_PROVIDER)) {
                    androidLocationManager.getLastKnownLocation(AndroidLocationManager.NETWORK_PROVIDER)
                } else {
                    null
                }
                
                callback(location)
            } catch (e: SecurityException) {
                callback(null)
            }
        }
    }
}
```

**Figura 42:** Actualizaciones de ubicación en tiempo real

```kotlin
// LocationManager.kt - Líneas 70-119 (Actualizaciones en tiempo real)
// Obtener actualizaciones de ubicación en tiempo real usando Flow
fun getLocationUpdates(): Flow<Location?> = callbackFlow {
    if (!hasLocationPermission()) {
        trySend(null)
        close()
        return@callbackFlow
    }
    
    val locationListener = object : LocationListener {
        override fun onLocationChanged(location: Location) {
            trySend(location)
        }
        
        override fun onProviderEnabled(provider: String) {}
        override fun onProviderDisabled(provider: String) {}
        override fun onStatusChanged(provider: String?, status: Int, extras: Bundle?) {}
    }
    
    try {
        if (androidLocationManager.isProviderEnabled(AndroidLocationManager.GPS_PROVIDER)) {
            androidLocationManager.requestLocationUpdates(
                AndroidLocationManager.GPS_PROVIDER,
                5000L, // Actualizar cada 5 segundos
                10f, // Mínimo 10 metros de cambio
                locationListener,
                Looper.getMainLooper()
            )
        } else if (androidLocationManager.isProviderEnabled(AndroidLocationManager.NETWORK_PROVIDER)) {
            androidLocationManager.requestLocationUpdates(
                AndroidLocationManager.NETWORK_PROVIDER,
                5000L,
                10f,
                locationListener,
                Looper.getMainLooper()
            )
        }
    } catch (e: SecurityException) {
        trySend(null)
        close()
    }
    
    awaitClose {
        try {
            androidLocationManager.removeUpdates(locationListener)
        } catch (e: Exception) {
            // Ignorar errores al remover updates
        }
    }
}
```

**Figura 43:** Detección de ubicaciones guardadas

```kotlin
// LocationManager.kt - Líneas 126-136 (Encontrar ubicación más cercana)
// Encontrar ubicación guardada más cercana a la ubicación actual
fun findNearestSavedLocation(
    currentLat: Double,
    currentLon: Double,
    savedLocations: List<SavedLocation>
): SavedLocation? {
    return savedLocations
        .filter { it.contains(currentLat, currentLon) }
        .minByOrNull { it.distanceTo(currentLat, currentLon) }
}
```

**Figura 44:** Solicitud de permisos de GPS

```kotlin
// MainActivity.kt - Líneas 123-132 (Solicitar permisos de GPS)
private val requestLocationPermissionLauncher = registerForActivityResult(
    ActivityResultContracts.RequestMultiplePermissions()
) { permissions ->
    if (permissions[Manifest.permission.ACCESS_FINE_LOCATION] == true ||
        permissions[Manifest.permission.ACCESS_COARSE_LOCATION] == true) {
        viewModel.checkLocationPermission()
        viewModel.getCurrentLocation()
    }
}
```

**Figura 45:** Mostrar ubicación actual en UI

```kotlin
// MainActivity.kt - Líneas 1336-1356 (Mostrar ubicación actual)
currentLocation?.let { loc ->
    Card(
        modifier = Modifier.fillMaxWidth(),
        colors = CardDefaults.cardColors(containerColor = Color(0xFFF5F5F5))
    ) {
        Column(
            modifier = Modifier.padding(16.dp)
        ) {
            Text(
                text = "Ubicación actual",
                fontWeight = FontWeight.Bold
            )
            Text(
                text = "Lat: ${String.format("%.6f", loc.latitude)}, Lon: ${String.format("%.6f", loc.longitude)}",
                fontSize = 12.sp,
                color = Color.Gray
            )
        }
    }
}
```

**Sugerencia de Interfaz:**
> 💡 **Pantalla "Ubicaciones"** - Card que muestra la ubicación GPS actual (latitud y longitud).
> 💡 **Pantalla "Ubicaciones"** - Card azul que sugiere una playlist cuando se detecta que el usuario está en una ubicación guardada.
> 💡 **Botón "Agregar Ubicación"** - Usa la ubicación GPS actual para guardar una nueva ubicación.

---

### 4. Sensores de Hardware (Acelerómetro y Giroscopio)

**Descripción:**
Los sensores de hardware como el acelerómetro y giroscopio detectan movimiento y orientación del dispositivo. El acelerómetro mide la aceleración en los ejes X, Y, Z, mientras que el giroscopio mide la velocidad de rotación.

**Implementación en el Reproductor:**

**Figura 46:** Lectura de datos del acelerómetro

```kotlin
// SensorManager.kt - Líneas 30-59 (Acelerómetro)
// Obtener datos del acelerómetro
fun getAccelerometerData(): Flow<FloatArray?> = callbackFlow {
    val sensor = sensorManager.getDefaultSensor(Sensor.TYPE_ACCELEROMETER)
    
    if (sensor == null) {
        trySend(null)
        close()
        return@callbackFlow
    }
    
    val listener = object : SensorEventListener {
        override fun onSensorChanged(event: SensorEvent?) {
            event?.values?.let { values ->
                trySend(values.clone())
            }
        }
        
        override fun onAccuracyChanged(sensor: Sensor?, accuracy: Int) {}
    }
    
    sensorManager.registerListener(
        listener,
        sensor,
        AndroidSensorManager.SENSOR_DELAY_NORMAL
    )
    
    awaitClose {
        sensorManager.unregisterListener(listener)
    }
}
```

**Figura 47:** Lectura de datos del giroscopio

```kotlin
// SensorManager.kt - Líneas 61-90 (Giroscopio)
// Obtener datos del giroscopio
fun getGyroscopeData(): Flow<FloatArray?> = callbackFlow {
    val sensor = sensorManager.getDefaultSensor(Sensor.TYPE_GYROSCOPE)
    
    if (sensor == null) {
        trySend(null)
        close()
        return@callbackFlow
    }
    
    val listener = object : SensorEventListener {
        override fun onSensorChanged(event: SensorEvent?) {
            event?.values?.let { values ->
                trySend(values.clone())
            }
        }
        
        override fun onAccuracyChanged(sensor: Sensor?, accuracy: Int) {}
    }
    
    sensorManager.registerListener(
        listener,
        sensor,
        AndroidSensorManager.SENSOR_DELAY_NORMAL
    )
    
    awaitClose {
        sensorManager.unregisterListener(listener)
    }
}
```

**Figura 48:** Detección de movimiento del dispositivo

```kotlin
// SensorManager.kt - Líneas 92-131 (Detectar movimiento)
// Detectar si el dispositivo está en movimiento basado en el acelerómetro
fun isDeviceMoving(threshold: Float = 0.5f): Flow<Boolean> = callbackFlow {
    val sensor = sensorManager.getDefaultSensor(Sensor.TYPE_ACCELEROMETER)
    
    if (sensor == null) {
        trySend(false)
        close()
        return@callbackFlow
    }
    
    var lastValues: FloatArray? = null
    
    val listener = object : SensorEventListener {
        override fun onSensorChanged(event: SensorEvent?) {
            event?.values?.let { currentValues ->
                if (lastValues != null) {
                    val deltaX = kotlin.math.abs(currentValues[0] - lastValues!![0])
                    val deltaY = kotlin.math.abs(currentValues[1] - lastValues!![1])
                    val deltaZ = kotlin.math.abs(currentValues[2] - lastValues!![2])
                    val totalDelta = deltaX + deltaY + deltaZ
                    
                    trySend(totalDelta > threshold)
                }
                lastValues = currentValues.clone()
            }
        }
        
        override fun onAccuracyChanged(sensor: Sensor?, accuracy: Int) {}
    }
    
    sensorManager.registerListener(
        listener,
        sensor,
        AndroidSensorManager.SENSOR_DELAY_NORMAL
    )
    
    awaitClose {
        sensorManager.unregisterListener(listener)
    }
}
```

**Sugerencia de Interfaz:**
> 💡 **Funcionalidad en Segundo Plano** - Los sensores pueden usarse para detectar cuando el usuario está caminando/corriendo y ajustar la reproducción automáticamente (aunque esta funcionalidad específica no está implementada en la UI actual).

---

## 📚 Recursos Adicionales

### Comparación: XML vs Jetpack Compose

Este proyecto usa **Jetpack Compose** en lugar de XML tradicional. Aquí está la equivalencia:

| Concepto XML | Equivalente en Compose |
|--------------|------------------------|
| Fragment | Composable condicional |
| RecyclerView | LazyColumn / LazyRow |
| ViewHolder | @Composable item |
| CardView | Card |
| TextView | Text |
| Button | Button |
| EditText | TextField / OutlinedTextField |
| ImageView | Image / Icon |
| LinearLayout | Column / Row |
| FrameLayout | Box |
| ConstraintLayout | Box + Modifier |
| Toast | Snackbar / Card con mensaje |

### Arquitectura de Managers

```
utils/
├── LocationManager.kt       # Gestión de GPS
├── SensorManager.kt         # Gestión de sensores
├── MusicScanner.kt          # Escaneo de archivos
├── PermissionManager.kt     # Gestión de permisos
├── Song.kt                  # Modelo de canción
├── Playlist.kt              # Modelo de playlist
└── Location.kt              # Modelo de ubicación
```

---

## 🎯 Resumen de Implementaciones

| Semana | Tema | Implementación Principal | Interfaz Sugerida |
|--------|------|-------------------------|-------------------|
| **5** | Fragment | Composables condicionales | Navegación entre pantallas |
| **5** | Ciclo de Vida | remember, LaunchedEffect | Indicador de páginas |
| **5** | Context | getSystemService, SharedPreferences | Toda la aplicación |
| **5** | Toast/SnackBar | Cards con mensajes | Mensajes de error/sugerencias |
| **6** | RecyclerView | LazyColumn | Listas de canciones/ubicaciones |
| **6** | Material Design | Cards, colores, tipografía | Toda la aplicación |
| **6** | CardView | Card component | Items de lista, mensajes |
| **9** | Hardware | Sensores, GPS | Pantalla de ubicaciones |
| **9** | Sonido | MusicPlayerService | Controles de reproducción |
| **9** | GPS | LocationManager | Ubicación actual, detección |
| **9** | Sensores | SensorManager | Detección de movimiento |

---

**Nota:** Este reproductor utiliza **Jetpack Compose** y arquitectura **MVVM** moderna, lo que hace que algunos conceptos tradicionales (Fragment, RecyclerView) se implementen de manera diferente pero con la misma funcionalidad.

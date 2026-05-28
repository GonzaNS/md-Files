# Guía de Implementación - Reproductor de Música KDM

Esta guía documenta la implementación de los temas cubiertos en las semanas 2, 3 y 4 del curso de Aplicaciones Móviles.

---

## 📱 SEMANA 2: Fundamentos de Android

### 1. Activity - Ciclo de Vida

**Descripción:**
Una Activity representa una pantalla única con interfaz de usuario en Android. El ciclo de vida de una Activity incluye estados como `onCreate()`, `onStart()`, `onResume()`, `onPause()`, `onStop()`, y `onDestroy()`. Comprender este ciclo es fundamental para gestionar recursos y mantener el estado de la aplicación.

**Implementación en el Reproductor:**

**Figura 1:** Ciclo de vida de Activity en MainActivity

```kotlin
// MainActivity.kt - Líneas 134-285
class MainActivity : ComponentActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        enableEdgeToEdge()
        
        setContent {
            Reproductor1Theme {
                val viewModel: MusicPlayerViewModel = viewModel()
                val hasStoragePermission by viewModel.hasStoragePermission.collectAsState()
                val showWelcomeScreens by viewModel.showWelcomeScreens.collectAsState()
                
                // Configuración inicial de la interfaz
                if (showWelcomeScreens) {
                    WelcomeScreens(
                        onComplete = { viewModel.completeWelcome() }
                    )
                } else {
                    // Pantalla principal del reproductor
                    MusicPlayerScreen(
                        viewModel = viewModel,
                        onRequestPermission = { requestStoragePermission() }
                    )
                }
            }
        }
    }
}
```

**Sugerencia de Interfaz:**
> 💡 **Pantalla de Bienvenida (WelcomeScreens)** - Las tres pantallas de bienvenida que se muestran la primera vez que se abre la aplicación, implementadas con el ciclo de vida de Activity.

---

### 2. Referencias en Android

**Descripción:**
Las referencias en Android permiten acceder a recursos (drawables, strings, layouts) y componentes de la UI. En Jetpack Compose, las referencias se manejan mediante estados reactivos (`State`, `MutableState`) y `remember` para mantener valores entre recomposiciones.

**Implementación en el Reproductor:**

**Figura 2:** Referencias con remember para mantener estado

```kotlin
// WelcomeScreens.kt - Líneas 28-29
@Composable
fun WelcomeScreens(
    onComplete: () -> Unit,
    modifier: Modifier = Modifier
) {
    var currentPage by remember { mutableStateOf(0) }
    
    // La variable currentPage mantiene su referencia entre recomposiciones
    // y permite navegar entre las diferentes pantallas de bienvenida
}
```

**Sugerencia de Interfaz:**
> 💡 **Indicador de Páginas** - Los puntos indicadores en la parte superior de las pantallas de bienvenida que muestran la página actual (líneas 37-51 de WelcomeScreens.kt).

---

### 3. Arquitectura para Aplicaciones Móviles

**Descripción:**
La arquitectura de aplicaciones móviles define cómo se organizan los componentes. Este proyecto utiliza **MVVM (Model-View-ViewModel)** con Jetpack Compose, separando la lógica de negocio (ViewModel) de la interfaz de usuario (Composables).

**Implementación en el Reproductor:**

**Figura 3:** Arquitectura MVVM del reproductor

```kotlin
// Estructura del proyecto:
// - Model: Song.kt, Playlist.kt, Location.kt (datos)
// - View: MainActivity.kt, WelcomeScreens.kt (UI con Compose)
// - ViewModel: MusicPlayerViewModel.kt (lógica de negocio)

// MainActivity.kt - Línea 140
val viewModel: MusicPlayerViewModel = viewModel()

// El ViewModel gestiona el estado de toda la aplicación
val hasStoragePermission by viewModel.hasStoragePermission.collectAsState()
val showWelcomeScreens by viewModel.showWelcomeScreens.collectAsState()
```

**Sugerencia de Interfaz:**
> 💡 **Pantalla Principal del Reproductor** - La interfaz completa que muestra la canción actual, controles de reproducción y navegación, todo gestionado por el ViewModel.

---

### 4. Modelos de Negocios con Aplicaciones Móviles

**Descripción:**
Los modelos de datos representan las entidades del negocio. En este reproductor, los modelos incluyen canciones, listas de reproducción, ubicaciones y usuarios.

**Implementación en el Reproductor:**

**Figura 4:** Modelos de datos del reproductor

```kotlin
// Song.kt - Modelo de canción
data class Song(
    val id: Int,
    val title: String,
    val artist: String,
    val album: String,
    val duration: Long,
    val path: String,
    val albumArt: ByteArray? = null
)

// Playlist.kt - Modelo de lista de reproducción
data class Playlist(
    val id: String,
    val name: String,
    val songIds: List<Int>,
    val locationId: String? = null
)

// Location.kt - Modelo de ubicación GPS
data class SavedLocation(
    val id: String,
    val name: String,
    val latitude: Double,
    val longitude: Double,
    val radius: Double,
    val category: LocationCategory
)
```

**Sugerencia de Interfaz:**
> 💡 **Pantalla "Mi Música"** - Lista de canciones que muestra todos los modelos de Song cargados desde el dispositivo.

---

## 🎨 SEMANA 3: Controles de Interfaz

### 1. ConstraintLayout

**Descripción:**
ConstraintLayout permite crear layouts complejos y flexibles mediante restricciones entre elementos. En Jetpack Compose, se utilizan `Box`, `Column`, `Row` y modificadores de alineación para lograr layouts similares.

**Implementación en el Reproductor:**

**Figura 5:** Layout de la pantalla principal con Column

```kotlin
// MainActivity.kt - Líneas 357-659 (MusicPlayerScreen)
@Composable
fun MusicPlayerScreen(
    viewModel: MusicPlayerViewModel,
    onRequestPermission: () -> Unit,
    modifier: Modifier = Modifier
) {
    Surface(modifier = modifier.fillMaxSize()) {
        Column(
            modifier = Modifier
                .fillMaxSize()
                .padding(16.dp),
            horizontalAlignment = Alignment.CenterHorizontally,
            verticalArrangement = Arrangement.spacedBy(24.dp)
        ) {
            // Carátula del álbum
            SongCover(song = currentSong, modifier = Modifier.weight(1f))
            
            // Información de la canción
            SongInfo(song = currentSong, viewModel = viewModel)
            
            // Barra de progreso
            TrackProgress(
                progress = progress,
                onProgressChanged = { viewModel.seekTo(it) },
                duration = currentSong.duration
            )
            
            // Controles de reproducción
            PlayerControls(
                isPlaying = isPlaying,
                onPlayPause = { viewModel.togglePlayPause() },
                onPrevious = { viewModel.playPrevious() },
                onNext = { viewModel.playNext() }
            )
        }
    }
}
```

**Sugerencia de Interfaz:**
> 💡 **Pantalla Principal del Reproductor** - Layout vertical con carátula, información de canción, barra de progreso y controles, todo organizado con Column y restricciones de peso.

---

### 2. Controles de Entrada (EditText/OutlinedTextField)

**Descripción:**
Los controles de entrada permiten al usuario ingresar texto. En Jetpack Compose se utiliza `OutlinedTextField` para campos de texto con bordes y etiquetas.

**Implementación en el Reproductor:**

**Figura 6:** Campos de texto en pantalla de autenticación

```kotlin
// MainActivity.kt - Líneas 1763-1794 (AuthScreen)
@Composable
fun AuthScreen(
    viewModel: MusicPlayerViewModel,
    authError: String?,
    isLoadingAuth: Boolean
) {
    // Campo de email
    OutlinedTextField(
        value = email,
        onValueChange = { email = it },
        label = { Text("Email") },
        modifier = Modifier.fillMaxWidth(),
        enabled = !isLoadingAuth,
        singleLine = true
    )
    
    // Campo de contraseña
    OutlinedTextField(
        value = password,
        onValueChange = { password = it },
        label = { Text("Contraseña") },
        modifier = Modifier.fillMaxWidth(),
        enabled = !isLoadingAuth,
        singleLine = true
    )
    
    // Campo de nombre (solo en modo registro)
    if (!isLoginMode) {
        OutlinedTextField(
            value = displayName,
            onValueChange = { displayName = it },
            label = { Text("Nombre de usuario") },
            modifier = Modifier.fillMaxWidth(),
            enabled = !isLoadingAuth,
            singleLine = true
        )
    }
}
```

**Sugerencia de Interfaz:**
> 💡 **Pantalla de Autenticación (Mi Nube)** - Formulario de login/registro con campos de texto para email, contraseña y nombre de usuario.

---

### 3. Button (Botones)

**Descripción:**
Los botones son elementos interactivos que ejecutan acciones cuando se presionan. En Compose, `Button` es altamente personalizable con colores, formas y contenido.

**Implementación en el Reproductor:**

**Figura 7:** Botón de navegación en pantallas de bienvenida

```kotlin
// WelcomeScreens.kt - Líneas 113-129 (Botón de navegación)
Button(
    onClick = onNext,
    modifier = Modifier
        .fillMaxWidth(0.8f)
        .height(56.dp),
    colors = ButtonDefaults.buttonColors(
        containerColor = Color.White,
        contentColor = Color(0xFF1976D2)
    ),
    shape = RoundedCornerShape(28.dp)
) {
    Text(
        text = "Siguiente",
        fontSize = 18.sp,
        fontWeight = FontWeight.Bold
    )
}
```

**Figura 8:** Botón de autenticación con indicador de carga

```kotlin
// MainActivity.kt - Líneas 1816-1842 (Botón de autenticación)
Button(
    onClick = {
        viewModel.clearAuthError()
        if (isLoginMode) {
            viewModel.signInUser(email, password)
        } else {
            if (displayName.isNotBlank()) {
                viewModel.registerUser(email, password, displayName)
            }
        }
    },
    modifier = Modifier.fillMaxWidth(),
    enabled = !isLoadingAuth && email.isNotBlank() && password.isNotBlank(),
    colors = ButtonDefaults.buttonColors(
        containerColor = Color(0xFF1976D2)
    )
) {
    if (isLoadingAuth) {
        CircularProgressIndicator(
            modifier = Modifier.size(20.dp),
            color = Color.White
        )
    } else {
        Text(if (isLoginMode) "Iniciar Sesión" else "Registrarse")
    }
}
```

**Sugerencia de Interfaz:**
> 💡 **Pantallas de Bienvenida** - Botones "Siguiente", "Anterior" y "Comenzar" con diseño redondeado y colores personalizados.
> 💡 **Pantalla de Autenticación** - Botón de login/registro con indicador de carga.

---

### 4. Checkboxes con Evento Clic

**Descripción:**
Los checkboxes permiten seleccionar múltiples opciones. Responden a eventos de clic para cambiar su estado entre seleccionado y no seleccionado.

**Implementación en el Reproductor:**

**Figura 9:** Checkboxes para selección múltiple de canciones

```kotlin
// MainActivity.kt - Líneas 1632-1660 (AddPlaylistDialog)
@Composable
fun AddPlaylistDialog(
    location: SavedLocation,
    allSongs: List<Song>,
    onDismiss: () -> Unit,
    onConfirm: (String, List<Int>) -> Unit
) {
    var selectedSongs by remember { mutableStateOf<Set<Int>>(emptySet()) }
    
    LazyColumn {
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
}
```

**Sugerencia de Interfaz:**
> 💡 **Diálogo "Crear Lista de Reproducción"** - Lista de canciones con checkboxes para seleccionar múltiples canciones al crear una playlist asociada a una ubicación.

---

### 5. Radio Buttons con Evento Clic

**Descripción:**
Los radio buttons permiten seleccionar una única opción de un grupo. Solo un radio button puede estar seleccionado a la vez dentro del mismo grupo.

**Implementación en el Reproductor:**

**Figura 10:** Radio buttons para selección de categoría

```kotlin
// MainActivity.kt - Líneas 1558-1573 (AddLocationDialog)
@Composable
fun AddLocationDialog(
    onDismiss: () -> Unit,
    onConfirm: (String, LocationCategory) -> Unit
) {
    var selectedCategory by remember { mutableStateOf(LocationCategory.HOME) }
    
    Column(verticalArrangement = Arrangement.spacedBy(12.dp)) {
        Text("Categoría:", fontWeight = FontWeight.Medium)
        
        LocationCategory.values().forEach { category ->
            Row(
                modifier = Modifier
                    .fillMaxWidth()
                    .clickable { selectedCategory = category }
                    .padding(8.dp),
                verticalAlignment = Alignment.CenterVertically
            ) {
                RadioButton(
                    selected = selectedCategory == category,
                    onClick = { selectedCategory = category }
                )
                Spacer(modifier = Modifier.width(8.dp))
                Text("${category.icon} ${category.displayName}")
            }
        }
    }
}
```

**Sugerencia de Interfaz:**
> 💡 **Diálogo "Agregar Ubicación"** - Sección de categorías (🏠 Casa, 💼 Trabajo, 🏋️ Gimnasio, etc.) con radio buttons para seleccionar una categoría de ubicación.

---

## 📍 SEMANA 4: Funcionalidades Avanzadas

### 1. Intent Explícitos

**Descripción:**
Los Intents explícitos inician componentes específicos de la aplicación, como Activities o Services. Se especifica exactamente qué componente debe ejecutarse.

**Implementación en el Reproductor:**

**Figura 11:** Declaración de Activity en AndroidManifest

```xml
<!-- AndroidManifest.xml - Líneas 35-45 -->
<activity
    android:name=".MainActivity"
    android:exported="true"
    android:label="@string/app_name"
    android:theme="@style/Theme.Reproductor1">
    <intent-filter>
        <action android:name="android.intent.action.MAIN" />
        <category android:name="android.intent.category.LAUNCHER" />
    </intent-filter>
</activity>
```

**Figura 12:** Declaración de servicio de música

```xml
<!-- AndroidManifest.xml - Líneas 49-53 (Servicio de música) -->
<service
    android:name=".service.MusicPlayerService"
    android:enabled="true"
    android:exported="false"
    android:foregroundServiceType="mediaPlayback" />
```

**Sugerencia de Interfaz:**
> 💡 **Icono de la Aplicación** - El launcher que inicia MainActivity mediante un intent explícito cuando el usuario toca el icono de la app.

---

### 2. ListView (LazyColumn en Compose)

**Descripción:**
ListView muestra una lista desplazable de elementos. En Jetpack Compose, se utiliza `LazyColumn` para renderizar eficientemente listas grandes, creando solo los elementos visibles.

**Implementación en el Reproductor:**

**Figura 13:** LazyColumn para lista de ubicaciones

```kotlin
// MainActivity.kt - Líneas 1384-1483 (LocationsScreen)
@Composable
fun LocationsScreen(
    viewModel: MusicPlayerViewModel,
    onRequestLocationPermission: () -> Unit,
    modifier: Modifier = Modifier
) {
    val savedLocations by viewModel.savedLocations.collectAsState()
    val playlists by viewModel.playlists.collectAsState()
    
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
}
```

**Figura 14:** LazyColumn para lista de canciones

```kotlin
// MainActivity.kt - Líneas 897-1089 (MyMusicScreen - Lista de canciones)
LazyColumn(
    modifier = Modifier.weight(1f)
) {
    itemsIndexed(filteredSongs) { index, song ->
        val isCurrentSong = currentSong?.id == song.id
        
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
                // Contenido de cada canción
                Column(modifier = Modifier.weight(1f)) {
                    Text(song.title, fontWeight = FontWeight.Bold)
                    Text(song.artist, fontSize = 14.sp, color = Color.Gray)
                }
            }
        }
    }
}
```

**Sugerencia de Interfaz:**
> 💡 **Pantalla "Ubicaciones"** - Lista desplazable de ubicaciones GPS guardadas con sus listas de reproducción asociadas.
> 💡 **Pantalla "Mi Música"** - Lista de todas las canciones disponibles en el dispositivo.

---

### 3. Evento Clic

**Descripción:**
Los eventos de clic permiten que los elementos de la UI respondan a las interacciones del usuario. En Compose, se utiliza el modificador `clickable` y callbacks como `onClick`.

**Implementación en el Reproductor:**

**Figura 15:** Controles de reproducción con eventos de clic

```kotlin
// MainActivity.kt - Líneas 823-888 (PlayerControls)
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
                modifier = Modifier.size(48.dp),
                tint = Color(0xFF1976D2)
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
                modifier = Modifier.size(64.dp),
                tint = Color(0xFF1976D2)
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
                modifier = Modifier.size(48.dp),
                tint = Color(0xFF1976D2)
            )
        }
    }
}
```

**Figura 16:** Botón de favoritos con evento de clic

```kotlin
// MainActivity.kt - Líneas 696-747 (SongInfo - Botón de favoritos)
IconButton(
    onClick = {
        if (isFavorite) {
            viewModel.removeFavorite(song.id)
        } else {
            viewModel.addFavorite(song.id)
        }
    }
) {
    Icon(
        imageVector = if (isFavorite) Icons.Filled.Favorite else Icons.Filled.FavoriteBorder,
        contentDescription = if (isFavorite) "Quitar de favoritos" else "Agregar a favoritos",
        tint = if (isFavorite) Color(0xFFE91E63) else Color.Gray,
        modifier = Modifier.size(28.dp)
    )
}
```

**Sugerencia de Interfaz:**
> 💡 **Controles de Reproducción** - Botones de anterior, play/pause y siguiente que responden a eventos de clic.
> 💡 **Botón de Favoritos** - Icono de corazón junto al título de la canción que cambia de estado al hacer clic.

---

## 📚 Recursos Adicionales

### Permisos Implementados

**Figura 17:** Permisos declarados en AndroidManifest

```xml
<!-- AndroidManifest.xml -->
<!-- Permisos de almacenamiento -->
<uses-permission android:name="android.permission.READ_EXTERNAL_STORAGE" />
<uses-permission android:name="android.permission.READ_MEDIA_AUDIO" />

<!-- Permisos de ubicación GPS (SEMANA 3) -->
<uses-permission android:name="android.permission.ACCESS_FINE_LOCATION" />
<uses-permission android:name="android.permission.ACCESS_COARSE_LOCATION" />

<!-- Permisos de internet para Firebase (SEMANA 4) -->
<uses-permission android:name="android.permission.INTERNET" />

<!-- Permisos para servicio en segundo plano -->
<uses-permission android:name="android.permission.FOREGROUND_SERVICE" />
```

### Arquitectura del Proyecto

```
app/
├── src/main/java/com/example/reproductor1/
│   ├── MainActivity.kt              # Activity principal
│   ├── ui/
│   │   ├── WelcomeScreens.kt       # Pantallas de bienvenida
│   │   └── theme/                   # Temas y colores
│   ├── viewmodel/
│   │   └── MusicPlayerViewModel.kt # Lógica de negocio
│   ├── service/
│   │   └── MusicPlayerService.kt   # Servicio de reproducción
│   └── utils/
│       ├── Song.kt                  # Modelo de canción
│       ├── Playlist.kt              # Modelo de playlist
│       ├── Location.kt              # Modelo de ubicación
│       ├── MusicScanner.kt          # Escaneo de música
│       ├── LocationManager.kt       # Gestión de GPS
│       └── PermissionManager.kt     # Gestión de permisos
```

---

## 🎯 Resumen de Implementaciones por Semana

| Semana | Tema | Implementación Principal | Interfaz Sugerida |
|--------|------|-------------------------|-------------------|
| **2** | Activity | MainActivity.kt | Pantalla de Bienvenida |
| **2** | Referencias | remember, mutableStateOf | Indicador de páginas |
| **2** | Arquitectura | MVVM con ViewModel | Toda la aplicación |
| **2** | Modelos | Song, Playlist, Location | Pantalla "Mi Música" |
| **3** | ConstraintLayout | Column, Row, Box | Reproductor principal |
| **3** | EditText | OutlinedTextField | Pantalla de autenticación |
| **3** | Button | Button con estilos | Pantallas de bienvenida |
| **3** | Checkbox | Selección múltiple | Crear playlist |
| **3** | Radio Button | Selección única | Categorías de ubicación |
| **4** | Intent | MainActivity, Service | Launcher de la app |
| **4** | ListView | LazyColumn | Listas de ubicaciones/canciones |
| **4** | Evento Clic | onClick, clickable | Controles de reproducción |

---

**Nota:** Este reproductor utiliza **Jetpack Compose**, el framework moderno de UI de Android, en lugar de XML tradicional. Los conceptos son los mismos, pero la sintaxis es declarativa y más concisa.

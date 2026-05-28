# Guía de Implementación - Reproductor de Música KDM
## Semanas 10, 11, 12, 13 y 14

Esta guía documenta la implementación de los temas cubiertos en las semanas 10-14 del curso de Aplicaciones Móviles.

---

## 🔧 SEMANA 10: Servicios Background e Hilos

### 1. Servicios Background

**Descripción:**
Los servicios en Android son componentes que ejecutan operaciones en segundo plano sin interfaz de usuario. Un **ForegroundService** es un tipo especial de servicio que muestra una notificación persistente y tiene mayor prioridad, ideal para tareas como reproducción de música.

**Implementación en el Reproductor:**

**Figura 49:** Declaración del servicio en AndroidManifest

```xml
<!-- AndroidManifest.xml - Líneas 47-53 (Declaración del servicio) -->
<!-- Servicio en segundo plano (ForegroundService) para reproducción de música -->
<!-- Permite reproducir música incluso cuando la app está en segundo plano -->
<service
    android:name=".service.MusicPlayerService"
    android:enabled="true"
    android:exported="false"
    android:foregroundServiceType="mediaPlayback" />
```

**Figura 50:** Clase MusicPlayerService

```kotlin
// MusicPlayerService.kt - Líneas 19-47 (Servicio de música)
// Servicio en segundo plano (ForegroundService) para reproducir música
// Permite reproducir música incluso cuando la app está en segundo plano
class MusicPlayerService : Service() {
    
    private val binder = MusicBinder()
    private var mediaPlayer: MediaPlayer? = null
    private var currentSong: Song? = null
    private var isPlaying = false
    
    // ExecutorService para manejar operaciones de audio en hilos separados
    private val executorService = Executors.newSingleThreadExecutor()
    
    inner class MusicBinder : Binder() {
        fun getService(): MusicPlayerService = this@MusicPlayerService
    }
    
    override fun onCreate() {
        super.onCreate()
        createNotificationChannel()
    }
    
    override fun onBind(intent: Intent?): IBinder {
        return binder
    }
    
    override fun onStartCommand(intent: Intent?, flags: Int, startId: Int): Int {
        return START_STICKY // Reiniciar servicio si se cierra
    }
}
```

**Figura 51:** Reproducción de música en background

```kotlin
// MusicPlayerService.kt - Líneas 49-72 (Reproducir canción en background)
// Reproducir canción
fun playSong(song: Song, filePath: String) {
    executorService.execute {
        try {
            stopCurrentSong()
            
            currentSong = song
            mediaPlayer = MediaPlayer().apply {
                setDataSource(filePath)
                prepare()
                setOnCompletionListener {
                    this@MusicPlayerService.isPlaying = false
                    this@MusicPlayerService.updateNotification()
                }
                start()
            }
            isPlaying = true
            
            updateNotification()
        } catch (e: Exception) {
            e.printStackTrace()
        }
    }
}
```

**Figura 52:** Creación de notificación persistente

```kotlin
// MusicPlayerService.kt - Líneas 120-156 (Notificación persistente)
private fun createNotificationChannel() {
    if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.O) {
        val channel = NotificationChannel(
            CHANNEL_ID,
            "Reproductor de Música",
            NotificationManager.IMPORTANCE_LOW
        ).apply {
            description = "Controles del reproductor de música"
            setShowBadge(false)
        }
        
        val notificationManager = getSystemService(NotificationManager::class.java)
        notificationManager.createNotificationChannel(channel)
    }
}

private fun updateNotification() {
    val song = currentSong ?: return
    
    val intent = Intent(this, MainActivity::class.java)
    val pendingIntent = PendingIntent.getActivity(
        this, 0, intent,
        PendingIntent.FLAG_UPDATE_CURRENT or PendingIntent.FLAG_IMMUTABLE
    )
    
    val notification = NotificationCompat.Builder(this, CHANNEL_ID)
        .setSmallIcon(R.drawable.ic_launcher_foreground)
        .setContentTitle(song.title)
        .setContentText(song.artist)
        .setPriority(NotificationCompat.PRIORITY_LOW)
        .setOngoing(isPlaying)
        .setVisibility(NotificationCompat.VISIBILITY_PUBLIC)
        .setContentIntent(pendingIntent)
        .build()
    
    startForeground(NOTIFICATION_ID, notification)
}
```

**Sugerencia de Interfaz:**
> 💡 **Notificación del Sistema** - Notificación persistente que muestra la canción actual y permite controlar la reproducción desde fuera de la app.

---

### 2. Aplicativos en Segundo Plano

**Descripción:**
Los aplicativos en segundo plano permiten que la aplicación continúe ejecutando tareas cuando el usuario no está interactuando directamente con ella. El reproductor usa un servicio foreground para mantener la música reproduciéndose incluso cuando la app está minimizada.

**Implementación en el Reproductor:**

**Figura 53:** Controles de reproducción en background

```kotlin
// MusicPlayerService.kt - Líneas 74-98 (Controles de reproducción en background)
// Pausar reproducción
fun pause() {
    executorService.execute {
        mediaPlayer?.pause()
        isPlaying = false
        updateNotification()
    }
}

// Reanudar reproducción
fun resume() {
    executorService.execute {
        mediaPlayer?.start()
        isPlaying = true
        updateNotification()
    }
}

// Detener reproducción
fun stop() {
    executorService.execute {
        stopCurrentSong()
        updateNotification()
    }
}
```

**Figura 54:** Estado y control del servicio

```kotlin
// MusicPlayerService.kt - Líneas 100-109 (Estado y control)
// Obtener estado actual
fun getCurrentSong(): Song? = currentSong
fun isPlaying(): Boolean = isPlaying
fun getCurrentPosition(): Int = mediaPlayer?.currentPosition ?: 0
fun getDuration(): Int = mediaPlayer?.duration ?: 0
fun seekTo(position: Int) {
    executorService.execute {
        mediaPlayer?.seekTo(position)
    }
}
```

**Figura 55:** Limpieza de recursos del servicio

```kotlin
// MusicPlayerService.kt - Líneas 158-162 (Limpieza de recursos)
override fun onDestroy() {
    super.onDestroy()
    stopCurrentSong()
    executorService.shutdown()
}
```

**Sugerencia de Interfaz:**
> 💡 **Reproducción Continua** - La música sigue reproduciéndose cuando minimizas la app o bloqueas la pantalla.

---

### 3. Hilos (Threads)

**Descripción:**
Los hilos permiten ejecutar operaciones en paralelo sin bloquear el hilo principal de la UI. Android proporciona **ExecutorService** para gestionar hilos de manera eficiente. El reproductor usa hilos para operaciones de audio y GPS.

**Implementación en el Reproductor:**

**Figura 56:** ExecutorService para operaciones de audio

```kotlin
// MusicPlayerService.kt - Líneas 17, 29-30 (ExecutorService para audio)
import java.util.concurrent.Executors

// ExecutorService para manejar operaciones de audio en hilos separados
private val executorService = Executors.newSingleThreadExecutor()

// Uso del ExecutorService para operaciones de audio
executorService.execute {
    try {
        stopCurrentSong()
        currentSong = song
        mediaPlayer = MediaPlayer().apply {
            setDataSource(filePath)
            prepare()
            start()
        }
        isPlaying = true
        updateNotification()
    } catch (e: Exception) {
        e.printStackTrace()
    }
}
```

**Figura 57:** ExecutorService para operaciones de GPS

```kotlin
// LocationManager.kt - Líneas 15, 25-26 (ExecutorService para GPS)
import java.util.concurrent.Executors

// ExecutorService para manejar operaciones de GPS en hilos separados
private val executorService = Executors.newSingleThreadExecutor()
```

**Figura 58:** Obtención de ubicación en hilo separado

```kotlin
// LocationManager.kt - Líneas 46-68 (Obtener ubicación en hilo separado)
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
```

**Sugerencia de Interfaz:**
> 💡 **Operaciones en Background** - Las operaciones de audio y GPS se ejecutan en hilos separados para no bloquear la interfaz de usuario.

---

## 📋 SEMANA 11: Menú de Opciones

### 1. Menú de Opciones (DropdownMenu)

**Descripción:**
El menú de opciones proporciona acciones secundarias y opciones de configuración. En Jetpack Compose, se utiliza **DropdownMenu** que se muestra al presionar un botón de menú (generalmente con el icono de tres puntos verticales).

**Implementación en el Reproductor:**

**Figura 59:** Menú de opciones en "Mi Música"

```kotlin
// MainActivity.kt - Líneas 1024-1069 (Menú de opciones en "Mi Música")
Box {
    IconButton(onClick = { menuExpanded = true }) {
        Icon(imageVector = Icons.Filled.MoreVert, contentDescription = "Más opciones")
    }
    DropdownMenu(
        expanded = menuExpanded,
        onDismissRequest = { menuExpanded = false }
    ) {
        var showSort by remember { mutableStateOf(false) }
        
        // Opción: Clasificar por
        DropdownMenuItem(
            text = { Text("Clasificar por") },
            onClick = { showSort = !showSort }
        )
        
        // Submenú de clasificación
        if (showSort) {
            DropdownMenuItem(
                text = { Text("Título") },
                onClick = {
                    viewModel.changeSort(MusicPlayerViewModel.SortBy.TITLE)
                    menuExpanded = false
                }
            )
            DropdownMenuItem(
                text = { Text("Artista") },
                onClick = {
                    viewModel.changeSort(MusicPlayerViewModel.SortBy.ARTIST)
                    menuExpanded = false
                }
            )
            DropdownMenuItem(
                text = { Text("Álbum") },
                onClick = {
                    viewModel.changeSort(MusicPlayerViewModel.SortBy.ALBUM)
                    menuExpanded = false
                }
            )
        }
        
        // Opción: Detalles de biblioteca
        DropdownMenuItem(
            text = { Text("Detalles") },
            onClick = {
                detailsDialog = true
                menuExpanded = false
            }
        )
    }
}
```

**Figura 60:** Menú lateral de navegación (NavigationDrawer)

```kotlin
// MainActivity.kt - Líneas 158-230 (Menú lateral de navegación)
ModalNavigationDrawer(
    drawerState = drawerState,
    drawerContent = {
        ModalDrawerSheet(
            modifier = Modifier.width(screenWidth * 0.8f),
            drawerContainerColor = Color.White
        ) {
            Text(
                text = "Menú",
                modifier = Modifier.padding(16.dp),
                fontWeight = FontWeight.Bold,
                fontSize = 18.sp
            )
            
            // Opción: Inicio
            NavigationDrawerItem(
                label = { Text("Inicio") },
                selected = currentScreen == AppScreen.Home,
                onClick = {
                    currentScreen = AppScreen.Home
                    scope.launch { drawerState.close() }
                },
                icon = { Icon(imageVector = Icons.Filled.PlayArrow, contentDescription = null) }
            )
            
            // Opción: Mi música
            NavigationDrawerItem(
                label = { Text("Mi música") },
                selected = currentScreen == AppScreen.MyMusic,
                onClick = {
                    currentScreen = AppScreen.MyMusic
                    scope.launch { drawerState.close() }
                },
                icon = { Icon(imageVector = Icons.Filled.Search, contentDescription = null) }
            )
            
            // Opción: Ubicaciones
            NavigationDrawerItem(
                label = { Text("Ubicaciones") },
                selected = currentScreen == AppScreen.Locations,
                onClick = {
                    currentScreen = AppScreen.Locations
                    scope.launch { drawerState.close() }
                },
                icon = { Icon(imageVector = Icons.Filled.LocationOn, contentDescription = null) }
            )
            
            // Opción: Configuración
            NavigationDrawerItem(
                label = { Text("Configuración") },
                selected = currentScreen == AppScreen.Settings,
                onClick = {
                    currentScreen = AppScreen.Settings
                    scope.launch { drawerState.close() }
                },
                icon = { Icon(imageVector = Icons.Filled.Settings, contentDescription = null) }
            )
            
            // Opción: Mi Nube (Firebase)
            NavigationDrawerItem(
                label = { Text("Mi Nube") },
                selected = currentScreen == AppScreen.Cloud,
                onClick = {
                    currentScreen = AppScreen.Cloud
                    scope.launch { drawerState.close() }
                },
                icon = { Icon(imageVector = Icons.Filled.Cloud, contentDescription = null) }
            )
        }
    }
)
```

**Sugerencia de Interfaz:**
> 💡 **Menú Lateral (Drawer)** - Menú deslizable desde la izquierda con opciones de navegación: Inicio, Mi música, Ubicaciones, Configuración y Mi Nube.
> 💡 **Menú de Opciones en "Mi Música"** - Icono de tres puntos verticales que despliega opciones para clasificar canciones y ver detalles de la biblioteca.

---

## ☁️ SEMANA 12: Firebase y Authentication

### 1. Servicios de Google

**Descripción:**
Google proporciona servicios en la nube como Firebase para autenticación, base de datos, almacenamiento y más. Firebase es una plataforma completa para desarrollo de aplicaciones móviles y web.

**Implementación en el Reproductor:**

**Figura 61:** Permisos de internet para Firebase

```xml
<!-- AndroidManifest.xml - Líneas 22-23 (Permisos de internet para Firebase) -->
<!-- Firebase: Permisos de internet -->
<uses-permission android:name="android.permission.INTERNET" />
```

**Figura 62:** Importaciones de Firebase

```kotlin
// MusicPlayerViewModel.kt - Líneas 33-36 (Importaciones de Firebase)
// Importaciones para Firebase Authentication y Firestore
import com.google.firebase.auth.FirebaseAuth
import com.google.firebase.auth.FirebaseUser
import com.google.firebase.firestore.FirebaseFirestore
```

**Figura 63:** Inicialización de Firebase

```kotlin
// MusicPlayerViewModel.kt - Líneas 127-129 (Inicialización de Firebase)
// Firebase Authentication y Firestore
private val firebaseAuth = FirebaseAuth.getInstance()
private val firestore = FirebaseFirestore.getInstance()
```

**Sugerencia de Interfaz:**
> 💡 **Pantalla "Mi Nube"** - Integración completa con servicios de Firebase para autenticación y almacenamiento.

---

### 2. Firebase

**Descripción:**
Firebase es la plataforma de desarrollo de aplicaciones de Google que proporciona herramientas para autenticación, base de datos en tiempo real, almacenamiento en la nube, analytics y más.

**Implementación en el Reproductor:**

**Figura 64:** Estados de usuario de Firebase

```kotlin
// MusicPlayerViewModel.kt - Líneas 132-133 (Estados de usuario)
private val _currentUser = MutableStateFlow<FirebaseUser?>(null)
val currentUser: StateFlow<FirebaseUser?> = _currentUser.asStateFlow()
```

**Sugerencia de Interfaz:**
> 💡 **Toda la sección "Mi Nube"** - Funcionalidad completa de Firebase integrada en la aplicación.

---

### 3. Authentication (Autenticación)

**Descripción:**
Firebase Authentication proporciona servicios de autenticación de usuarios con email/contraseña, Google, Facebook, etc. Permite registrar usuarios, iniciar sesión y gestionar sesiones de manera segura.

**Implementación en el Reproductor:**

**Figura 65:** Registro de usuario con Firebase

```kotlin
// MusicPlayerViewModel.kt - Líneas 766-798 (Registrar usuario)
// Registrar nuevo usuario con email y contraseña en Firebase
fun registerUser(email: String, password: String, displayName: String) {
    _isLoadingAuth.value = true
    _authError.value = null
    
    viewModelScope.launch {
        try {
            val result = firebaseAuth.createUserWithEmailAndPassword(email, password).await()
            val user = result.user
            if (user != null) {
                // Actualizar perfil con nombre de usuario
                val profileUpdates = com.google.firebase.auth.UserProfileChangeRequest.Builder()
                    .setDisplayName(displayName)
                    .build()
                user.updateProfile(profileUpdates).await()
                
                // Guardar datos adicionales en Firestore
                saveUserDataToFirestore(user.uid, email, displayName)
                
                _currentUser.value = user
                _isAuthenticated.value = true
                _userEmail.value = email
                _userDisplayName.value = displayName
                // Cargar favoritos después de registrar usuario
                loadFavoriteSongs()
            }
        } catch (e: Exception) {
            _authError.value = e.message ?: "Error al registrar usuario"
        } finally {
            _isLoadingAuth.value = false
        }
    }
}
```

**Figura 66:** Inicio de sesión con Firebase

```kotlin
// MusicPlayerViewModel.kt - Líneas 800-822 (Iniciar sesión)
// Iniciar sesión con email y contraseña en Firebase
fun signInUser(email: String, password: String) {
    _isLoadingAuth.value = true
    _authError.value = null
    
    viewModelScope.launch {
        try {
            val result = firebaseAuth.signInWithEmailAndPassword(email, password).await()
            val user = result.user
            if (user != null) {
                _currentUser.value = user
                _isAuthenticated.value = true
                _userEmail.value = user.email
                _userDisplayName.value = user.displayName
                loadUserDataFromFirestore(user.uid)
            }
        } catch (e: Exception) {
            _authError.value = e.message ?: "Error al iniciar sesión"
        } finally {
            _isLoadingAuth.value = false
        }
    }
}
```

**Figura 67:** Cerrar sesión de Firebase

```kotlin
// MusicPlayerViewModel.kt - Líneas 824-836 (Cerrar sesión)
// Cerrar sesión de Firebase
fun signOut() {
    firebaseAuth.signOut()
    _currentUser.value = null
    _isAuthenticated.value = false
    _userEmail.value = null
    _userDisplayName.value = null
    _authError.value = null
    // Limpiar favoritos y listener al cerrar sesión
    removeFavoritesListener()
    _favoriteSongs.value = emptyList()
    _favoriteSongIds.value = emptySet()
}
```

**Figura 68:** Pantalla de autenticación

```kotlin
// MainActivity.kt - Líneas 1717-1863 (Pantalla de autenticación)
// Pantalla de autenticación con opciones de login y registro
@Composable
fun AuthScreen(
    viewModel: MusicPlayerViewModel,
    authError: String?,
    isLoadingAuth: Boolean
) {
    var isLoginMode by remember { mutableStateOf(true) } // true = login, false = registro
    var email by remember { mutableStateOf("") }
    var password by remember { mutableStateOf("") }
    var displayName by remember { mutableStateOf("") }
    
    Column(
        modifier = Modifier
            .fillMaxSize()
            .padding(24.dp),
        horizontalAlignment = Alignment.CenterHorizontally,
        verticalArrangement = Arrangement.Center
    ) {
        Icon(
            imageVector = Icons.Filled.Cloud,
            contentDescription = "Mi nube",
            modifier = Modifier.size(80.dp),
            tint = Color(0xFF1976D2)
        )
        
        Text(
            text = if (isLoginMode) "Iniciar Sesión" else "Registrarse",
            fontSize = 28.sp,
            fontWeight = FontWeight.Bold
        )
        
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
        
        // Botón de acción principal
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
            enabled = !isLoadingAuth && email.isNotBlank() && password.isNotBlank()
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
    }
}
```

**Sugerencia de Interfaz:**
> 💡 **Pantalla "Mi Nube" - Formulario de Login** - Campos de email y contraseña con botón de "Iniciar Sesión".
> 💡 **Pantalla "Mi Nube" - Formulario de Registro** - Campos de email, contraseña y nombre de usuario con botón de "Registrarse".

---

## 🗺️ SEMANA 13: Google Maps y Cloud Storage

### 1. Google Maps

**Descripción:**
Google Maps permite integrar mapas interactivos en aplicaciones Android. Aunque el reproductor no muestra mapas visuales, utiliza las coordenadas GPS para detectar ubicaciones guardadas.

**Implementación en el Reproductor:**

**Figura 69:** Modelo de ubicación con coordenadas GPS

```kotlin
// Location.kt - Modelo de ubicación con coordenadas GPS
data class SavedLocation(
    val id: Int,
    val name: String,
    val latitude: Double,
    val longitude: Double,
    val radius: Float,
    val category: LocationCategory
) {
    // Calcular distancia a otra ubicación (fórmula de Haversine)
    fun distanceTo(lat: Double, lon: Double): Double {
        val R = 6371000.0 // Radio de la Tierra en metros
        val dLat = Math.toRadians(lat - latitude)
        val dLon = Math.toRadians(lon - longitude)
        val a = Math.sin(dLat / 2) * Math.sin(dLat / 2) +
                Math.cos(Math.toRadians(latitude)) * Math.cos(Math.toRadians(lat)) *
                Math.sin(dLon / 2) * Math.sin(dLon / 2)
        val c = 2 * Math.atan2(Math.sqrt(a), Math.sqrt(1 - a))
        return R * c
    }
    
    // Verificar si una coordenada está dentro del radio
    fun contains(lat: Double, lon: Double): Boolean {
        return distanceTo(lat, lon) <= radius
    }
}
```

**Figura 70:** Mostrar coordenadas GPS en UI

```kotlin
// MainActivity.kt - Líneas 1336-1356 (Mostrar coordenadas GPS)
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
> 💡 **Pantalla "Ubicaciones"** - Card que muestra las coordenadas GPS actuales (latitud y longitud).
> 💡 **Lista de Ubicaciones Guardadas** - Cada ubicación muestra su nombre, categoría y radio de detección.

---

### 2. Interfaces en Android Studio

**Descripción:**
Android Studio proporciona herramientas para diseñar interfaces de usuario. En Jetpack Compose, las interfaces se crean mediante código declarativo en lugar de XML visual.

**Implementación en el Reproductor:**

**Figura 71:** Composables para interfaces

```kotlin
// Todas las pantallas están creadas con Jetpack Compose
// Ejemplo: MainActivity.kt - Composables para cada pantalla

@Composable
fun MusicPlayerScreen(...) { /* Pantalla principal */ }

@Composable
fun MyMusicScreen(...) { /* Pantalla de biblioteca */ }

@Composable
fun LocationsScreen(...) { /* Pantalla de ubicaciones */ }

@Composable
fun SettingsScreen(...) { /* Pantalla de configuración */ }

@Composable
fun CloudScreen(...) { /* Pantalla de Firebase */ }
```

**Sugerencia de Interfaz:**
> 💡 **Todas las Pantallas** - Diseñadas con Jetpack Compose en Android Studio.

---

### 3. Almacenamiento de Imágenes y Cloud Storage

**Descripción:**
Cloud Storage permite almacenar y servir archivos (imágenes, videos, audio) en la nube. Firestore se usa para almacenar datos estructurados como información de usuarios y canciones favoritas.

**Implementación en el Reproductor:**

**Figura 72:** Guardar datos de usuario en Firestore

```kotlin
// MusicPlayerViewModel.kt - Líneas 838-856 (Guardar datos en Firestore)
// Guardar datos de usuario en Firestore
private fun saveUserDataToFirestore(uid: String, email: String, displayName: String) {
    val userData = hashMapOf(
        "email" to email,
        "displayName" to displayName,
        "createdAt" to com.google.firebase.Timestamp.now()
    )
    
    firestore.collection("users")
        .document(uid)
        .set(userData)
        .addOnSuccessListener {
            // Datos guardados exitosamente
        }
        .addOnFailureListener { e ->
            // Error al guardar
            e.printStackTrace()
        }
}
```

**Figura 73:** Guardar canción favorita en Firestore

```kotlin
// MusicPlayerViewModel.kt - Líneas 888-930 (Guardar canción favorita en Firestore)
// Agregar canción a favoritos y guardar en Firestore
fun addToFavorites(song: Song) {
    if (!_isAuthenticated.value || _currentUser.value == null) {
        return // No se puede agregar si no está autenticado
    }
    
    val userId = _currentUser.value!!.uid
    val favoriteSongIds = _favoriteSongIds.value.toMutableSet()
    favoriteSongIds.add(song.id)
    _favoriteSongIds.value = favoriteSongIds
    
    // Guardar en Firestore
    val favoriteData = hashMapOf(
        "songId" to song.id,
        "title" to song.title,
        "artist" to song.artist,
        "album" to song.album,
        "duration" to song.duration,
        "filePath" to song.filePath,
        "addedAt" to com.google.firebase.Timestamp.now()
    )
    
    firestore.collection("users")
        .document(userId)
        .collection("favorites")
        .document(song.id.toString())
        .set(favoriteData)
        .addOnSuccessListener {
            // La actualización se hará automáticamente por el listener en tiempo real
            val favorites = _favoriteSongs.value.toMutableList()
            if (!favorites.any { it.id == song.id }) {
                favorites.add(song)
                _favoriteSongs.value = favorites
            }
        }
        .addOnFailureListener { e ->
            // Revertir cambio local si falla
            favoriteSongIds.remove(song.id)
            _favoriteSongIds.value = favoriteSongIds
            e.printStackTrace()
        }
}
```

**Sugerencia de Interfaz:**
> 💡 **Pantalla "Mi Nube" - Perfil de Usuario** - Muestra información del usuario almacenada en Firestore.
> 💡 **Pantalla "Mi Nube" - Favoritos** - Lista de canciones favoritas sincronizadas con Firestore.

---

## ⚡ SEMANA 14: Tiempo Real y Cloud Storage

### 1. Tiempo Real usando Realtime Database y Cloud Storage

**Descripción:**
Firebase Realtime Database y Firestore permiten sincronización de datos en tiempo real. Los cambios en la base de datos se reflejan automáticamente en todos los clientes conectados mediante **listeners**.

**Implementación en el Reproductor:**

**Figura 74:** Listener de Firestore para tiempo real

```kotlin
// MusicPlayerViewModel.kt - Líneas 972 (Listener de Firestore)
// Listener de Firestore para actualización en tiempo real
private var favoritesListener: com.google.firebase.firestore.ListenerRegistration? = null
```

**Figura 75:** Cargar favoritos en tiempo real

```kotlin
// MusicPlayerViewModel.kt - Líneas 974-1023 (Cargar favoritos en tiempo real)
// Cargar canciones favoritas desde Firestore con listener en tiempo real
fun loadFavoriteSongs() {
    val userId = _currentUser.value?.uid ?: return
    
    // Cancelar listener anterior si existe
    favoritesListener?.remove()
    
    // Configurar listener en tiempo real para actualización automática
    favoritesListener = firestore.collection("users")
        .document(userId)
        .collection("favorites")
        .addSnapshotListener { snapshot, error ->
            if (error != null) {
                error.printStackTrace()
                return@addSnapshotListener
            }
            
            val favorites = mutableListOf<Song>()
            val favoriteIds = mutableSetOf<Int>()
            
            snapshot?.documents?.forEach { document ->
                try {
                    val songId = document.getLong("songId")?.toInt() ?: return@forEach
                    val title = document.getString("title") ?: ""
                    val artist = document.getString("artist") ?: ""
                    val album = document.getString("album") ?: "Álbum desconocido"
                    val duration = document.getLong("duration") ?: 0L
                    val filePath = document.getString("filePath") ?: ""
                    
                    val favoriteSong = Song(
                        id = songId,
                        title = title,
                        artist = artist,
                        album = album,
                        duration = duration,
                        coverResId = R.drawable.ic_launcher_foreground,
                        filePath = filePath
                    )
                    
                    favorites.add(favoriteSong)
                    favoriteIds.add(songId)
                } catch (e: Exception) {
                    e.printStackTrace()
                }
            }
            
            _favoriteSongs.value = favorites
            _favoriteSongIds.value = favoriteIds
        }
}
```

**Figura 76:** Limpiar listener de tiempo real

```kotlin
// MusicPlayerViewModel.kt - Líneas 1025-1029 (Limpiar listener)
// Limpiar listener cuando se cierra sesión o se destruye el ViewModel
private fun removeFavoritesListener() {
    favoritesListener?.remove()
    favoritesListener = null
}
```

**Figura 77:** Listener de autenticación en tiempo real

```kotlin
// MusicPlayerViewModel.kt - Líneas 744-763 (Listener de autenticación)
// Escuchar cambios en el estado de autenticación de Firebase
firebaseAuth.addAuthStateListener { auth ->
    val currentUser = auth.currentUser
    _currentUser.value = currentUser
    _isAuthenticated.value = currentUser != null
    if (currentUser != null) {
        _userEmail.value = currentUser.email
        _userDisplayName.value = currentUser.displayName
        loadUserDataFromFirestore(currentUser.uid)
        // Cargar favoritos cuando el usuario inicia sesión
        loadFavoriteSongs()
    } else {
        _userEmail.value = null
        _userDisplayName.value = null
        // Limpiar favoritos y listener cuando el usuario cierra sesión
        removeFavoritesListener()
        _favoriteSongs.value = emptyList()
        _favoriteSongIds.value = emptySet()
    }
}
```

**Sugerencia de Interfaz:**
> 💡 **Pantalla "Mi Nube" - Lista de Favoritos** - Se actualiza automáticamente en tiempo real cuando agregas o quitas favoritos.
> 💡 **Botón de Favoritos** - El icono de corazón cambia instantáneamente al hacer clic y se sincroniza con la nube.

---

## 📚 Recursos Adicionales

### Arquitectura de Servicios y Firebase

```
Servicios:
├── MusicPlayerService.kt        # Servicio foreground para reproducción
│   ├── ExecutorService          # Hilos para operaciones de audio
│   ├── MediaPlayer              # Reproducción de música
│   └── Notification             # Notificación persistente

Firebase:
├── Authentication               # Autenticación de usuarios
│   ├── registerUser()          # Registro con email/contraseña
│   ├── signInUser()            # Inicio de sesión
│   └── signOut()               # Cerrar sesión
│
└── Firestore                    # Base de datos en la nube
    ├── users/                   # Colección de usuarios
    │   ├── {uid}/              # Documento de usuario
    │   │   ├── email
    │   │   ├── displayName
    │   │   └── favorites/      # Subcolección de favoritos
    │   │       └── {songId}/   # Documento de canción favorita
    │   │           ├── title
    │   │           ├── artist
    │   │           └── addedAt
```

### Flujo de Autenticación

```
1. Usuario abre "Mi Nube"
   ↓
2. Si no está autenticado → Mostrar formulario de login/registro
   ↓
3. Usuario ingresa email y contraseña
   ↓
4. Firebase Authentication valida credenciales
   ↓
5. Si es exitoso → Guardar datos en Firestore
   ↓
6. Cargar favoritos con listener en tiempo real
   ↓
7. Mostrar perfil de usuario y favoritos
```

### Flujo de Favoritos en Tiempo Real

```
1. Usuario marca canción como favorita
   ↓
2. Actualización local inmediata (UI)
   ↓
3. Guardar en Firestore
   ↓
4. Listener detecta cambio en Firestore
   ↓
5. Actualizar lista de favoritos automáticamente
   ↓
6. Si hay error → Revertir cambio local
```

---

## 🎯 Resumen de Implementaciones

| Semana | Tema | Implementación Principal | Interfaz Sugerida |
|--------|------|-------------------------|-------------------|
| **10** | Servicios Background | MusicPlayerService | Notificación persistente |
| **10** | Aplicativos en Segundo Plano | ForegroundService | Reproducción continua |
| **10** | Hilos | ExecutorService | Operaciones en background |
| **11** | Menú de Opciones | DropdownMenu, NavigationDrawer | Menú lateral y de opciones |
| **12** | Servicios de Google | Firebase | Pantalla "Mi Nube" |
| **12** | Firebase | Auth + Firestore | Toda la sección "Mi Nube" |
| **12** | Authentication | registerUser, signInUser | Formularios de login/registro |
| **13** | Google Maps | Coordenadas GPS | Ubicación actual |
| **13** | Interfaces | Jetpack Compose | Todas las pantallas |
| **13** | Cloud Storage | Firestore | Perfil y favoritos |
| **14** | Tiempo Real | addSnapshotListener | Lista de favoritos sincronizada |

---

**Nota:** Este reproductor implementa una arquitectura moderna con **Jetpack Compose**, **Firebase** y **servicios en segundo plano**, demostrando las mejores prácticas de desarrollo Android actual.

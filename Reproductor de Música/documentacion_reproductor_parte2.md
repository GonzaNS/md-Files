# Documentación del Proyecto: Reproductor de Música (GonZound) - Parte 2

Este documento cubre la fase avanzada del desarrollo (Semanas 9-14) del proyecto "Reproductor1", integrando hardware, servicios en segundo plano y la nube con Firebase.

---

## SEMANA 9: Hardware y Multimedia

### 1. Conceptos Clave
- **MediaPlayer**: API nativa de Android para controlar la reproducción de audio/video. Es una máquina de estados compleja (Idle, Initialized, Prepared, Started, etc.).
- **Permisos en Tiempo de Ejecución**: Desde Android 6.0, los permisos peligrosos (como ubicación o almacenamiento) deben solicitarse al usuario mientras usa la app, no solo al instalarla.

### 2. Implementación

**Archivo: `service/MusicPlayerService.kt`**
```kotlin
fun playSong(song: Song, filePath: String) {
    // Ejecutamos en un hilo secundario para no bloquear la UI
    executorService.execute {
        try {
            stopCurrentSong() // Limpieza previa
            
            currentSong = song
            // Configuración del MediaPlayer
            mediaPlayer = MediaPlayer().apply {
                setDataSource(filePath) // Define qué archivo reproducir
                prepare() // Decodifica los primeros buffers de audio (operación costosa)
                
                // Listener para saber cuándo termina la canción
                setOnCompletionListener {
                    this@MusicPlayerService.isPlaying = false
                    this@MusicPlayerService.updateNotification()
                }
                start() // Comienza el audio
            }
            isPlaying = true
            updateNotification() // Actualiza la notificación con la nueva canción
        } catch (e: Exception) {
            e.printStackTrace()
        }
    }
}
```
**Análisis del Código:**
- `executorService.execute`: La preparación del `MediaPlayer` (`prepare()`) puede tardar milisegundos perceptibles. Si lo hiciéramos en el hilo principal, la app se congelaría momentáneamente. Por eso usamos un `Executor`.
- `setDataSource`: Vincula el archivo físico (MP3) con el reproductor.
- `setOnCompletionListener`: Es crucial para la lógica de reproducción continua. Nos avisa cuando la canción terminó naturalmente para actualizar la UI (poner el botón de Play en pausa) o saltar a la siguiente.

**Archivo: `MainActivity.kt` (Solicitud de Permisos)**
```kotlin
// Registro del contrato para solicitar permisos
private val requestPermissionLauncher = registerForActivityResult(
    ActivityResultContracts.RequestPermission()
) { isGranted ->
    if (isGranted) {
        // Si el usuario acepta, procedemos a escanear
        viewModel.requestStoragePermission()
        viewModel.scanDeviceMusic()
    }
}

// Uso en la UI
Button(onClick = { requestPermissionLauncher.launch(Manifest.permission.READ_MEDIA_AUDIO) }) {
    Text("Conceder Permisos")
}
```
**Análisis del Código:**
Usamos la API moderna `registerForActivityResult`. Esto elimina la necesidad de sobrescribir `onRequestPermissionsResult` en la Activity. El callback `{ isGranted -> ... }` maneja la respuesta del usuario directamente donde se define, haciendo el código más limpio y local.

### 3. Resultado Visual
Si la app no tiene permisos, muestra una pantalla explicativa ("Empty State") con un botón. Al pulsarlo, aparece el diálogo nativo del sistema pidiendo permiso. Si se concede, la lista de música aparece mágicamente.

[IMAGEN SUGERIDA: Diálogo del sistema Android solicitando permiso de acceso a audio]

---

## SEMANA 10: Servicios y Hilos (Threads)

### 1. Conceptos Clave
- **Foreground Service**: Un servicio que tiene una notificación visible y que el sistema trata con alta prioridad. Es obligatorio para apps de música para evitar que el sistema mate la reproducción al apagar la pantalla.
- **ExecutorService**: Una abstracción de Java para manejar un pool de hilos. En este caso, usamos `newSingleThreadExecutor` para asegurar que las órdenes de reproducción se procesen en orden (serializadas) pero fuera del hilo principal.

### 2. Implementación

**Archivo: `service/MusicPlayerService.kt`**
```kotlin
class MusicPlayerService : Service() {
    // Crea un único hilo dedicado para tareas de audio
    private val executorService = Executors.newSingleThreadExecutor()

    override fun onStartCommand(intent: Intent?, flags: Int, startId: Int): Int {
        // START_STICKY: Si el sistema mata el servicio por falta de memoria,
        // lo recreará automáticamente cuando sea posible.
        return START_STICKY 
    }

    fun pause() {
        executorService.execute {
            mediaPlayer?.pause()
            isPlaying = false
            updateNotification() // Refresca la notificación para mostrar el icono de "Play"
        }
    }
    
    override fun onDestroy() {
        super.onDestroy()
        // Importante: Liberar recursos para evitar fugas de memoria
        mediaPlayer?.release()
        executorService.shutdown()
    }
}
```
**Análisis del Código:**
- `START_STICKY`: Es la configuración ideal para reproductores. Queremos que la música sea resiliente.
- `executorService`: Todas las acciones públicas (`play`, `pause`, `stop`) envían tareas a este ejecutor. Esto garantiza "Thread Safety": no habrá dos hilos intentando modificar el `MediaPlayer` al mismo tiempo, lo cual causaría errores.
- `onDestroy`: La limpieza es vital. Un `MediaPlayer` no liberado puede mantener bloqueado el hardware de audio del dispositivo.

### 3. Resultado Visual
El usuario puede salir de la app, bloquear el teléfono o abrir otra aplicación, y la música sigue sonando sin interrupciones. La notificación en la barra de estado es la prueba visual de que el servicio está corriendo.

[IMAGEN SUGERIDA: Pantalla de bloqueo del teléfono mostrando los controles multimedia de la app]

---

## SEMANA 11: Menús y Navegación

### 1. Conceptos Clave
- **TopAppBar**: La barra superior estándar. En Compose, es un componente flexible que puede contener títulos, navegación y acciones.
- **DropdownMenu**: Menú flotante que aparece sobre la interfaz, anclado a un elemento (generalmente un botón de "tres puntos").

### 2. Implementación

**Archivo: `MainActivity.kt` (CustomTopBar)**
```kotlin
@Composable
fun CustomTopBar(onMenuClick: () -> Unit) {
    Surface(color = Color(0xFF1976D2)) {
        Row(
            modifier = Modifier.fillMaxWidth().padding(16.dp),
            verticalAlignment = Alignment.CenterVertically
        ) {
            // Botón de menú "Hamburguesa"
            IconButton(onClick = onMenuClick) {
                Icon(Icons.Filled.Menu, contentDescription = "Menú", tint = Color.White)
            }
            
            Spacer(modifier = Modifier.size(16.dp))
            
            Text(
                text = "GonZound",
                color = Color.White,
                fontSize = 20.sp,
                fontWeight = FontWeight.Bold
            )
        }
    }
}
```
**Análisis del Código:**
Creamos una barra personalizada en lugar de usar la `TopAppBar` por defecto para tener control total del diseño. Usamos un `Row` para alinear el icono de menú y el título. El color azul (`0xFF1976D2`) define la identidad de la marca.

**Archivo: `MainActivity.kt` (Menú Contextual en MyMusicScreen)**
```kotlin
Box {
    IconButton(onClick = { menuExpanded = true }) {
        Icon(Icons.Filled.MoreVert, "Opciones")
    }
    
    // El menú se dibuja solo si menuExpanded es true
    DropdownMenu(
        expanded = menuExpanded,
        onDismissRequest = { menuExpanded = false }
    ) {
        DropdownMenuItem(
            text = { Text("Clasificar por Título") },
            onClick = {
                viewModel.changeSort(SortBy.TITLE)
                menuExpanded = false // Cerrar menú al seleccionar
            }
        )
        // ... más opciones
    }
}
```
**Análisis del Código:**
El `Box` es necesario para anclar el menú. `DropdownMenu` se superpone a la UI. La lógica es simple: una variable booleana `menuExpanded` controla la visibilidad. Al hacer clic en una opción, ejecutamos la acción (ej. ordenar lista) y *debemos* poner `menuExpanded = false` manualmente para cerrar el menú.

### 3. Resultado Visual
Una barra superior azul consistente en toda la app. En la lista de música, un botón discreto de tres puntos permite acceder a funciones avanzadas sin saturar la interfaz principal.

[IMAGEN SUGERIDA: Menú desplegable "Más opciones" abierto sobre la lista de canciones]

---

## SEMANA 12: Autenticación y Firebase

### 1. Conceptos Clave
- **Firebase Auth**: Backend-as-a-Service que maneja la complejidad de sesiones, encriptación de contraseñas y proveedores de identidad.
- **Estado de Autenticación**: La UI debe reaccionar en tiempo real a si el usuario está logueado o no.

### 2. Implementación

**Archivo: `MainActivity.kt` (AuthScreen)**
```kotlin
@Composable
fun AuthScreen(viewModel: MusicPlayerViewModel, ...) {
    // Estado local para alternar entre Login y Registro
    var isLoginMode by remember { mutableStateOf(true) }
    
    Column(...) {
        // Título dinámico
        Text(text = if (isLoginMode) "Iniciar Sesión" else "Registrarse")
        
        // Campos de texto
        OutlinedTextField(value = email, label = { Text("Email") }, ...)
        OutlinedTextField(value = password, label = { Text("Contraseña") }, ...)
        
        // Botón de acción principal
        Button(
            onClick = {
                if (isLoginMode) {
                    viewModel.signInUser(email, password)
                } else {
                    viewModel.registerUser(email, password, displayName)
                }
            }
        ) {
            Text(if (isLoginMode) "Entrar" else "Crear Cuenta")
        }
        
        // Toggle para cambiar de modo
        TextButton(onClick = { isLoginMode = !isLoginMode }) {
            Text(if (isLoginMode) "¿No tienes cuenta? Regístrate" else "Ya tengo cuenta")
        }
    }
}
```
**Análisis del Código:**
En lugar de crear dos pantallas separadas (`LoginActivity` y `RegisterActivity`), usamos una sola pantalla `AuthScreen` que muta su contenido basándose en `isLoginMode`. Esto reduce la duplicación de código y hace la transición más fluida para el usuario.
El `ViewModel` maneja la llamada asíncrona a Firebase. Si hay error (ej. "password incorrecto"), actualiza un estado `authError` que la UI muestra en una tarjeta roja (ver código completo).

### 3. Resultado Visual
Una pantalla de acceso limpia. Si el usuario se equivoca, aparece un mensaje de error claro. Al autenticarse correctamente, la pantalla desaparece automáticamente y muestra el perfil, gracias a la observación del estado `currentUser`.

[IMAGEN SUGERIDA: Pantalla de Login y Pantalla de Registro lado a lado]

---

## SEMANA 13: Google Maps y Ubicaciones

### 1. Conceptos Clave
- **FusedLocationProvider**: La API recomendada por Google para obtener ubicación. Combina GPS, Wi-Fi y redes móviles para ahorrar batería y mejorar precisión.
- **Lógica de Proximidad**: Calcular la distancia entre la ubicación actual y una guardada para activar acciones automáticas.

### 2. Implementación

**Archivo: `MainActivity.kt` (LocationsScreen)**
```kotlin
// Mostrar ubicación actual
currentLocation?.let { loc ->
    Card(...) {
        Text("Lat: ${loc.latitude}")
        Text("Lon: ${loc.longitude}")
    }
}

// Detección de cercanía (Lógica simplificada en UI)
suggestedPlaylist?.let { playlist ->
    Card(colors = CardDefaults.cardColors(containerColor = Color(0xFFE3F2FD))) {
        Text("📍 Estás en: ${detectedLocation?.name}")
        Button(onClick = { viewModel.playSuggestedPlaylist() }) {
            Text("Reproducir ${playlist.name}")
        }
    }
}
```
**Análisis del Código:**
La UI es reactiva a la ubicación. No necesitamos un botón "Refrescar". Si el `ViewModel` actualiza `currentLocation` (porque el usuario se movió), la UI se redibuja y muestra las nuevas coordenadas.
Si el sistema detecta que estamos cerca de una ubicación guardada (ej. "Gimnasio"), `suggestedPlaylist` deja de ser nulo, y aparece mágicamente una tarjeta azul sugiriendo música. Esto es un excelente ejemplo de **Context-Aware Computing**.

### 3. Resultado Visual
Una pantalla que lista lugares favoritos. Al llegar a uno de ellos, aparece una notificación visual destacada invitando a poner música adecuada para ese lugar.

[IMAGEN SUGERIDA: Pantalla de Ubicaciones mostrando la tarjeta de "Ubicación detectada" activa]

---

## SEMANA 14: Base de Datos en Tiempo Real (Firestore)

### 1. Conceptos Clave
- **Firestore**: Base de datos NoSQL orientada a documentos. Es rápida y permite sincronización en tiempo real.
- **Colecciones y Documentos**: Estructura de datos flexible (similar a JSON). Guardamos usuarios en una colección y sus favoritos como sub-colección o campo array.

### 2. Implementación

**Archivo: `MainActivity.kt` (UserProfileScreen)**
```kotlin
@Composable
fun UserProfileScreen(viewModel: MusicPlayerViewModel, ...) {
    // 'collectAsState' suscribe la UI al flujo de datos de favoritos
    val favoriteSongs by viewModel.favoriteSongs.collectAsState()
    
    LazyColumn {
        items(favoriteSongs) { song ->
            Card(...) {
                Row(...) {
                    Icon(Icons.Filled.Favorite, tint = Color(0xFFE91E63)) // Corazón rosa
                    Text(song.title)
                }
            }
        }
    }
    
    // Botón de actualización manual (aunque Firestore puede ser realtime)
    IconButton(onClick = { viewModel.loadFavoriteSongs() }) {
        Icon(Icons.Filled.Refresh, "Actualizar")
    }
}
```
**Análisis del Código:**
Esta pantalla muestra los datos persistentes en la nube.
- `favoriteSongs` no es una lista estática local; viene de Firestore. Si el usuario marca una canción como favorita en otro dispositivo, esta lista podría actualizarse automáticamente (dependiendo de la implementación del ViewModel).
- La UI maneja el estado vacío: si `favoriteSongs` está vacío, muestra un icono y mensaje amigable ("No tienes favoritos aún"), en lugar de una pantalla en blanco.

### 3. Resultado Visual
Pantalla "Mi Nube" que actúa como el perfil del usuario. Muestra su nombre (traído de Auth) y su lista de éxitos personales (traída de Firestore), accesible desde cualquier dispositivo donde inicie sesión.

[IMAGEN SUGERIDA: Pantalla de Perfil de Usuario con la lista de favoritos cargada desde la nube]

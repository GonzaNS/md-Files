# 📊 Exposición Sprint 6 - Zity

## 🎯 Objetivo
Implementar **notificaciones en tiempo real** (Realtime + Email), **auditoría centralizada**, **foto de cierre técnico**, **cambio de contraseña seguro** y **filtros de notificaciones**.

---

## ✅ PBIs Implementados (7 tareas · 13 horas)

### 1️⃣ **PBI-12: Notificaciones Supabase Realtime + Email** (Gonza Morales · 4h)
**¿Qué se agregó?**
- Canal Realtime único por usuario para actualizaciones en vivo
- Suscripción automática a tabla `notificaciones` con filtro por `usuario_id`
- Actualización optimista (UI responde al instante)
- Reconexión exponencial ante desconexiones

**¿Cómo funciona?**
- Listener en `postgres_changes` escucha INSERTs y UPDATEs
- Al llegar notificación: suma contador `noLeidasCount` sin esperar servidor
- Al marcar como leída: actualización inmediata en lista

**📁 Ubicación en código:**
```
src/lib/notificaciones.ts (líneas 38-74)
  - useNotificaciones() hook
  - Suscripción channel: supabase.channel(`notificaciones:${usuarioId}`)
  - Payload listener con INSERT/UPDATE handling
```

**Edge Function (email simulado):**
```
supabase/functions/notificar-cambio-estado/index.ts (líneas 88-101)
  - Resend API integration
  - HTML template minimalista
  - Modo dry-run si no hay RESEND_API_KEY (desarrollo local)
```

**💻 Código en Detalle - PBI-12:**

```typescript
// src/lib/notificaciones.ts (líneas 33-74)
// ┌─ Hook que escucha cambios en tiempo real de la tabla notificaciones
useEffect(() => {
  void Promise.resolve().then(() => fetchNotificaciones())  // ← Carga inicial sin esperar

  if (!usuarioId) return  // ← Si no hay usuario, no suscribirse

  // Suscripción Realtime (PBI-12)
  // ┌─ Crea canal único por usuario: "notificaciones:uuid-del-usuario"
  const channel = supabase
    .channel(`notificaciones:${usuarioId}`)
    // ┌─ Escucha TODOS los eventos (INSERT, UPDATE, DELETE)
    // en la tabla 'notificaciones' de esquema 'public'
    // SOLO para filas donde usuario_id = usuarioId actual
    .on(
      'postgres_changes',
      {
        event: '*',  // ← Todos los eventos
        schema: 'public',  // ← Esquema BD
        table: 'notificaciones',  // ← Tabla monitoreada
        filter: `usuario_id=eq.${usuarioId}`,  // ← Solo las del usuario actual
      },
      (payload) => {  // ← Callback cuando hay cambio
        // Si es nuevo INSERT
        if (payload.eventType === 'INSERT') {
          const nueva = payload.new as Notificacion
          // Agregar al inicio de lista y limitar a 50 items (optimistic update)
          setNotificaciones(prev => [nueva, ...prev].slice(0, 50))
          // Incrementar contador de no leídas
          setNoLeidasCount(prev => prev + 1)
        } 
        // Si es UPDATE (ej: marcada como leída)
        else if (payload.eventType === 'UPDATE') {
          const actualizada = payload.new as Notificacion
          // Reemplazar la notificación actualizada en la lista
          setNotificaciones(prev =>
            prev.map(n => (n.id === actualizada.id ? actualizada : n))
          )
          // Recalcular contador: si cambió de no-leída a leída, restar
          setNoLeidasCount(prev => {
            const old = payload.old as Partial<Notificacion>
            // Si era no-leída (false) y ahora es leída (true)
            if (old.leida === false && actualizada.leida === true) 
              return Math.max(0, prev - 1)  // ← Restar 1 al contador
            // Si era leída (true) y ahora es no-leída (false)
            if (old.leida === true && actualizada.leida === false) 
              return prev + 1  // ← Sumar 1 al contador
            return prev  // ← Sin cambio
          })
        }
      }
    )
    .subscribe()  // ← Activar suscripción

  // Cleanup: remover canal al desmontar componente
  return () => {
    supabase.removeChannel(channel)
  }
}, [usuarioId, fetchNotificaciones])
```

**📧 Edge Function - Email (líneas 88-101 en index.ts):**

```typescript
// supabase/functions/notificar-cambio-estado/index.ts (líneas 88-101)

const resendApiKey = Deno.env.get("RESEND_API_KEY")  // ← Obtener API key del secreto

if (!resendApiKey) {  // ← Si NO hay API key (desarrollo local)
  console.log("----- DRY-RUN EMAIL -----")  // ← Log de debug
  console.log(`To: ${residente.email}`)
  console.log(`Subject: [Zity] Actualización de la solicitud ${solicitud.codigo}`)
  console.log("Body HTML:")
  console.log(emailHtml)
  console.log("-------------------------")
  return jsonResponse(req, { success: true, mode: "dry-run" })  // ← Simular éxito
}

// Llamada a Resend API (servicio email real)
const res = await fetch("https://api.resend.com/emails", {
  method: "POST",
  headers: {
    "Authorization": `Bearer ${resendApiKey}`,  // ← Autorización Bearer
    "Content-Type": "application/json",
  },
  body: JSON.stringify({
    from: "Zity Condominios <no-reply@zity.site>",  // ← Remitente
    to: [residente.email],  // ← Destinatario
    subject: `[Zity] Actualización de la solicitud ${solicitud.codigo}`,  // ← Asunto
    html: emailHtml,  // ← Cuerpo en HTML (generado arriba)
  }),
})

if (!res.ok) {  // ← Si respuesta no es 200-299
  const errorText = await res.text()
  throw new Error(`Fallo al enviar correo vía Resend: ${errorText}`)
}

const resData = await res.json()  // ← Parsear respuesta JSON
return jsonResponse(req, { success: true, messageId: resData.id })  // ← Retornar éxito
```

---

### 2️⃣ **HU-NOTIF-01: Centro de Notificaciones con Badge** (Santiago Flores · 1.5h)
**¿Qué se agregó?**
- Página `/notificaciones` con lista paginada (50 máximo)
- Ícono animado en navbar con badge de contador
- Dropdown en navbar (últimas 10 notificaciones)
- Marca visual (fondo celeste) para no leídas

**¿Cómo funciona?**
- Badge pulsa cuando hay no leídas
- Dropdown cierra al hacer click afuera
- Página completa permite ver todas + filtros

**📁 Ubicación en código:**
```
src/pages/Notificaciones.tsx
  - Página completa con ícono por tipo (líneas 9-42)
  - Filtros estado + rango de fechas

src/components/shared/CampanaNotificaciones.tsx
  - Componente dropdown con últimas 10 (líneas 65-157)
  - Badge animado (línea 76)
  - Link a página completa (líneas 145-151)
```

**💻 Código en Detalle - HU-NOTIF-01:**

```typescript
// src/components/shared/CampanaNotificaciones.tsx (líneas 65-80)
// ┌─ Componente dropdown con campana y badge

return (
  <div className="relative" ref={menuRef}>  {/* ← Ref para detectar click afuera */}
    <button
      type="button"
      onClick={() => setIsOpen(!isOpen)}  {/* ← Alternar estado del dropdown */}
      className="relative p-2 text-warm-500 hover:text-primary-700 hover:bg-primary-50 rounded-full transition-colors cursor-pointer"
      aria-label="Notificaciones"
    >
      <svg className="w-6 h-6" fill="none" viewBox="0 0 24 24" stroke="currentColor" strokeWidth={1.8}>
        <path strokeLinecap="round" strokeLinejoin="round" d="M15 17h5l-1.405-1.405A2.032 2.032 0 0118 14.158V11a6.002 6.002 0 00-4-5.659V5a2 2 0 10-4 0v.341C7.67 6.165 6 8.388 6 11v3.159c0 .538-.214 1.055-.595 1.436L4 17h5m6 0v1a3 3 0 11-6 0v-1m6 0H9" />
      </svg>
      
      {/* Badge rojo con contador de no leídas */}
      {noLeidasCount > 0 && (
        <span className="absolute top-1.5 right-1.5 flex h-4 w-4 items-center justify-center rounded-full bg-error text-[9px] font-bold text-white shadow-sm ring-2 ring-white animate-pulse-once">
          {noLeidasCount > 99 ? '99+' : noLeidasCount}  {/* ← Mostrar '99+' si > 99 */}
        </span>
      )}
    </button>

    {isOpen && (  {/* ← Mostrar dropdown solo si está abierto */}
      <div className="absolute right-0 mt-2 w-80 sm:w-96 bg-white border border-warm-200 rounded-xl shadow-xl z-50 overflow-hidden animate-fade-in-down">
        {/* Header con título y botón marcar todas */}
        <div className="px-4 py-3 bg-warm-50 border-b border-warm-200 flex items-center justify-between">
          <h3 className="font-semibold text-primary-900">Notificaciones</h3>
          {noLeidasCount > 0 && (
            <button
              onClick={() => marcarTodasComoLeidas()}  {/* ← Marcar TODAS como leídas */}
              className="text-[0.6875rem] font-medium text-primary-600 hover:text-primary-800 underline cursor-pointer"
            >
              Marcar todas leídas
            </button>
          )}
        </div>

        {/* Área scrollable con últimas 10 notificaciones */}
        <div className="max-h-[28rem] overflow-y-auto">
          {topNotificaciones.length === 0 ? (
            // Mensaje vacío
            <div className="px-4 py-8 text-center">
              <p className="text-sm text-warm-500 font-medium">Estás al día</p>
            </div>
          ) : (
            // Lista de notificaciones
            <div className="divide-y divide-warm-100">
              {topNotificaciones.map(notif => (  {/* ← Mapear últimas 10 */}
                <div
                  key={notif.id}
                  className={`p-4 flex gap-3 hover:bg-warm-50 transition-colors ${!notif.leida ? 'bg-primary-50/40' : ''}`}
                  onClick={() => {
                    if (!notif.leida) marcarComoLeida(notif.id)  {/* ← Al hacer click: marcar como leída */}
                  }}
                >
                  {/* Ícono según tipo de notificación */}
                  <div className="shrink-0 mt-0.5">
                    <div className={`p-2 rounded-full ${!notif.leida ? 'bg-white shadow-sm ring-1 ring-warm-200' : 'bg-warm-100'}`}>
                      {getIconForTipo(notif.tipo)}
                    </div>
                  </div>
                  
                  {/* Contenido de notificación */}
                  <div className="min-w-0 flex-1 cursor-default">
                    <p className={`text-sm ${!notif.leida ? 'font-semibold text-primary-900' : 'font-medium text-warm-700'}`}>
                      {notif.titulo}
                    </p>
                    <p className="text-xs text-warm-500 mt-0.5 line-clamp-2 leading-relaxed">
                      {notif.mensaje}
                    </p>
                    <p className="text-[0.6875rem] text-warm-400 mt-1.5 font-medium">
                      {tiempoTranscurrido(notif.created_at)}  {/* ← Ej: "hace 5 minutos" */}
                    </p>
                  </div>
                  
                  {/* Punto indicador si no leída */}
                  {!notif.leida && (
                    <div className="shrink-0 flex items-center">
                      <div className="w-2 h-2 bg-primary-500 rounded-full" />  {/* ← Punto azul */}
                    </div>
                  )}
                </div>
              ))}
            </div>
          )}
        </div>

        {/* Footer con link a página completa */}
        <div className="px-4 py-3 bg-warm-50 border-t border-warm-200 text-center">
          <Link
            to="/notificaciones"
            onClick={() => setIsOpen(false)}  {/* ← Cerrar dropdown al hacer click */}
            className="text-xs font-semibold text-primary-600 hover:text-primary-800 uppercase tracking-wider"
          >
            Ver centro de notificaciones
          </Link>
        </div>
      </div>
    )}
  </div>
)
```

---

### 3️⃣ **HU-NOTIF-02: Marcar Notificación como Leída** (Santiago Flores · 1.5h)
**¿Qué se agregó?**
- Función `marcarComoLeida()` con UPDATE optimista
- Botón "Marcar como leída" en cada notificación
- Rollback automático si falla la sincronización

**¿Cómo funciona?**
- UI actualiza al instante (optimistic update)
- En background: `UPDATE notificaciones SET leida=true WHERE id=?`
- Si falla: re-fetch para recuperar estado real

**📁 Ubicación en código:**
```
src/lib/notificaciones.ts (líneas 76-90)
  - marcarComoLeida(): optimistic + rollback
  - update({ leida: true }).eq('id', id)

src/pages/Notificaciones.tsx (líneas 176-185)
  - Botón solo visible si no leída
  
src/components/shared/CampanaNotificaciones.tsx (líneas 113-115)
  - Al hacer click marca como leída
```

**💻 Código en Detalle - HU-NOTIF-02:**

```typescript
// src/lib/notificaciones.ts (líneas 76-90)
// ┌─ Función para marcar UNA notificación como leída con optimistic update

const marcarComoLeida = async (id: string) => {
  // ┌─ PASO 1: Optimistic update (actualizar UI inmediatamente)
  // No esperar respuesta del servidor, el usuario ve cambio al instante
  setNotificaciones(prev => 
    prev.map(n => n.id === id ? { ...n, leida: true } : n)  // ← Cambiar leida=true
  )
  // Decrementar contador de no leídas
  setNoLeidasCount(prev => Math.max(0, prev - 1))

  // ┌─ PASO 2: En background, enviar UPDATE a Supabase
  const { error } = await supabase
    .from('notificaciones')
    .update({ leida: true })  // ← Marcar como leída en BD
    .eq('id', id)  // ← Solo la notificación con este ID

  // ┌─ PASO 3: Si falla, hacer rollback (re-fetch estado real)
  if (error) {
    // La UPDATE falló, por lo que re-traemos los datos reales de la BD
    // para sincronizar UI con estado real
    fetchNotificaciones()
  }
}

// ┌─ Similar pero para TODAS las notificaciones no leídas
const marcarTodasComoLeidas = async () => {
  if (!usuarioId) return

  // Optimistic update: marcar todas como leídas en UI
  setNotificaciones(prev => prev.map(n => ({ ...n, leida: true })))
  setNoLeidasCount(0)  // ← Poner contador a 0

  // En background, UPDATE en la BD
  const { error } = await supabase
    .from('notificaciones')
    .update({ leida: true })
    .eq('usuario_id', usuarioId)  // ← Usuario actual
    .eq('leida', false)  // ← Solo las que NO están leídas

  // Si falla, rollback
  if (error) {
    fetchNotificaciones()
  }
}
```

**📄 Uso en Página (src/pages/Notificaciones.tsx - líneas 176-185):**

```typescript
// ┌─ Botón "Marcar como leída" solo aparece si notificación NO está leída

{!notif.leida && (  {/* ← Condición: mostrar solo si no leída */}
  <div className="mt-4 flex">
    <button
      onClick={() => marcarComoLeida(notif.id)}  {/* ← Llamar función */}
      className="text-xs font-medium text-primary-600 hover:text-primary-800 bg-white border border-primary-200 shadow-sm px-3 py-1.5 rounded-md transition-colors cursor-pointer"
    >
      Marcar como leída
    </button>
  </div>
)}
```

**💡 Flujo Completo:**

```
Usuario hace click en "Marcar como leída"
    ↓
marcarComoLeida(notificationId)
    ├─ UI actualiza INMEDIATAMENTE (optimistic)
    │  ├─ Cambiar notificación.leida = true
    │  └─ Restar 1 al contador
    └─ En background:
       └─ await supabase.from('notificaciones').update({ leida: true }).eq('id', id)
          ├─ Si OK ✅ → UI ya está actualizada (no hace nada)
          └─ Si Error ❌ → Llamar fetchNotificaciones() para recuperar estado real
```

---

### 4️⃣ **PBI-S3-E01: Foto de Cierre Técnico** (Cortez Zamora · 1.5h)
**¿Qué se agregó?**
- Subida opcional de foto al cerrar solicitud (máx. 5 MB)
- Serialización de ruta en nota como JSON: `[cierre] {"archivo": "path/to/file"}`
- Ruta en historial para reproducir flujo antes/después

**¿Cómo funciona?**
1. Técnico sube foto + nota al marcar como "resuelta"
2. Foto se guarda en bucket `solicitudes-fotos/{usuario_id}/{solicitud_id}/cierre_{timestamp}`
3. Ruta se serializa en `nota` como JSON
4. Residente ve foto original + foto de cierre lado a lado

**📁 Ubicación en código:**
```
src/components/tecnico/solicitudes/SeccionActualizarEstado.tsx
  - UploadFoto component (línea 18)
  - subirFotoCierre() + serializarNotaCierre() (líneas 74-77)
  - Lógica de subida (líneas 74-77)

src/hooks/useSolicitudesTecnico.ts
  - subirFotoCierre(): subida a Storage
  - serializarNotaCierre(): JSON encoding
```

**💻 Código en Detalle - PBI-S3-E01:**

```typescript
// src/components/tecnico/solicitudes/SeccionActualizarEstado.tsx (líneas 64-100)
// ┌─ Componente que maneja actualización de estado + foto de cierre

async function handleGuardar() {
  if (!estadoDestino || !user || botonDeshabilitado) return

  setGuardando(true)  // ← Mostrar spinner
  setError(null)
  setExito(false)

  try {
    let finalNota = nota  // ← Comenzar con la nota ingresada

    // ┌─ PASO 1: Si se pasa a "resuelta" Y hay foto seleccionada
    if (estadoDestino === 'resuelta' && fotoFile) {
      // Subir foto a Storage y obtener su ruta
      // En lugar de residenteId, usamos user.id para cumplir con RLS
      // (técnico debe estar como propietario de la carpeta en Storage)
      const filePath = await subirFotoCierre(fotoFile, user.id, solicitudId)
      
      // ┌─ PASO 2: Serializar la ruta de foto como JSON en la nota
      // Formato: "[cierre] {\"archivo\": \"path/to/file\"}"
      // Esto permite que residente vea foto original + foto cierre
      finalNota = serializarNotaCierre(nota, filePath)
    }

    // ┌─ PASO 3: Actualizar estado de la solicitud en BD
    // Se envía: estado nuevo, nota final (con foto si existe), técnico ID
    const resultado = await actualizarEstadoTecnico({
      solicitudId,
      estadoAnterior: estadoActual,  // ← Ej: "en_progreso"
      estadoNuevo: estadoDestino,    // ← Ej: "resuelta"
      nota: finalNota,               // ← Nota + foto serializada
      tecnicoId: user.id,
    })

    // ┌─ PASO 4: Verificar si actualización fue exitosa
    if (!resultado.ok) {
      setError(resultado.error ?? 'Error al actualizar el estado.')
      return
    }

    // ┌─ PASO 5: Si todo OK, limpiar formulario y notificar éxito
    setExito(true)
    setNota('')  // ← Limpiar textarea
    setFotoFile(null)  // ← Limpiar foto seleccionada
    onEstadoActualizado()  // ← Callback para refrescar lista de solicitudes
  } catch (err) {
    setError((err as Error).message || 'Error al subir la foto o actualizar el estado.')
  } finally {
    setGuardando(false)  // ← Ocultar spinner
  }
}
```

**📸 Funciones de Upload y Serialización (en useSolicitudesTecnico.ts):**

```typescript
// ┌─ Función para subir foto de cierre al bucket de Storage
export async function subirFotoCierre(
  file: File, 
  usuarioId: string, 
  solicitudId: string
): Promise<string> {
  // ┌─ Validación: máximo 5 MB
  const MAX_MB = 5
  if (file.size > MAX_MB * 1024 * 1024) {
    throw new Error(`Foto debe ser menor a ${MAX_MB} MB`)
  }

  // ┌─ Generar ruta en Storage: solicitudes-fotos/{usuario_id}/{solicitud_id}/cierre_{timestamp}
  // Ejemplo: solicitudes-fotos/uuid-usuario/uuid-solicitud/cierre_1715123456789.jpg
  const timestamp = Date.now()
  const filePath = `${usuarioId}/${solicitudId}/cierre_${timestamp}_${file.name}`

  // ┌─ Subir archivo a bucket "solicitudes-fotos"
  const { error } = await supabase.storage
    .from('solicitudes-fotos')  // ← Nombre del bucket
    .upload(filePath, file)     // ← Ruta + contenido

  if (error) {
    throw new Error(`Fallo al subir foto: ${error.message}`)
  }

  // ┌─ Retornar ruta para guardarla en BD
  return filePath
}

// ┌─ Función para serializar nota + ruta de foto como JSON
export function serializarNotaCierre(nota: string, filePath: string): string {
  // ┌─ Formato: "[cierre] {\"archivo\": \"path\"}"
  // Permite que frontend pueda parsear y mostrar foto original + cierre
  return `[cierre] ${JSON.stringify({ archivo: filePath })}\n${nota}`
}
```

**🖼️ Cómo el Residente Ve la Foto (flujo posterior):**

```
Tabla historial_estados:
├─ nota: "[cierre] {\"archivo\": \"uuid-usuario/uuid-solicitud/cierre_1715123456789.jpg\"}\nNota del técnico aquí"
│
└─ Al renderizar, el frontend:
   ├─ Parsear JSON de la nota
   ├─ Obtener ruta del archivo
   ├─ Mostrar foto ORIGINAL (almacenada en solicitud.imagen_url)
   └─ Mostrar foto CIERRE (almacenada en Storage con ruta del JSON)
```

---

### 5️⃣ **PBI-S4-E01: Notificación Admin Rechazo Residente** (Cortez Zamora · 1h)
**¿Qué se agregó?**
- Trigger `after_solicitud_estado_changed` en BD
- Condición: si estado = `en_progreso` Y origen = `rechazo_residente`
- Insert automático en `notificaciones` dirigida a TODOS los admins

**¿Cómo funciona?**
```sql
-- Trigger en BD (PBI-S4-E01)
IF NEW.estado_nuevo = 'en_progreso' AND NEW.origen = 'rechazo_residente'
  INSERT INTO notificaciones (usuario_id, tipo, titulo, mensaje)
    SELECT admin.id, 'alerta_rechazo', 
           'Rechazo: ' || solicitud.codigo,
           'Residente rechazó solicitud'
    FROM usuarios admin WHERE admin.rol = 'admin'
```

**📁 Ubicación en código:**
- Trigger en BD (migration/Supabase)
- No hay código frontend explícito (sucede en backend automáticamente)

---

### 6️⃣ **PBI-S5-E02: Botón "Ver Auditoría" en Drawer Admin** (Santiago Flores · 1h)
**¿Qué se agregó?**
- Botón en drawer de solicitud del admin que abre modal de auditoría
- Query parámetro: `entity_id=solicitud_id AND entidad='solicitudes'`
- Modal muestra todas las acciones sobre esa solicitud

**¿Cómo funciona?**
- Admin abre drawer de solicitud → botón "Ver auditoría"
- Modal lista: quién, qué acción, cuándo, detalles JSON
- Tabla filtrada por `entity_id` + `entidad` + índices

**📁 Ubicación en código:**
```
src/components/admin/solicitudes/ModalAsignarTecnico.tsx (o drawer)
  - Botón "Ver auditoría" que abre modal
  
src/components/admin/auditoria/ModalDetalleAudit.tsx
  - Modal con query filtrada por entity_id
```

---

### 7️⃣ **PBI-S5-E03: Cambio de Contraseña con Reauth** (Cortez Zamora · 1h)
**¿Qué se agregó?**
- Tab "Seguridad" en página Perfil (`/perfil`)
- Re-autenticación silenciosa: `signInWithPassword(currentPassword)`
- Rate limiting: 3 intentos fallidos = bloqueo 5 minutos
- `updateUser()` vía Supabase Auth (no Supabase BD)

**¿Cómo funciona?**
1. Usuario ingresa contraseña actual → se verifica con `signInWithPassword()`
2. Si falla 3 veces: UI bloquea botón por 5 minutos
3. Si OK: `updateUser({ password: newPassword })`
4. Sesión se mantiene (no cierra automático)

**📁 Ubicación en código:**
```
src/pages/Perfil.tsx
  - Estado para intentos: intentosFallidos, bloqueadoHasta (líneas 47-49)
  - useEffect timer para desbloqueo (líneas 51-65)
  - handleGuardarSeguridad() (líneas 130-171)
  - Re-auth: supabase.auth.signInWithPassword() (línea 148)
  - Update: supabase.auth.updateUser({ password }) (línea 155)
  - Rate limit: 3 intentos = bloqueo (líneas 166-170)
```

**💻 Código en Detalle - PBI-S5-E03:**

```typescript
// src/pages/Perfil.tsx (líneas 38-65)
// ┌─ Estado para contraseña y rate limiting

const [currentPassword, setCurrentPassword] = useState('')      // ← Contraseña actual
const [newPassword, setNewPassword] = useState('')             // ← Nueva contraseña
const [confirmPassword, setConfirmPassword] = useState('')     // ← Confirmación

// Rate Limiting
const [intentosFallidos, setIntentosFallidos] = useState(0)    // ← Contador de intentos
const [bloqueadoHasta, setBloqueadoHasta] = useState<number | null>(null)  // ← Timestamp de desbloqueo
const isBloqueado = bloqueadoHasta !== null && Date.now() < bloqueadoHasta  // ← Verificar si está bloqueado

// ┌─ useEffect: si está bloqueado, esperar hasta que se desbloquee
useEffect(() => {
  if (bloqueadoHasta) {
    const remaining = bloqueadoHasta - Date.now()  // ← Milisegundos restantes
    
    if (remaining > 0) {
      // Aún está bloqueado, poner timer para cuando se desbloquee
      const timer = setTimeout(() => {
        setBloqueadoHasta(null)  // ← Permitir nuevos intentos
        setIntentosFallidos(0)   // ← Resetear contador
      }, remaining)
      
      return () => clearTimeout(timer)  // ← Cleanup si componente desmonta
    } else {
      // Ya pasó el tiempo, desbloquear
      setBloqueadoHasta(null)
      setIntentosFallidos(0)
    }
  }
}, [bloqueadoHasta])
```

**🔐 Función Cambio de Contraseña (líneas 130-171):**

```typescript
// src/pages/Perfil.tsx - handleGuardarSeguridad()

async function handleGuardarSeguridad(e: React.FormEvent) {
  e.preventDefault()
  
  // ┌─ Si está bloqueado, no permitir (botón debe estar disabled)
  if (isBloqueado) return

  // ┌─ Validaciones básicas en cliente
  if (newPassword.length < 8) {
    setErrorSeguridad('La nueva contraseña debe tener al menos 8 caracteres.')
    return
  }
  if (newPassword !== confirmPassword) {
    setErrorSeguridad('Las contraseñas nuevas no coinciden.')
    return
  }

  setGuardandoSeguridad(true)
  setErrorSeguridad(null)

  try {
    // ┌─ PASO 1: Re-autenticar silenciosamente
    // Verificar que la contraseña actual es correcta
    const { error: authError } = await supabase.auth.signInWithPassword({
      email: profile!.email,       // ← Email del usuario
      password: currentPassword,   // ← Contraseña actual que ingresó
    })

    // ┌─ Si fallo en re-auth
    if (authError) {
      // Incrementar contador de intentos fallidos
      const nuevoIntento = intentosFallidos + 1
      setIntentosFallidos(nuevoIntento)

      // Si llega a 3 intentos fallidos, bloquear por 5 minutos
      if (nuevoIntento >= 3) {
        const bloqueadoHasta = Date.now() + 5 * 60 * 1000  // ← 5 minutos = 300000ms
        setBloqueadoHasta(bloqueadoHasta)
        setErrorSeguridad('Demasiados intentos fallidos. Bloqueado por 5 minutos.')
      } else {
        // Mostrar intentos restantes
        const restantes = 3 - nuevoIntento
        setErrorSeguridad(`Contraseña actual incorrecta. ${restantes} intento(s) restante(s).`)
      }
      
      setGuardandoSeguridad(false)
      return
    }

    // ┌─ PASO 2: Si re-auth OK, actualizar contraseña en Supabase Auth
    const { error: updateError } = await supabase.auth.updateUser({
      password: newPassword,  // ← Nueva contraseña
    })

    if (updateError) {
      setErrorSeguridad(updateError.message)
      setGuardandoSeguridad(false)
      return
    }

    // ┌─ PASO 3: Éxito - limpiar campos y resetear intentos
    setCurrentPassword('')
    setNewPassword('')
    setConfirmPassword('')
    setIntentosFallidos(0)  // ← Resetear contador en caso de cambios posteriores
    setToast('Contraseña cambiada correctamente.')  // ← Mostrar toast de éxito
    setTimeout(() => setToast(null), 3000)  // ← Auto-desaparecer después de 3s

  } catch (err) {
    setErrorSeguridad((err as Error).message || 'Error inesperado.')
  } finally {
    setGuardandoSeguridad(false)  // ← Ocultar spinner
  }
}
```

**🎨 Render del Tab Seguridad (estructura HTML aproximada):**

```typescript
// ┌─ Botón de guardar seguridad: deshabilitado si está bloqueado o validaciones fallan

<button
  type="submit"
  disabled={isBloqueado || guardandoSeguridad}  {/* ← Desabilitar si bloqueado */}
  className={`px-4 py-2 rounded-lg font-medium transition-colors ${
    isBloqueado || guardandoSeguridad
      ? 'bg-warm-200 text-warm-500 cursor-not-allowed'  {/* ← Gris si deshabilitado */}
      : 'bg-primary-600 text-white hover:bg-primary-700'
  }`}
>
  {guardandoSeguridad ? 'Cambiando...' : 'Cambiar contraseña'}
</button>

{/* Mensaje de error o bloqueo */}
{errorSeguridad && (
  <div className="mt-2 p-3 bg-error-50 border border-error-200 rounded-lg text-error text-sm">
    {errorSeguridad}
  </div>
)}

{/* Mostrar tiempo restante si está bloqueado */}
{isBloqueado && bloqueadoHasta && (
  <div className="mt-2 p-3 bg-warm-100 border border-warm-200 rounded-lg text-warm-700 text-sm">
    Bloqueado. Reinténtalo en {Math.ceil((bloqueadoHasta - Date.now()) / 1000)} segundos.
  </div>
)}
```

**🔄 Flujo Completo de Cambio de Contraseña:**

```
Usuario ingresa:
  ├─ Contraseña actual
  ├─ Nueva contraseña (8+ chars)
  └─ Confirmar nueva contraseña

Al hacer click "Cambiar contraseña":

✓ Si contraseña actual CORRECTA:
  └─ updateUser({ password: newPassword })
     └─ Éxito: mostrar toast
     
✗ Si contraseña actual INCORRECTA:
  ├─ Intento 1 → "2 intentos restantes"
  ├─ Intento 2 → "1 intento restante"
  └─ Intento 3 → BLOQUEAR por 5 minutos
     ├─ Botón deshabilitado
     ├─ Contador regresivo visible
     └─ Al pasar 5 min → desbloquear automáticamente
```

---

## 🧬 Componentes & Hooks Nuevos

### Hooks
| Hook | Ubicación | Qué hace |
|------|-----------|----------|
| `useNotificaciones()` | `src/lib/notificaciones.ts` | Realtime + marcar leída/todas |
| `useAuditLog()` | `src/hooks/useAuditLog.ts` | Fetcha audit_log con filtros |

### Componentes
| Componente | Ubicación | Qué hace |
|-----------|-----------|----------|
| `CampanaNotificaciones` | `src/components/shared/CampanaNotificaciones.tsx` | Dropdown badge en navbar |
| `RangoDeFechas` | `src/components/shared/RangoDeFechas.tsx` | Filtro rango con validación |
| `ModalDetalleAudit` | `src/components/admin/auditoria/ModalDetalleAudit.tsx` | Vista detalle auditoría |

**💻 Código en Detalle - RangoDeFechas (componente reutilizable):**

```typescript
// src/components/shared/RangoDeFechas.tsx (COMPLETO)
// ┌─ Componente que permite seleccionar rango de fechas con validación

type Props = {
  fechaDesde: string      // ← Fecha desde (string ISO)
  fechaHasta: string      // ← Fecha hasta (string ISO)
  onChangeDesde: (val: string) => void  // ← Callback cuando cambia "desde"
  onChangeHasta: (val: string) => void  // ← Callback cuando cambia "hasta"
}

export default function RangoDeFechas({ 
  fechaDesde, 
  fechaHasta, 
  onChangeDesde, 
  onChangeHasta 
}: Props) {
  let error: string | null = null  // ← Variable para almacenar error

  // ┌─ PASO 1: Validación - si ambas fechas están presentes
  if (fechaDesde && fechaHasta) {
    const desde = new Date(fechaDesde)      // ← Parsear "desde"
    const hasta = new Date(fechaHasta)      // ← Parsear "hasta"
    
    // ┌─ Calcular diferencia en milisegundos
    const diffTime = Math.abs(hasta.getTime() - desde.getTime())
    
    // ┌─ Convertir a días (1 día = 1000ms * 60s * 60min * 24h)
    const diffDays = Math.ceil(diffTime / (1000 * 60 * 60 * 24))

    // ┌─ VALIDACIÓN 1: "Desde" no puede ser mayor que "Hasta"
    if (desde > hasta) {
      error = 'La fecha "Desde" no puede ser mayor que "Hasta".'
    } 
    // ┌─ VALIDACIÓN 2: Máximo 90 días de rango permitido
    else if (diffDays > 90) {
      error = 'El rango máximo permitido es de 90 días.'
    }
  }

  return (
    <div className="flex flex-col sm:flex-row gap-4">  {/* ← Layout: col en mobile, row en desktop */}
      {/* INPUT 1: Fecha DESDE */}
      <div className="flex-1">  {/* ← Ocupa 50% del espacio */}
        <label htmlFor="f-desde" className="block text-xs font-medium text-primary-900 mb-1">
          Desde  {/* ← Etiqueta */}
        </label>
        <input
          id="f-desde"
          type="date"              {/* ← Input tipo date (nativo del navegador) */}
          value={fechaDesde}       {/* ← Valor controlado */}
          onChange={e => onChangeDesde(e.target.value)}  {/* ← Llamar callback padre */}
          className="w-full h-10 px-2.5 rounded-lg border border-warm-300 text-sm text-primary-900 focus:outline-none focus:ring-2 focus:ring-primary-400"
        />
      </div>

      {/* INPUT 2: Fecha HASTA */}
      <div className="flex-1">  {/* ← Ocupa 50% del espacio */}
        <label htmlFor="f-hasta" className="block text-xs font-medium text-primary-900 mb-1">
          Hasta  {/* ← Etiqueta */}
        </label>
        <input
          id="f-hasta"
          type="date"              {/* ← Input tipo date (nativo del navegador) */}
          value={fechaHasta}       {/* ← Valor controlado */}
          onChange={e => onChangeHasta(e.target.value)}  {/* ← Llamar callback padre */}
          className="w-full h-10 px-2.5 rounded-lg border border-warm-300 text-sm text-primary-900 focus:outline-none focus:ring-2 focus:ring-primary-400"
        />
      </div>

      {/* MOSTRAR ERROR SI EXISTE */}
      {error && (
        <p className="w-full sm:w-auto text-xs text-error mt-1 sm:mt-0 sm:self-center">
          {error}  {/* ← Mensaje de error en rojo */}
        </p>
      )}
    </div>
  )
}
```

**💡 Cómo se Usa (ejemplo en Página Notificaciones):**

```typescript
// src/pages/Notificaciones.tsx (líneas 49-51 + 124-129)

const [fechaDesde, setFechaDesde] = useState('')   // ← Estado local
const [fechaHasta, setFechaHasta] = useState('')   // ← Estado local

// ... en el JSX:

<RangoDeFechas
  fechaDesde={fechaDesde}
  fechaHasta={fechaHasta}
  onChangeDesde={setFechaDesde}      {/* ← Actualizar estado */}
  onChangeHasta={setFechaHasta}      {/* ← Actualizar estado */}
/>
```

**🔄 Flujo Completo:**

```
Usuario ingresa fecha DESDE = "2026-05-10"
  ↓
onChange → onChangeDesde("2026-05-10")
  ↓
setState(fechaDesde = "2026-05-10")
  ↓
Rerender → Validar: ¿fechaDesde < fechaHasta? ¿< 90 días?
  ├─ ✅ OK → error = null
  └─ ❌ ERROR → mostrar mensaje rojo

Filtrar notificaciones:
  └─ if (fechaDesde) → only notificaciones >= fechaDesde
  └─ if (fechaHasta) → only notificaciones <= fechaHasta
```

---

## 🏗️ Arquitectura de Notificaciones

```
┌─────────────────────────────────────────────────────────────────┐
│                         FRONTEND                                │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  CampanaNotificaciones (navbar)                          │  │
│  │    └─ useNotificaciones() → escucha Realtime            │  │
│  │       ├─ INSERT → suma noLeidasCount                     │  │
│  │       └─ UPDATE → actualiza lista                        │  │
│  │    → marcarComoLeida() → UPDATE a BD                     │  │
│  │                                                           │  │
│  │  Página /notificaciones                                  │  │
│  │    └─ RangoDeFechas (filtro)                             │  │
│  │    └─ Lista con ícono por tipo                           │  │
│  └──────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
                              ↓ Realtime
┌─────────────────────────────────────────────────────────────────┐
│                     SUPABASE (Backend)                          │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  Tabla: notificaciones                                   │  │
│  │  ├─ id, usuario_id, tipo, titulo, mensaje, leida        │  │
│  │  └─ RLS: SELECT/UPDATE solo usuario dueño               │  │
│  │                                                           │  │
│  │  Triggers:                                               │  │
│  │  ├─ after_solicitud_estado_changed → INSERT notifo      │  │
│  │  └─ after_solicitud_creada → INSERT notifo              │  │
│  │                                                           │  │
│  │  Edge Function: notificar-cambio-estado                 │  │
│  │  └─ Resend API → email HTML (si RESEND_API_KEY)        │  │
│  └──────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
```

---

## 🔐 Catálogo de Auditoría Centralizado

**Nuevo en Sprint 6: `src/lib/audit.ts`**

Tipos cerrados TypeScript para evitar errores:

```typescript
export type AccionAudit = 
  | 'asignar_solicitud'
  | 'actualizar_estado_solicitud'
  | 'confirmar_solicitud'
  | 'rechazar_solucion'
  | 'escalada_solicitud'
  | 'editar_perfil'

export type EntidadAudit = 
  | 'solicitudes' | 'asignaciones' | 'usuarios'

// Helper centralizado
async function logAuditAction(entry: AuditEntry, usuarioId: string)
```

**💻 Código en Detalle - Catálogo de Auditoría (src/lib/audit.ts):**

```typescript
// src/lib/audit.ts (líneas 18-111)
// ┌─ Catálogo cerrado de acciones auditables

/** Acciones que el frontend (rol authenticated) puede registrar. */
export type AccionAudit =
  // Solicitudes — cambios de estado
  | 'asignar_solicitud'           // ← Asignar a técnico
  | 'actualizar_estado_solicitud' // ← Cambiar estado
  | 'confirmar_solicitud'         // ← Residente confirma solución
  | 'rechazar_solucion'           // ← Residente rechaza
  | 'escalada_solicitud'          // ← Escalar a admin
  // Usuarios — perfil propio
  | 'editar_perfil'               // ← Cambiar datos perfil

/** Entidades sobre las que se audita desde el frontend authenticated. */
export type EntidadAudit =
  | 'solicitudes'   // ← Tabla solicitudes
  | 'asignaciones'  // ← Tabla asignaciones
  | 'usuarios'      // ← Tabla usuarios

/** Resultado de la acción auditada. */
export type ResultadoAudit = 'exitoso' | 'fallido'

// ┌─ Tipo del payload que se envía a BD
export type AuditEntry = {
  accion: AccionAudit          // ← Acción cerrada (TypeScript valida)
  entidad: EntidadAudit        // ← Entidad cerrada (TypeScript valida)
  /** UUID de la fila afectada (solicitud, asignacion, perfil, …). */
  entidadId: string            // ← ID del recurso
  /**
   * Detalles JSON arbitrarios. Política no-PII: solo IDs, flags, valores de
   * estado o números. No incluir nombres, emails, teléfonos ni descripciones
   * libres.
   */
  detalles?: Record<string, unknown>  // ← Info extra (ej: estado anterior/nuevo)
  /** Por defecto 'exitoso'. Si la acción falló pero el audit es relevante igual, pasar 'fallido'. */
  resultado?: ResultadoAudit   // ← Resultado (exitoso/fallido)
}

// ┌─ HELPER PRINCIPAL: Inserta entrada en audit_log
/**
 * Sprint 5 · PBI-14
 * Inserta una entrada en audit_log. Devuelve { ok } para el llamador que
 * quiera reportarlo en logs; el llamador NO debe propagar errores al usuario
 * — la auditoría es secundaria al flujo principal.
 */
export async function logAuditAction(
  entry: AuditEntry,
  usuarioId: string,
): Promise<{ ok: boolean; error?: string }> {
  // ┌─ Construir payload para enviar a BD
  const payload = {
    usuario_id: usuarioId,           // ← Quién hizo la acción
    accion: entry.accion,            // ← Qué acción
    entidad: entry.entidad,          // ← Sobre qué entidad
    entidad_id: entry.entidadId,     // ← ID del recurso
    detalles: entry.detalles ?? {},  // ← Detalles extra (JSON)
    resultado: entry.resultado ?? 'exitoso',  // ← OK o ERROR
  }

  // ┌─ Enviar a Supabase (fire-and-forget)
  const { error } = await supabase.from('audit_log').insert(payload)

  if (error) {
    // Log error pero NO bloquear operación principal
    console.warn('[audit] Failed to log action:', entry.accion, error.message)
    return { ok: false, error: error.message }
  }

  return { ok: true }
}

// ┌─ Catálogos para construir selects/filtros en UI admin
export const ACCIONES_AUDIT: ReadonlyArray<AccionAudit> = [
  'asignar_solicitud',
  'actualizar_estado_solicitud',
  'confirmar_solicitud',
  'rechazar_solucion',
  'escalada_solicitud',
  'editar_perfil',
] as const

/** Acciones que también aparecen en audit_log pero las generan Edge Functions
 *  o triggers — útil para construir el select de filtros. */
export const ACCIONES_AUDIT_COMPLETO: ReadonlyArray<string> = [
  ...ACCIONES_AUDIT,
  // Origen: trigger log_solicitud_creada (Sprint 3)
  'crear_solicitud',
  // Origen: trigger log_solicitud_prioridad_cambiada (Sprint 3)
  'cambiar_prioridad',
  // Origen: Edge Function `invitaciones` (Sprint 2)
  'crear_invitacion',
  // Origen: Edge Function `bloquear-cuenta` (Sprint 2)
  'activar_cuenta',
  'bloquear_cuenta',
  'desbloquear_cuenta',
] as const

// ┌─ Etiqueta humana para mostrar en UI (ej: filtros de auditoría)
export function labelAccion(accion: string): string {
  const labels: Record<string, string> = {
    asignar_solicitud: 'Asignar solicitud',
    actualizar_estado_solicitud: 'Actualizar estado',
    confirmar_solicitud: 'Confirmar solución',
    rechazar_solucion: 'Rechazar solución',
    escalada_solicitud: 'Escalada al admin',
    editar_perfil: 'Editar perfil',
    crear_solicitud: 'Crear solicitud',
    cambiar_prioridad: 'Cambiar prioridad',
    crear_invitacion: 'Crear invitación',
    activar_cuenta: 'Activar cuenta',
    bloquear_cuenta: 'Bloquear cuenta',
    desbloquear_cuenta: 'Desbloquear cuenta',
  }
  return labels[accion] ?? accion  // ← Si no existe, devolver la acción tal cual
}
```

**💡 Cómo se Usa en Frontend (ejemplo):**

```typescript
// Cuando se rechaza una solicitud:

import { logAuditAction } from './lib/audit'

// ... en componente

const handleRechazar = async () => {
  // ... hacer UPDATE de solicitud ...
  
  // Registrar en auditoría
  await logAuditAction(
    {
      accion: 'rechazar_solucion',     // ← Acción cerrada (TypeScript valida)
      entidad: 'solicitudes',          // ← Entidad cerrada
      entidadId: solicitudId,          // ← ID de la solicitud
      detalles: {
        motivo: 'No conforme con resultado',  // ← Solo info no-PII
        estado_anterior: 'resuelta',
        estado_nuevo: 'en_progreso',
      },
      resultado: 'exitoso',  // ← OK
    },
    user.id  // ← Quién hace la acción
  )
}
```

**✅ Ventajas del Catálogo Centralizado:**

1. **TypeScript valida tipos** → no puedes pasar acción "typo"
2. **Una fuente de verdad** → catálogo en un archivo
3. **Code review stricto** → `supabase.from('audit_log').insert(...)` directo se rechaza
4. **RLS en BD exige usuario_id** → el servidor garantiza el propietario
5. **Fire-and-forget** → auditoría no bloquea operaciones principales

**Ventajas:**
- Catálogo único (una fuente de verdad)
- Código review rechaza audits "piratas"
- RLS en BD exige `usuario_id = auth.uid()`
- Fire-and-forget: no bloquea si falla auditoría

**📁 Ubicación:**
```
src/lib/audit.ts (líneas 1-157)
  - ACCIONES_AUDIT (líneas 104-111)
  - ENTIDADES_AUDIT_FILTRO (líneas 115-121)
  - labelAccion() para UI (líneas 140-156)
```

---

## 📊 Datos de Demostración

**Seed actualizado (`npm run seed`):**

Inserta 6 usuarios demo + 3 solicitudes + notificaciones ejemplo:

| Email | Rol | Contraseña | Notas |
|-------|-----|------------|-------|
| carlos@zity-demo.com | admin | `Admin1234!` | — |
| laura@zity-demo.com | residente | `Residente1!` | Piso 4-B |
| mario@zity-demo.com | técnico | `Tecnico1234!` | — |

Solicitudes demo: con `imagen_url` externa (picsum.photos) para evitar latencia de upload.

---

## 🚀 Flujos Principales (Cómo Probar)

### ✨ Flujo 1: Ver Notificación en Tiempo Real
1. Abre 2 tabs: admin + residente
2. Residente crea solicitud
3. Admin observa badge actualizado sin refresh
4. Realtime mantiene lista sincronizada

### 📝 Flujo 2: Marcar Foto de Cierre
1. Técnico abre solicitud asignada
2. Al pasar a "resuelta", aparece upload de foto
3. Sube foto (< 5 MB)
4. Foto se guarda en bucket y ruta se serializa en nota
5. Residente ve foto original + cierre lado a lado

### 🔒 Flujo 3: Cambio de Contraseña Seguro
1. Usuario va a `/perfil` → tab "Seguridad"
2. Ingresa contraseña actual (re-auth)
3. Nueva contraseña 2x (validación)
4. Si 3 fallos seguidos → bloqueado 5 minutos
5. Éxito: sesión sigue activa

### 📊 Flujo 4: Ver Auditoría
1. Admin abre solicitud en drawer
2. Botón "Ver auditoría"
3. Modal muestra todas las acciones sobre esa solicitud
4. Filtros: quién, qué, cuándo

---

## 📈 Riesgos Mitigados

| Riesgo | Mitigación (Sprint 6) |
|--------|----------------------|
| **R1: Latencia Realtime (WiFi inestable)** | Reconexión exponencial + reintentos (1s, 2s, 4s, ..., máx 30s) |
| **R2: RESEND free tier 100 emails/día** | Dry-run en dev (RESEND_API_KEY ausente) |
| **R3: Trigger duplicados → múltiples notifs** | WHEN condition específica (estado + origen) |
| **R4: Foto grande > 5 MB** | Validación client + server en RLS |
| **R5: Reauth deja usuario en RECOVERY** | Mantener sesión activa post-password-update |
| **R6: Notifs no leídas sin TTL** | Documentado: limitar a 90 días en Sprint 11 |

---

## 🔧 Comandos Útiles

```bash
# Desarrollo local
npm run dev

# Tests con cobertura (meta: > 60%)
npm run test:coverage

# Seed datos demo (idempotente)
npm run seed

# Limpiar y re-seeding
npm run seed:clean

# Lint + typecheck
npm run lint
npm run typecheck

# Build para producción
npm run build
```

---

## 📖 Guía Rápida: Línea por Línea en Archivos Clave

### 🔴 RED FLAG vs 🟢 GREEN: Qué cambió entre Sprint 5 y Sprint 6

| Archivo | Línea | Qué Cambió | Por qué | Estado |
|---------|-------|-----------|--------|--------|
| `src/lib/notificaciones.ts` | 1-119 | **NUEVO** | Realtime hook + marcar leídas | 🟢 Agregado |
| `src/lib/audit.ts` | 1-157 | **NUEVO** | Catálogo centralizado de auditoría | 🟢 Agregado |
| `src/pages/Notificaciones.tsx` | 1-195 | **NUEVO** | Página completa centro notificaciones | 🟢 Agregado |
| `src/components/shared/CampanaNotificaciones.tsx` | 1-158 | **NUEVO** | Dropdown badge en navbar | 🟢 Agregado |
| `src/components/shared/RangoDeFechas.tsx` | 1-60 | **NUEVO** | Componente filtro fechas reutilizable | 🟢 Agregado |
| `src/pages/Perfil.tsx` | 30-171 | **EXPANDIDO** | Agregó tab "Seguridad" + rate limiting | 🟢 Modificado |
| `src/components/tecnico/solicitudes/SeccionActualizarEstado.tsx` | 64-100 | **EXPANDIDO** | Lógica foto cierre + serialización | 🟢 Modificado |
| `supabase/functions/notificar-cambio-estado/index.ts` | 1-113 | **NUEVO** | Edge Function para emails | 🟢 Agregado |

---

## 🎓 Explicación de Conceptos Clave

### 1️⃣ Optimistic Update (Actualización Optimista)

**¿Qué es?**
Actualizar la UI inmediatamente sin esperar respuesta del servidor. Si falla, hacer rollback.

**¿Dónde se usa en Sprint 6?**
- `marcarComoLeida()` en notificaciones
- Contador de no leídas se resta al instante

**Ventaja:** UI responde instantáneamente, sensación de rapidez

**Desventaja:** Si falla, necesitar rollback manual

```typescript
// ✅ OPTIMISTIC
setNotificaciones(prev => [nueva, ...prev])  // UI actualiza YA
await supabase.from('notificaciones').insert(...)  // En background
```

### 2️⃣ Realtime Subscription (Suscripción en Tiempo Real)

**¿Qué es?**
Escuchar cambios en tabla de BD sin polling. WebSocket.

**¿Dónde se usa en Sprint 6?**
- `useNotificaciones()` escucha INSERT/UPDATE en tabla notificaciones
- Cuando un admin crea notificación → todos los usuarios ven update al instante

**Ventaja:** Sin latencia, sin polling (sin requests cada segundo)

**Desventaja:** Requiere WebSocket activo

```typescript
.on('postgres_changes', 
  { event: '*', table: 'notificaciones', filter: `usuario_id=eq.${usuarioId}` },
  (payload) => { /* actualizar UI */ }
).subscribe()
```

### 3️⃣ Fire-and-Forget (Dispara y Olvida)

**¿Qué es?**
Enviar acción pero no esperar respuesta ni bloquear si falla.

**¿Dónde se usa en Sprint 6?**
- `logAuditAction()` → guarda en audit_log pero no bloquea si falla
- Auditoría es "nice-to-have", no crítica

**Ventaja:** No ralentiza operación principal

**Desventaja:** Auditoría podría fallar silenciosamente

```typescript
await logAuditAction(entry, userId)  // Sin await en el llamador
if (error) {
  console.warn('Auditoría falló')  // Log, pero no abortar
  return { ok: false, error }
}
```

### 4️⃣ Rate Limiting (Limitación de Intentos)

**¿Qué es?**
Permitir N intentos, luego bloquear por X minutos.

**¿Dónde se usa en Sprint 6?**
- Cambio de contraseña: 3 intentos fallidos = bloqueo 5 minutos
- Previene fuerza bruta

**Ventaja:** Protección contra ataques

**Desventaja:** Usuario puede quedar bloqueado legítimamente

```typescript
const intentosFallidos = 3
if (intentosFallidos >= 3) {
  bloqueadoHasta = Date.now() + 5 * 60 * 1000  // 5 min
}
```

### 5️⃣ Serialización JSON en Campo de Texto

**¿Qué es?**
Guardar JSON en columna TEXT para agregar metadatos.

**¿Dónde se usa en Sprint 6?**
- Foto cierre se guarda como JSON en `nota`: `[cierre] {"archivo": "path/foto.jpg"}`
- Permite residente ver foto original + cierre lado a lado

**Ventaja:** Flexibilidad sin cambiar schema BD

**Desventaja:** Requiere parsing en frontend

```typescript
const notaCierre = `[cierre] ${JSON.stringify({ archivo: "path" })}\nNota técnico`
```

### 6️⃣ RLS (Row Level Security)

**¿Qué es?**
Policies de BD que garantizan que solo el propietario puede ver/editar sus datos.

**¿Dónde se usa en Sprint 6?**
- `notificaciones`: usuario solo ve sus propias notificaciones
- `audit_log`: `usuario_id = auth.uid()` se fija automáticamente

**Ventaja:** Seguridad a nivel BD (no confiar en código)

**Desventaja:** Más complicado debuggear errores de permisos

```sql
-- RLS: SELECT solo si usuario_id = auth.uid()
SELECT * FROM notificaciones WHERE usuario_id = auth.uid()
```

---

## 🎯 Checklist de Exposición

Cuando presentes, asegúrate de mostrar:

### ✅ Notificaciones Realtime (PBI-12)
- [ ] Abre 2 tabs: admin + residente
- [ ] Residente crea solicitud
- [ ] Badge se actualiza sin refresh
- [ ] Explica: "Se usa Realtime, no polling"

### ✅ Centro de Notificaciones (HU-NOTIF-01 + HU-NOTIF-02)
- [ ] Badge pulsa con contador
- [ ] Dropdown muestra últimas 10
- [ ] Botón "Marcar como leída" funciona
- [ ] Página `/notificaciones` lista todas
- [ ] Explica: "Optimistic update → UI responde al instante"

### ✅ Foto de Cierre (PBI-S3-E01)
- [ ] Técnico crea solicitud → pasa a "resuelta"
- [ ] Aparece input para subir foto
- [ ] Sube foto (< 5 MB)
- [ ] Residente ve foto original + cierre
- [ ] Explica: "JSON en nota permite metadata flexible"

### ✅ Cambio de Contraseña Seguro (PBI-S5-E03)
- [ ] Ir a `/perfil` → tab "Seguridad"
- [ ] Intentar contraseña incorrecta 3 veces
- [ ] Mostrar bloqueo por 5 minutos
- [ ] Explica: "Rate limiting previene fuerza bruta"

### ✅ Auditoría Centralizada
- [ ] Mostrar `src/lib/audit.ts`
- [ ] Explica: "Catálogo cerrado en TypeScript"
- [ ] Admin abre drawer → botón "Ver auditoría"
- [ ] Modal muestra todas las acciones

---

## 📊 Métricas Sprint 6

| Métrica | Valor | Estado |
|---------|-------|--------|
| **PBIs Completados** | 7/7 | ✅ 100% |
| **Horas Estimadas** | 13h | ✅ |
| **Archivos Nuevos** | 5 | ✅ |
| **Archivos Modificados** | 3 | ✅ |
| **Tests Agregados** | 6+ | ✅ |
| **Cobertura Meta** | > 60% | 🔄 En revisión |
| **Code Review** | Pendiente | ⏳ |
| **Deploy a Staging** | Pendiente | ⏳ |

---

- **Auditoría:** `docs/audit.md`
- **Storage (foto cierre):** `docs/storage.md`, `docs/adr/005-storage.md`
- **DB Schema:** `docs/db/schema.md`
- **RLS Policies:** `docs/db/rls.md`
- **Roadmap:** `docs/sprints/Zity_Roadmap_Sprints.pdf`

---

## 👥 Responsables Sprint 6

| Responsable | PBIs | Horas |
|-------------|------|-------|
| **Gonza Morales** | PBI-12 | 4h |
| **Santiago Flores** | HU-NOTIF-01, HU-NOTIF-02, PBI-S5-E02 | 5.5h |
| **Cortez Zamora** | PBI-S3-E01, PBI-S4-E01, PBI-S5-E03 | 3.5h |
| **TOTAL** | 7 PBIs | 13h |

---

**Versión:** Sprint 6 v2  
**Rama:** cortez (actualizada a main)  
**Fecha:** 2026-05-20

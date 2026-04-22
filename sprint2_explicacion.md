# Sprint 2 — Explicación con Código

> Cada sección muestra **qué archivo** se creó o modificó y **qué hace el código**.
> Las partes de Gonza están marcadas con 🟡.

---

## 1. 🏗️ PBI-S2-01 — Modelado de la Base de Datos
**Responsable:** Cortez Zamora Leonardo

### `src/types/database.ts`

Define la "forma" de todos los datos del sistema. No guarda nada, solo describe cómo luce cada tabla.

```typescript
// Los 3 roles posibles en el sistema
export type Rol = 'residente' | 'admin' | 'tecnico'

// Los 3 estados que puede tener una cuenta
export type EstadoCuenta = 'pendiente' | 'activo' | 'bloqueado'

// Así luce un usuario en la base de datos
export type Profile = {
  id: string
  email: string
  nombre: string
  apellido: string
  telefono: string
  rol: Rol                        // residente | admin | tecnico
  piso: string
  departamento: string
  estado_cuenta: EstadoCuenta     // pendiente | activo | bloqueado
  empresa_tercero: string | null  // solo aplica para técnicos
  created_at: string
  updated_at: string
}

// Así luce una solicitud de mantenimiento
export type Solicitud = {
  id: string
  residente_id: string
  tipo: 'mantenimiento' | 'reparacion' | 'queja' | 'sugerencia' | 'otro'
  categoria: 'plomeria' | 'electricidad' | 'limpieza' | 'seguridad' | 'areas_comunes' | 'otro'
  descripcion: string
  estado: 'pendiente' | 'asignada' | 'en_progreso' | 'resuelta' | 'cerrada'
  // ...
}

// Registro de todas las acciones importantes (auditoría)
export type AuditLog = {
  id: string
  usuario_id: string | null
  accion: string        // ej: 'bloquear_cuenta', 'crear_invitacion'
  entidad: string | null
  resultado: string | null
  created_at: string
}
```

> **Nota:** Este archivo solo *describe* los datos. La base de datos real vive en Supabase
> y fue armada por Cortez con migraciones SQL y reglas de seguridad (RLS).

---

## 2. 🟡 PBI-AUTH-06 — Invitar Usuarios por Email
**Responsable:** Gonza Morales Yoel

> ⚠️ **GONZA** — Esta funcionalidad completa es de Gonza. Son dos archivos: uno de pantalla y uno de servidor.

### `src/components/admin/ModalInvitacion.tsx`

El popup que ve el admin para invitar a alguien.

```tsx
// El modal recibe dos funciones: qué hacer al enviar y qué hacer al cerrar
type Props = {
  onEnviado: (email: string) => void  // se llama cuando el envío fue exitoso
  onCerrar: () => void                // se llama al cerrar sin enviar
}

export default function ModalInvitacion({ onEnviado, onCerrar }: Props) {
  const { enviarInvitacion, cargando, error } = useInvitacion()

  // Estado del formulario (todos los campos)
  const [form, setForm] = useState({
    email: '',
    nombre: '',
    rol: 'residente' as Rol,   // por defecto: residente
    piso: '',
    departamento: '',
    empresa_tercero: '',
  })

  async function handleSubmit(e: React.FormEvent) {
    e.preventDefault()
    const ok = await enviarInvitacion({
      email: form.email,
      rol: form.rol,
      nombre: form.nombre,
      // Si el rol es técnico, incluye la empresa; si no, lo omite
      empresa_tercero: form.rol === 'tecnico' ? form.empresa_tercero : undefined,
    })
    if (ok) onEnviado(form.email)  // ✅ Avisa al padre que salió bien
  }

  return (
    <div className="fixed inset-0 z-50 flex items-center justify-center">
      {/* Fondo oscuro que cierra el modal al hacer clic */}
      <div className="absolute inset-0 bg-black/40" onClick={onCerrar} />
      <div className="relative bg-white rounded-xl p-6 w-full max-w-lg">
        <form onSubmit={handleSubmit}>
          <input type="email" placeholder="correo@ejemplo.com" />
          <input type="text" placeholder="Nombre" />
          <select>{/* Residente / Técnico / Admin */}</select>
          {/* Si es técnico, aparece campo de empresa */}
          {form.rol === 'tecnico' && <input placeholder="TecnoEdif SAC" />}
          <input placeholder="Piso" />
          <input placeholder="Departamento (4B)" />
          <button type="submit">Enviar invitación</button>
        </form>
      </div>
    </div>
  )
}
```

---

### `supabase/functions/invitaciones/index.ts`

El servidor que procesa la invitación. Corre en la nube (Supabase Edge Functions).

```typescript
Deno.serve(async (req: Request) => {
  // 1. Verificar que quien llama está autenticado
  const token = req.headers.get('authorization').replace('Bearer ', '')
  const { data: { user } } = await supabaseAdmin.auth.getUser(token)
  if (!user) return Response(401) // ❌ No autorizado

  // 2. Verificar que es admin (no cualquiera puede invitar)
  const { data: profile } = await supabaseAdmin
    .from('usuarios').select('rol').eq('id', user.id).single()
  if (profile?.rol !== 'admin') return Response(403) // ❌ Solo admins

  // 3. Leer los datos del formulario
  const { email, rol, nombre, piso, departamento, empresa_tercero } = await req.json()

  // 4. Supabase crea el usuario y manda el email automáticamente
  //    El link del email lleva al usuario a /activar para crear su contraseña
  await supabaseAdmin.auth.admin.inviteUserByEmail(email, {
    data: { nombre, rol, piso, departamento, empresa_tercero },
    redirectTo: `https://zity.site/activar`,
  })

  // 5. Guardar en tabla 'invitaciones' para rastrearla después
  await supabaseAdmin.from('invitaciones').insert({
    email, rol, nombre, piso, departamento,
    token: crypto.randomUUID(),
    creada_por: user.id,  // quién la creó
  })

  // 6. Registrar en audit_log (historial de acciones)
  await supabaseAdmin.from('audit_log').insert({
    usuario_id: user.id,
    accion: 'crear_invitacion',
    resultado: 'exitoso',
  })

  return Response({ success: true }) // ✅
})
```

---

## 3. 📋 PBI-S2-02 — Panel Admin: Lista de Usuarios
**Responsable:** Santiago Flores Carlos

### `src/hooks/useUsuarios.ts`

El "conector" con la base de datos. Busca usuarios aplicando filtros.

```typescript
export function useUsuarios(filtros: FiltrosState) {
  const [usuarios, setUsuarios] = useState<Profile[]>([])
  const [loading, setLoading] = useState(true)

  useEffect(() => {
    // Construir la consulta base
    let query = supabase
      .from('usuarios')
      .select('*')
      .order('created_at', { ascending: false }) // más recientes primero

    // Aplicar filtros solo si fueron seleccionados
    if (filtros.rol)    query = query.eq('rol', filtros.rol)
    if (filtros.estado) query = query.eq('estado_cuenta', filtros.estado)

    // Ejecutar con timeout de 8 segundos por seguridad
    const { data } = await query
    setUsuarios(data ?? [])
  }, [filtros.rol, filtros.estado]) // se re-ejecuta si cambia algún filtro

  return { usuarios, loading, refetch }
}
```

---

### `src/components/admin/FiltrosUsuarios.tsx`

Dos menús desplegables para filtrar la tabla por rol y estado.

```tsx
export default function FiltrosUsuarios({ filtros, onChange }) {
  return (
    <div className="flex gap-3">
      {/* Filtro por rol */}
      <select onChange={e => onChange({ ...filtros, rol: e.target.value })}>
        <option value="">Todos los roles</option>
        <option value="admin">Admin</option>
        <option value="residente">Residente</option>
        <option value="tecnico">Técnico</option>
      </select>

      {/* Filtro por estado */}
      <select onChange={e => onChange({ ...filtros, estado: e.target.value })}>
        <option value="">Todos los estados</option>
        <option value="activo">Activo</option>
        <option value="pendiente">Pendiente</option>
        <option value="bloqueado">Bloqueado</option>
      </select>

      {/* Botón limpiar, solo aparece si hay algún filtro activo */}
      {(filtros.rol || filtros.estado) && (
        <button onClick={() => onChange({ rol: '', estado: '' })}>
          Limpiar filtros
        </button>
      )}
    </div>
  )
}
```

---

### `src/components/admin/BadgeEstado.tsx`

La etiquetita de colores para el estado de la cuenta.

```tsx
const config = {
  activo:    { label: 'activo',    classes: 'bg-green/10 text-green'   }, // 🟢
  pendiente: { label: 'pendiente', classes: 'bg-yellow/10 text-yellow' }, // 🟡
  bloqueado: { label: 'bloqueado', classes: 'bg-red/10 text-red'       }, // 🔴
}

export default function BadgeEstado({ estado }: { estado: EstadoCuenta }) {
  const { label, classes } = config[estado]
  return (
    <span className={`rounded-full px-2.5 py-0.5 text-xs font-medium ${classes}`}>
      {label}
    </span>
  )
}
```

---

### `src/components/admin/TablaUsuarios.tsx`

La tabla principal con todos los usuarios del edificio.

```tsx
export default function TablaUsuarios({ usuarios, loading, onBloquear, onDesbloquear }) {
  // Muestra "hace 3 días", "hace 2 horas", etc.
  function tiempoTranscurrido(fechaISO: string): string {
    const diffMs = Date.now() - new Date(fechaISO).getTime()
    const rtf = new Intl.RelativeTimeFormat('es', { numeric: 'auto' })
    const diffDays = Math.floor(diffMs / 86_400_000)
    if (diffDays >= 1) return rtf.format(-diffDays, 'day') // "hace 3 días"
    // ... también calcula horas y minutos
  }

  return (
    <table>
      <thead>
        <tr>
          <th>Nombre</th> <th>Email</th> <th>Rol</th>
          <th>Piso / Depto</th> <th>Estado</th> <th>Registro</th> <th />
        </tr>
      </thead>
      <tbody>
        {usuarios.map(usuario => (
          <tr key={usuario.id}>
            <td>{usuario.nombre} {usuario.apellido}</td>
            <td>{usuario.email}</td>
            <td>
              {usuario.rol}
              {/* Técnicos muestran su empresa debajo del rol */}
              {usuario.rol === 'tecnico' && <span>{usuario.empresa_tercero}</span>}
            </td>
            <td>{usuario.piso && `Piso ${usuario.piso} — ${usuario.departamento}`}</td>
            <td><BadgeEstado estado={usuario.estado_cuenta} /></td>
            <td>{tiempoTranscurrido(usuario.created_at)}</td>
            <td>
              {/* El botón cambia según el estado actual del usuario */}
              {usuario.estado_cuenta === 'bloqueado'
                ? <button onClick={() => onDesbloquear(usuario)}>Desbloquear</button>
                : <button onClick={() => onBloquear(usuario)}>Bloquear</button>
              }
            </td>
          </tr>
        ))}
      </tbody>
    </table>
  )
}
```

---

## 4. 🪜 PBI-AUTH-01b — Registro en 2 Pasos con Barra de Progreso
**Responsable:** Cortez Zamora Leonardo

### `src/components/BarraProgreso.tsx`

Componente que muestra el porcentaje completado del registro.

```tsx
type Props = { pasoActual: number; totalPasos: number }

export default function BarraProgreso({ pasoActual, totalPasos }: Props) {
  // Paso 1 de 2 → 50%. Paso 2 de 2 → 100%
  const porcentaje = Math.round((pasoActual / totalPasos) * 100)

  return (
    <div>
      <div className="flex justify-between">
        <span>Paso {pasoActual} de {totalPasos}</span>
        <span>{porcentaje}%</span>
      </div>
      {/* La barra: animación suave de 500ms al cambiar de paso */}
      <div className="h-1.5 bg-gray-200 rounded-full">
        <div
          className="h-full bg-gradient-to-r from-blue-500 to-purple-500 transition-all duration-500"
          style={{ width: `${porcentaje}%` }}  // ← ancho dinámico
        />
      </div>
    </div>
  )
}
```

---

### `src/pages/Register.tsx` (fragmento clave)

Formulario de registro dividido en 2 pantallas, con validación en cada paso.

```tsx
export default function Register() {
  const [step, setStep] = useState<1 | 2>(1) // paso actual

  // Datos del paso 1 (credenciales de acceso)
  const [step1, setStep1] = useState({ email: '', password: '', confirmPassword: '' })

  // Datos del paso 2 (información personal y del edificio)
  const [step2, setStep2] = useState({
    nombre: '', apellido: '',
    telefono: '+51 ',  // prefijo fijo para Perú
    piso: '', departamento: '',
    rol: 'residente',
  })

  // Antes de pasar al paso 2, valida las credenciales
  function handleNext() {
    const errors = validateStep1()
    if (Object.keys(errors).length > 0) { setStep1Errors(errors); return }
    setStep(2) // ✅ avanza
  }

  // Al enviar el paso 2, crea la cuenta en Supabase
  async function handleSubmit(e) {
    e.preventDefault()
    const errors = validateStep2()
    if (Object.keys(errors).length > 0) { setStep2Errors(errors); return }

    await signUp(step1.email, step1.password, {
      nombre: step2.nombre, apellido: step2.apellido,
      telefono: step2.telefono, piso: step2.piso,
      departamento: step2.departamento, rol: step2.rol,
    })
    navigate('/verify-email') // → pantalla "verificá tu correo"
  }

  return (
    <AuthLayout>
      {/* Barra de progreso siempre visible en la parte superior */}
      <BarraProgreso pasoActual={step} totalPasos={2} />

      {/* ① ── ② indicador de puntos */}
      <div className="step-indicator">
        <div className={step === 1 ? 'active' : 'completed'}>{step > 1 ? '✓' : '1'}</div>
        <div className="line" />
        <div className={step === 2 ? 'active' : 'pending'}>2</div>
      </div>

      {step === 1 && (
        <div>
          {/* Email, contraseña, confirmar contraseña */}
          <button onClick={handleNext}>Continuar →</button>
        </div>
      )}

      {step === 2 && (
        <form onSubmit={handleSubmit}>
          {/* Nombre, apellido, teléfono (+51), piso, departamento, rol */}
          <button type="button" onClick={handleBack}>← Atrás</button>
          <button type="submit">Crear cuenta</button>
        </form>
      )}
    </AuthLayout>
  )
}
```

---

## 5. 🟡 PBI-S2-03 — Bloquear / Desbloquear Cuentas
**Responsable:** Gonza Morales Yoel

> ⚠️ **GONZA** — El modal de confirmación y la Edge Function del servidor son de Gonza.

### `src/components/admin/ModalConfirmacion.tsx`

Popup de "¿Estás seguro?" que aparece antes de bloquear o desbloquear.

```tsx
type Props = {
  titulo: string
  mensaje: string
  labelConfirmar: string           // "Bloquear" o "Desbloquear"
  variante: 'peligro' | 'primario' // rojo para bloquear, azul para desbloquear
  cargando: boolean
  onConfirmar: () => void
  onCancelar: () => void
}

export default function ModalConfirmacion({ titulo, mensaje, labelConfirmar, variante, cargando, onConfirmar, onCancelar }) {
  return (
    <div className="fixed inset-0 z-50 flex items-center justify-center">
      {/* Clic en el fondo oscuro → cancela */}
      <div className="absolute inset-0 bg-black/40" onClick={onCancelar} />

      <div className="relative bg-white rounded-xl p-6 max-w-md">
        <h3>{titulo}</h3>   {/* ej: "Bloquear cuenta" */}
        <p>{mensaje}</p>    {/* ej: "¿Estás seguro de bloquear a Juan García?" */}
        <div className="flex gap-3 justify-end">
          <button onClick={onCancelar}>Cancelar</button>
          <button
            onClick={onConfirmar}
            // Rojo si es acción peligrosa (bloquear), azul si no (desbloquear)
            className={variante === 'peligro' ? 'bg-red-500 text-white' : 'btn-primary'}
          >
            {cargando ? <Spinner /> : labelConfirmar}
          </button>
        </div>
      </div>
    </div>
  )
}
```

---

### `supabase/functions/bloquear-cuenta/index.ts`

El servidor que ejecuta el bloqueo o desbloqueo efectivo. Corre en la nube.

```typescript
Deno.serve(async (req: Request) => {
  // 1. Verificar autenticación
  const token = req.headers.get('authorization').replace('Bearer ', '')
  const { data: { user } } = await supabaseAdmin.auth.getUser(token)
  if (!user) return Response(401)

  // 2. Verificar que quien actúa es admin
  const { data: profile } = await supabaseAdmin
    .from('usuarios').select('rol').eq('id', user.id).single()
  if (profile?.rol !== 'admin') return Response(403)

  // 3. Leer qué se quiere hacer
  const { usuario_id, accion } = await req.json()
  // accion = 'bloquear' | 'desbloquear'

  const nuevoEstado = accion === 'bloquear' ? 'bloqueado' : 'activo'

  // 4. Cambiar estado en la tabla 'usuarios'
  await supabaseAdmin
    .from('usuarios')
    .update({ estado_cuenta: nuevoEstado })
    .eq('id', usuario_id)

  // 5. Invalidar la sesión activa del usuario (si está logueado, lo echa)
  //    ban_duration '87600h' = 10 años (efectivamente permanente hasta desbloqueo)
  if (accion === 'bloquear') {
    await supabaseAdmin.auth.admin.updateUserById(usuario_id, {
      ban_duration: '87600h',  // expulsa inmediatamente
    })
  } else {
    await supabaseAdmin.auth.admin.updateUserById(usuario_id, {
      ban_duration: 'none',    // levanta el ban → puede volver a entrar
    })
  }

  // 6. Registrar en audit_log
  await supabaseAdmin.from('audit_log').insert({
    usuario_id: user.id,
    accion: accion === 'bloquear' ? 'bloquear_cuenta' : 'desbloquear_cuenta',
    entidad: 'usuarios',
    entidad_id: usuario_id,
    resultado: 'exitoso',
  })

  return Response({ success: true, estado: nuevoEstado }) // ✅
})
```

---

### `src/pages/admin/Usuarios.tsx` — Cómo se conectan los dos modales de Gonza

Esta página une el trabajo de Santiago (la tabla) con los modales de Gonza.

```tsx
export default function AdminUsuarios() {
  const [tipoAccion, setTipoAccion] = useState<'bloquear' | 'desbloquear' | null>(null)
  const [usuarioAccion, setUsuarioAccion] = useState<Profile | null>(null)
  const [mostrarInvitacion, setMostrarInvitacion] = useState(false)

  // Cuando el admin confirma en el modal → llama a la Edge Function de Gonza
  async function confirmarAccion() {
    const { data } = await supabase.functions.invoke('bloquear-cuenta', {
      body: { usuario_id: usuarioAccion.id, accion: tipoAccion },
    })
    if (data?.success) refetch() // refresca la tabla automáticamente
  }

  return (
    <div>
      {/* Botón que abre el ModalInvitacion de Gonza */}
      <button onClick={() => setMostrarInvitacion(true)}>
        Invitar usuario
      </button>

      {/* Tabla de Santiago con filtros */}
      <TablaUsuarios
        usuarios={usuarios}
        onBloquear={usuario => { setUsuarioAccion(usuario); setTipoAccion('bloquear') }}
        onDesbloquear={usuario => { setUsuarioAccion(usuario); setTipoAccion('desbloquear') }}
      />

      {/* 🟡 GONZA — ModalInvitacion aparece al hacer clic en "Invitar usuario" */}
      {mostrarInvitacion && (
        <ModalInvitacion
          onEnviado={email => { setMostrarInvitacion(false); mostrarConfirmacion(email) }}
          onCerrar={() => setMostrarInvitacion(false)}
        />
      )}

      {/* 🟡 GONZA — ModalConfirmacion aparece al hacer clic en "Bloquear"/"Desbloquear" */}
      {tipoAccion && usuarioAccion && (
        <ModalConfirmacion
          titulo={tipoAccion === 'bloquear' ? 'Bloquear cuenta' : 'Desbloquear cuenta'}
          mensaje={`¿Estás seguro de ${tipoAccion} la cuenta de ${usuarioAccion.nombre}?`}
          variante={tipoAccion === 'bloquear' ? 'peligro' : 'primario'}
          onConfirmar={confirmarAccion}
          onCancelar={() => setUsuarioAccion(null)}
        />
      )}
    </div>
  )
}
```

---

## 6. 📄 PBI-14b — Actualizar README y Seed
**Responsable:** Santiago Flores Carlos

### `README.md`

```markdown
## Stack
| Capa       | Tecnología                              |
|------------|-----------------------------------------|
| Frontend   | React + Vite + TailwindCSS              |
| Auth + DB  | Supabase (Postgres + Auth + Realtime)   |
| Deploy     | Vercel                                  |
| Testing    | Vitest + Testing Library                |
```

### `scripts/seed.js`

Script que carga datos ficticios para probar la app sin datos reales.

```bash
npm run seed  # Inserta usuarios, solicitudes y edificios de demo
```

---

## 🗺️ Mapa de Archivos del Sprint 2

```
Zity-main/
│
├── src/
│   ├── types/
│   │   └── database.ts                ← 🏗️ Cortez: tipos y estructura de la BD
│   │
│   ├── hooks/
│   │   └── useUsuarios.ts             ← 📋 Santiago: consulta de usuarios con filtros
│   │
│   ├── components/
│   │   ├── BarraProgreso.tsx          ← 🪜 Cortez: barra de progreso del registro
│   │   └── admin/
│   │       ├── BadgeEstado.tsx        ← 📋 Santiago: etiqueta de colores de estado
│   │       ├── FiltrosUsuarios.tsx    ← 📋 Santiago: menús de filtro
│   │       ├── TablaUsuarios.tsx      ← 📋 Santiago: tabla de usuarios
│   │       ├── ModalConfirmacion.tsx  ← 🟡 GONZA: popup "¿estás seguro?"
│   │       └── ModalInvitacion.tsx    ← 🟡 GONZA: popup de invitar usuario
│   │
│   └── pages/
│       ├── Register.tsx               ← 🪜 Cortez: formulario de 2 pasos
│       └── admin/
│           └── Usuarios.tsx           ← 📋 Santiago + 🟡 Gonza: página de gestión
│
├── supabase/functions/
│   ├── invitaciones/index.ts          ← 🟡 GONZA: servidor de invitaciones por email
│   └── bloquear-cuenta/index.ts      ← 🟡 GONZA: servidor de bloqueo/desbloqueo
│
├── scripts/
│   └── seed.js                        ← 📄 Santiago: datos de prueba
│
└── README.md                          ← 📄 Santiago: documentación actualizada
```

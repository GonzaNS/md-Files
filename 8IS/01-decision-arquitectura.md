# Decisiones de Arquitectura: Proyecto Zity

Este documento explica en detalle la arquitectura elegida para Zity, el stack tecnológico subyacente y la justificación detrás de cada elección. El objetivo de este documento es servir como guía para defender y explicar las decisiones técnicas del proyecto ante stakeholders o evaluadores.

---

## 1. Patrón Arquitectónico: Serverless / BaaS (Backend as a Service)

Zity no utiliza una arquitectura tradicional de tres capas (Frontend ↔ Backend propio Node.js/Java ↔ Base de datos). En su lugar, utiliza un modelo **BaaS (Backend as a Service)** mediante Supabase.

### ¿Por qué esta arquitectura y no otra (ej. Node.js + Express)?
* **Velocidad de iteración (Time-to-market):** El proyecto es un MVP diseñado para desarrollarse en 16 semanas. Construir un backend propio implica programar desde cero la autenticación (JWT), reseteo de contraseñas, envío de emails y endpoints CRUD. Supabase da todo esto listo y expone una API segura automática desde Postgres.
* **Reducción de Código Boilerplate:** Nos permite enfocarnos en la lógica de negocio en lugar de la infraestructura.
* **Tiempo Real:** Se requería notificar a los residentes en tiempo real. Implementar WebSockets (Socket.io) requiere gestión compleja de conexiones; Supabase Realtime lo incluye de forma nativa.

---

## 2. El Stack Tecnológico: Justificaciones e Innovaciones (2026)

### 2.1 Frontend: React 19
* **¿Por qué React?** Es el estándar de la industria, con un ecosistema masivo y excelente soporte para SPAs (Single Page Applications).
* **Innovaciones de la v19:** 
  * **React Compiler:** Hemos dejado de usar `useMemo` y `useCallback` manualmente. El compilador de React 19 optimiza automáticamente el renderizado analizando las dependencias en tiempo de compilación. Esto reduce errores humanos y mejora el rendimiento por defecto (levanta el "suelo de rendimiento").
  * **Actions API:** Simplifica enormemente el manejo de estados asíncronos (cargas, errores, actualizaciones optimistas) sin necesidad de escribir docenas de variables `useState` para spinners de carga.

### 2.2 Herramienta de Build: Vite 8
* **¿Por qué Vite y no Webpack / Create React App?** Webpack empaqueta todo el código antes de servirlo, lo cual es lento a medida que el proyecto crece. Vite utiliza módulos ES nativos del navegador durante el desarrollo. El servidor arranca en milisegundos y el Hot Module Replacement (HMR) es instantáneo. 

### 2.3 Estilos: TailwindCSS v4
* **¿Por qué Tailwind?** Es "Utility-First". Permite construir diseños a medida y responsivos directamente en el HTML/JSX sin tener que inventar nombres de clases CSS ni saltar entre archivos.
* **Innovaciones de la v4:** Tailwind v4 (lanzado recientemente) reescribió su motor (Oxide) en Rust y C++. Ya no requiere Node.js ni el pesado archivo `tailwind.config.js`. Ahora todo se configura mediante CSS nativo (`@theme`), haciéndolo hasta 10 veces más rápido que la v3.

### 2.4 Base de Datos y Auth: Supabase (PostgreSQL)
* **¿Por qué Relacional (SQL) y no NoSQL (MongoDB)?** Zity maneja datos altamente estructurados (Usuarios pertenecen a Edificios, Solicitudes a Técnicos). PostgreSQL garantiza **integridad referencial**.
* **Seguridad (RLS):** Supabase delega la seguridad a la base de datos mediante **Row Level Security (RLS)**. La validación del token JWT y el ID del usuario (`auth.uid()`) se hace a nivel de fila en la base de datos, lo que significa que el backend es seguro independientemente de los fallos del frontend.

---

## 3. Seguridad e Incidentes Reales en la Industria

Al utilizar servicios administrados (BaaS/PaaS), es crítico entender cómo protegerse ante vulnerabilidades de la cadena de suministro o configuraciones erróneas.

### 3.1 El Incidente de Vercel (Abril 2026)
Recientemente (abril de 2026), Vercel sufrió un incidente de seguridad grave, pero es vital entender **cómo ocurrió y por qué Zity está protegido**:
* **El Ataque:** Fue un ataque de cadena de suministro. Un empleado de una herramienta de terceros (Context.ai) fue infectado con malware (Lumma Stealer), robando tokens OAuth. Los atacantes usaron esto para secuestrar la cuenta de Google Workspace de un empleado de Vercel, evadiendo el MFA (Múltiple Factor de Autenticación).
* **El Impacto:** Los atacantes lograron acceder a la red interna de Vercel y extrajeron **variables de entorno no sensibles** de algunos clientes.
* **La Defensa de Zity:** Vercel encripta las variables marcadas como "Sensibles" de tal forma que ni siquiera sus propios empleados (o atacantes en su red interna) pueden leerlas en texto plano. En Zity, llaves maestras como `SUPABASE_SERVICE_ROLE_KEY` o tokens de base de datos **están estrictamente marcados como "Sensibles" en el panel de Vercel**, por lo que este ataque no habría comprometido nuestros datos críticos.

### 3.2 Los "Hackeos" a Proyectos de Supabase
Aunque Supabase como plataforma no ha sido vulnerada a nivel estructural, muchos proyectos que usan Supabase han sufrido exposiciones masivas de datos.
* **Causa:** Estos incidentes siempre han sido de "Capa 8" (error humano del desarrollador). Ocurren cuando:
  1. No se habilita RLS (Row Level Security), dejando la base de datos como una API pública abierta a todo el internet.
  2. El desarrollador filtra por error la llave maestra (`service_role_key`) en el código frontend de React.
* **Nuestra Mitigación:** En Zity, el RLS es mandatorio para toda tabla nueva desde el Sprint 0. Además, la `service_role_key` jamás se expone al cliente; solo se usa en las Edge Functions seguras del servidor (ej. para la función de bloquear cuentas).

---

## 4. Conclusión Estratégica

La arquitectura elegida prioriza la **velocidad de entrega, la modernidad del stack y la seguridad por diseño**. Al usar herramientas automatizadas (React Compiler, Vite, Tailwind v4) y delegar la infraestructura a expertos (Supabase, Vercel), el equipo de desarrollo puede centrarse en el valor del negocio: el flujo de trabajo de mantenimiento del edificio, asegurando al mismo tiempo que los datos están protegidos contra vectores de ataque modernos.

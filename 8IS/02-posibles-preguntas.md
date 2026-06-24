# Posibles Preguntas y Respuestas para la Defensa del Proyecto Zity

Este documento contiene un listado de las preguntas técnicas más probables (y difíciles) que evaluadores, profesores o stakeholders podrían hacerte sobre la arquitectura, código y decisiones de Zity. Se incluyen las respuestas recomendadas para defender sólidamente tu trabajo.

---

## 1. Arquitectura y Backend (Supabase)

### Pregunta 1: "¿Por qué elegiste un BaaS como Supabase en lugar de programar tu propia API y servidor en Node.js, Python o Java?"
**Tu respuesta:** 
"Fue una decisión puramente de eficiencia y *time-to-market*. Nuestro objetivo principal en 16 semanas era construir un flujo de valor funcional para residentes y administradores, no reinventar la rueda construyendo sistemas de login o tokens JWT desde cero. Supabase nos entrega PostgreSQL, Autenticación y Storage integrados. Si hubiéramos hecho un backend propio en Node.js, habríamos tardado al menos 3 a 4 sprints solo en tener un CRUD básico y un sistema de login seguro. Con Supabase, logramos eso en el Sprint 1."

### Pregunta 2: "Como usas Supabase, la lógica está en el frontend de React. ¿No es eso una falla de seguridad masiva? ¿Cualquier usuario puede abrir la consola de Chrome y borrar la base de datos entera?"
**Tu respuesta:**
"No, porque la seguridad no reside en React, sino directamente en PostgreSQL a través de **RLS (Row Level Security)**. La clave que tenemos en el frontend (la `anon_key`) solo permite conectarse a Supabase, pero **no da permisos por sí sola**. 
Cuando un usuario intenta borrar o leer una solicitud, Supabase lee su token JWT. Si el usuario intenta ver una solicitud de otro departamento, la política RLS en la base de datos evalúa `auth.uid() = residente_id` y rechaza la consulta devolviendo un arreglo vacío. Es imposible saltarse esta protección desde el cliente."

---

## 2. Decisiones de Frontend

### Pregunta 3: "¿Por qué decidieron usar Vite en lugar de herramientas clásicas como Create React App (Webpack) o usar un framework completo como Next.js?"
**Tu respuesta:**
*   **Vite vs Create React App:** Vite es inmensamente superior porque utiliza los módulos de ECMAScript (ES modules) nativos del navegador durante el desarrollo. Webpack tiene que empaquetar todo el proyecto cada vez que guardas un archivo, lo que se vuelve muy lento. Vite arranca en milisegundos y el Hot Module Replacement (HMR) es instantáneo, ahorrando horas de tiempo de desarrollo.
*   **Vite vs Next.js:** Next.js es excelente, pero está pensado para aplicaciones con Server-Side Rendering (SSR) o SEO complejo (como blogs o e-commerce). Zity es un "Dashboard" privado que requiere inicio de sesión obligatorio; no necesitamos SEO público. Por lo tanto, una SPA (Single Page Application) clásica con Vite y React Router era la herramienta más directa y con menos fricción para nuestro caso de uso.

### Pregunta 4: "Veo que usan TailwindCSS v4. Hay mucha crítica de que Tailwind 'ensucia' el HTML y rompe la separación de responsabilidades. ¿Por qué lo eligieron?"
**Tu respuesta:**
"Elegimos Tailwind porque el enfoque de 'Utility-First' acelera drásticamente la maquetación. Es cierto que el JSX se llena de clases, pero el concepto de separación de responsabilidades en React ocurre **a nivel de Componente**, no de archivo de estilos. En React, un Botón es su propio archivo, por lo que las clases de Tailwind están encapsuladas ahí y no se repiten. Además, elegimos la recién lanzada **v4 con el motor Oxide**, que eliminó la dependencia de Node.js en la configuración de estilos, usando CSS nativo, lo que hizo nuestros builds 10 veces más rápidos."

### Pregunta 5: "En aplicaciones React grandes, los renders innecesarios son un problema grave. ¿Cómo manejaron el rendimiento en Zity?"
**Tu respuesta:**
"Gracias a que construimos sobre **React 19**, nuestro paradigma de rendimiento cambió. En versiones anteriores hubiésemos llenado el código de `useMemo` y `useCallback` manualmente para evitar renders innecesarios. En React 19, utilizamos el **React Compiler**, que automatiza la optimización y memoización analizando el código durante el proceso de build. Esto nos dio un rendimiento excelente por defecto (levantó el 'suelo de rendimiento') sin complicar la lectura del código."

---

## 3. Pruebas (Testing) y Despliegue

### Pregunta 6: "¿Por qué usar dos herramientas diferentes para testing (Vitest y Playwright)? ¿No bastaba con una?"
**Tu respuesta:**
"Tienen propósitos completamente distintos que se complementan en nuestra pirámide de pruebas:
*   **Vitest:** Lo usamos para pruebas unitarias. Prueba piezas lógicas aisladas del código muy rápido (ej. 'si el estado cambia a cerrada, verificar que la fecha de actualización se registre').
*   **Playwright:** Es para pruebas End-to-End (E2E). Levanta un navegador Chromium real simulando a un usuario real que hace clic, sube una foto y espera una respuesta. Nos asegura que todos los sistemas (Vercel, Frontend, Supabase, Base de Datos) están funcionando bien en conjunto. Una prueba unitaria no puede detectar si se cayó el servidor, Playwright sí."

### Pregunta 7: "Hubo un incidente importante en Vercel hace poco (Abril 2026) que vulneró proyectos. ¿Por qué Zity sigue alojado ahí y cómo nos aseguras que los datos están a salvo?"
**Tu respuesta:**
"Sí, estamos al tanto. Para explicarlo de forma muy sencilla: imagina que Vercel es un banco. Los hackers no rompieron la seguridad principal del banco, sino que engañaron a un proveedor externo para robarle llaves de acceso (como si le robaran las llaves al personal de limpieza).
Una vez adentro, solo pudieron llevarse cosas sin importancia (lo que llamamos 'variables no sensibles'). Vercel tiene una caja fuerte virtual con un candado irrompible para las contraseñas de verdad (las variables 'Sensibles'). En Zity, nuestras llaves maestras de la base de datos están estrictamente guardadas en esa caja fuerte. Por eso, aunque los hackers entraron, jamás pudieron leer nuestras contraseñas. Además, Vercel solo guarda el código de la pantalla; nuestra base de datos completa con los datos de residentes vive en otro lado (Supabase), totalmente aislada y a salvo."

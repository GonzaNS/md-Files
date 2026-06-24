# Decisiones de Arquitectura y Stack Tecnológico: Proyecto Zity

Este documento explica en detalle **todas las tecnologías** y decisiones arquitectónicas que componen el proyecto Zity. El objetivo de este documento es servir como guía exhaustiva para defender y explicar las decisiones técnicas (qué es cada cosa y por qué se eligió frente a otras alternativas) ante stakeholders o evaluadores.

---

## 1. Patrón Arquitectónico Global: Serverless / BaaS (Backend as a Service)

Zity no utiliza una arquitectura tradicional de servidor (Frontend ↔ Backend propio en Node.js/Java ↔ Base de datos). En su lugar, utiliza un modelo **BaaS**.

* **¿Qué es?** Es un modelo donde la infraestructura del backend, las bases de datos y la autenticación son gestionados por un proveedor de la nube de terceros (en este caso, Supabase).
* **¿Por qué se eligió?** Velocidad de iteración (Time-to-market). Zity es un MVP diseñado para desarrollarse en 16 semanas (sprints). Construir un backend propio implica programar desde cero la autenticación (JWT), reseteo de contraseñas, envío de emails y todos los endpoints (CRUD). Este patrón nos permite omitir el "código repetitivo" (boilerplate) y enfocarnos exclusivamente en la lógica de negocio del mantenimiento de edificios.

---

## 2. Frontend y Ecosistema Cliente

### 2.1 TypeScript (v6)
* **¿Qué es?** Es un superconjunto de JavaScript que añade tipado estático opcional al lenguaje.
* **¿Por qué se eligió?** JavaScript puro es propenso a errores en tiempo de ejecución. Al usar TypeScript, los errores (como intentar leer propiedades de una "solicitud" que no existe) se detectan en el editor de código antes de correr la aplicación. Asegura que el frontend se comunique perfectamente con los tipos de datos de la base de datos de Supabase.

### 2.2 React (v19)
* **¿Qué es?** Una librería de JavaScript para construir interfaces de usuario interactivas basadas en componentes reutilizables.
* **¿Por qué se eligió?** Es el estándar de la industria, garantizando soporte a largo plazo. 
* **Innovaciones recientes:** En la versión 19 introduce el **React Compiler**, que automatiza la optimización del rendimiento (ya no hay que usar funciones complejas como `useMemo` a mano), y la **Actions API** que simplifica drásticamente el manejo de carga de datos y formularios asíncronos.

### 2.3 Vite (v8)
* **¿Qué es?** Una herramienta de construcción (build tool) y servidor de desarrollo local para proyectos frontend. Reemplaza al antiguo "Create React App" o Webpack.
* **¿Por qué se eligió?** Webpack empaqueta todo el código antes de mostrarlo, haciéndose lento a medida que el proyecto crece. Vite utiliza los módulos ES (ECMAScript) nativos del navegador. Esto significa que el servidor de desarrollo arranca en milisegundos y los cambios en el código se reflejan instantáneamente (Hot Module Replacement instantáneo).

### 2.4 TailwindCSS (v4)
* **¿Qué es?** Un framework CSS "Utility-First" que proporciona clases de bajo nivel (ej. `flex`, `pt-4`, `text-center`) para construir diseños directamente en el HTML/JSX.
* **¿Por qué se eligió?** Permite maquetar interfaces extremadamente rápido sin la fricción de tener que inventar nombres de clases semánticas ni crear cientos de archivos `.css` separados.
* **Innovaciones recientes:** La recién lanzada versión 4 ha reescrito su motor (llamado Oxide) en Rust/C++. Ya no necesita Node.js ni el pesado archivo de configuración `tailwind.config.js`. Todo se maneja a través de CSS nativo (`@theme`), haciéndolo hasta 10 veces más rápido que la v3.

### 2.5 React Router (v7)
* **¿Qué es?** La librería estándar de enrutamiento para React. Permite navegar entre diferentes páginas de la aplicación (`/login`, `/residente`, `/admin`) sin recargar el navegador.
* **¿Por qué se eligió?** Proporciona herramientas nativas para proteger rutas según el rol del usuario (guardas de ruta) y evita accesos no autorizados a paneles administrativos.

---

## 3. Backend, Datos y Autenticación

### 3.1 Supabase
* **¿Qué es?** Una plataforma BaaS de código abierto basada enteramente en PostgreSQL. Es el núcleo del backend de Zity.
* **¿Por qué se eligió?** Proporciona toda la infraestructura necesaria en un solo paquete:
  * **PostgreSQL:** Base de datos relacional. Elegida sobre NoSQL (como MongoDB) porque los datos de condominios son altamente relacionales (Usuarios -> Edificios -> Solicitudes) y necesitamos **integridad referencial**.
  * **Supabase Auth:** Gestiona el registro, verificación de emails, inicio de sesión y perfiles de usuario.
  * **Supabase Storage:** Un sistema de almacenamiento de archivos (buckets) configurado para alojar las fotos de las solicitudes de mantenimiento.
  * **Supabase Realtime:** Permite que las notificaciones y los cambios de estado en las solicitudes aparezcan en la pantalla del usuario en tiempo real sin recargar la página.
  * **Edge Functions:** Pequeños scripts (escritos en Deno) que corren en servidores globales para realizar tareas de alta seguridad que no pueden hacerse en el frontend (ej. bloquear cuentas de usuarios forzosamente).

---

## 4. Despliegue y DevOps (CI/CD)

### 4.1 Vercel
* **¿Qué es?** Una plataforma en la nube (PaaS) diseñada para hospedar frameworks frontend y sitios estáticos. Es donde se aloja la página web de Zity para que cualquier persona pueda acceder desde internet.
* **¿Por qué se eligió?** Está diseñado específicamente para React/Vite. Ofrece características esenciales:
  * **Edge Network (CDN Global):** Distribuye la app en servidores alrededor del mundo, cargando rápido sin importar de dónde se acceda.
  * **Preview Deployments:** Cada vez que hacemos una rama nueva de código, Vercel genera un enlace temporal único para probar los cambios antes de mandarlos a producción.
  * **Despliegue sin fricción:** Se conecta a GitHub y se despliega automáticamente con cada nuevo "commit".

### 4.2 GitHub Actions
* **¿Qué es?** Plataforma de Integración Continua y Despliegue Continuo (CI/CD) integrada en GitHub.
* **¿Por qué se eligió?** Actúa como nuestra "barrera de calidad". Se configuró para que, antes de que cualquier código nuevo llegue a Vercel, pase por procesos automáticos que revisan si hay errores de sintaxis (ESLint) o si algún test falló. Si algo falla, GitHub Actions bloquea el despliegue.

---

## 5. Calidad y Testing (Aseguramiento)

### 5.1 Vitest
* **¿Qué es?** Un framework para realizar pruebas unitarias y de integración, muy similar a Jest pero diseñado específicamente para funcionar con Vite.
* **¿Por qué se eligió?** Es inmensamente más rápido que Jest porque usa la misma cadena de construcción que nuestro frontend (Vite), lo que ahorra tiempo de cómputo en nuestro pipeline de CI/CD. Se usa para probar la lógica core del sistema (ej. calcular métricas financieras o estados de solicitudes).

### 5.2 Playwright
* **¿Qué es?** Una herramienta para realizar pruebas End-to-End (E2E).
* **¿Por qué se eligió?** A diferencia de las pruebas unitarias, Playwright levanta navegadores reales (Chrome, Safari) y simula el comportamiento humano real (hacer clics, llenar formularios). Lo elegimos para automatizar la verificación de los flujos críticos (ej. "Un residente inicia sesión, crea una solicitud y le aparece al admin").

---

## 6. Seguridad e Incidentes Reales en la Industria

Al utilizar servicios administrados (BaaS/PaaS) como Vercel y Supabase, es crítico entender cómo protegerse ante vulnerabilidades modernas.

### 6.1 El Incidente de Vercel (Abril 2026)
* **¿Qué pasó?** Vercel sufrió un incidente real debido a un ataque de *cadena de suministro*. Empleados de una herramienta de terceros (Context.ai) fueron infectados con malware (Lumma Stealer), robando tokens y permitiendo a los atacantes secuestrar una cuenta interna de Vercel, evadiendo múltiples factores de autenticación (MFA).
* **El Impacto:** Los atacantes lograron entrar a la red interna y robar **variables de entorno no sensibles** de algunos clientes.
* **La Defensa de Zity:** Vercel posee un sistema de encriptación que protege las variables marcadas explícitamente como "Sensibles" (Sensitive). Ni siquiera los empleados de Vercel o los atacantes infiltrados pudieron desencriptarlas. En Zity, hemos mitigado este riesgo asegurando que todas las credenciales maestras (como la `SUPABASE_SERVICE_ROLE_KEY`) estén marcadas estrictamente como Sensibles, por lo que nuestros datos críticos nunca estuvieron en riesgo ante esta brecha.

### 6.2 Los "Hackeos" a Proyectos de Supabase
* **¿Qué pasó?** Aunque la plataforma de Supabase no ha sufrido hackeos estructurales que extraigan información desde el núcleo, la prensa ha reportado decenas de exposiciones masivas de datos en startups que usan Supabase.
* **La Causa (Capa 8 / Error humano):** Estos incidentes ocurren exclusivamente cuando los desarrolladores cometen dos errores fatales:
  1. Dejar apagado el RLS (Row Level Security), lo que expone toda la base de datos a internet de forma pública a través de la API REST generada.
  2. Filtrar o incluir por error la llave maestra (`service_role_key`) en el código frontend (React/Vite), regalándole la base de datos a cualquier usuario que inspeccione la web.
* **La Defensa de Zity:** Nuestro diseño requiere RLS mandatorio para absolutamente toda tabla nueva desde el Sprint 0. Toda solicitud que llega al backend intercepta el token criptográfico del usuario y solo devuelve la información que le corresponde. Además, la `service_role_key` jamás se empaqueta en el frontend; se ejecuta únicamente en entornos de servidor seguros (Edge Functions).

---

## 7. Conclusión Estratégica

La arquitectura elegida para Zity prioriza la **velocidad de entrega, la modernidad del stack y la seguridad por diseño**. Al usar herramientas avanzadas (React Compiler, Vite, Tailwind v4 Oxide) y delegar la infraestructura a expertos de nivel empresarial (Supabase, Vercel), el equipo de desarrollo puede centrarse puramente en entregar valor de negocio y nuevas funcionalidades a los residentes y la administración, manteniendo los más altos estándares de resiliencia frente a ataques cibernéticos modernos.

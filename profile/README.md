# Manual Maestro de Contexto: Proyecto Evento Institucional PWA

## 1. Contexto de Negocio y Audiencia
- **Evento:** Encuentro presencial de las Primeras Damas de los 32 departamentos de Colombia.
- **Objetivo:** Aplicación PWA interactiva para registrar las memorias del evento (fotografías y videos) en tiempo real.
- **Audiencia:** Altas dignatarias y sus equipos técnicos. La estabilidad, seguridad, privacidad de datos y una experiencia de usuario (UI/UX) impecable y fluida son críticas.

## 2. Infraestructura y Entorno de Despliegue
- **Alojamiento:** Servidor físico local ubicado en las instalaciones de la **Casa del Libro Total**.
- **Base de Datos:** PostgreSQL corriendo de forma local (`base-datos-evento`).
- **Condición de Red:** El ancho de banda de salida a internet en el recinto puede presentar fluctuaciones. Se debe priorizar el tráfico en red local (LAN) y garantizar resiliencia offline mediante Service Workers.

## 3. Arquitectura del Proyecto (Repositorios Separados)
El proyecto está dividido en componentes independientes distribuidos en las carpetas adyacentes:
1. `evento-pwa-api/`: Servidor backend construido con **NestJS** y **TypeScript**. Se encarga de la lógica de negocio, autenticación, validación de datos y persistencia en Postgres.
2. `evento-pwa-web/`: Aplicación frontend PWA construida con **Next.js (App Router)**, **React** y **ECMAScript/TypeScript**. Encargada de la interfaz, compresión de medios en el cliente y sincronización offline.
3. `evento-pwa-worker/`: Microservicio en segundo plano construido con **NestJS** para manejar procesos asíncronos y WebSockets (**Socket.io**) para la actualización de métricas en tiempo real.

---

## 3. Matriz de Requisitos Oficiales (Single Source of Truth)

### Requisitos Funcionales (RF)
- **RF-01: Landing:** Mostrar página principal institucional con acceso a los módulos.
- **RF-02: Autenticación:** Permitir inicio de sesión de usuarios autorizados mediante código OTP al correo. El backend debe emitir un token JWT. Sesión extendida para el evento.
- **RF-03: Feed:** Listar publicaciones (foto/video + texto + autor + fecha). Debe mostrar dinámicamente sus reacciones y comentarios.
- **RF-04: Captura/Publicación:** Permitir subir imagen/video con descripción opcional. Mostrar loader durante el envío y un toast con el resultado.
- **RF-05: Reacciones:** Permitir registrar reacciones por publicación (`LIKE`, `APPLAUSE`, `HEART`). Restricción: Máximo una de cada tipo por usuario por publicación (sistema Toggle). Restricción `UNIQUE` compuesta en Postgres. Requiere **Optimistic UI** en el frontend.
- **RF-06: Comentarios:** Permitir agregar comentarios de texto en las publicaciones (Rutas protegidas).
- **RF-07: Memoria Digital:** Sección de consolidación de mejores momentos. Permitir exportación posterior (Fase 2: PDF/HTML).
- **RF-08: PWA:** Aplicación instalable desde navegadores compatibles. Funcionar con caché básico para navegación y lectura reciente (Offline-ready).

### Requisitos No Funcionales (RNF)
- **RNF-01: Usability (Usabilidad):** Interfaz mobile-first, sumamente clara para usuarios no técnicos (Primeras Damas). Cumplir accesibilidad mínima (contraste, tamaño de texto legible, labels correctos).
- **RNF-02: Performance (Rendimiento):** Carga inicial objetivo < 3 segundos en red 4G promedio. Exige compresión estricta de imágenes/videos en el cliente (`evento-pwa-web`) antes de subirlos.
- **RNF-03: Security (Seguridad):** Uso estricto de JWT para rutas protegidas. Validación exhaustiva de entradas en la API mediante DTOs y `class-validator`. Validación estricta en backend de tipos de archivo (MIME types) y tamaños máximos permitidos.
- **RNF-04: Availability (Disponibilidad):** Operación 100% estable y redundante durante las ventanas de ejecución del evento presencial.
- **RNF-05: Maintainability (Mantenibilidad):** Separación estricta de responsabilidades entre los 3 repositorios (web/api/worker).


### Flujo de Comentarios
- Las publicaciones de fotos y videos permiten comentarios de texto. 
- Debido al perfil institucional de las invitadas, los endpoints de comentarios deben requerir autenticación estricta.

### Autenticación del Evento
- **Mecanismo:** Envío de código de verificación (OTP) de un solo uso al correo electrónico institucional previamente registrado en la base de datos.
- **Persistencia de Sesión:** Duración de sesión JWT extendida para asegurar que las invitadas no tengan que reautenticarse durante los días del evento.
- **Contingencia:** El sistema debe contar con un bypass o rol de administrador para soporte técnico presencial en caso de fallos con los servidores de correo institucionales de las gobernaciones.

---

## 4. Instrucciones de Operación para el Agente (Claude)
- **Foco de Trabajo:** Cuando se trabaje dentro de una subcarpeta específica, lee el archivo `agents.md` local de ese repositorio para aplicar las reglas de sintaxis y dependencias correspondientes (NestJS, Next.js, etc.).
- **Validación de Datos:** Aplica tipado estricto extremo a extremo aprovechando que el stack completo se apoya en TypeScript.
- **Optimización de Recursos:** Toda propuesta de código que involucre subida de archivos debe incluir compresión previa en el cliente para proteger la red local del evento.

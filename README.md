Post-mortem: el error de CORS que casi frena mi CV ATS Optimizer

Contexto:

Como proyecto final del módulo de IA aplicada de mi diplomatura en Desarrollo Full Stack (Coderhouse), desarrollé CV ATS Optimizer, una herramienta web que ayuda a optimizar currículums para que superen los filtros ATS (Applicant Tracking System) que usan muchas empresas antes de que un CV llegue a manos humanas.

La arquitectura del proyecto es la siguiente:

Un formulario HTML donde el usuario carga los datos de su CV y la descripción del puesto al que aplica.
Un webhook de Make (antes Integromate) que recibe esos datos y orquesta el flujo.
Una llamada a Google Gemini (gemini-2.5-flash-lite) que analiza y reescribe el contenido optimizado.
Generación del CV final en PDF usando jsPDF, descargable directamente desde el navegador.

Todo el flujo corre sin backend propio: el "servidor" es el escenario de Make, y el frontend es un archivo estático.

Problema:
Durante las pruebas integrales del flujo me encontré con dos problemas técnicos encadenados:

Errores de filtrado en el módulo Router de Make: al intentar decidir qué rama del escenario ejecutar según el tipo de análisis solicitado, el Router fallaba con errores de ID de módulo de filtro, y las variables de un módulo no llegaban correctamente a los módulos posteriores.
Bloqueo por CORS: una vez resuelto el enrutamiento interno, al testear el formulario abriendo el HTML directamente desde el sistema de archivos (file://), el navegador bloqueaba la petición al webhook con un error de política de CORS, impidiendo que la respuesta de Gemini llegara al frontend.

Este segundo problema era el más crítico, porque no era un error de lógica sino de entorno: funcionaba en algunos contextos y en otros no, lo que generaba confusión sobre si el problema estaba en Make, en Gemini o en el propio navegador.
Acciones
Paso 1 — Resolver el enrutamiento en Make. Diagnostiqué que el error de filtro del Router se debía a que el módulo no tenía un ID de filtro válido asignado al momento de duplicar ramas del escenario. Lo solucioné reconstruyendo las condiciones del filtro desde cero en lugar de duplicar el módulo existente.

Paso 2 — Resolver la propagación de contexto entre módulos. Para que los datos generados en un módulo temprano (por ejemplo, el texto ya procesado por Gemini) estuvieran disponibles en módulos posteriores del escenario, incorporé un módulo intermedio de "Set Multiple Variables". Esto centralizó el contexto en un solo punto en lugar de depender de referencias directas entre módulos, que se rompían cada vez que reordenaba el flujo.

Paso 3 — Diagnosticar el problema de CORS (post-mortem constructivo). Acá es donde vino el aprendizaje más importante. Mi primer instinto fue asumir que el webhook de Make estaba mal configurado. Pero al revisar la consola del navegador con más atención, identifiqué que el error solo aparecía cuando el HTML se abría con el protocolo file://, no cuando se serviría desde un dominio real (http:// o https://). Es decir: el problema no era el webhook, era el entorno de pruebas que yo mismo había elegido por comodidad.

Si tuviera que hacer una autocrítica honesta: perdí tiempo buscando el error del lado equivocado (backend) cuando la causa raíz estaba en cómo estaba testeando (frontend/entorno). La lección de proceso es clara: antes de asumir que el problema está en la integración externa, hay que descartar el entorno local como variable.

Paso 4 — Definir la solución definitiva. La solución no es un parche, sino un cambio de entorno: desplegar el HTML en GitHub Pages o Netlify para que se sirva desde un origen HTTP real, y agregar el header Access-Control-Allow-Origin: * en la respuesta del webhook de Make. Esta es la tarea que estoy terminando de implementar antes de la entrega final de julio.

Aprendizajes:

CORS no es un error de "todo o nada": puede aparecer solo en ciertos entornos de prueba (como file://) y desaparecer en producción, lo que puede llevar a diagnósticos equivocados si no se aísla bien la variable de entorno.
Centralizar el contexto entre módulos (con un paso explícito tipo "Set Multiple Variables") es más mantenible que encadenar referencias directas entre módulos en herramientas de automatización no-code como Make.
Reconstruir en lugar de duplicar módulos con lógica de filtrado evita arrastrar configuraciones corruptas o IDs inválidos.
Testear en el entorno más parecido posible al de producción (un servidor HTTP real, aunque sea local) desde etapas tempranas ahorra ciclos de debugging innecesarios.

Control de versiones:

Repositorio del proyecto: (agregar acá el link a tu repo de GitHub, por ejemplo https://github.com/rodrivalentini/cv-ats-optimizer)
Commit con la corrección del Router y el módulo Set Multiple Variables: (pegar link al commit específico)
Commit o PR con el despliegue a GitHub Pages/Netlify y el ajuste de headers CORS en el webhook: (pegar link al commit o PR)

Nota: reemplazá los placeholders de arriba por los links reales antes de publicar. Si el repo es nuevo, un solo commit inicial ya es evidencia válida de control de versiones, pero lo ideal es mostrar al menos 2-3 commits que reflejen el antes/después del fix.

Reflexión sobre feedback radicalmente sincero:

Durante este proceso, la parte más valiosa del feedback sincero no vino de otra persona, sino de ser honesto conmigo mismo sobre dónde estaba realmente el error. Fue más cómodo, al principio, pensar "el webhook está mal" que aceptar que mi metodología de testeo (abrir el archivo local) era la causa. Aplicar un enfoque de feedback radicalmente sincero significó cuestionar mi propio diagnóstico inicial en lugar de defenderlo, revisar la evidencia (la consola del navegador) sin filtro, y corregir el rumbo aunque implicara reconocer que había estado buscando en el lugar equivocado. Ese cambio de perspectiva fue lo que finalmente destrabó la solución.



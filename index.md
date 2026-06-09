Cuando la base de datos se cayó en producción: anatomía de un post-mortem que cambió a mi equipo

Un deploy de rutina, una migración mal validada y 52 minutos de downtime. Esta es la historia de cómo convertimos el peor martes del trimestre en el mejor aprendizaje del año.


Contexto
Trabajo como desarrollador backend en un equipo distribuido de 6 personas que mantiene la plataforma de e-commerce de una empresa mediana. Operamos con despliegues continuos: varias veces por semana, el código que pasa los tests se despliega automáticamente a producción.
El entorno es el típico de un equipo ágil y remoto: trabajamos de manera asincrónica, en distintas zonas horarias, coordinándonos por Slack y revisando código mediante Pull Requests en GitHub. La velocidad es una ventaja, pero también significa que un error puede llegar a producción rápido.
Este post documenta un incidente real que vivimos, el análisis post-mortem que hicimos, cómo usamos el control de versiones durante la resolución y qué aprendí sobre comunicación técnica y feedback en el proceso.

Problema
Era un martes a las 14:10. Hicimos merge de un Pull Request que incluía una migración de base de datos: agregaba una columna nueva a la tabla de pedidos y renombraba otra. El pipeline de CI pasó todos los tests en verde. El deploy automático se ejecutó.
A las 14:18, las alertas de monitoreo se dispararon. El endpoint de checkout devolvía errores 500 de forma masiva. Los usuarios no podían completar sus compras.
El problema técnico de fondo: la migración renombró una columna que el código de la versión anterior todavía estaba usando. Durante la ventana entre que la migración corrió y que todos los servidores tomaron el código nuevo, la aplicación buscaba una columna que ya no existía con ese nombre. El resultado fue un downtime de 52 minutos en la función más crítica del negocio: el pago.
Pero el problema real no fue solo técnico. Fue que:

No teníamos un proceso definido para migraciones que renombran o eliminan columnas.
Durante el incidente, la comunicación fue caótica: varias personas investigando en paralelo sin coordinación.
Nadie sabía quién estaba "al mando" del incidente.


Acciones
Una vez detectado el incidente, seguimos (improvisando sobre la marcha, hay que admitirlo) estos pasos. Después los formalizamos en un protocolo.
1. Contención inmediata
La primera decisión fue hacer rollback en lugar de intentar arreglar hacia adelante. Revertimos al commit anterior estable usando control de versiones:
bash# Identificamos el último commit estable
git log --oneline -5

# Revertimos el merge problemático
git revert -m 1 <hash-del-merge>

# Push que dispara el deploy de la versión estable
git push origin main
A las 14:45 el rollback estaba en producción. A las 15:02, el monitoreo confirmó error rate en 0%. Servicio restaurado.
2. El post-mortem constructivo
Acá es donde aplicamos lo aprendido sobre post-mortems blameless (sin culpables). En lugar de preguntar "¿quién rompió producción?", preguntamos "¿qué falló en nuestro sistema que permitió que esto pasara?".
Seguimos la estructura de cuatro partes:
Descripción objetiva del incidente:
El deploy de la versión 2.4.1 incluyó una migración que renombró la columna order_status a status. El código de la versión 2.4.0, todavía corriendo en algunos servidores durante el despliegue gradual, seguía consultando order_status, causando errores 500 en el checkout durante 52 minutos.
Análisis de causas (los 5 porqués):
PreguntaRespuesta¿Por qué falló el checkout?El código buscaba una columna que había sido renombrada¿Por qué buscaba una columna inexistente?El renombrado no fue retrocompatible con la versión anterior¿Por qué no fue retrocompatible?No teníamos una política de migraciones en dos fases¿Por qué no había política?Nunca habíamos tenido un incidente así, "siempre funcionó"¿Por qué nunca lo previmos?No existía un checklist de revisión para migraciones de riesgo
Causa raíz: ausencia de un proceso estandarizado para migraciones de base de datos que alteran el esquema de forma no retrocompatible.
Acciones correctivas y preventivas:

 Implementar política de migraciones en dos fases (expand/contract): primero agregar lo nuevo manteniendo lo viejo, deployar, y recién después eliminar lo viejo. Responsable: equipo backend · Plazo: 1 semana
 Agregar un checklist obligatorio en el template de PR para migraciones de riesgo. Responsable: tech lead · Plazo: 3 días
 Configurar una alerta temprana de error rate que notifique en el primer minuto, no en el octavo. Responsable: DevOps · Plazo: 1 semana

3. Documentación del flujo con control de versiones
Todo el proceso quedó registrado en el control de versiones, lo que nos dio trazabilidad completa:

El PR original con la migración problemática.
El commit de revert que contuvo el incidente.
Un nuevo PR con la migración corregida usando el patrón expand/contract.
El post-mortem documentado en el repositorio, en /postmortems/2025-06-15-checkout-down.md.

Este es justamente el valor del control de versiones: cada decisión quedó registrada, fechada y atribuida, y cualquier persona del equipo (presente o futura) puede entender exactamente qué pasó y por qué.

Aprendizajes
Lo técnico
La lección técnica fue clara: las migraciones de base de datos que alteran el esquema deben ser retrocompatibles. Adoptamos el patrón expand/contract, que separa el cambio en fases seguras. Nunca más renombramos una columna en un solo paso.
Lo humano: comunicación y feedback
Pero el aprendizaje más profundo no fue técnico. Fue sobre cómo nos comunicamos bajo presión.
Durante el incidente, el caos de comunicación nos costó tiempo. Aprendimos a:

Abrir siempre un canal dedicado al incidente y nombrar un coordinador.
Dar actualizaciones cada 15 minutos, aunque no haya novedades. El silencio genera más ansiedad que una mala noticia.
Comunicar a las áreas no técnicas en su lenguaje: no "renombramos una columna", sino "el sistema de pagos estuvo caído 52 minutos y ya está resuelto".

Reflexión sobre el feedback radicalmente sincero
Cuando hicimos la retrospectiva, tuve que dar un feedback incómodo: el PR con la migración lo había revisado yo, y lo aprobé sin notar el problema de retrocompatibilidad. Era tentador quedarme callado.
Pero el concepto de feedback radicalmente sincero (Radical Candor) propone una intersección entre importarte la persona y ser directo. Lo apliqué conmigo mismo primero: dije abiertamente en la retro que había aprobado el PR sin la diligencia necesaria, no para culparme, sino para que el equipo entendiera que el problema no era de una persona, sino de un proceso de review que no incluía verificar retrocompatibilidad.
Esa sinceridad cambió la conversación. En lugar de buscar culpables, el equipo se enfocó en mejorar el proceso de revisión. La autora de la migración, que al principio estaba a la defensiva, se relajó cuando vio que nadie la estaba señalando. Salimos de esa retro con un checklist nuevo y, sobre todo, con más confianza entre nosotros.
Esa es la conexión que entendí en este curso: la mentalidad de crecimiento (ver el error como aprendizaje) y la comunicación efectiva (decir las cosas difíciles con cuidado y claridad) no son dos habilidades separadas. Son la misma cosa. Un equipo que comunica bien aprende rápido. Un equipo que comunica mal esconde sus errores hasta que explotan.

Cierre
Aquel martes perdimos 52 minutos de ventas. Pero ganamos un protocolo de migraciones, un checklist de review, un sistema de comunicación de incidentes y, lo más valioso, un equipo que ya no le tiene miedo a hablar de sus errores.
El error no fue el problema. El error fue el maestro.

Este artículo fue escrito como entregable final de la diplomatura en Mentalidad de Crecimiento y Comunicación en Entornos Digitales. El repositorio con el código, los commits y el post-mortem completo está disponible en el enlace del repositorio.

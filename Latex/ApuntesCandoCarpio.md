Apuntes de la revisión técnica recibida

Después de revisar las observaciones realizadas por Farinango y Urbina a nuestro proyecto de Soporte Técnico ISP, entendimos que nuestro sistema tiene una arquitectura distribuida que posee varias decisiones técnicas correctas, pero también existen puntos importantes que debemos mejorar. La revisión no se enfocó solamente en encontrar errores de funcionamiento, sino en analizar la calidad del código, la organización de las responsabilidades, la comunicación entre los microservicios, la seguridad y la capacidad que tiene el sistema para registrar y manejar los errores que puedan ocurrir.

Uno de los primeros aspectos que comprendimos es que nuestro proyecto sí presenta características propias de un sistema distribuido. Tenemos diferentes servicios que cumplen funciones específicas, como auth-service, svc-principal, report-service, notification-service y ai-service. También utilizamos Kafka para la comunicación mediante eventos y CockroachDB para manejar bases de datos separadas. Por esta razón, entendimos que la arquitectura general del proyecto tiene una buena base y que no estamos utilizando una sola base de datos compartida entre todos los servicios.

También comprendimos que existen decisiones que fueron consideradas positivas por los revisores. Por ejemplo, la separación de los datos por servicio fue considerada correcta. El modelo de lectura mediante CQRS también fue identificado como una buena implementación, porque report-service construye su propia información a partir de eventos y no necesita consultar directamente la base de datos del servicio principal. Igualmente, el mecanismo de reintento utilizado cuando ocurre un conflicto de concurrencia fue considerado adecuado porque el sistema realiza solamente un reintento y evita entrar en un ciclo infinito.

Sin embargo, la revisión también nos permitió identificar problemas que debemos corregir.

Responsabilidades de TicketService

Uno de los puntos más importantes que anotamos fue que la clase TicketService actualmente realiza demasiadas tareas. No solamente contiene la lógica relacionada con los tickets, sino que también participa en autorización por roles, persistencia, creación y serialización de eventos, publicación hacia Kafka y actualización de métricas.

Entendimos que esto afecta el principio de responsabilidad única de SOLID, porque una misma clase tiene diferentes motivos por los cuales podría necesitar modificaciones. Por ejemplo, si queremos cambiar solamente la manera en que se publican los eventos, actualmente tendríamos que modificar la misma clase donde también tenemos las reglas de negocio y los permisos.

Por esta razón, anotamos que debemos intentar separar esas responsabilidades. La publicación de eventos podría encontrarse en otro componente y las reglas de autorización podrían encontrarse en una política o servicio diferente. De esa manera TicketService podría concentrarse principalmente en la lógica relacionada con los tickets.

Manejo y centralización de roles

Otro aspecto que entendimos de la revisión es que actualmente las reglas correspondientes a los roles se encuentran repetidas en diferentes métodos mediante condiciones como if.

Esto significa que, si posteriormente agregamos un nuevo rol, tendríamos que revisar y modificar varios lugares diferentes del código. El problema no es solamente escribir más código, sino que podríamos modificar tres lugares y olvidarnos de un cuarto, provocando comportamientos diferentes para el mismo rol.

Por esa razón anotamos que sería mejor centralizar la matriz de permisos o crear una política de autorización que permita definir en un solo lugar qué operaciones puede realizar cada rol.

Separación entre dominio y persistencia

También entendimos que existe un acoplamiento entre nuestro modelo de dominio y la tecnología utilizada para guardar la información. La clase Ticket contiene anotaciones pertenecientes a Jakarta Persistence y funciona directamente como entidad de JPA.

Comprendimos que esto no significa que nuestro sistema deje de funcionar, pero sí produce dependencia entre la lógica del dominio y la tecnología utilizada para persistir los datos.

Como mejora anotamos que podríamos separar el modelo que representa al ticket dentro del negocio de la entidad utilizada específicamente para JPA. Esto permitiría que los cambios relacionados con la base de datos tengan menor impacto sobre la lógica principal del sistema.

Repositorios con operaciones innecesarias

También anotamos la observación realizada sobre TicketRepository. Al extender directamente JpaRepository, nuestra interfaz obtiene muchas operaciones que realmente no utilizamos.

Comprendimos que esto se relaciona con el principio de segregación de interfaces, debido a que los componentes terminan teniendo acceso a operaciones que no necesitan. Aunque este punto fue considerado de menor gravedad, entendimos que es una oportunidad para tener interfaces más específicas y controlar mejor las operaciones permitidas sobre los tickets.

Falta de un punto de entrada único

Un aspecto importante que comprendimos es que actualmente nuestro frontend conoce directamente las direcciones de varios servicios. Es decir, el navegador se comunica con diferentes puertos para autenticación, tickets y reportes.

La revisión nos hizo entender que sería conveniente tener una puerta de enlace única o API Gateway. De esta manera, el frontend tendría un solo punto de entrada y sería la puerta de enlace la encargada de dirigir las solicitudes al servicio correspondiente.

También entendimos que actualmente parte de la validación de autenticación se encuentra repetida en diferentes servicios. Con una puerta de enlace podríamos reducir este tipo de duplicación y centralizar determinadas responsabilidades transversales.

Dependencia entre la capa de servicio y los DTO

Otro punto que anotamos es que algunas clases de la capa de servicio están utilizando directamente DTO pertenecientes a la capa web.

Comprendimos que esto genera dependencia entre capas. Por ejemplo, si posteriormente modificamos un objeto utilizado para recibir información HTTP, podríamos terminar modificando también nuestra lógica de aplicación.

Como posible solución entendimos que podríamos utilizar comandos u objetos propios de la capa de aplicación y realizar la conversión desde el DTO recibido por el controlador hacia esos objetos internos.

Seguridad de secretos y contraseñas

Uno de los aspectos que consideramos más importantes fue la presencia de valores por defecto para el secreto JWT y para una cuenta administrativa.

Entendimos que, aunque estos valores hayan sido colocados pensando solamente en desarrollo, existe un riesgo porque, si las variables de entorno no están correctamente configuradas, el sistema podría iniciar utilizando esos valores conocidos.

Por esta razón anotamos que los secretos, contraseñas y datos sensibles deben proporcionarse externamente y que sería mejor impedir el inicio de la aplicación si una variable obligatoria de seguridad no se encuentra definida.

URLs escritas directamente en el frontend

También entendimos que las direcciones de los diferentes servicios no deberían encontrarse escritas directamente dentro del código JavaScript.

Esto complica el cambio entre diferentes ambientes, por ejemplo desarrollo, pruebas o producción. Por esta razón anotamos que las URLs deben manejarse mediante variables de entorno o mediante una configuración externa.

Problema crítico: comunicación con auth-service

El hallazgo que más llamó nuestra atención fue el relacionado con la comunicación entre svc-principal y auth-service.

Actualmente se realiza una llamada HTTP para validar el token, pero la revisión identificó que no existe un tiempo de espera configurado explícitamente.

Entendimos que esto puede convertirse en un problema serio si auth-service se encuentra lento o deja de responder. Las solicitudes podrían permanecer esperando y ocupar los recursos del servicio principal. Si llegan muchas solicitudes mientras auth-service está presentando problemas, el fallo podría extenderse al servicio encargado de gestionar los tickets.

Por esta razón anotamos que debemos configurar tiempos máximos de conexión y lectura y diferenciar correctamente entre un token inválido y un servicio de autenticación temporalmente no disponible.

También comprendimos la recomendación de utilizar en el futuro un patrón como Circuit Breaker, porque permitiría dejar de realizar temporalmente solicitudes hacia un servicio que ya sabemos que está presentando múltiples fallos.

Manejo de mensajes de error

Otra observación que entendimos claramente fue que no debemos devolver directamente ex.getMessage() al usuario.

El problema es que una excepción puede contener información interna del sistema, como nombres de clases, información de la base de datos o mensajes técnicos que no deberían ser visibles para el usuario.

Anotamos que los mensajes que recibe el usuario deben ser controlados y generales, mientras que la información técnica completa del error debe permanecer únicamente en los registros internos del sistema.

Logging e identificador de correlación

También comprendimos que nuestros diferentes servicios utilizan mecanismos de registro distintos. Algunos utilizan bibliotecas de logging, mientras que en otros casos se utiliza directamente console.log o console.error.

Pero el aspecto más importante fue entender la necesidad de implementar un identificador de correlación.

Como una operación puede pasar por svc-principal, auth-service, Kafka, ai-service, report-service y notification-service, necesitamos alguna forma de saber que todos esos registros corresponden a la misma operación.

Actualmente podríamos intentar relacionarlos manualmente mediante la hora o mediante el identificador del ticket, pero eso dificulta encontrar un problema.

Por lo tanto, anotamos que debemos generar un identificador cuando comienza una operación y propagarlo durante todo el recorrido entre servicios y eventos. De esta manera podríamos buscar ese identificador en los logs y reconstruir fácilmente todo lo ocurrido.

Información sensible dentro de los logs

También entendimos que debemos tener cuidado con la información que almacenamos en los registros. En algunos casos se está registrando el payload completo de los eventos, donde pueden encontrarse datos como el identificador del cliente y la descripción escrita dentro de un ticket.

Anotamos que debemos sanitizar los datos antes de escribirlos en los logs y registrar únicamente la información necesaria para diagnosticar el sistema.

Además, entendimos que existen operaciones importantes de autenticación que deberían quedar registradas como parte de una auditoría, por ejemplo intentos de inicio de sesión, reutilización de tokens o determinadas acciones administrativas.

Conclusión de nuestros apuntes

En conclusión, de la revisión realizada por Farinango y Urbina entendimos que nuestro proyecto no está mal estructurado en su totalidad. Al contrario, posee elementos positivos como la separación de datos por servicio, el uso de CQRS, Kafka y una política de reintento controlada.

Lo que principalmente debemos fortalecer es la manera en que el sistema responde cuando uno de sus componentes falla, además de mejorar la seguridad, la separación de responsabilidades y la trazabilidad de las operaciones.

De todos los aspectos revisados anotamos como prioridad solucionar primero el problema de tiempo de espera con auth-service, posteriormente eliminar los secretos y contraseñas por defecto, controlar correctamente los mensajes de error y mejorar los logs. Después debemos trabajar en aspectos estructurales como dividir responsabilidades de TicketService, centralizar los permisos de los roles, mejorar la separación entre capas y establecer una puerta de enlace única para nuestros servicios.

La revisión nos ayudó a comprender que en un sistema distribuido no es suficiente con que cada servicio funcione individualmente. También debemos pensar qué sucede cuando un servicio se demora, deja de responder, devuelve un error o cuando necesitamos seguir una operación que pasó por varios componentes. Por eso entendimos que conceptos como tolerancia a fallos, trazabilidad, seguridad y separación de responsabilidades son importantes para que nuestro sistema sea más mantenible y confiable.





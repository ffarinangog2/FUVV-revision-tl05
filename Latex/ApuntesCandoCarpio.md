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

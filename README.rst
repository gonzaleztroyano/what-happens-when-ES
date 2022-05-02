Este repositorio es la versión traducida al castellano del recurso de `Alex Waynor`_, `"What happens when..."`_. 

Alex Waynor no está relacionado con este trabajo ni ha revisado la traducción. 

¿Qué pasa cuando...?
=====================

Este repositorio es un intento de responder a la antigua pregunta "¿Qué ocurre cuando escribes *google.com* en tu la barra de direcciones de tu navegador y pulsas *Enter*?"

A excepción de las respuesta típica, intentaremos tratar de responder a esta pregunta con el mayor detalle posible. Sin dejar de lado ningún detalle. 

Este es un proceso colaborativo, ¡así que investiga sobre la sección que más te guste y ayúdanos! Hay muchísimos detalles faltantes, ¡están esperando a que los añadas! ¡Envíanos un *pull request*, por favor!

Este trabajo está licenciado bajo la licencia `Creative Commons Zero`_.

También puedes leerlo en  `简体中文`_ (chino simplificado), `日本語`_ (japonés) y `한국어`_ (koreano). También, por supuesto, la versión original de `"What happens when..."`_ en inglés. 

Tabla de contenidos
====================

.. contents::
   :backlinks: none
   :local:

La tecla "g" es presionada
----------------------------

Las siguientes secciones explican el comportamiento físico del teclado y las interrupciones del sistema operativo. 

Cuando presionas la tecla "g" el navegador recibe el evento y la función de auto-completado del navegador entra en acción. 

Dependiendo del algoritmo del navegador y si estás en una ventana de incógnito (también llamada Privada o InPrivate) o no, varias sugerencias te serán mostradas debajo de la barra de direcciones. La mayoría de estos algoritmos ordenan y priorizan los resultados basándose en el historial de búsqueda, marcadores, cookies, y consultas populares en todo Internet. 

Según continúas escribiendo "google.com" muchos bloques de código son ejecutados y las sugerencias serán mejoradas con cada tecla presionada. Incluso te acabará sugieriendo "google.com" antes de que termines de escribirlo. Es importante aclarar que las sugerencias mostradas no son únicamente las que empiezan por la letra ``g``, sino que también pueden aparecer sugerencias que el algoritmos considere como útiles, aún sin contenener la letra g. 


La tecla "enter" toca fondo
-----------------------------

Por elegir un punto de inicio, elijamos la tecla Enter del teclado tocando fondo. En este momento, un circuito específico para la tecla Enter, es cerrado (de forma directa, mecánica, o capacitiva). Esto permite que una pequeña cantidad de corriente fluya al circuito lógico del teclado, que escanea el estado del interruptor de la tecla, trata la señal recibida y la convierte a un código numérico. En este caso el número 13. 

El controlador del teclado a continuación codifica el código numérico obtenido y lo envía al ordenador. Cómo lo envia es, a día de hoy, prácticamente universal. Ya esté conectado mediante Universal Serial Bus (USB) o Bluetooth. Otro gallo cantaba en época de las conexiones PS/2 y ADB.

*En el caso de un teclado Bluetooth:*

- El circuito USB del teclado está encendido y funcionando gracias a los 5V de electricidad facilitados por el ordenador al que está conectado el USB, concretamente al pin número 1 del conector. 

- El código numérico generado es guardado por el circuito interno del teclado en un registro llamado "endpoint".

- El equipo al que está conectado el USB "pregunta" cada 10ms aproximadamente (valor mínimo declarado por el teclado), y de esta forma obtiene el código numérico guardado en el "endpoint".

- Este valor va hasta el SIE (Serial Interface Engine) del USB, que serán convertidos en uno o más paquetes USB que seguirán el protocolo de bajo nivel que gobierna la comunicación USB.

- Esos paquetes son enviados por una señal eléctrica diferencial a través de los pines D+ y D+ (los dos del centro del conector) a un máximo de 1.5Mb/s. Como dispositivo HID (*Human Digital Interface*, Interfaz Digital Humana) es siempre declarado como un "dispositivo de baja velocidad" (complimiento del protocolo USB 2.0).

- Esta señal en serie es después decodificado en el equipo USB al que está conectado el teclado, e interpretado por el driver *Human Interface Device* (HID) instalado en el equipo. El valor de la tecla es transmido a la capa de absracción hardware del sistema operativo. 

*En el caso de teclados virtuales (como en dispositivos con pantallas táctiles):*

- Cuando el usuario pone sus dedos en una pantalla capacitiva moderna, una pequeña cantidad de corriente se transmite al dedo. Esto completa un cirtuto a través del campo electrostático de la capa conductiva y crea una caída de voltaje que crea una caída de voltaje en ese punto de la pantalla. El ``controlador de la pantalla`` genera una interrupción reportando las coordenadas de la pulsación. 

- El sistema operativo móvil notifica a la aplicación que en ese momento tiene el foco de la pulsación de uno de sus elementos GUI (que es en este momento son los botones del teclado virtual). 

- El teclado virtual puede ahora notificar el evento de "tecla pulsada" de vuelta al Sistema Operativo. 

Lanzamiento de interrupciones [NO para teclados USB]
-----------------------------------------------------

El teclado envía señales a su *interrupt request line (IRQ)*, que es mapeada a un ``vector de interrupción`` (integral, *integer*) por el controlador de interrupciones. La CPU usa la ``Interrupt Descriptor Table`` (IDT) para mapear el vector de interrupción hacia las funciones (``manejadores de interrupciones``) que son proporcionados por el kernel. Cuando una interrupción es generada, la CPU busca en la IDT el vector de interrupción y ejecuta el manejador apropiado. Por lo tanto, se ingresa al kernel.
 

(En Windows) El mensaje ``WM_KEYDOWN`` es enviado a la aplicación
-------------------------------------------------------------------

El transporte de HID pasa el evento de tecla pulsada al controlador ``KBDHID.sys`` que convierte el uso de HID en un código de escaneo. En este caso, el código de escaneo es ``VK_RETURN`` (``0x0D``). El controlador ``KBDHID.sys`` interactúa con ``KBDCLASS.sys`` (controlador de clase de teclado). Este controlador es responsable de gestionar todas las entradas del teclado y del teclado numérico de manera segura. Luego llama a ``Win32K.sys`` (después de potencialmente pasar el mensaje a través de filtros de teclado de terceros que están instalados). Todo esto sucede en modo kernel.

``Win32K.sys`` determina qué ventana es la ventana activa a través de la API ``GetForegroundWindow()``. Esta API proporciona el identificador de ventana del cuadro de dirección del navegador. La "message pump" principal de Windows luego llama ``SendMessage(hWnd, WM_KEYDOWN, VK_RETURN, lParam)``. ``lParam`` es una máscara de bits que indica más información sobre la pulsación de tecla: número de repeticiones (0 en este caso), el código de escaneo real (puede depender del OEM, pero generalmente no sería para ``VK_RETURN``), si teclas extendidas (por ejemplo, alt, shift, ctrl) también fueron presionadas (no lo fueron), y algún otro estado.

La API ``SendMessage`` de Windows es una función sencilla que agrega el mensaje a una cola para el identificador de ventana en particular (``hWnd``). Más tarde, se llama a la función principal de procesamiento de mensajes (llamada ``WindowProc``) asignada a ``hWnd`` para procesar cada mensaje en la cola.

La ventana (``hWnd``) que está activa es en realidad un control de edición y ``WindowProc`` en este caso tiene un controlador de mensajes para mensajes ``WM_KEYDOWN``. Este código busca dentro del tercer parámetro que se pasó a ``SendMessage`` (``wParam``) y, debido a que es ``VK_RETURN``, sabe que el usuario ha presionado la tecla ENTER.

(En OS X) El NSEvent ``KeyDown`` NSEvent es enviado a la apliacación
----------------------------------------------------------------------

La señal de interrupción desencadena un evento de interrupción en el controlador de teclado I/O Kit kext. El controlador traduce la señal en un código clave que se pasa al proceso ``WindowServer`` de OS X. Como resultado, ``WindowServer`` envía un evento a cualquier aplicación adecuada (por ejemplo, activa o escuchando) a través de su puerto Mach, donde se coloca en una cola de eventos. Los eventos pueden ser leídos desde esta cola por subprocesos con suficientes privilegios llamando a la función ``mach_ipc_dispatch``. Esto ocurre más comúnmente a través de un bucle de eventos principal ``NSApplication`` y es manejado por ``NSApplication``, a través de ``NSEvent`` de ``NSEventType`` ``KeyDown``.

(En GNU/Linux) El servdor Xorg en escucha de "keycodes"
-------------------------------------------------------

Cuando se utiliza un ``servidor X`` gráfico, ``X`` utilizará el controlador de eventos genérico ``evdev`` para adquirir la pulsación de tecla. Se realiza una reasignación de códigos clave a códigos de escaneo con reglas y mapas de teclas específicos del ``servidor X``.

Cuando se completa la asignación del código de escaneo de la tecla presionada, el ``X Server`` envía el carácter al ``administrador de ventanas`` (DWM, metacity, i3, etc.), por lo que el ``administrador de ventanas`` a su vez envía el carácter a la ventana enfocada. La API gráfica de la ventana que recibe el carácter imprime el símbolo de fuente apropiado en el campo enfocado apropiado.


Parsear la URL
---------------

* El navegador tiene en este momento la siguiente información contenida en la URL (Uniform Resource Locator, *Localizador de recursos uniforme*):

    - ``Protocolo``  "http"
        Usa 'Hyper Text Transfer Protocol', HTTP
    
    - ``Dominio`` "google.com"
        El servidor es google.com

    - ``Recurso``  "/"
        Recupera la página principal (index)

Una URL/URI se puede parsear de ls siguiente forma:

.. image:: https://upload.wikimedia.org/wikipedia/commons/thumb/d/d6/URI_syntax_diagram.svg/800px-URI_syntax_diagram.svg.png
    :width: 300
    :alt: Esquema de parseo de una URL

¿Es una URL o un término de búsqueda?
-------------------------------------

Cuando no se ha introducido en el navegador un protocolo o dominio (DNS) válido, este le pasa el término al buscador web predeterminado. En muchos casos, la URL tendrá un texto especial en ella para decirle al motor de búsqueda para informarle desde qué navegador es realizada la consulta.

Convertir caracteres no ASCII en el nombre de dominio
-------------------------------------------------------

* El navegador comprueba el nombre de dominio en busca de caracteres que no son ``a-z``,
  ``A-Z``, ``0-9``, ``-``, o ``.``.
* Puesto que el nombre de dominio es ``google.com`` no habrá caracteres especiales fuera de los arriba indicados. Si los hubiera, el navegador aplicaría la codificación `Punycode`_ a la parte del dominio de la URL.

Comprobar la lista HSTS
--------------------------
* El navegador comprueba su lista HSTS (HTTP Strict Transport Security) precargada. Esta es una lista the sitios web que han solicitado que sean contactados únicamente mediante HTTPS. 
* Si el sitio web está en la lista, el navegador envia su petición mediante HTTPS en vez de HTTP. De otro modo, la petición inicial será enviada por HTTP (esto puede depender también de las políticas y la configuración del propio navegador). Debemos tener en cuenta que los sitios web siguen pudiendo utiliza HSTS sin estar en estas listas. La primera petición enviada por el cliente es respondida con una respuesta solicitando que el cliente únicamente envíe peticiones HTTPS. Son embargo, esta única petición HTTP podría dejar al usuario vulnerable a los `downgrade attack`_, también llamados ataques de degradación, que es el motivo por el cual las listas HSTS fueron añadidas a los navegadores web. Como ejemplo, esta es la `lista HSTS`_ precargada en Chrome. 


Búsqueda DNS
------------

* El navegador comprueba si el dominio está en su caché. (Para ver el caché DNS en Chrome, podemos acceder a `chrome://net-internals/#dns <chrome://net-internals/#dns>`_. Para verlo en Firefox, puedes acceder a `about:networking#dns <about:networking#dns>`_. En el navegador Edge basado en chromium, puedes acceder a `edge://net-internals/#dns`_).

* Si no es encontrado, el navegador llama a la función ``gethostbyname`` (varía según el sistema operativo) para hacer la búsqueda DNS.

* ``gethostbyname`` comprueba si el nombre de dominio puede ser resuelto buscando en el archivo ``hosts`` local (cuya localización `puede variar por OS`_) antes de intentar su resolución mediante DNS.

* Si ``gethostbyname`` no tiene la respuesta en caché o no la ha podido encontrar en el archivo ``hosts``, realiza una petición al servidor DNS configurado en los ajustes de red. Normalmente, es el *router* de nuestro operador o su servidor de cacheo DNS. En Windows usará un algoritmo que determina qué servidor DNS consultar primera para resolver el nombre de dominio. (Véase [este enlace](http://technet.microsoft.com/en-us/library/dd197552(WS.10).aspx))

* Si el servidor DNS está en la misma subred, la librería de red sigue el ``Proceso ARP`` a continuación indicado para encontrar el servidor DNS. En el caso de que la red esté trabajando con IPv6, se usa el protocolo de ``neighbor discovery``, que es ligeramente diferente. 

* Si el servidor DNS se encuentra en una subred diferente, la librería de red sigue el ``Proceso ARP`` debajo indicado para encontrar la puerta de enlace hacia esa red (que normalmente será la puerta de enlace por defecto).

* Prácticamente la gran mayoría de veces el servidor DNS definido en la red no mantiene la zona de "google.com", a esto lo conocemos como "Servidor autoritativo". La única excepción para esto, sería que quizá un equipo dentro del propio centro de datos de Google esté solicitando la respuesta (este no será seguramente nuestro caso...), así que el servidor DNS local intentará averiguar qué servidor DNS "posee" el dominio google.com. 

* Todos los equipos que utilizan DNS poseen una lista de "servidores raíz" predefinidos. Utilizando su propio algoritmo, elegirá un servidor raíz para encontrar el servidor SOA (Start Of Authority).

* Una vez que se elige el servidor raíz, se realiza una solicitud del TLD (Top-Level Domain). En este caso, es "com". Entonces, la solicitud de NS para "com". se le pregunta al servidor raíz.

* Una respuesta generará una lista de servidores para el TLD "com", al momento de escribir esto,  [a-m].gtld-servers.net (servido por Verisgn)

* Se envía otra solicitud de NS a uno de los [a-m].gtld-servers.net para "google.com".

* El servidor dns de Verisign responderá con los 4 servidores DNS de google, ns1.google.com a ns4.google.com y también incluirá las (direcciones IPv4) para llegar a ellos directamente. Si no los incluyera en la respuesta, el servidor DNS deberá volver a preguntar sobre estos. 

* El servidor DNS solicitante utilizará esta información para llegar al servidor DNS "real" de google.com (el que posee la SOA del dominio) y pide una resolución A (o AAAA si es IPv6) con "www.google.com". como la solicitud.

* El servidor DNS de Google utilizará la dirección IP de conexión remota y la resolverá a través de un instantánea reciente de la red BGP para identificar el origen ASN (Número de Sistema Autónomo) de la solicitud (el número único de su ISP, proveedor de Internet).

* El ASN se verifica contra una base de datos para saber qué centro de datos de Google se considera el mejor para responder a una solicitud de su ISP.

* El servidor DNS de Google devuelve la dirección IP del centro de datos más cercano según la ubicación estimada del usuario, en base al su dirección IP y el ASN al que pertenece esta.

* El servidor DNS recursivo/local devolverá la dirección IP al sistema operativo.

Proceso ARP
------------

Para enviar una solicitud ARP (Address Resolution Protocol) de broadcast, la librería de red necesita conocer la dirección IP a buscar. También necsita conocer la dirección MAC de la interfaz por la que va a enviar la solicitud ARP. Este proceso es diferente en IPv6. 

El caché ARP es primeramente comprobado en busca de una entrada ARP para la dirección IP objetivo. Si se encuentra en la caché, devuelve el resultado: IP objetico = Dirección MAC.

Si la entrada no se encuentra en la caché ARP:

* Se busca en la tabla de enrutado para ver si la dirección IP objetivo está en alguna de las subredes en la tabla de enrutado local (esto significa que el dispositivo está directamente conectado a estas redes). Si lo está, la librería utiliza la interfaz asociada con esa subred. Si no está, la librería usa la interfaz asociada a la puerta de enlace por defecto configurada en el equipo.
* La dirección MAC de la interfaz de red de la subred seleccionada es buscada.

* La librería de red enbía una solicitud ARP de capa 2 (capa de enlace de datos en el `modelo OSI`_):

``Solicitud ARP``::

    MAC Origen : dirección:MAC:origen:aquí
    IP Origen  : direccion.ip.origen.aquí
    MAC Destino: FF:FF:FF:FF:FF:FF (Broadcast)
    IP Destino : direccion.ip.destino.aquí

Dependiendo qué dispositivos se encuentren entre el equipo y equipo de destino:

Directamente conectado:

* Si el equipo está conectado directamente al equipo destino, este responde con una ``ARP Reply``, una respuesta ARP (ver a continación).

Hub:

* Si el ordenador está conectado a un hub, este enviará la petición ARP por todos los puertos (excepto por el que lo ha recibido). Si el router está conectado a este, responderá con una ``ARP Reply``, una respuesta ARP (ver a continación).

Switch:

* Si el equipo está conectado a un switch, el switch comprobará su tabla MAC/CAM para ver a qué puerto está conectada la IP que se está buscando. Si el switch no tiene ninguna entrada para esta MAC, la enviará por todos los otros puertos. 

* Si el switch tiene una entrada en la tabla MAC/CAM, enviará la petición ARP únicamente por el puerto al que está conectado el equipo con la MAC solicitada. 

* Si el router está conectado en la misma red, responderá con una respuesta ARP, ``ARP Reply``

``ARP Reply``::

    MAC Origen : dirección:MAC:origen:aquí
    IP Origen  : direccion.ip.origen.aquí
    MAC Destino: FF:FF:FF:FF:FF:FF (Broadcast)
    IP Destino : direccion.ip.destino.aquí

Continúa proceso DNS
---------------------


Ahora que la biblioteca de red tiene la dirección IP de nuestro servidor DNS o la puerta de enlace predeterminada, el equipo puede reanudar su proceso de DNS:

* El cliente abre un socket con destino al puerto 53/UDP en el servidor DNS, utilizando un puerto de origen por encima de 1023.
* Si el cliente estuviera configurado para utilzar DNSoverHTTPS o DNSoverTLS, el destino del socket sería 53/TCP.
* Si el servidor DNS local, o el de nuestro ISP, no dispone de la respuesta en su caché, entonces realiza una petición recursiva. Esta petición recursiva avanza hasta que se encuentra el SOA (``Start Of Authority``) y devuelve la respuesta de este. 

Abrir un socket
-------------------
Una vez que el navegador recibe la dirección IP del servidor de destino, la almacena, junto con el  número de puerto dado en la URL (el protocolo HTTP predeterminado es el puerto 80 y HTTPS el puerto 443). Realiza una llamada a la función de la biblioteca del sistema llamada ` `socket`` y solicita un flujo de socket TCP: ``AF_INET/AF_INET6`` y ``SOCK_STREAM``. Una vez el socket cliente es creado, la aplicación llama a la función ``connect`` con el socket, la dirección IP del servidor HTTP y el puerto.

* Esta solicitud se pasa primero a la capa de transporte donde se crea un segmento TCP. El puerto de destino se agrega al encabezado y se elige un puerto de origen dentro del rango de puertos dinámicos del kernel (ip_local_port_range en Linux).

* Este segmento se envía a la capa de Red (Nivel 3 de OSI), que añade un encabezado IP adicional. La dirección IP del servidor de destino, así como la de la máquina actual, se insertan para formar un paquete.

* El paquete llega después a la capa de Enlace (Nivel 2 de OSI). Se agrega un encabezado de "frame" que incluye la dirección MAC de la NIC de la máquina, así como la dirección MAC de la puerta de enlace (router local). Como antes, si el núcleo no conoce la dirección MAC de la puerta de enlace, debe transmitir una consulta ARP para encontrarla.

En este momento, el paquete está listo para ser transferido por cualquier método físico:

* `Ethernet`_
* `WiFi`_
* `Datos móviles`_

Para la mayoría de las conexiones a Internet domésticas o de pequeñas empresas, el paquete pasará desde su equipo, posiblemente a través de una red local, y luego a través de un módem (MOdulator/DEModulator) que convierte los 1 y 0 digitales en una señal analógica adecuada para la transmisión por teléfono, cable, o conexiones de telefonía inalámbrica. En el otro extremo de la conexión hay otro módem que vuelve a convertir la señal analógica en datos digitales para ser procesados por el siguiente `nodo de la red`_ donde se analizarán más a fondo las direcciones de origen y destino.

La mayoría de las empresas más grandes y algunas conexiones residenciales más nuevas tendrán conexiones de fibra o Ethernet directa, en cuyo caso los datos permanecen digitales y pasan directamente al siguiente `nodo de la red`_ para su procesamiento. En España, al menos en las áreas metropolitanas, es común la conexión mediante FTTH (Fiber to the home).

En algún momento, el paquete llegará al router que administra la subred local. Desde allí, continuará viajando a los routers de borde del sistema autónomo (AS), otros AS y finalmente al servidor de destino. Cada enrutador en el camino extrae la dirección de destino del encabezado IP y la enruta al próximo salto apropiado. El campo de tiempo de vida (TTL) en el encabezado IP se reduce en uno por cada enrutador que pasa. El paquete se descartará si el campo TTL llega a cero o si el enrutador actual no tiene espacio en su cola (quizás debido a la congestión de la red).


Este "enviar y recibir" ocurre múltiples veces siguiendo el esquema de conexión TCP:

* El cliente elige un número de secuencia inicial (ISN) y envía el paquete al servidor con el bit SYN establecido para indicar que está configurando el ISN.
* El servidor recibe SYN y si puede atender la petición:
   * El servidor elige su propio número de secuencia inicial.
   * El servidor establece SYN para indicar que está eligiendo su ISN.
   * El servidor copia el (cliente ISN + 1) en su campo ACK y agrega el indicador ACK para indicar que está acusando recibo del primer paquete.
* El cliente reconoce la conexión enviando un paquete:
   * Aumenta su propio número de secuencia.
   * Aumenta el número de *acknowledgment* del receptor.
   * Establece el campo ACK.
* Los datos se transfieren de la siguiente manera:
   * A medida que un lado envía N bytes de datos, aumenta su SEQ en ese número.
   * Cuando el otro lado acusa recibo de ese paquete (o una cadena de paquetes), envía un paquete ACK con el valor ACK igual al último secuencia recibida del otro lado.
* Para cerrar la conexión:
   * El lado que desea cerrar la conexión envía un paquete FIN.
   * El otro lado acepta ("ACK") el paquete FIN y envía su propio FIN.
   * El lado que cierra la conexión reconoce el FIN del otro lado con un ACK.

Handshake TLS
-------------

El protocolo TSL es el sucesor de SSL, que provee un mecanismo seguro para autenticación utilizando certificados x509.
También provee un canal de comunicación bidireccional entre dos partes, como un navegador y un cliente web, para establecer los detalles de su comunicación.
Un "TLS Handshake", que se podría traducir como "apretón de manos TLS", ocurre cuando usuario accede a un sitio a través de HTTPS (HTTP Seguro. Muy por encima, HTTP es el protocolo sobre el que se comunican las páginas web). El navegador comienza a consultar el servidor de origen del sitio web, también ocurre cada vez que cualquier otra comunicación utiliza HTTPS, incluidas las llamadas API y DNS over HTTPS/DNS over TLS.

Durante el curso de un "TLS Handshake", el cliente y el servidor juntos harán lo siguiente:

* Especificar qué versión de TLS (TLS 1.0, 1.2, 1.3, etc.) usarán.
* Decidir qué suites de cifrado utilizarán.
* Autenticar la identidad del servidor a través de la clave pública del servidor
y la firma digital de la autoridad certificadora SSL.
* Generar claves de sesión para usar encriptación simétrica (más eficiente y rápida que la asimétrica) después de completar el *TLS Handshake*.

Y los pasos para realizar esto son los siguientes:

* La computadora cliente envía un mensaje ``ClientHello`` al servidor con su versión Transport Layer Security (TLS), lista de algoritmos de cifrado y métodos de compresión disponibles.

* El servidor responde con un mensaje ``ServerHello`` al cliente con la versión de TLS, el cifrado seleccionado, los métodos de compresión seleccionados y el certificado público del servidor firmado por una CA (Autoridad de Certificación, *Certificate Authority*). El certificado contiene una clave pública que utilizará el cliente para cifrar el resto del protocolo de enlace hasta que se pueda acordar una clave simétrica.

* El cliente verifica el certificado digital del servidor con su lista de CA de confianza. Si se puede establecer la confianza en función de la CA, el cliente genera una cadena de bytes pseudoaleatorios y la cifra con la clave pública del servidor. Estos bytes aleatorios se pueden utilizar para determinar la clave simétrica.

* El servidor descifra los bytes aleatorios utilizando su clave privada y utiliza estos bytes para generar su propia copia de la clave maestra simétrica.

* El cliente envía un mensaje ``Finished`` al servidor, encriptando un hash de la transmisión hasta este punto con la clave simétrica.

* El servidor genera su propio hash y luego descifra el hash enviado por el cliente para verificar que coincida. Si lo hace, envía su propio mensaje ``Finished`` al cliente, también encriptado con la clave simétrica.

* A partir de ese momento, la sesión TLS transmite los datos de la aplicación (HTTP) encriptados con la clave simétrica acordada.

Si un paquete se pierde
------------------------

A veces, debido a la congestión de la red o conexiones de hardware inestables, los paquetes TLS se descartarán antes de que lleguen a su destino final. El remitente entonces tiene que decidir cómo reaccionar. El algoritmo para esto se llama `control de congestión TCP`_. Esto varía según el remitente; los algoritmos más comunes son `cubic`_ en los sistemas operativos más nuevos y `New Reno`_ en casi todos los demás.

* El cliente elige una `congestion window`_ ("ventana de congestión", en castellano) basada en el `maximum segment size`_ (MSS, tamaño máximo del segmento) de la conexión.

* Por cada paquete reconocido, la ventana se duplica en tamaño hasta que alcanza el 'umbral de inicio lento', *slow-start threshold*. En algunas implementaciones, este umbral es adaptativo.

* Después de alcanzar el umbral de inicio lento, la ventana aumenta de manera adicional para cada paquete reconocido. Si se descarta un paquete, la ventana se reduce exponencialmente hasta que se reconoce otro paquete.

Protocolo HTTP
---------------

Si el navegador que está utilizando fue escrito por Google y tiene una versióna anterior al año 2016, en lugar de enviar una solicitud HTTP para recuperar la página, enviará una solicitud para intentar negociar con el servidor una "actualización" de HTTP al protocolo SPDY.

Este protocolo, SPDY fue un intento de Google para mejorar la navegación web. Fue un protocolo experimental que quedó `descontinuado`_ en favor de HTTP/2.

Si el cliente está utilizando el protocolo HTTP y no es compatible con SPDY, envía una solicitud al servidor de la forma (esto ocurrirá en la mayoría de los casos, pues es el estandar)::

    GET / HTTP/1.1
    Host: google.com
    Connection: close
    [otras cabeceras]

Donde ``[otras cabeceras]`` se refiere a una serie de pares clave-valor separadas por dos puntos, ":", formateadas siguiendo el estándar HTTP y separadas por retornos de carro, también llamados saltos de línea. 
(Esto asume que el navegador web que se está utilizando no tiene ningún error que infrinja la especificación HTTP. Esto también supone que el navegador web está utilizando ``HTTP/1.1``; de lo contrario, es posible que no incluya el encabezado ``Host`` en el y la versión especificada en la solicitud ``GET`` será ``HTTP/1.0`` o ``HTTP/0.9``.)

HTTP/1.1 define la opción de conexión "cerrada" para que el remitente señale que la conexión se cerrará después de completar la respuesta. Por ejemplo,

    Connection: close

Las aplicaciones HTTP/1.1 que no admiten conexiones persistentes DEBEN incluir la opción de conexión "close" en cada mensaje.

Después de enviar la solicitud y las cabeceras, el navegador web envía una nueva línea en blanco al servidor para indicar que el contenido de la solicitud está listo.

El servidor responde con un código de respuesta que indica el estado de la solicitud y responde con una respuesta de la forma::

    200 OK
    [cabeceras de respuesta]

Seguido de un solo salto de línea, envía un *payload* del contenido HTML de ``www.google.com``. Luego, el servidor puede cerrar la conexión o, si los encabezados enviados por el cliente lo solicitaron, mantener la conexión abierta para reutilizarla para futuras solicitudes.

Si los encabezados HTTP enviados por el navegador web incluían información suficiente para que el servidor web determinara si la versión del archivo almacenado en caché por el navegador web no se ha modificado desde la última recuperación (es decir, si el navegador web incluía un encabezado ``ETag`` ), en su lugar, puede responder con una solicitud de la forma:

    304 Not Modified
    [cabeceras de respuesta]

sin ningún contenido adicional, y el navegador web en su lugar recupera el HTML de su caché.

Después de analizar el HTML, el navegador web (y el servidor) repite este proceso para cada recurso (imagen, CSS, favicon.ico, etc.) al que hace referencia la página HTML, excepto que en lugar de ``GET / HTTP/1.1``, la solicitud será ser ``GET /$(URL relativa a www.google.com) HTTP/1.1``.

Si el HTML hace referencia a un recurso en un dominio diferente a ``www.google.com``, el navegador web vuelve a los pasos involucrados en la resolución del otro dominio y sigue todos los pasos hasta este punto para ese dominio. El encabezado ``Host`` en la solicitud se configurará con el nombre de servidor apropiado en lugar de ``google.com``.

Manejo de peticiones HTTP en servidor
--------------------------------------

El servidor HTTPD (HTTP Daemon) es el que maneja las solicitudes/respuestas en el lado del servidor. Los servidores HTTPD más comunes son Apache o nginx para Linux e IIS para Windows.

* El servidor HTTP recibe la petición.
* El servidor desglosa la solicitud en los siguientes parámetros:
   * Método de petición HTTP (que puede ser ``GET``, ``HEAD``, ``POST``, ``PUT``,
     ``PATCH``, ``DELETE``, ``CONNECT``, ``OPTIONS``, o ``TRACE``). En el caso de una URL ingresada directamente en la barra de direcciones, el método será ``GET``.
   * Dominio, en este caso google.com.
   * Página o ruta solicitada.  En este caso,  */* (puesto que no se solicitó una ruta/página específica, / es la ruta por defecto).
* A menudo (siempre es cierto para Google y páginas webs grandes), el servidor que recibe la solicitud inicialmente es un balanceador de carga. Este equipo "leerá" la petición y la enviará a uno de los servidores web con los que mantiene comunicación. La decisión de a qué servidor web enviarlo variará según la utilizanción y el estado de cada uno de ellos, así como la petición en sí.
* El servidor verifica que haya un host virtual configurado en el servidor que se corresponda con google.com.
* El servidor verifica que google.com puede aceptar solicitudes GET.
* El servidor verifica que el cliente tiene permiso para usar este método (por IP, autenticación, etc.).
* Si el servidor tiene instalado un módulo de reescritura (como *mod_rewrite* para Apache o *URL Rewrite* para IIS), intenta hacer coincidir la solicitud con una de las reglas configuradas. Si se encuentra una regla coincidente, el servidor usa esa regla para reescribir la solicitud.
* El servidor extrae el contenido que corresponde con la solicitud, en nuestro caso, recurrirá al archivo de índice, ya que "/" es el archivo principal (algunos casos pueden anular esto, pero este es el método más común).
* El servidor analiza el archivo según el controlador. Si Google se ejecuta en PHP, el servidor usa PHP para interpretar el archivo de índice y transmite la salida al cliente.

El trabajo del navegador
-------------------------

Una vez que el servidor proporciona los recursos (HTML, CSS, JS, imágenes, etc.) al navegador, se inicia el siguiente proceso:

* Parseado - HTML, CSS, JS
* Renderizado - Construir árbol DOM → Árbol de renderizado → Diseño del árbol de renderizado → Pintar el árbol de renderizado

Navegador
-----------

La funcionalidad del navegador es presentar el recurso web que elija, solicitándolo al servidor y mostrándolo en la ventana del navegador. El recurso suele ser un documento HTML, pero también puede ser un PDF, una imagen o algún otro tipo de contenido. El usuario especifica la ubicación del recurso mediante un URI (identificador uniforme de recursos).

La forma en que el navegador interpreta y muestra los archivos HTML se recoge en las especificaciones de HTML y CSS. Estas especificaciones son mantenidas por la organización W3C (World Wide Web Consortium), que es la organización de estándares para la web.

Las interfaces de usuario del navegador tienen mucho en común entre sí.
Los elementos comunes de la interfaz de usuario son:

* Una barra de direcciones para insertar un URI
* Botones de avance y retroceso
* Opciones de marcadores
* Botones Actualizar y Detener para actualizar o detener la carga de documentos actuales
* Botón de inicio que te lleva a tu página de inicio

**Estructura del navegador (en alto nivel)**

The components of the browsers are:

* **Interfaz de usuario:** La interfaz de usuario incluye la barra de direcciones, el botón de avance/retroceso, el menú de favoritos, etc. Todas las partes de la pantalla del navegador excepto la ventana donde se ve la página solicitada.

* **Motor del navegador:** el motor del navegador ordena las acciones entre la interfaz de usuario y el motor de renderizado.

* **Motor de renderizado:** El motor de renderizado es responsable de mostrar el contenido solicitado. Por ejemplo, si el contenido solicitado es HTML, el motor de representación analiza HTML y CSS y muestra el contenido analizado en la pantalla.

* **Redes:** Las redes manejan llamadas de red, como solicitudes HTTP, utilizando diferentes implementaciones para diferentes plataformas detrás de una interfaz independiente de la plataforma.

* **Backend de la interfaz de usuario:** el backend de la interfaz de usuario se usa para dibujar widgets básicos como cuadros combinados y ventanas. Este backend expone una interfaz genérica que no es específica de la plataforma. Debajo, utiliza métodos de interfaz de usuario del sistema operativo.

* **Motor JavaScript:** El motor JavaScript se utiliza para analizar y ejecutar código JavaScript.

* **Almacenamiento de datos:** El almacenamiento de datos es una capa de persistencia. Es posible que el navegador necesite guardar todo tipo de datos localmente, como cookies. Los navegadores también admiten mecanismos de almacenamiento como localStorage, IndexedDB, WebSQL y FileSystem.

Análisis de HTML
-------------------

El motor de renderizado comienza a obtener  de la capa de red el contenido del documento solicitado. Esto generalmente se hará en fragmentos de 8kB.

El trabajo principal del analizador HTML es analizar el marcado HTML en un árbol de análisis.

El árbol de salida (el "*parse tree*") es un árbol de elementos nodos y atributos DOM. DOM es la abreviatura de *Document Object Model*, Modelo de Objetos de Documento. 

Es la presentación de objetos del documento HTML y la interfaz de los elementos HTML con el mundo exterior como JavaScript. La raíz del árbol es el objeto "Document". Antes de cualquier manipulación a través de secuencias de comandos, el DOM tiene una relación casi uno a uno con el marcado.

**El algoritmo de análisis ("parseo")**

HTML no se puede analizar con los analizadores normales: de arriba hacia abajo o de abajo hacia arriba.

Las razones son:

* La naturaleza indulgente del lenguaje.
* El hecho de que los navegadores tengan una tolerancia a errores tradicional para admitir casos bien conocidos de HTML no válido.
* El proceso de análisis es reentrante. Para otros idiomas, la fuente no cambia durante el análisis, pero en HTML, el código dinámico (como los elementos de *scripts* que contienen llamadas `document.write()`) pueden agregar tokens adicionales, por lo que el proceso de análisis en realidad modifica la entrada.

Incapaz de utilizar las técnicas de análisis habituales, el navegador utiliza un analizador personalizado para analizar HTML. El algoritmo de análisis se describe en detalle en la especificación HTML5.

El algoritmo consta de dos etapas: tokenización y construcción del árbol.

**Acciones cuando finaliza el análisis**

El navegador comienza a obtener recursos externos vinculados a la página (CSS, imágenes, archivos JavaScript, etc.).

En esta etapa, el navegador marca el documento como interactivo y comienza a analizar los scripts que están en modo "diferido": aquellos que deben ejecutarse después de analizar el documento. El estado del documento se establece en "*complete*" (completo, en castellano) y se activa un evento de "*load*" (cargado, en castellano).

Ten en cuenta que nunca aparece un error de "Sintaxis no válida" en una página HTML. Los navegadores corrigen cualquier contenido no válido y continúan.

Interpretación de CSS
----------------------

* Analizar archivos CSS, contenidos de etiquetas ``<style>`` y valores de atributo ``style`` utilizando la `"Sintaxis y léxico CSS"`_
* Cada archivo CSS se analiza en un ``StyleSheet object``, donde cada objeto contiene reglas CSS con selectores y objetos correspondientes a la gramática CSS.
* Un analizador CSS puede ser de arriba hacia abajo o de abajo hacia arriba cuando se usa un analizador específico.

Renderización de la página
---------------------------

* Se crea un "*Frame Tree*", "Árbol de marcos" en castellano, o "*Render Tree*", 'Árbol de procesamiento' en castellano, recorriendo los nodos DOM y calculando los valores de estilo CSS para cada nodo.
* Ce calcula el ancho preferido de cada nodo en el "*Frame Tree*" de abajo hacia arriba sumando el ancho preferido de los nodos secundarios y los márgenes horizontales (*margins*), bordes (*borders*)  y relleno (*padding*) del nodo.
* Se calcula el ancho real de cada nodo de arriba hacia abajo asignando el ancho disponible de cada nodo a sus hijos.
* Se calcula la altura de cada nodo de abajo hacia arriba aplicando ajuste de texto y sumando las alturas de los nodos secundarios y los márgenes, bordes y relleno del nodo.
* Se calculan las coordenadas de cada nodo utilizando la información calculada anteriormente.
* Se toman pasos más complicados cuando los elementos están "flotados" (``float``), posicionados ``absolutamente`` o ``relativamente``, u otras características complejas son utilizados. Consulte https://dev.w3.org/csswg/css2/ y https://www.w3.org/Style/CSS/current-work para obtener más detalles.
* Se crean capas para describir qué partes de la página se pueden animar como un grupo sin volver a ser rasterizados. Cada frame/render de renderizado se asigna a una capa.
* Las texturas se asignan para cada capa de la página.
* Los objetos de frame/render para cada capa se recorren y los comandos de dibujo se ejecutan para cada capa. Esto puede ser rasterizado por la CPU o dibujado en la GPU directamente usando D2D/SkiaGL.
* Todos los pasos anteriores pueden reutilizar los valores calculados desde la última vez que se representó la página web, por lo que los cambios incrementales requieren menos carga de trabajo.
* Las capas de la página se envían al proceso de composición donde se combinan con capas para otro contenido visible como el iframes y paneles adicionales.
* Las capas finales se calculan y los comandos de composición se emiten a través de Direct3D/OpenGL. Los búferes de comandos de la GPU se vuelcan en la GPU para la representación asíncrona y el resultado se envía al servidor de ventanas.

Rendización GPU
----------------

* Durante el proceso de renderizado, las capas de computación gráfica pueden usar la ``CPU`` de propósito general o también el procesador gráfico, la ``GPU``.

* Cuando se usa la ``GPU`` para cálculos de representación gráfica, las capas de software gráfico dividen la tarea en varias partes, por lo que puede aprovechar el paralelismo masivo de ``GPU`` para los cálculos de punto flotante necesarios para el proceso de renderizado.


Servidor de ventanas
-----------------------

Post-procesado y ejecuciones a causa del usuario
--------------------------------------------------
Una vez que se ha completado el procesamiento, el navegador ejecuta el código JavaScript como resultado de algún mecanismo de tiempo (como una animación de Google Doodle) o la interacción del usuario (escribir una consulta en el cuadro de búsqueda y recibir sugerencias). Los complementos como Flash o Java también pueden ejecutarse, aunque no en este momento en la página de inicio de Google. Los scripts pueden hacer que se realicen solicitudes de red adicionales, así como modificar la página o su diseño, lo que provoca otra ronda de renderizado y pintado de la página.

.. _`"What happens when..."`: https://github.com/alex/what-happens-when
.. _`Alex Waynor`: https://github.com/alex
.. _`Creative Commons Zero`: https://creativecommons.org/publicdomain/zero/1.0/
.. _`"Sintaxis y léxico CSS"`: http://www.w3.org/TR/CSS2/grammar.html
.. _`Punycode`: https://en.wikipedia.org/wiki/Punycode
.. _`Ethernet`: http://en.wikipedia.org/wiki/IEEE_802.3
.. _`WiFi`: https://en.wikipedia.org/wiki/IEEE_802.11
.. _`Datos móviles`: https://en.wikipedia.org/wiki/Cellular_data_communication_protocol
.. _`analog-to-digital converter`: https://en.wikipedia.org/wiki/Analog-to-digital_converter
.. _`nodo de la red`: https://en.wikipedia.org/wiki/Computer_network#Network_nodes
.. _`control de congestión TCP`: https://en.wikipedia.org/wiki/TCP_congestion_control
.. _`cubic`: https://en.wikipedia.org/wiki/CUBIC_TCP
.. _`New Reno`: https://en.wikipedia.org/wiki/TCP_congestion_control#TCP_New_Reno
.. _`congestion window`: https://en.wikipedia.org/wiki/TCP_congestion_control#Congestion_window
.. _`maximum segment size`: https://en.wikipedia.org/wiki/Maximum_segment_size
.. _`puede variar por OS` : https://en.wikipedia.org/wiki/Hosts_%28file%29#Location_in_the_file_system
.. _`简体中文`: https://github.com/skyline75489/what-happens-when-zh_CN
.. _`한국어`: https://github.com/SantonyChoi/what-happens-when-KR
.. _`日本語`: https://github.com/tettttsuo/what-happens-when-JA
.. _`downgrade attack`: http://en.wikipedia.org/wiki/SSL_stripping
.. _ `ataques de degradación`: https://encyclopedia.kaspersky.com/glossary/downgrade-attack/
.. _`modelo OSI`: https://es.wikipedia.org/wiki/Modelo_OSI
.. _`lista HSTS`: https://source.chromium.org/chromium/chromium/src/+/main:net/http/transport_security_state_static.json 
.. _`descontinuado`: https://blog.chromium.org/2016/02/transitioning-from-spdy-to-http2.html

Este repositorio es la versión traducida al castellano del recurso de `Alex Waynor`_, `"What happens when..."`_. 

Alex Waynor no está relacionado con este trabajo ni ha revisado la traducción. 

¿Qué pasa cuando...?
=====================

Este repositorio es un intento de responder a la antigua pregunta "¿Qué ocurre cuando escribes *google.com* en tu la barra de direcciones de tu navegador y pulsas *Enter*?"

A excepción de las respuesta típica, intentaremos tratar de responder a esta pregunta con el mayor detalle posible. Sin dejar de lado ningún detalle. 

Este es un proceso colaborativo, ¡así que investiga sobre la sección que más te guste y ayúdanos! Hay muchísimos detalles faltantes, ¡están esperando a que los añadas! ¡Envíanos un *pull request*, por favor!

Este trabajo está licenciado bajo la licencia `Creative Commons Zero`_.

También puedes leerlo en  `简体中文`_ (chino simplificado), `日本語`_ (japonés) y `한국어`_(koreano). También, por supuesto, la versión original de `"What happens when..."`_ en inglés. 

Table of Contents
====================

.. contents::
   :backlinks: none
   :local:

La tecla "g" es presionada
----------------------------

Las siguientes secciones explican el comportamiento físico del teclado y las interrupciones del sistema operativo. 

Cuando presionas la tecla "g" el navegador recibe el evento y la función de auto-completado del navegador entra en acción. 

Dependiendo del algoritmo del navegador y si estás en una ventana de incógnito (también llamada Privada o InPrivate) o no, varias sugerencias te serán mostradas debajo de la barra de direcciones. La mayoría de estos algoritmos ordenan y priorizan los resultados basándose en el historial de búsqueda, marcadores, cookies, y consultas populares en todo Internet. 

Según continúas escribiendo "google.com" muchos bloques de código son ejecutados y las sugerencias serán mejoradas con cada tecla presionada. Incluso te acabará sugieriendo "google.com" antes de que termines de escribirlo. 


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

    - ``Recurso``  "/"
        Recupera la página principal (index)


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
* Si el sitio web está en la lista, el navegador envia su petición mediante HTTPS en vez de HTTP. De otro modo, la petición inicial será enviada por HTTP (esto puede depender también de las políticas y la configuración del propio navegador). Debemos tener en cuenta que los sitios web siguen pudiendo utiliza HSTS sin estar en estas listas. La primera petición enviada por el cliente es respondida con una respuesta solicitando que el cliente únicamente envíe peticiones HTTPS. Son embargo, esta única petición HTTP podría dejar al usuario vulnerable a los `downgrade attack`_, también llamados `ataques de degradación`_, que es el motivo por el cual las listas HSTS fueron añadidas a los navegadores web. Como ejemplo, esta es la `lista HSTS`_ precargada en Chrome. 


Búsqueda DNS
------------

* El navegador comprueba si el dominio está en su caché. (Para ver el caché DNS en Chrome, podemos acceder a `chrome://net-internals/#dns <chrome://net-internals/#dns>`_).
* Si no es encontrado, el navegador llama a la función ``gethostbyname`` (varía según el sistema operativo) para hacer la búsqueda DNS.
* ``gethostbyname`` comprueba si el nombre de dominio puede ser resuelto buscando en el archivo ``hosts`` local (cuya localización `puede variar por OS`_) antes de intentar su resolución mediante DNS.
* Si ``gethostbyname`` no tiene la respuesta en caché o no la ha podido encontrar en el archivo ``hosts``, realiza una petición al servidor DNS configurado en los ajustes de red. Normalmente, es el *router* de nuestro operador o su servidor de cacheo DNS.
* Si el servidor DNS está en la misma subred, la librería de red sigue ``Proceso ARP`` a continuación indicado para encontrar el servidor DNS.
* Si el servidor DNS se encuentra en una subred diferente, la librería de red sigue el ``Proceso ARP`` debajo indicado para encontrar la puerta de enlace hacia esa red (que normalmente será la puerta de enlace por defecto).

Proceso ARP
------------

Para enviar una solicitud ARP (Address Resolution Protocol) de broadcast, la librería de red necesita conocer la dirección IP a buscar. También necsita conocer la dirección MAC de la interfaz por la que va a enviar la solicitud ARP. 

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

Dependiendo qué dispositivos se encuentren entre el equipo y el router:

Directamente conectado:

* Si el equipo está conectado directamente al router, el router responde con una ``ARP Reply``, una respuesta ARP (ver a continación).

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

Ahora que la biblioteca de red tiene la dirección IP de nuestro servidor DNS o la puerta de enlace predeterminada, el equipo puede reanudar su proceso de DNS:

* El cliente abre un socket con destino al puerto 53/UDP en el servidor DNS, utilizando un puerto de origen por encima de 1023.
* Si el cliente estuviera configurado para utilzar DNSoverHTTPS o DNSoverTLS, el destino del socket sería 53/TCP.
* Si el servidor DNS local, o el de nuestro ISP, no dispone de la respuesta en su caché, entonces realiza una petición recursiva. Esta petición recursiva avanza hasta que se encuentra el SOA (``Start Of A uthority``) y devuelve la respuesta de este. 

Opening of a socket
-------------------
Once the browser receives the IP address of the destination server, it takes
that and the given port number from the URL (the HTTP protocol defaults to port
80, and HTTPS to port 443), and makes a call to the system library function
named ``socket`` and requests a TCP socket stream - ``AF_INET/AF_INET6`` and
``SOCK_STREAM``.

* This request is first passed to the Transport Layer where a TCP segment is
  crafted. The destination port is added to the header, and a source port is
  chosen from within the kernel's dynamic port range (ip_local_port_range in
  Linux).
* This segment is sent to the Network Layer, which wraps an additional IP
  header. The IP address of the destination server as well as that of the
  current machine is inserted to form a packet.
* The packet next arrives at the Link Layer. A frame header is added that
  includes the MAC address of the machine's NIC as well as the MAC address of
  the gateway (local router). As before, if the kernel does not know the MAC
  address of the gateway, it must broadcast an ARP query to find it.

At this point the packet is ready to be transmitted through either:

* `Ethernet`_
* `WiFi`_
* `Cellular data network`_

For most home or small business Internet connections the packet will pass from
your computer, possibly through a local network, and then through a modem
(MOdulator/DEModulator) which converts digital 1's and 0's into an analog
signal suitable for transmission over telephone, cable, or wireless telephony
connections. On the other end of the connection is another modem which converts
the analog signal back into digital data to be processed by the next `network
node`_ where the from and to addresses would be analyzed further.

Most larger businesses and some newer residential connections will have fiber
or direct Ethernet connections in which case the data remains digital and
is passed directly to the next `network node`_ for processing.

Eventually, the packet will reach the router managing the local subnet. From
there, it will continue to travel to the autonomous system's (AS) border
routers, other ASes, and finally to the destination server. Each router along
the way extracts the destination address from the IP header and routes it to
the appropriate next hop. The time to live (TTL) field in the IP header is
decremented by one for each router that passes. The packet will be dropped if
the TTL field reaches zero or if the current router has no space in its queue
(perhaps due to network congestion).

This send and receive happens multiple times following the TCP connection flow:

* Client chooses an initial sequence number (ISN) and sends the packet to the
  server with the SYN bit set to indicate it is setting the ISN
* Server receives SYN and if it's in an agreeable mood:
   * Server chooses its own initial sequence number
   * Server sets SYN to indicate it is choosing its ISN
   * Server copies the (client ISN +1) to its ACK field and adds the ACK flag
     to indicate it is acknowledging receipt of the first packet
* Client acknowledges the connection by sending a packet:
   * Increases its own sequence number
   * Increases the receiver acknowledgment number
   * Sets ACK field
* Data is transferred as follows:
   * As one side sends N data bytes, it increases its SEQ by that number
   * When the other side acknowledges receipt of that packet (or a string of
     packets), it sends an ACK packet with the ACK value equal to the last
     received sequence from the other
* To close the connection:
   * The closer sends a FIN packet
   * The other sides ACKs the FIN packet and sends its own FIN
   * The closer acknowledges the other side's FIN with an ACK

TLS handshake
-------------
* The client computer sends a ``ClientHello`` message to the server with its
  Transport Layer Security (TLS) version, list of cipher algorithms and
  compression methods available.

* The server replies with a ``ServerHello`` message to the client with the
  TLS version, selected cipher, selected compression methods and the server's
  public certificate signed by a CA (Certificate Authority). The certificate
  contains a public key that will be used by the client to encrypt the rest of
  the handshake until a symmetric key can be agreed upon.

* The client verifies the server digital certificate against its list of
  trusted CAs. If trust can be established based on the CA, the client
  generates a string of pseudo-random bytes and encrypts this with the server's
  public key. These random bytes can be used to determine the symmetric key.

* The server decrypts the random bytes using its private key and uses these
  bytes to generate its own copy of the symmetric master key.

* The client sends a ``Finished`` message to the server, encrypting a hash of
  the transmission up to this point with the symmetric key.

* The server generates its own hash, and then decrypts the client-sent hash
  to verify that it matches. If it does, it sends its own ``Finished`` message
  to the client, also encrypted with the symmetric key.

* From now on the TLS session transmits the application (HTTP) data encrypted
  with the agreed symmetric key.

If a packet is dropped
----------------------

Sometimes, due to network congestion or flaky hardware connections, TLS packets
will be dropped before they get to their final destination. The sender then has
to decide how to react. The algorithm for this is called `TCP congestion
control`_. This varies depending on the sender; the most common algorithms are
`cubic`_ on newer operating systems and `New Reno`_ on almost all others.

* Client chooses a `congestion window`_ based on the `maximum segment size`_
  (MSS) of the connection.
* For each packet acknowledged, the window doubles in size until it reaches the
  'slow-start threshold'. In some implementations, this threshold is adaptive.
* After reaching the slow-start threshold, the window increases additively for
  each packet acknowledged. If a packet is dropped, the window reduces
  exponentially until another packet is acknowledged.

HTTP protocol
-------------

If the web browser used was written by Google, instead of sending an HTTP
request to retrieve the page, it will send a request to try and negotiate with
the server an "upgrade" from HTTP to the SPDY protocol.

If the client is using the HTTP protocol and does not support SPDY, it sends a
request to the server of the form::

    GET / HTTP/1.1
    Host: google.com
    Connection: close
    [other headers]

where ``[other headers]`` refers to a series of colon-separated key-value pairs
formatted as per the HTTP specification and separated by single newlines.
(This assumes the web browser being used doesn't have any bugs violating the
HTTP spec. This also assumes that the web browser is using ``HTTP/1.1``,
otherwise it may not include the ``Host`` header in the request and the version
specified in the ``GET`` request will either be ``HTTP/1.0`` or ``HTTP/0.9``.)

HTTP/1.1 defines the "close" connection option for the sender to signal that
the connection will be closed after completion of the response. For example,

    Connection: close

HTTP/1.1 applications that do not support persistent connections MUST include
the "close" connection option in every message.

After sending the request and headers, the web browser sends a single blank
newline to the server indicating that the content of the request is done.

The server responds with a response code denoting the status of the request and
responds with a response of the form::

    200 OK
    [response headers]

Followed by a single newline, and then sends a payload of the HTML content of
``www.google.com``. The server may then either close the connection, or if
headers sent by the client requested it, keep the connection open to be reused
for further requests.

If the HTTP headers sent by the web browser included sufficient information for
the webserver to determine if the version of the file cached by the web
browser has been unmodified since the last retrieval (ie. if the web browser
included an ``ETag`` header), it may instead respond with a request of
the form::

    304 Not Modified
    [response headers]

and no payload, and the web browser instead retrieve the HTML from its cache.

After parsing the HTML, the web browser (and server) repeats this process
for every resource (image, CSS, favicon.ico, etc) referenced by the HTML page,
except instead of ``GET / HTTP/1.1`` the request will be
``GET /$(URL relative to www.google.com) HTTP/1.1``.

If the HTML referenced a resource on a different domain than
``www.google.com``, the web browser goes back to the steps involved in
resolving the other domain, and follows all steps up to this point for that
domain. The ``Host`` header in the request will be set to the appropriate
server name instead of ``google.com``.

HTTP Server Request Handle
--------------------------
The HTTPD (HTTP Daemon) server is the one handling the requests/responses on
the server-side. The most common HTTPD servers are Apache or nginx for Linux
and IIS for Windows.

* The HTTPD (HTTP Daemon) receives the request.
* The server breaks down the request to the following parameters:
   * HTTP Request Method (either ``GET``, ``HEAD``, ``POST``, ``PUT``,
     ``PATCH``, ``DELETE``, ``CONNECT``, ``OPTIONS``, or ``TRACE``). In the
     case of a URL entered directly into the address bar, this will be ``GET``.
   * Domain, in this case - google.com.
   * Requested path/page, in this case - / (as no specific path/page was
     requested, / is the default path).
* The server verifies that there is a Virtual Host configured on the server
  that corresponds with google.com.
* The server verifies that google.com can accept GET requests.
* The server verifies that the client is allowed to use this method
  (by IP, authentication, etc.).
* If the server has a rewrite module installed (like mod_rewrite for Apache or
  URL Rewrite for IIS), it tries to match the request against one of the
  configured rules. If a matching rule is found, the server uses that rule to
  rewrite the request.
* The server goes to pull the content that corresponds with the request,
  in our case it will fall back to the index file, as "/" is the main file
  (some cases can override this, but this is the most common method).
* The server parses the file according to the handler. If Google
  is running on PHP, the server uses PHP to interpret the index file, and
  streams the output to the client.

Behind the scenes of the Browser
----------------------------------

Once the server supplies the resources (HTML, CSS, JS, images, etc.)
to the browser it undergoes the below process:

* Parsing - HTML, CSS, JS
* Rendering - Construct DOM Tree → Render Tree → Layout of Render Tree →
  Painting the render tree

Browser
-------

The browser's functionality is to present the web resource you choose, by
requesting it from the server and displaying it in the browser window.
The resource is usually an HTML document, but may also be a PDF,
image, or some other type of content. The location of the resource is
specified by the user using a URI (Uniform Resource Identifier).

The way the browser interprets and displays HTML files is specified
in the HTML and CSS specifications. These specifications are maintained
by the W3C (World Wide Web Consortium) organization, which is the
standards organization for the web.

Browser user interfaces have a lot in common with each other. Among the
common user interface elements are:

* An address bar for inserting a URI
* Back and forward buttons
* Bookmarking options
* Refresh and stop buttons for refreshing or stopping the loading of
  current documents
* Home button that takes you to your home page

**Browser High-Level Structure**

The components of the browsers are:

* **User interface:** The user interface includes the address bar,
  back/forward button, bookmarking menu, etc. Every part of the browser
  display except the window where you see the requested page.
* **Browser engine:** The browser engine marshals actions between the UI
  and the rendering engine.
* **Rendering engine:** The rendering engine is responsible for displaying
  requested content. For example if the requested content is HTML, the
  rendering engine parses HTML and CSS, and displays the parsed content on
  the screen.
* **Networking:** The networking handles network calls such as HTTP requests,
  using different implementations for different platforms behind a
  platform-independent interface.
* **UI backend:** The UI backend is used for drawing basic widgets like combo
  boxes and windows. This backend exposes a generic interface that is not
  platform-specific.
  Underneath it uses operating system user interface methods.
* **JavaScript engine:** The JavaScript engine is used to parse and
  execute JavaScript code.
* **Data storage:** The data storage is a persistence layer. The browser may
  need to save all sorts of data locally, such as cookies. Browsers also
  support storage mechanisms such as localStorage, IndexedDB, WebSQL and
  FileSystem.

HTML parsing
------------

The rendering engine starts getting the contents of the requested
document from the networking layer. This will usually be done in 8kB chunks.

The primary job of the HTML parser is to parse the HTML markup into a parse tree.

The output tree (the "parse tree") is a tree of DOM element and attribute
nodes. DOM is short for Document Object Model. It is the object presentation
of the HTML document and the interface of HTML elements to the outside world
like JavaScript. The root of the tree is the "Document" object. Prior to
any manipulation via scripting, the DOM has an almost one-to-one relation to
the markup.

**The parsing algorithm**

HTML cannot be parsed using the regular top-down or bottom-up parsers.

The reasons are:

* The forgiving nature of the language.
* The fact that browsers have traditional error tolerance to support well
  known cases of invalid HTML.
* The parsing process is reentrant. For other languages, the source doesn't
  change during parsing, but in HTML, dynamic code (such as script elements
  containing `document.write()` calls) can add extra tokens, so the parsing
  process actually modifies the input.

Unable to use the regular parsing techniques, the browser utilizes a custom
parser for parsing HTML. The parsing algorithm is described in
detail by the HTML5 specification.

The algorithm consists of two stages: tokenization and tree construction.

**Actions when the parsing is finished**

The browser begins fetching external resources linked to the page (CSS, images,
JavaScript files, etc.).

At this stage the browser marks the document as interactive and starts
parsing scripts that are in "deferred" mode: those that should be
executed after the document is parsed. The document state is
set to "complete" and a "load" event is fired.

Note there is never an "Invalid Syntax" error on an HTML page. Browsers fix
any invalid content and go on.

CSS interpretation
------------------

* Parse CSS files, ``<style>`` tag contents, and ``style`` attribute
  values using `"CSS lexical and syntax grammar"`_
* Each CSS file is parsed into a ``StyleSheet object``, where each object
  contains CSS rules with selectors and objects corresponding CSS grammar.
* A CSS parser can be top-down or bottom-up when a specific parser generator
  is used.

Page Rendering
--------------

* Create a 'Frame Tree' or 'Render Tree' by traversing the DOM nodes, and
  calculating the CSS style values for each node.
* Calculate the preferred width of each node in the 'Frame Tree' bottom-up
  by summing the preferred width of the child nodes and the node's
  horizontal margins, borders, and padding.
* Calculate the actual width of each node top-down by allocating each node's
  available width to its children.
* Calculate the height of each node bottom-up by applying text wrapping and
  summing the child node heights and the node's margins, borders, and padding.
* Calculate the coordinates of each node using the information calculated
  above.
* More complicated steps are taken when elements are ``floated``,
  positioned ``absolutely`` or ``relatively``, or other complex features
  are used. See
  http://dev.w3.org/csswg/css2/ and http://www.w3.org/Style/CSS/current-work
  for more details.
* Create layers to describe which parts of the page can be animated as a group
  without being re-rasterized. Each frame/render object is assigned to a layer.
* Textures are allocated for each layer of the page.
* The frame/render objects for each layer are traversed and drawing commands
  are executed for their respective layer. This may be rasterized by the CPU
  or drawn on the GPU directly using D2D/SkiaGL.
* All of the above steps may reuse calculated values from the last time the
  webpage was rendered, so that incremental changes require less work.
* The page layers are sent to the compositing process where they are combined
  with layers for other visible content like the browser chrome, iframes
  and addon panels.
* Final layer positions are computed and the composite commands are issued
  via Direct3D/OpenGL. The GPU command buffer(s) are flushed to the GPU for
  asynchronous rendering and the frame is sent to the window server.

GPU Rendering
-------------

* During the rendering process the graphical computing layers can use general
  purpose ``CPU`` or the graphical processor ``GPU`` as well.

* When using ``GPU`` for graphical rendering computations the graphical
  software layers split the task into multiple pieces, so it can take advantage
  of ``GPU`` massive parallelism for float point calculations required for
  the rendering process.


Window Server
-------------

Post-rendering and user-induced execution
-----------------------------------------

After rendering has been completed, the browser executes JavaScript code as a result
of some timing mechanism (such as a Google Doodle animation) or user
interaction (typing a query into the search box and receiving suggestions).
Plugins such as Flash or Java may execute as well, although not at this time on
the Google homepage. Scripts can cause additional network requests to be
performed, as well as modify the page or its layout, causing another round of
page rendering and painting.

.. _`"What happens when..."`: https://github.com/alex/what-happens-when
.. _`Alex Waynor`: https://github.com/alex
.. _`Creative Commons Zero`: https://creativecommons.org/publicdomain/zero/1.0/
.. _`"CSS lexical and syntax grammar"`: http://www.w3.org/TR/CSS2/grammar.html
.. _`Punycode`: https://en.wikipedia.org/wiki/Punycode
.. _`Ethernet`: http://en.wikipedia.org/wiki/IEEE_802.3
.. _`WiFi`: https://en.wikipedia.org/wiki/IEEE_802.11
.. _`Cellular data network`: https://en.wikipedia.org/wiki/Cellular_data_communication_protocol
.. _`analog-to-digital converter`: https://en.wikipedia.org/wiki/Analog-to-digital_converter
.. _`network node`: https://en.wikipedia.org/wiki/Computer_network#Network_nodes
.. _`TCP congestion control`: https://en.wikipedia.org/wiki/TCP_congestion_control
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

# Configurar PKI


## 1.- Ámbito del problema

Se quiere configurar una infraestructura de clave pública (Public Key Infrastructure) con el objetivo de autenticar a otros en la red. 

Una PKI, resumiendo algo, es un conjunto de máquinas, que certifican ante el resto de usuarios de la red que un usuario o servicio es quien dice ser. Esto se hace mediante el uso de claves públicas y privadas.  

En el problema que nos atañe, se creará una entidad certificadora raíz (root CA) que sea capaz de certificar a otras entidades certificadoras subordinadas (intermediate CA) para que estas  hagan  el  trabajo  de  certificar  a  terceros.  Una  de  las  entidades  certificadoras subordinadas, certificará a un servidor https con el que más tarde se realizará una conexión para ver si todo ha salido bien. 

### 1.1.- ¿Por qué dos tipos de entidad? 

Que haya dos entidades certificadoras, una raíz y otra subordinada supone una capa de seguridad más a la infraestructura, ya que si una de las entidades cae, por el motivo que sea, hay que revocar en cadena todos los certificados que haya firmado esta entidad.  

Si se es precavido y se protege a la entidad certificadora más importante, la raíz, apagando y desconectando esta máquina de la red, al producirse un accidente, no habría que volver a montar  la  infraestructura.  Bastaría  con  revocar  los  certificados  a  partir  de  la  entidad comprometida, levantar la CA raíz, volver a certificar alguna entidad intermedia y volver a apagarla. 
# 
## 2.- Configurar CA Raíz 

Voy a usar una máquina virtual en Microsoft Azure. No tiene mucha capacidad de cómputo ni especificaciones técnicas, pero tampoco es necesario, ya que en el momento que  esta entidad certifique a una subordinada, se apagará y desconectará de la red. También voy a usar OpenSSL para configurar la entidad certificadora. 

Una vez montada e iniciada la máquina virtual (Debian 10 en mi caso) hay que configurar el sistema de clave privada/pública. 

Lo primero es actualizar los repositorios y  actualizar el sistema. ![](Docs/Configura\_PKI.002.png)  

Una  cosa  a  tener  en  cuenta  tras  actualizar  el  sistema,  es  que si las entidades raíz y subordinada están en husos horarios distintos, sus horas locales podrían diferir, por lo que podría haber problemas. Para solucionar este inconveniente, simplemente hay que instalar ntp en ambas máquinas. 

![](Docs/Configura\_PKI.003.png)

Ahora hay que crear la estructura de directorios y archivos para la CA raíz. Esta estructura la voy a crear en el directorio / por tanto voy a necesitar cambiar al usuario root por unos instantes. 

![](Docs/Configura\_PKI.004.png)

Los  archivos  y  directorios  creados  son  necesarios  para  la  posterior  configuración  de OpenSSL. 

Ahora hay que llevar un archivo de configuración a la máquina. Este archivo es el siguiente: [openssl_root.cnf](https://raw.githubusercontent.com/cotelus/ConfigurePKI/main/plantillas_configuracion/openssl_root.cnf). Para ello, puede hacerse con scp, o ayudarnos del repositorio ya que está subido ahí. Para descargarlo del repositorio, se requiere la visualización de los archivos sin el código html propio de la página. 

Con el método que sea, se ubica el archivo *openssl\_root.conf* en el directorio que hemos preparado /root/ca 

![](Docs/Configura\_PKI.005.png)El  archivo  *openssl\_root.conf*  es  una  plantilla.  En  mi  caso  solo  necesito  modificar *dir* y DOMAINNAME para que coincida con mi estructura de directorios y su ubicación. 

![](Docs/Configura\_PKI.006.png)

![](Docs/Configura\_PKI.007.png)

### 2.1.- Generar clave privada CA raíz y firmar certificado 

En el directorio anterior, donde está ubicada toda la estructura de ficheros y archivos, se genera con openssl la clave privada. 

![](Docs/Configura\_PKI.008.png)

Ahora se genera el certificado de la CA raíz con la clave privada que se acaba de generar. 

![](Docs/Configura\_PKI.009.png)
(Copio la orden aquí ya que la imagen se ve muy pequeña) 
*$  openssl  req  -config  openssl\_root.cnf  -new  -x509  -sha512  -extensions  v3\_ca  -key /root/ca/private/ca.COTELO.key.pem -out /root/ca/certs/ca.COTELO.crt.pem -days 3650 -set\_serial 0* 

La orden anterior, lo que hace es generar un certificado, pero con los siguientes requisitos: 

- El archivo de configuración de OpenSSL va a ser *openssl\_root.cnf* 
- El algoritmo de encriptación va a ser *sha512*  
- La clave para firmar el certificado va a ser */root/ca/private/ca.COTELO.key.pem*  
- El certificado generado va a ser */root/ca/certs/ca.COTELO.crt.pem* 
- El certificado va a durar 3650 días (aproximadamente 10 años) 
# 
## 3.- Configurar CA Subordinada

De forma muy similar a la CA Raíz, esta máquina va a ser configurada en Azure con las mismas características. 

Después de actualizar la máquina e instalar ntp, se procede a crear la estructura de archivos y directorios de manera similar a la CA raíz. 

![](Docs/Configura\_PKI.010.png)

En la dirección /root/subordinada voy a descargar el archivo [openssl_intermediate.cnf](https://raw.githubusercontent.com/cotelus/ConfigurePKI/main/plantillas_configuracion/openssl_intermediate.cnf) 

![](Docs/Configura\_PKI.011.png)

De manera similar a la CA raíz, modifico los valores dir y DOMAINNAME para que se ajusten a mi problema. 

![](Docs/Configura\_PKI.012.png)

### 3.1.- Generar clave privada y petición de firma CA subordinada

Al contrario que en la CA raíz, esta entidad subordinada no puede firmarse a sí misma, ya que depende de otra entidad. En este caso, se genera un certificado de petición de firma que se enviará a la CA raíz, esta lo firmará y lo devolverá ya firmado a la CA subordinada. 

Aún así, lo primero es generar la clave privada de esta CA subordinada. 

![](Docs/Configura\_PKI.013.png)

### 3.2.- Enviar petición de CA subordinada a CA raíz 

Uno de los archivos que generó la orden anterior fué */root/subordinada/csr/int.COTELO.csr*. Este archivo tiene una extensión .csr, esta extensión significa “solicitud de firma de certificado” en nuestro idioma. Este tipo de archivos se generan en el servidor donde van a ser utilizados para enviarlos a la entidad certificadora correspondiente para que acredite la identidad. 

Para este caso, hay que enviar este archivo a la máquina CA raíz. Para ello voy a usar el comando scp, haciendo que la máquina CA raíz sea la que se encargue del tema del transporte.

![](Docs/Configura\_PKI.014.png)

Ahora, desde la máquina CA raíz se firma el archivo .csr 

![](Docs/Configura\_PKI.015.png)

Las opciones de openssl son muy similares a cuando se autofirma la CA raíz, siendo la diferencia más notable la opción  *“-extensions v3\_intermediate\_ca”* que es la que dirá que se está firmando un certificado para una CA subordinada. 

### 3.3.- Cadena de certificados

Hecho esto hay que crear la cadena de certificados. Esto es un documento en el que se almacena también el certificado de la CA raíz para dar autoridad al documento. Al requerir del certificado de la CA raíz, esta operación se hace también en la CA raíz. 

![](Docs/Configura\_PKI.016.png)

Finalmente,  lo  que  hay  que  hacer  es  enviar  los  archivos */root/ca/requests/int.COTELO.crt.pem*  y  */root/ca/requests/chain.COTELO.crt.pem*  a  la máquina que va a ser la CA subordinada. 

![](Docs/Configura\_PKI.017.png)

![](Docs/Configura\_PKI.018.png)

Como  scp  no  puede  ejecutarse  como  sudo  en  la  máquina  destino  y  el  directorio /root/subordinada/ de este, pertenece a root, envío los certificados al directorio personal del usuario cotelo. Suponiendo que este usuario se ha creado sola y exclusivamente para el manejo de los certificados. 

Una vez en la máquina CA subordinada, copio los certificados que se han transferido desde la CA raíz a la estructura de directorios que se montó anteriormente para ello. 

![](Docs/Configura\_PKI.019.png)

En este punto, no va a ser necesaria la CA raíz hasta que haya que renovar los certificados de las CA subordinadas. Es por tanto, que puede apagarse la máquina. 

# 
## 4.- CA subordinada certifica servidor HTTPS

En  este  punto,  ya  se  puede  montar  un  servidor  https  con  apache  y  que  la  entidad certificadora subordinada que se montó anteriormente, certifique al servidor. 

Hacer esto es muy similar a que la CA raíz firme a la CA subordinada. De hecho, es el mismo  procedimiento.  En  este  caso  es  el  servidor  https  el  que  crea el documento de petición de firma y se lo envía a la CA subordinada para que lo firme. 

En la máquina que hará de servidor https, se crea la siguiente estructura de directorios: 

![](Docs/Configura\_PKI.020.png)

Se genera la clave privada para el servidor https y la petición de firma: 

![](Docs/Configura\_PKI.021.png)

![](Docs/Configura\_PKI.022.png)

Mediante scp se lleva el archivo *root/auth/csr/servidor\_https.csr.pem* a la CA subordinada para  que  lo  firme.  Además  he  copiado  de  forma  temporal  el  archivo  al  directorio /home/cotelo/ para que el usuario que hace el scp tenga acceso. 

![](Docs/Configura\_PKI.023.png)

Ahora se firma la petición desde la CA subordinada.

![](Docs/Configura\_PKI.024.png)

Finalmente, se envía desde la CA subordinada al servidor HTTPS el certificado firmado. 

![](Docs/Configura\_PKI.025.png)

Se envía también el certificado de la CA subordinada (el certificado encadenado Ca raíz y CA subordinada). 

![](Docs/Configura\_PKI.026.png)

Se comprueba que el módulo del certificado y de la clave privada del servidor https son iguales. 

![](Docs/Configura\_PKI.027.png)Si todo coincide, se sigue adelante. Desde el servidor https ahora se mueve el certificado firmado ![](Configura\_PKI.001.png)a la estructura de directorios que se creó. 

![](Docs/Configura\_PKI.028.png)

Sólo falta enviar el certificado de la CA subordinada, el certificado del servidor y la clave privada del servidor al directorio del que los vaya a cargar apache. 

![](Docs/Configura\_PKI.029.png)

Se modifica el archivo de configuración de ssl de apache2. 

![](Docs/Configura\_PKI.030.png)

Se  activa  el  módulo  ssl  de  apache2  y  se  reinicia el servicio para que los cambios se apliquen. 

![](Configura\_PKI.031.png)

Se activa ssl en apache2 y se recarga el servicio 

![](Docs/Configura\_PKI.032.png)

# 
## 5.- Para finalizar

Si todo ha ido bien, el servidor apache2 va a enviar los certificados con las consultas https. No obstante, el navegador que se use no va a tener instalados los certificados de la PKI que se ha creado. 

![](Docs/Configura\_PKI.034.png)

Para que el certificado sea válido en el navegador, habrá que instalarlo en los certificados de confianza del ordenador. 

![](Docs/Configura\_PKI.035.png)

` `El certificado raíz puede encontrarse aqui: [caCotelo](https://raw.githubusercontent.com/cotelus/ConfigurePKI/main/certificado_ca/ca.COTELO.crt.pem)

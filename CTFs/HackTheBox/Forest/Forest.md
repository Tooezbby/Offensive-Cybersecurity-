Task 1

For which domain is this machine a Domain Controller?


Primero como siempre enumeramos con nmap para ver que puertos tiene abiertos la máquina. Aqui ya podemos fijarnos que seguramente hay un DC debido a los puertos abiertos y los servicios que corren en ellos: 53:UDP, 88:Kerberos, 389:LDAP,445:SMB,3268:CATLOG_LDAP,5985:WinRM

(images/namp.png)

Al ver que tenemos el puerto de SMB intentamos enumerarlo.

(images/2_intento_smb_enum.png)

Sin credenciales no podemos aceder a nada.


Después de la primera enumeración con Nmap, observamos varios servicios relacionados con Active Directory, como DNS, Kerberos, LDAP y SMB, por lo que podemos sospechar que estamos ante un Domain Controller.

Para confirmar esta información y obtener el nombre del dominio podemos utilizar diferentes herramientas. En este caso utilizaremos dos métodos:

NetExec realiza una conexión contra el servicio SMB y obtiene información que Windows expone durante la negociación, como el hostname de la máquina, el dominio asociado y el sistema operativo.

En la salida podemos observar:

- **Hostname:** FOREST
- **Dominio:** htb.local

(images/task1_netexec_domain_name.png)

(images/task1_nmap_domain_name.png)

Una de las formas de obtener información básica del dominio es consultando el servicio LDAP mediante el script `ldap-rootdse` de Nmap. Que devuelve lo mismo que utilizando la herramienta ldapsearch y preguntando por los datos basicos al dominio.

Task 2

Which of the following services allows for anonymous authentication and can provide us with valuable information about the machine? FTP, LDAP, SMB, WinRM

Aqui como hemos visto antenriormente SMB no lo permite. Y también al ejecutar el nmap con el script ldap-rootsde he comparado salidas con el ldapsearch
`ldapsearch -x -H ldap://10.129.95.210 -s base`

Y la salida exitoso nos dice si que da información útil de la máquina.

*No pongo captura por que literalmente es la de nmap con el script de ldap*


Task 3

Which user has Kerberos Pre-Authentication disabled?

Lo primero que tenemos que hacer para ver que usuario tiene desactivado el pre-auth es saber que usuario existen en el dominio.

(images/elchurroquenomeva.png)


(images/probamos con el otro churro.png)

(images/creamosUsers.png)

HAY QUE HACER CAPTURA DEL USER.TXT BUENO EL QUE TIENE 20 USUARIOS.
metemos Administrator y Guest


---
NADA DE ESTO FUNCIONA DONDE COÑO ESTA SVC_aLFRESCO

enumeramos rpd

(images/aquiestaalfresco)

por que puedo enumerar en rpd, se puede siempre? esta mal configurado?
por que me sale alfrescoaqui y no en el otro lado




Task 4

What is the password of the user svc-alfresco?

(images/task4_aalfresco_hash)

Despues de utilizar herramientas como hashcat o john ripper logramos sacar la contraseña:

S3rvice

Task 5

To what port can we connect with these creds to get an interactive shell?

Aqui volvemos a mirar NMAP vemos que nuestro puerto de WinRM está abierto (5985)

intentamos conectarnos remotamente con evil-WinRM

(images/task5_WinRM)

Task 6

Submit the flag located on the svc-alfresco user's desktop.
*Navegar al escritorio y mirar la flag, no hay que hacer "nada"*





Task 7

Which group has WriteDACL permissions over the HTB.LOCAL domain? Give the group name without the `@htb.local`.




task 8
Dsync y explicar todo

task 9

paso 1 - añadir y comprobar
Paso 2 — Añade a svc-alfresco al grupo Exchange Windows Permissions:

paso 3- otorgarte a ti mismo los derechos de DCSync:

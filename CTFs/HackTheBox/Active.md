
# HackTheBox - Active

## Tasks

> ## Task 1
> **How many SMB shares are shared by the target?**

Lo primero fue enumerar los recursos compartidos SMB del servidor para identificar posibles puntos de acceso a información sensible.


```
smbclient -L //10.10.10.100 -N
```
 - -L para listar
 - -N para que intente autenticarse sin contraseña

Al listar los recursos compartidos aparecieron varios shares, entre ellos:

ADMIN$
C$
IPC$
NETLOGON
SYSVOL
Replication

En este paso simplemente estamos realizando reconocimiento.


> [!danger] Task 2
> **What is the name of the share that allows anonymous read access?**

Tras enumerar los shares, comprobamos cuáles permiten acceso sin credenciales.


```
smbmap -H 10.10.10.100
```

El resultado mostró que el recurso **Replication** tenía permisos de lectura para usuarios anónimos.


> [!danger] Task 3
> **Which file has encrypted account credentials in it?**

Accedemos al share **Replication** para revisar su contenido.

La carpeta contiene archivos utilizados para la gestión automática de políticas de grupo (GPP - Group Policy Preferences). Durante la enumeración encontramos un archivo llamado:

```
Groups.xml
```

![[Pasted image 20260712054811.png]]

Este archivo contiene credenciales almacenadas mediante el atributo **cpassword**.

> [!danger] Task 4
> **What is the decrypted password for the SVC_TGS account?**

Una vez localizado el valor **cpassword** dentro de _Groups.xml_, utilizamos una herramienta capaz de descifrarlo.

En este caso:

```
gpp-decrypt <cpassword>
```

![[Pasted image 20260712054825.png]]
- **Algoritmo:** AES-256
- **Modo:** CBC (Cipher Block Chaining)
- **Clave:** una clave AES pública proporcionada por Microsoft

En este punto conseguimos nuestras primeras credenciales válidas dentro del dominio.



> [!danger] User Flag
> **Submit the flag located on the security user's desktop.**

Con las credenciales obtenidas del usuario **SVC_TGS**, podemos acceder a recursos del dominio y enumerar usuarios disponibles.

![[Pasted image 20260712054842.png]]
Posteriormente establecemos una sesión remota, en este caso utilizando smbclient, las credenciales del SVC_TGS y accediendo al recurso /Users.

![[Pasted image 20260712054918.png]]
![[Pasted image 20260712054931.png]]
Una vez estamos dentro navegamos un poco por los directorios y nos topamos con user.txt en el escritorio SVC_TGS. Hacemos un mget para poder traer el archivo a nuestro equipo y hacer un cat.

> [!danger] Task 6
> **Which service account on Active is vulnerable to Kerberoasting?**

Después de obtener las credenciales del usuario SVC_TGS mediante la vulnerabilidad de Group Policy Preferences, se realizó una consulta LDAP contra el controlador de dominio para enumerar usuarios activos del dominio.

El objetivo de esta enumeración es identificar cuentas asociadas a servicios mediante sus SPN, ya que estas cuentas pueden ser vulnerables a ataques Kerberoasting. La información obtenida permitirá solicitar tickets Kerberos y realizar un ataque offline para recuperar posibles contraseñas.
![[Pasted image 20260712054950.png|697]]
![[Pasted image 20260712055009.png]]

Una vez obtenidas las credenciales de SVC_TGS, utilicé GetADUsers para enumerar los usuarios del dominio y obtener algo más de información sobre ellos, como la fecha del último cambio de contraseña o el último inicio de sesión.

Una vez enumerados los usuarios del dominio, utilicé `GetUserSPNs` para buscar cuentas con un Service Principal Name (SPN) asociado. La herramienta identificó que la cuenta `Administrator` tenía registrado el SPN `active/CIFS:445`, lo que la convertía en candidata para un ataque Kerberoasting.
![[Pasted image 20260712055101.png]]

A continuación ejecuté la misma herramienta con la opción -request, que además de enumerar los SPN solicita el ticket Kerberos correspondiente. Como resultado se obtuvo un ticket TGS en formato compatible con Hashcat ($krb5tgs$), que posteriormente fue utilizado para intentar recuperar la contraseña de la cuenta Administrator mediante cracking offline.

> [!danger] Task 7
> **What is the plaintext password for the administrator account?**

Después de obtener el ticket Kerberos mediante Kerberoasting, guardamos el hash en un archivo.

Ejemplo:

```
nano hash.txt
```

A continuación utilizamos Hashcat para intentar recuperar la contraseña:

```
hashcat -m 13100 hash.txt rockyou.txt
```

Donde:

- `13100` corresponde al modo Kerberos TGS-REP RC4.
- `hash.txt` contiene el ticket obtenido.
- `rockyou.txt` es el diccionario utilizado.

Tras el proceso de cracking obtenemos la contraseña en texto plano del usuario **Administrator**.

> [!danger] Root Flag
> **Submit the flag located on the administrator's desktop.**

Una vez obtenida la contraseña de la cuenta **Administrator** mediante Kerberoasting, procedí a utilizar la herramienta `wmiexec` de Impacket para conectarme remotamente al controlador de dominio utilizando las credenciales recuperadas.

![[Pasted image 20260712055143.png]]
wmiexec permite ejecutar comandos remotos a través de WMI (Windows Management Instrumentation) sin necesidad de abrir una sesión gráfica. Tras autenticarnos correctamente, obtuvimos una consola semi-interactiva sobre el sistema objetivo.

![[Pasted image 20260712055154.png]]

Después de navegar un poco... TACHAAAN! root.txt Finalizado el CTF.

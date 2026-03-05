tags:
___
## 🧠 Concepto clave
Los miembros del grupo **Hyper-V Administrators** tienen acceso completo a todas las funcionalidades de Hyper-V. Si los Domain Controllers están virtualizados, estos administradores deben considerarse equivalentes a Domain Admins — pueden clonar el DC en vivo, montar el disco virtual offline y extraer el `NTDS.dit` con todos los hashes del dominio. Adicionalmente, existe un vector de escalada local basado en el comportamiento de `vmms.exe` al eliminar máquinas virtuales. 📄 [Hyper-V Administrators — Microsoft Docs](https://docs.microsoft.com/en-us/windows/security/identity-protection/access-control/active-directory-security-groups#hyper-v-administrators)

| Concepto | Qué es | Riesgo |
|---|---|---|
| **Hyper-V Administrators** | Grupo built-in con acceso total a todas las funciones de Hyper-V del sistema | Si hay DCs virtualizados, equivale a Domain Admin — acceso directo a NTDS.dit clonando el DC |
| **Hard Link (NTFS)** | Entrada del sistema de archivos que apunta al mismo bloque de datos que otro archivo, comportándose como si fuera el archivo original | Puede usarse para redirigir operaciones privilegiadas del sistema hacia archivos protegidos, obteniendo permisos sobre ellos |
| **vmms.exe** | Servicio de gestión de máquinas virtuales de Hyper-V, corre como `NT AUTHORITY\SYSTEM` | Al eliminar una VM, intenta restaurar permisos sobre el `.vhdx` como SYSTEM sin impersonar al usuario — esto puede explotarse con un hard link para obtener permisos sobre archivos protegidos |

## 📌 Puntos importantes

### Vector 1 — Clonado de DC virtualizado

Si los Domain Controllers están virtualizados en Hyper-V, un miembro de este grupo puede crear una copia del DC en vivo y montar su disco `.vhdx` offline para acceder directamente a `NTDS.dit` sin necesidad de pasar por el sistema operativo activo. El proceso es el mismo que en el ataque de Backup Operators: extraer NTDS.dit + SYSTEM del registro y procesarlos con secretsdump o DSInternals para obtener todos los hashes del dominio. 📄 [Gestión de VMs — Windows Admin Center](https://docs.microsoft.com/en-us/windows-server/manage/windows-admin-center/use/manage-virtual-machines)

### Vector 2 — Escalada local mediante vmms.exe y Hard Links

Este vector, documentado en detalle en 📄 [From Hyper-V Admin to SYSTEM — decoder.cloud](https://decoder.cloud/2020/01/20/from-hyper-v-admin-to-system/), se basa en el siguiente comportamiento: cuando se elimina una VM, `vmms.exe` intenta restaurar los permisos originales del archivo `.vhdx` asociado, ejecutándolo como `NT AUTHORITY\SYSTEM` **sin impersonar al usuario**. Si antes de que se produzca esa restauración se elimina el `.vhdx` y se crea un **hard link NTFS** en su lugar apuntando a un archivo protegido del sistema, `vmms.exe` aplicará los permisos sobre ese archivo protegido — otorgando control total sobre él al atacante.

El PoC que implementa este hard link está disponible en: 📄 [hyperv-eop.ps1 — decoder-it](https://raw.githubusercontent.com/decoder-it/Hyper-V-admin-EOP/master/hyperv-eop.ps1)

### Condiciones para el ataque y CVEs relacionados

Si el sistema es vulnerable a **CVE-2018-0952** 📄 [CVE-2018-0952 — Tenable](https://www.tenable.com/cve/CVE-2018-0952) o **CVE-2019-0841** 📄 [CVE-2019-0841 — Tenable](https://www.tenable.com/cve/CVE-2019-0841), el ataque es directo. En sistemas parcheados, se puede buscar una aplicación instalada en el servidor que cumpla dos condiciones:

- Tiene un servicio que corre como SYSTEM
- Ese servicio puede ser iniciado por un usuario sin privilegios

Un ejemplo clásico es **Firefox**, que instala el **Mozilla Maintenance Service** — un servicio que corre como SYSTEM y puede iniciarse sin privilegios de administrador. El flujo consiste en usar el hard link para obtener permisos sobre el ejecutable del servicio, reemplazarlo por un binario malicioso y arrancar el servicio para obtener ejecución como SYSTEM.

> ⚠️ **Este vector fue mitigado por las actualizaciones de seguridad de Windows de marzo de 2020**, que modificaron el comportamiento del sistema respecto a los hard links. En sistemas completamente parcheados este ataque específico no funcionará, pero la lógica del hard link puede aplicarse a otros escenarios según el entorno.

---

### Flujo de explotación — Hard Link + Mozilla Maintenance Service

**Paso 1 — Ejecutar el PoC para obtener permisos sobre el ejecutable del servicio**

Tras ejecutar el script `hyperv-eop.ps1`, el hard link redirige la operación de `vmms.exe` hacia el ejecutable del Mozilla Maintenance Service, otorgando control total sobre él al usuario actual.

El archivo objetivo es:
```
C:\Program Files (x86)\Mozilla Maintenance Service\maintenanceservice.exe
```

**Paso 2 — Tomar posesión del archivo**

> [!tip] Tomar posesión del ejecutable del Mozilla Maintenance Service
> ```cmd
> takeown /F "C:\Program Files (x86)\Mozilla Maintenance Service\maintenanceservice.exe"
> ```
> `takeown` — Utilidad nativa de Windows para cambiar el propietario de un archivo o directorio.
> `/F` — Especifica la ruta completa del archivo del que se tomará posesión.

**Paso 3 — Reemplazar el ejecutable por uno malicioso y arrancar el servicio**

Con control total sobre el archivo, se reemplaza `maintenanceservice.exe` por un binario malicioso (una reverse shell, un payload de Meterpreter, o simplemente un `cmd.exe` renombrado) y se inicia el servicio para obtener ejecución como SYSTEM:

> [!tip] Iniciar el Mozilla Maintenance Service con el ejecutable malicioso
> ```cmd
> sc.exe start MozillaMaintenance
> ```
> `sc.exe start` — Inicia el servicio especificado. Al arrancar, ejecutará el binario malicioso que reemplazó al `maintenanceservice.exe` original, en el contexto de `NT AUTHORITY\SYSTEM`.
> `MozillaMaintenance` — Nombre interno del servicio Mozilla Maintenance Service.

---

## 🔗 Relaciones / Contexto

El grupo **Hyper-V Administrators** es otro ejemplo de grupo que, aunque no parece crítico a primera vista, otorga en la práctica un nivel de acceso equivalente al de Domain Admin cuando hay DCs virtualizados. Durante un assessment, cualquier cuenta en este grupo debe marcarse como hallazgo de alto riesgo. El vector del hard link, aunque parcheado en su forma original, ilustra una clase de ataques más amplia basada en abusar de operaciones privilegiadas del sistema sobre rutas controlables por el atacante — una técnica que reaparece en distintas formas en otros contextos de escalada de privilegios en Windows.
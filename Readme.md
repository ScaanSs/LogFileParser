# LogFileParser — by scaan

Analiza el **`$LogFile` de NTFS** (el registro de transacciones del sistema de archivos) y muestra qué archivos y carpetas se **crearon, renombraron o borraron**. También muestra los registros del **`$UsnJrnl` que siguen guardados dentro del log**.

Usa las mismas columnas que JournalParser, porque es como mejor se lee la información:

| Name | Date | Reason | Directory |
|------|------|--------|-----------|
| Nombre del archivo o carpeta | Fecha y hora (hora local) | Qué pasó | Carpeta donde estaba |

---

## Requisitos

- Windows 10 / 11.
- Una unidad **NTFS**.
- **Ejecutar como administrador**: para leer el `$LogFile` hay que leer el disco directamente. El programa ya pide los permisos al abrirse.

---

## Uso

1. Abre `LogFileParser.exe` y acepta el aviso de administrador.
2. Al arrancar analiza automáticamente la unidad del sistema (normalmente `C:`).
3. Para analizar otra unidad, elígela en el desplegable de arriba a la derecha y pulsa **Parse**.
4. Para analizar un `$LogFile` exportado de otro equipo, por ejemplo con FTK Imager, tienes tres opciones:
   - pulsar **Open file...**,
   - arrastrar el archivo encima del `.exe`,
   - pasarlo por línea de comandos: `LogFileParser.exe C:\ruta\$LogFile`.

### En la tabla

- **Clic** en una fila: la deja marcada, para seguirla con la vista.
- **Doble clic** en una fila: abre el **historial de ese archivo** (todo lo que le pasó y que aún queda en el log).
- **Clic derecho** en una fila:
  - Copy name
  - Copy folder path
  - Copy whole row
  - Open folder
  - Show file history
- **Pasar el ratón** por encima de una fila muestra el nombre, la ruta, el origen del evento, el LSN y el registro MFT.
- **Clic en el título de una columna** ordena por esa columna.

### Filtros (barra lateral)

| Filtro | Reason que muestra |
|--------|--------------------|
| Created | `File Create` |
| Deleted | `File Delete` |
| Renamed: new name | `Rename New Name` |
| Renamed: old name | `Rename Old Name` |
| Attributes changed | `Basic Info Change` |
| Stream changed | `Stream Change` |
| Content truncated | `Data Truncation` |

**Source** permite ver todo junto o solo un origen:
- **$LogFile transactions**: eventos reconstruidos a partir de las operaciones de NTFS.
- **USN records in $LogFile**: entradas del `$UsnJrnl` que siguen dentro del log, con fecha exacta.

**Clear all filters** quita todos los filtros de una vez.

### Buscador

Usa la misma sintaxis que JournalParser. No distingue mayúsculas de minúsculas y la búsqueda se aplica al pulsar **Enter**.

- Sin indicar columna, se busca en el nombre.
- Columnas disponibles: `name:`, `reason:`, `directory:` (o `dir:`) y `date:`.
- `;` une varias condiciones, y deben cumplirse todas.
- Operadores dentro de una condición: `&&` (y), `||` (o), `!!` (y no).

```
.exe                          → archivos cuyo nombre contiene .exe
Name:.exe;Reason:file delete  → .exe que fueron borrados
.exe;reason:rename            → .exe que se renombraron
dir:temp;name:.ps1||.bat      → scripts dentro de una carpeta Temp
reason:create!!close          → creaciones sin "Close"
date:2026-10-22 14            → eventos del 22/10/2026 entre las 14:00 y las 14:59
```

### Accesibilidad

- **Zoom del texto** del 85% al 200%:
  - botones `A-` / `A+`,
  - `Ctrl +`, `Ctrl -` y `Ctrl 0` (volver al 100%),
  - `Ctrl` + rueda del ratón.
- Se adapta a la escala de Windows (125%, 150%…).
- **Tema claro y oscuro** con el botón *Light mode / Dark mode*.
- El zoom y el tema se guardan en `%LOCALAPPDATA%\LogFileParser\settings.ini`.
- Los colores de Reason los pueden distinguir también las personas daltónicas, y cada uno lleva además un símbolo:
  - `+` creado (azul)
  - `×` borrado (naranja)
  - `→` renombrado (amarillo)

### $LogFile Info

El botón **$LogFile Info** muestra:
- tamaño del log,
- desde y hasta qué fecha llegan los eventos,
- versión del log y tamaño de página,
- páginas válidas, vacías y dañadas,
- rango de LSN,
- fechas del propio `$LogFile`.

Si el `$LogFile` mide menos de los 64 MiB que pone Windows por defecto, muestra un **aviso**: alguien puede haberlo reducido con `chkdsk /L:<tamaño>` para que guarde menos historial.

---

## Qué detecta y cómo

| Reason | Cómo se obtiene |
|--------|-----------------|
| `File Create` | Se inicializa un registro de la MFT y se añade su nombre a la carpeta, en la misma transacción |
| `File Delete` | Se quita el nombre de la carpeta y se libera su registro de la MFT |
| `Rename Old Name` / `Rename New Name` | Se quita un nombre y se añade otro para el mismo registro de la MFT |
| `Hard Link Change \| Index Entry Add/Delete` | Se añade o quita un nombre sin que se cree ni se borre el archivo (enlaces duros, movimientos parciales) |
| Reasons del USN (`Data Extend`, `Close`, `Basic Info Change`…) | Registros `USN_RECORD_V2` que NTFS escribió en el `$UsnJrnl` a través del log |

Además:
- Las reorganizaciones internas de los índices de carpetas (una entrada que se quita y se vuelve a poner igual) **se descartan**, para que no aparezcan como cambios falsos.
- La columna **Directory** se calcula así:
  1. Se busca la carpeta en el disco a partir de su referencia de archivo.
  2. Si la carpeta ya no existe, la ruta se **reconstruye con los nombres que aparecen en el propio log**. Esto funciona también con carpetas borradas y con copias exportadas.
  3. Si falta algún tramo, aparece como `[MFT#número]`.
- El historial de un archivo se agrupa por **registro MFT + número de secuencia**. NTFS reutiliza los registros de los archivos borrados para archivos nuevos, pero cada vez sube el número de secuencia, así que un archivo nuevo que ocupa el registro de uno borrado sale en un historial aparte.

---

## Limitaciones importantes

- **El `$LogFile` no guarda el historial desde que se instaló Windows.**
  - Es un archivo **circular** de tamaño fijo (64 MiB por defecto) y se sobrescribe continuamente.
  - En el disco del sistema suele cubrir **desde unos minutos hasta unas pocas horas** de actividad.
  - LogFileParser lee el log **entero** (todas las páginas, no solo la parte activa), pero lo que ya se sobrescribió no se puede recuperar.
  - El periodo que cubre cada análisis se ve abajo (*Oldest / Newest*) y en **$LogFile Info**.
- **Fechas:** NTFS no guarda la hora de cada transacción en el `$LogFile`.
  - Las creaciones usan la fecha de creación del archivo.
  - El resto de transacciones (borrados, renombres) toman la hora del registro USN más cercano en el log, así que es **aproximada**.
  - Los eventos de origen *USN records in $LogFile* tienen fecha **exacta**.
- Para historial más antiguo hay que combinarlo con otros artefactos:
  - `$UsnJrnl` (JournalParser): días o semanas.
  - `$MFT`.

---

## Compilar

Código fuente:

```
LogFileParser/
├─ main.cpp                 Interfaz (ImGui + DirectX 11)
├─ logfile/
│  ├─ ntfs_volume.hh        Lectura del disco: boot sector, MFT registro 2, data runs
│  ├─ logfile_reader.hh     Páginas RSTR/RCRD, fixups, log records
│  └─ logfile_events.h      Reconstrucción de eventos, rutas, filtros
├─ d3d11/d3dx.hh            Dispositivo D3D11 y WndProc
├─ imgui/                   Dear ImGui 1.92.3
├─ app.manifest             Pide administrador
└─ resource.rc
```

- **Visual Studio**: abre *x64 Native Tools Command Prompt for VS* y ejecuta `build.bat`.
- **MSYS2 UCRT64**: ejecuta `sh build_mingw.sh` (necesita g++ 13 o superior, por `std::format`).

El resultado es `build\LogFileParser.exe`, que ya incluye todo lo necesario para ejecutarse (no requiere DLLs extra).

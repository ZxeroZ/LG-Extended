# LG Extended - Documentación Completa

## Tabla de Contenidos

1. [Resumen del Proyecto](#1-resumen-del-proyecto)
2. [Arquitectura](#2-arquitectura)
3. [Cómo Funcionan los Hooks](#3-cómo-funcionan-los-hooks)
   - 3.1 [Conceptos Fundamentales](#31-conceptos-fundamentales)
   - 3.2 [ModPrefs - Sistema de Comunicación](#32-modprefs---sistema-de-comunicación-entre-hooks-y-ui)
   - 3.3 [BatteryHook](#33-batteryhook---personalización-del-icono-de-batería)
   - 3.4 [DpiHook](#34-dpihook---cambio-de-dpi-por-aplicación)
   - 3.5 [RecentsHook](#35-recentshook---estilo-ios-para-multitasking)
   - 3.6 [FlagSecureHook](#36-flagsecurehook---bypass-de-restricciones)
   - 3.7 [SettingsHook](#37-settingshook---personalización-de-ajustes)
   - 3.8 [LauncherHook](#38-launcherhook---orden-automático-de-apps)
   - 3.9 [QSPanelHook](#39-qspanelhook---panel-de-quick-settings-miui)
   - 3.10 [ScrimHook](#310-scrimhook---efecto-blur-en-systemui)
   - 3.11 [NotificationHook](#311-notificationhook---estilo-de-notificaciones-miui)
   - 3.12 [SystemUIHook](#312-systemuihook---reemplazo-de-recursos)
   - 3.13 [Resumen de Todos los Hooks](#313-resumen-de-todos-los-hooks)
4. [Componentes UI](#4-componentes-ui)
5. [Gestión de Datos](#5-gestión-de-datos)
6. [Integración con Root](#6-integración-con-root)
7. [Configuración del Build](#7-configuración-del-build)
8. [Estructura de Archivos](#8-estructura-de-archivos)
9. [Dependencias](#9-dependencias)
10. [Guía de Instalación](#10-guía-de-instalación)
11. [Troubleshooting](#11-troubleshooting)

---

## 1. Resumen del Proyecto

**LG Extended** es un módulo Xposed diseñado específicamente para dispositivos LG (principalmente LG V60) que permite personalizar y modificar varios aspectos del sistema sin necesidad de modificar el firmware. El módulo interactúa con múltiples procesos del sistema para aplicar cambios en tiempo real.

### Información General

| Propiedad | Valor |
|-----------|-------|
| **Nombre** | LG Extended |
| **Paquete** | `com.zxerox.lg_extended` |
| **Versión** | 1.0 (versionCode: 1) |
| **Autor** | ZxeroX |
| **SDK Mínimo** | Android 10 (API 29) |
| **SDK Objetivo** | Android 15 (API 35) |
| **Framework** | Xposed API 82+ |
| **Requisitos** | LSPosed o similar + Root (Magisk/KernelSU/APatch) |

### Descripción

LG Extended proporciona una suite de mods para dispositivos LG que incluye:

- **Personalización del icono de batería** con múltiples estilos (iOS 26, iOS 17, OneUI 8, OneUI 9)
- **Cambio de DPI por aplicación** para ajustar la densidad de pantalla individualmente
- **Estilo iOS para recientes** que modifica el diseño del multitasking
- **Bypass de FLAG_SECURE** para permitir capturas de pantalla en apps restringidas
- **Personalización de iconos en Ajustes** con iconos estilo OneUI
- **Tarjeta de perfil personalizada** en la pantalla principal de Ajustes

---

## 2. Arquitectura

### Diagrama de Componentes

```
┌─────────────────────────────────────────────────────────────────────┐
│                        LG Extended App                              │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │                      MainHook.java                          │   │
│  │  (Punto de entrada Xposed - IXposedHookLoadPackage,         │   │
│  │   IXposedHookZygoteInit, IXposedHookInitPackageResources)   │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                              │                                      │
│     ┌────────────┬───────────┼───────────┬────────────┐            │
│     │            │           │           │            │            │
│     ▼            ▼           ▼           ▼            ▼            │
│ ┌────────┐ ┌────────┐ ┌──────────┐ ┌────────┐ ┌──────────┐      │
│ │Battery │ │ DpiHook│ │ Recents  │ │Flag    │ │Settings  │      │
│ │Hook    │ │        │ │ Hook     │ │Secure  │ │Hook      │      │
│ │(SysUI) │ │(Apps)  │ │(Launcher)│ │Hook    │ │(Settings)│      │
│ └────────┘ └────────┘ └──────────┘ │(android│ └──────────┘      │
│     │                              └────────┘       │            │
│     │                                               │            │
│ ┌───┴────┐ ┌────────┐ ┌──────────┐                  │            │
│ │QSPanel │ │Scrim   │ │Notification│                │            │
│ │Hook    │ │Hook    │ │Hook       │                │            │
│ │(SysUI) │ │(SysUI) │ │(SysUI)    │                │            │
│ └────────┘ └────────┘ └──────────┘                  │            │
│     │                                               │            │
│ ┌───────────┐                                       │            │
│ │LauncherHook│ ◀────────────────────────────────────┘            │
│ │(Launcher) │                                                     │
│ └───────────┘                                                     │
│     │                                                             │
│  ┌──┴──────────────────────────────────────────────────────┐    │
│  │                  SystemUIHook                            │    │
│  │         (IXposedHookInitPackageResources)                │    │
│  │         Reemplaza recursos: drawables, colores, dimens   │    │
│  └──────────────────────────────────────────────────────────┘    │
│                                                                     │
│  ┌───────────────────────────────────────────────────────────┐    │
│  │                     ModPrefs (ContentProvider)             │    │
│  │              Almacenamiento centralizado de prefs          │    │
│  │      ← Comunicación entre UI y todos los hooks →           │    │
│  └───────────────────────────────────────────────────────────┘    │
│                                                                     │
│  ┌───────────────────────────────────────────────────────────┐    │
│  │                    UI Layer (Activities)                   │    │
│  │  MainActivity | BatteryStyleActivity | DpiActivity | ...  │    │
│  └───────────────────────────────────────────────────────────┘    │
│                                                                     │
│  ┌───────────────────────────────────────────────────────────┐    │
│  │                  Root Utils (libsu)                        │    │
│  │            DeviceInfoProvider | RootUtils                   │    │
│  └───────────────────────────────────────────────────────────┘    │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### Ciclo de Vida del Hook

1. **Inicialización**: `MainHook.initZygote()` almacena la ruta del módulo (`MODULE_PATH`)
2. **Carga**: `MainHook.handleLoadPackage()` detecta el paquete objetivo y registra hooks según el paquete
3. **Registro**: Cada hook se registra para su paquete específico (con try/catch por si falla)
4. **Intercepción**: Los hooks modifican el comportamiento en tiempo real via `beforeHookedMethod` / `afterHookedMethod`
5. **Persistencia**: Las preferencias se almacenan vía `ModPrefs` ContentProvider
6. **Marcado**: MainHook marca cada hook exitoso con `hook_active_{nombre}` para que la UI sepa qué hooks están corriendo
7. **Logging**: Cada hook registrado es logueado con éxito/error vía `LogWriter`

#### Paquetes que Cada Hook Intercepta

```
handleLoadPackage(lpparam)
    │
    ├── "com.zxerox.lg_extended" → EXCLUIDO (no hook itself)
    │
    ├── CUALQUIER paquete:
    │       └── DpiHook (excepto la app misma)
    │
    ├── "com.android.systemui" o "com.lge.systemui":
    │       ├── QSPanelHook
    │       ├── ScrimHook
    │       ├── NotificationHook
    │       └── BatteryHook
    │
    ├── "com.lge.launcher3" o "com.android.launcher3":
    │       ├── RecentsHook
    │       └── LauncherHook
    │
    ├── "android":
    │       └── FlagSecureHook
    │
    └── "com.android.settings":
            └── SettingsHook
```

### Patrón de Comunicación

El módulo utiliza un **ContentProvider** (`ModPrefs`) como sistema de comunicación entre:
- La app de configuración (UI)
- Los hooks activos en diferentes procesos del sistema

```
┌──────────────────┐     ContentResolver      ┌──────────────────┐
│  MainActivity    │ ───────────────────────▶ │    ModPrefs      │
│  (UI Process)    │ ◀─────────────────────── │ (Module Process) │
└──────────────────┘     Query/Insert         └──────────────────┘
                                │
                                ▼
┌──────────────────┐     ContentObserver      ┌──────────────────┐
│  BatteryHook     │ ◀─────────────────────── │    ModPrefs      │
│  (SystemUI)      │     onChange()           │                  │
└──────────────────┘                          └──────────────────┘
```

---

## 3. Cómo Funcionan los Hooks

### 3.1 Conceptos Fundamentales

Un **hook** es una técnica que intercepta métodos de Android en tiempo de ejecución para modificar su comportamiento sin alterar el código fuente original. LG Extended usa el framework **Xposed** para esto.

#### Ciclo de Vida de un Hook

```
┌─────────────────────────────────────────────────────────────────────┐
│                    PROCESO DE CARGA DEL MÓDULO                       │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│  1. Zygote carga el módulo                                           │
│     └─▶ initZygote() almacena MODULE_PATH                           │
│                                                                       │
│  2. Android lanza cada proceso (app)                                 │
│     └─▶ handleLoadPackage(lpparam) se ejecuta por CADA paquete       │
│                                                                       │
│  3. MainHook verifica el nombre del paquete                          │
│     └─▶ Decide qué hooks registrar según el paquete                  │
│                                                                       │
│  4. Cada hook intercepta métodos específicos                         │
│     └─▶ XC_MethodHook.beforeHookedMethod() o afterHookedMethod()    │
│                                                                       │
│  5. Los hooks modifican comportamiento en tiempo real                │
│     └─▶ Cambian vistas, retornos, parámetros, etc.                   │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

#### Tipos de Hooks en Xposed

| Tipo | Cuándo se usa | Ejemplo |
|------|---------------|---------|
| `beforeHookedMethod` | Antes de que ejecute el método original | Modificar parámetros, cancelar ejecución |
| `afterHookedMethod` | Después de que ejecute el método original | Ocultar vistas, inyectar elementos |
| `XC_MethodReplacement` | Reemplazar completamente el método | Retornar un valor fijo, no hacer nada |

#### Patrón `hookSilently`

RecursHook usa un patrón helper que engloba los hooks en try/catch para evitar crashes si una clase no existe:

```java
// Si la clase no existe en esta versión de Android, simplemente se ignora
private void hookSilently(ClassLoader cl, String className, String method, Object... params) {
    try {
        XposedHelpers.findAndHookMethod(className, cl, method, params);
    } catch (Throwable t) {} // Silencioso - no crashea
}
```

---

### 3.2 ModPrefs - Sistema de Comunicación entre Hooks y UI

**Autoridad**: `com.zxerox.lg_extended.prefs`
**URI**: `content://com.zxerox.lg_extended.prefs/prefs`

Los hooks y la app de configuración se comunican a través de un **ContentProvider** llamado `ModPrefs`. Esto permite que la UI cambie preferencias y los hooks las lean en tiempo real.

```
┌──────────────────┐     ContentResolver      ┌──────────────────┐
│  MainActivity    │ ───────────────────────▶ │    ModPrefs      │
│  (UI Process)    │ ◀─────────────────────── │ (Module Process) │
└──────────────────┘     Query/Insert         └──────────────────┘
                                │
                                ▼
┌──────────────────┐     ContentObserver      ┌──────────────────┐
│  BatteryHook     │ ◀─────────────────────── │    ModPrefs      │
│  (SystemUI)      │     onChange()           │                  │
└──────────────────┘                          └──────────────────┘
```

#### Escribir Preferencias (desde la UI)

```java
ContentValues values = new ContentValues();
values.put("key", "battery_style");
values.put("type", "string");
values.put("value", "IOS_26");
contentResolver.insert(ModPrefs.CONTENT_URI, values);
// ModPrefs notifica a todos los ContentObserver registrados
```

#### Leer Preferencias (desde un hook)

```java
Cursor c = context.getContentResolver().query(
    ModPrefs.CONTENT_URI,
    new String[]{"battery_style"},  // key a leer
    "string",                        // tipo de dato
    new String[]{"ONEUI_8"},         // valor por defecto
    null
);
if (c != null && c.moveToFirst()) {
    String valor = c.getString(0);
    c.close();
}
```

#### Actualización en Tiempo Real (ContentObserver)

BatteryHook registra un observer para aplicar cambios de estilo/color instantáneamente sin reiniciar SystemUI:

```java
context.getContentResolver().registerContentObserver(
    Uri.parse("content://com.zxerox.lg_extended.prefs/prefs"),
    true, // Notificar si cambia cualquier descendiente
    new ContentObserver(new Handler(Looper.getMainLooper())) {
        @Override
        public void onChange(boolean selfChange) {
            // Re-leer preferencias y actualizar TODAS las vistas de batería
            BatteryIconView.Estilo nuevoEstilo = leerEstiloGuardado(context);
            for (BatteryIconView v : baterias.values()) {
                v.setEstilo(nuevoEstilo);
                aplicarColoresGuardados(context, v);
            }
        }
    }
);
```

#### Marcado de Hooks Activos

MainHook marca cada hook exitoso en ModPrefs con la key `hook_active_{nombre}`. Esto permite que la UI muestre qué hooks están corriendo:

```java
// En MainHook.markHookActive()
values.put("key", "hook_active_battery");
values.put("type", "boolean");
values.put("value", "true");
ctx.getContentResolver().insert(ModPrefs.CONTENT_URI, values);
```

---

### 3.3 BatteryHook - Personalización del Icono de Batería

**Paquete objetivo**: `com.android.systemui` / `com.lge.systemui`
**Archivo**: `hooks/BatteryHook.java`

#### Qué hace

Reemplaza el icono de batería nativo de LG (`LGBatteryMeterView`) por un `BatteryIconView` personalizado con múltiples estilos y colores.

#### Cómo funciona paso a paso

**Hook 1: `onAttachedToWindow`** - Se ejecuta cuando la vista de batería se adjunta al árbol de vistas

```
LGBatteryMeterView.onAttachedToWindow()
    │
    ├── 1. Obtener referencia a la vista original
    ├── 2. Obtener el padre (ViewGroup)
    ├── 3. Ocultar la vista original (setVisibility GONE)
    ├── 4. Crear BatteryIconView personalizado
    ├── 5. Insertar en el padre MISMA posición que la original
    ├── 6. Registrar en WeakHashMap para tracking
    ├── 7. Aplicar colores guardados desde ModPrefs
    └── 8. Registrar ContentObserver para cambios en tiempo real
```

**Hook 2: `onBatteryLevelChanged`** - Se ejecuta cuando cambia el nivel de batería

```
LGBatteryMeterView.onBatteryLevelChanged(level, plugged, charging)
    │
    ├── 1. Ocultar vista original (por si se vuelve visible)
    ├── 2. Ocultar texto de porcentaje (mBatteryLevel)
    ├── 3. Obtener BatteryIconView del WeakHashMap
    └── 4. Llamar actualizarEstado(nivel, cargando)
```

#### Estilos Disponibles

| Estilo | Descripción |
|--------|-------------|
| `ONEUI_8` | Estilo pill redondeado (default) |
| `ONEUI_9` | Estilo con círculo de carga |
| `IOS_26` | Estilo iOS con bolt integrado |
| `IOS_17` | Estilo iOS con borde y relleno |

#### Estados y Colores

| Estado | Color Fondo Default | Color Texto Default |
|--------|---------------------|---------------------|
| Normal | `#1C1C1E` | Blanco |
| Cargando | `#34C759` | Blanco |
| Batería Baja (≤20%) | `#FF3B30` | Blanco |

#### Preferencias Almacenadas

| Key | Tipo | Descripción |
|-----|------|-------------|
| `battery_style` | String | Estilo seleccionado (ONEUI_8, ONEUI_9, IOS_26, IOS_17) |
| `battery_color_fondo` | int | Color de fondo normal |
| `battery_color_texto` | int | Color de texto normal |
| `battery_color_borde` | int | Color de borde normal |
| `battery_color_fondo_cargando` | int | Color de fondo cargando |
| `battery_color_texto_cargando` | int | Color de texto cargando |
| `battery_color_borde_cargando` | int | Color de borde cargando |
| `battery_color_fondo_bajo` | int | Color de fondo batería baja |
| `battery_color_texto_bajo` | int | Color de texto batería baja |
| `battery_color_borde_bajo` | int | Color de borde batería baja |

---

### 3.4 DpiHook - Cambio de DPI por Aplicación

**Paquete objetivo**: Todas las aplicaciones (excepto `com.zxerox.lg_extended`)
**Archivo**: `hooks/DpiHook.java`

#### Qué hace

Modifica la densidad de pantalla (DPI) de cada aplicación individualmente antes de que Android renderice la interfaz.

#### Cómo funciona paso a paso

```
ResourcesImpl.updateConfiguration(config, metrics, compatInfo)
    │
    ├── 1. ¿Primera vez que se ejecuta? (dpiCache == -1)
    │       ├── SÍ: Leer DPI guardado en ModPrefs para este paquete
    │       │       └── Guardar en dpiCache (se cachea para no repetir queries)
    │       └── NO: Usar dpiCache ya almacenado
    │
    ├── 2. ¿dpiCache <= 0? → No hacer nada (usar DPI del sistema)
    │
    └── 3. Modificar Configuration y DisplayMetrics
            ├── config.densityDpi = dpiCache
            ├── metrics.densityDpi = dpiCache
            └── metrics.density = dpiCache * 0.00625f
```

#### Flujo de Datos

```
┌──────────────────┐
│  DpiActivity     │
│  (Seleccionar    │
│   DPI por app)   │
└────────┬─────────┘
         │  insert()
         ▼
┌──────────────────┐
│  ModPrefs        │
│  (Almacenar      │
│   paquete: dpi)  │
└────────┬─────────┘
         │  notifyChange()
         ▼
┌──────────────────┐
│  DpiHook         │
│  (Leer en cada   │
│   app al cargar) │
└──────────────────┘
```

#### Preferencias Almacenadas

| Key | Tipo | Descripción |
|-----|------|-------------|
| `{package_name}` | int | DPI específico para esa app (0 = default del sistema) |

---

### 3.5 RecentsHook - Estilo iOS para Multitasking

**Paquete objetivo**: `com.lge.launcher3` / `com.android.launcher3`
**Archivo**: `hooks/RecentsHook.java`

#### Qué hace

Modifica la vista de recientes (multitasking) para implementar un estilo similar a iOS con efectos de pila, escala progresiva y transparencia de headers.

#### Qué métodos intercepta

Este hook usa el patrón `hookSilently` para interceptar **10+ métodos** sin crashear si alguno no existe:

| Método | Clase | Tipo de Hook | Qué hace |
|--------|-------|--------------|----------|
| `get` | `TaskCornerRadius` | before | Redondea esquinas a 32dp |
| `setFullscreenProgress` | `TaskView` | after | Aplica escala basada en progreso |
| `setDimAlpha` | `TaskView` | before | Elimina oscurecimiento (fijo a 0) |
| `setDimAlpha` | `TaskThumbnailView` | before | Elimina oscurecimiento del thumbnail |
| `setDimAlphaMultipler` | `TaskThumbnailView` | before | Elimina multiplicador de oscurecimiento |
| `setStableAlpha` | `TaskView` (×2) | before | Fija alpha a 1.0 (sin transparencia) |
| `setContentAlpha` | `RecentsView` | before | Controla alpha del contenido |
| `onFinishInflate` | `TaskView` | after | Elimina elevación (elevation = 0) |
| `onTaskListVisibilityChanged` | `TaskView` | before | Fuerza visible = true |
| `updateStackLayout` | `RecentsView` | before | Cancela layout original (DO_NOTHING) |
| `updateStackProperties` | `RecentsView` | after | **Hook principal** - aplica efecto pila |
| `updateCurveProperties` | `RecentsView` (×2) | after | Reutiliza el mismo scrollHook |

#### Lógica del Efecto Pila (scrollHook)

```
RecentsView.updateStackProperties()
    │
    ├── 1. Obtener datos del scroll
    │       ├── count = número de task views
    │       ├── scrollX = posición actual del scroll
    │       └── width = ancho del RecentsView
    │
    ├── 2. Calcular spacing nativo
    │       └── Distancia real entre la primera y segunda task view
    │
    ├── 3. Para CADA task view:
    │       │
    │       ├── Calcular distanceToCenter (distancia al centro de pantalla)
    │       ├── Calcular progress (distancia / spacing)
    │       │
    │       ├── TRANSLACIÓN X:
    │       │   ├── Si está a la izquierda del centro:
    │       │   │   └── translationX = progress * stackGapBehind
    │       │   ├── Si está a la derecha del centro:
    │       │   │   └── translationX = progress * stackGapAhead
    │       │   └── Si está en el centro:
    │       │       └── translationX = 0
    │       │
    │       ├── ESCALA:
    │       │   ├── Primeras 4 apps: 1.0 → 0.97
    │       │   ├── Resto: 0.97 (mínimo 0.70)
    │       │   └── Se interpola con fullscreenProgress
    │       │
    │       ├── PROFUNDIDAD:
    │       │   └── translationZ = child.getLeft() * 0.01
    │       │
    │       └── HEADER (título):
    │           ├── Escala: 0.85
    │           ├── translateY: 15f
    │           └── Alpha: se desvanece con la distancia
    │               └── titleAlpha = max(0, 1 - |progress| * 2.5)
    │
    └── 4. Aplicar todas las transformaciones a la vista
```

#### Parámetros de Diseño

| Parámetro | Valor | Descripción |
|-----------|-------|-------------|
| `STACK_GAP` | 150f | Distancia entre apps en pila (behind) |
| `stackGapAhead` | 65% del spacing | Distancia hacia adelante |
| Scale inicial | 1.0 - 0.97 | Escala para primeras 4 apps |
| Scale mínimo | 0.70 | Escala mínima para apps lejanas |
| Header scale | 0.85 | Escala del título de la app |
| Header alpha threshold | 2.5 | Factor de desvanecimiento del título |

#### Preferencias Almacenadas

| Key | Tipo | Descripción |
|-----|------|-------------|
| `recents_enabled` | boolean | Habilitar estilo iOS en recientes |

---

### 3.6 FlagSecureHook - Bypass de Restricciones

**Paquete objetivo**: `android` (proceso del sistema)
**Archivo**: `hooks/FlagSecureHook.java`

#### Qué hace

Desactiva `FLAG_SECURE` que impide capturas de pantalla en aplicaciones que lo implementan (apps bancarias, Netflix, etc.).

#### Cómo funciona paso a paso

```
WindowState.isSecureLocked()
    │
    ├── 1. Recargar preferencias desde archivo XML
    │       └── prefs.reload() (lee directamente de disco)
    │
    ├── 2. Verificar si bypass está habilitado
    │       └── prefs.getBoolean("bypass_flag_secure", true)
    │
    └── 3. Si está habilitado:
            └── param.setResult(false)  ← SIEMPRE retorna false
                (el método original NUNCA se ejecuta)
```

#### Detalle Técnico Importante

Este hook usa `XSharedPreferences` en lugar de `ModPrefs` porque se ejecuta en el proceso `android` (system_server), donde el ContentProvider de la app no está disponible. Lee directamente el archivo XML de preferencias.

#### Preferencias Almacenadas

| Key | Tipo | Descripción |
|-----|------|-------------|
| `bypass_flag_secure` | boolean | Habilitar bypass (default: true) |

---

### 3.7 SettingsHook - Personalización de Ajustes

**Paquete objetivo**: `com.android.settings`
**Archivo**: `hooks/SettingsHook.java`

#### Qué hace

Realiza modificaciones extensas a la app de Ajustes de Android:

1. **Tarjeta de perfil personalizada** en la pantalla principal
2. **Reemplazo de iconos** con iconos estilo OneUI
3. **Eliminación de divisores** para un look más limpio
4. **Bloqueo de tinte de iconos** para mantener colores originales
5. **Modificación del AppBar** para agrandar la zona del título

#### Hook 1: Tarjeta de Perfil (DashboardFragment.refreshAllPreferences)

```
DashboardFragment.refreshAllPreferences()
    │
    ├── 1. Verificar que es TopLevelSettings (pantalla principal)
    ├── 2. Leer perfil desde ModPrefs (name, phrase, avatar)
    ├── 3. Construir vista custom (LinearLayout con avatar + texto)
    ├── 4. Crear LayoutPreference con la vista
    ├── 5. Insertar con order = -999 (siempre arriba)
    ├── 6. Inyectar divider category con order = -998
    └── 7. Modificar AppBar (altura 180dp, título 36sp)
```

#### Hook 2: Reemplazo de Iconos (PreferenceGroupAdapter.onBindViewHolder)

```
PreferenceGroupAdapter.onBindViewHolder(holder, position)
    │
    ├── 1. Obtener preferencia en esa posición
    ├── 2. Verificar que hook_settings_icons esté habilitado
    ├── 3. Obtener key de la preferencia
    ├── 4. Buscar icono en el iconMap (18 mapeos)
    ├── 5. Cargar drawable desde recursos del módulo
    │       └── XModuleResources.createInstance(MODULE_PATH)
    ├── 6. Reemplazar ImageView del holder
    └── 7. Quitar tintes y ajustar tamaño a 40dp
```

#### Mapeo de Iconos

| Key de Preferencia | Icono |
|-------------------|-------|
| `top_level_network` | `ic_red_e_internet` |
| `top_level_connected_devices` | `ic_bluetooth` |
| `top_level_sound` | `ic_sonido` |
| `top_level_notification` | `ic_notificaciones` |
| `top_level_display` | `ic_pantalla` |
| `top_level_theme` | `ic_fondo_pantalla_y_tema` |
| `top_level_security` | `ic_pantalla_de_bloqueo_y_seguridad` |
| `top_level_privacy` | `ic_privacidad` |
| `top_level_location` | `ic_ubicacion` |
| `top_level_useful_features` | `ic_extensiones` |
| `top_level_apps_and_notifs` | `ic_aplicaciones` |
| `top_level_digital_wellbeing` | `ic_bienestar_digital` |
| `top_level_battery` | `ic_bateria` |
| `top_level_storage` | `ic_almacenamiento` |
| `top_level_emergency` | `ic_seguridad_y_emergencia` |
| `top_level_accounts` | `ic_cuentas` |
| `top_level_system` | `ic_sistema` |
| `top_level_accessibility` | `ic_accesibilidad` |

#### Hook 3: Bloqueo de Tinte (DynamicIconColorDecorator)

```
DynamicIconColorDecorator.setIconTint() / setDynamicIconColor()
    │
    ├── 1. Obtener key de la preferencia
    ├── 2. ¿Está en el iconMap?
    │       └── SÍ: param.setResult(null) ← no aplicar tinte
    └── 3. NO: dejar comportamiento original
```

#### Hook 4: Eliminación de Divisores

```
PreferenceFragmentCompat.setDivider()        → DO_NOTHING (no-op)
PreferenceFragmentCompat.setDividerHeight()   → args[0] = 0
RecyclerView.addItemDecoration()              → bloquear si contiene "divider"
HIGHelper.isShowDivider()                     → returnConstant(false)
```

#### Preferencias Almacenadas

| Key | Tipo | Descripción |
|-----|------|-------------|
| `hook_settings_icons` | boolean | Reemplazar iconos de ajustes |
| `profile_name` | String | Nombre del perfil |
| `profile_phrase` | String | Frase del perfil |
| `profile_avatar_base64` | String | Avatar en Base64 |

---

### 3.8 LauncherHook - Orden Automático de Apps

**Paquete objetivo**: `com.lge.launcher3` / `com.android.launcher3`
**Archivo**: `hooks/LauncherHook.java`

#### Qué hace

Ordena automáticamente las apps alfabéticamente después de que se instala una nueva app.

#### Cómo funciona

```
AllAppsPagedView.addApps(apps, sessionId)
    │
    ├── 1. Recargar preferencias
    ├── 2. ¿hook_auto_sort_apps está habilitado?
    │       └── NO: No hacer nada
    │
    └── 3. SÍ:
            ├── Obtener enum SortType.NAME
            └── Llamar changeSortType(NAME)
```

#### Preferencias Almacenadas

| Key | Tipo | Descripción |
|-----|------|-------------|
| `hook_auto_sort_apps` | boolean | Ordenar apps automáticamente |

---

### 3.9 QSPanelHook - Panel de Quick Settings MIUI

**Paquete objetivo**: `com.android.systemui` / `com.lge.systemui`
**Archivo**: `hooks/QSPanelHook.java`

#### Qué hace

Hace transparente el fondo del panel de Quick Settings para un look estilo MIUI.

#### Cómo funciona

```
QSContainerImpl.onFinishInflate()
    │
    ├── 1. Verificar que pref_enable_miui_qs esté habilitado
    │       (también verifica archivo /data/local/tmp/lg_ext_miui_qs)
    │
    └── 2. Si está habilitado:
            ├── Buscar vista quick_settings_background
            └── setBackgroundColor(Color.TRANSPARENT)
```

#### Preferencias Almacenadas

| Key | Tipo | Descripción |
|-----|------|-------------|
| `pref_enable_miui_qs` | boolean | Habilitar estilo MIUI QS |

---

### 3.10 ScrimHook - Efecto Blur en SystemUI

**Paquete objetivo**: `com.android.systemui` / `com.lge.systemui`
**Archivo**: `hooks/ScrimHook.java`

#### Qué hace

Agrega efecto de blur (desenfoque) y ajusta la transparencia del scrim (capa superpuesta) en SystemUI.

#### Hooks que intercepta

**Hook 1: `ScrimController.attachViews`**

```
ScrimController.attachViews(mScrimBehind, mScrimInFront, mNotificationsScrim)
    │
    ├── 1. Verificar que MIUI QS esté habilitado
    └── 2. Si está habilitado:
            ├── mScrimBehind.setBackgroundBlurRadius(150)
            ├── mDefaultScrimAlpha = 0.4f
            └── mScrimBehindUnblurAlpha = 0.4f
```

**Hook 2: `ScrimController.updateScrims`**

```
ScrimController.updateScrims()
    │
    ├── 1. Verificar que MIUI QS esté habilitado
    └── 2. Si está habilitado:
            └── Si mBehindAlpha > 0.4f:
                    └── mBehindAlpha = 0.4f (limitar transparencia)
```

#### Preferencias Almacenadas

| Key | Tipo | Descripción |
|-----|------|-------------|
| `pref_enable_miui_qs` | boolean | Habilitar estilo MIUI QS |

---

### 3.11 NotificationHook - Estilo de Notificaciones MIUI

**Paquete objetivo**: `com.android.systemui` / `com.lge.systemui`
**Archivo**: `hooks/NotificationHook.java`

#### Qué hace

Modifica la apariencia de las notificaciones para un estilo MIUI: blur en fondos, colores personalizados, y ocultar flecha de expandir.

#### Hooks que intercepta

| Método | Clase | Tipo | Qué hace |
|--------|-------|------|----------|
| Constructor | `NotificationBackgroundView` | after | Aplica blurRadius(50) al fondo |
| `setTint` | `NotificationBackgroundView` | before | Aplica colorFilter personalizado |
| `setPinned` | `ExpandableNotificationRow` | after | Ajusta opacidad de notificaciones fijadas |
| `onFinishInflate` | `NotificationExpandButton` | after | Oculta flecha de expandir |
| `updateColors` | `NotificationExpandButton` | after | Oculta flecha de expandir |
| `updateViewWithShelf` | `StackScrollAlgorithm` | after | Oculta notificaciones casi fuera de pantalla |

#### Flujo del hook de tinting

```
NotificationBackgroundView.setTint(color)
    │
    ├── 1. Verificar que MIUI QS esté habilitado
    ├── 2. Obtener Drawable mBackground
    └── 3. Si color != 0:
            └── bg.setColorFilter(color, SRC)
        Si color == 0:
            └── bg.clearColorFilter()
```

#### Preferencias Almacenadas

| Key | Tipo | Descripción |
|-----|------|-------------|
| `pref_enable_miui_qs` | boolean | Habilitar estilo MIUI QS |
| `pref_hide_notification_arrow` | boolean | Ocultar flecha de expandir notificación |
| `pref_separate_notification_cards` | boolean | Separar tarjetas de notificación |

---

### 3.12 SystemUIHook - Reemplazo de Recursos

**Paquete objetivo**: `com.android.systemui` / `com.lge.systemui`
**Archivo**: `hooks/SystemUIHook.java`

#### Qué hace

Implementa `IXposedHookInitPackageResources` para reemplazar recursos (drawables, colores, dimensiones) del sistema por recursos personalizados del módulo.

#### Qué reemplaza

| Recurso Original | Recurso del Módulo |
|------------------|--------------------|
| `ic_indi_noti_button_on` | `qs_tile_background` |
| `ic_indi_noti_button_off` | `qs_tile_background` |
| `notification_material_background_color` | `notification_material_background_color_miui` |
| `notification_material_background_dimmed_color` | `notification_material_background_dimmed_color_miui` |
| `notification_corner_radius` | `notification_corner_radius_miui` |
| `notification_divider_height` | `notification_divider_height_miui` |
| `qs_icon_size` | `qs_icon_size_miui` |

#### Diferencia con otros hooks

Este hook NO intercepta métodos Java. Implementa `IXposedHookInitPackageResources` que se ejecuta cuando Android carga los recursos de un paquete. Permite reemplazar drawables, colores y dimensiones directamente.

---

### 3.13 Resumen de Todos los Hooks

| Hook | Paquete | Tipo | Métodos Interceptados |
|------|---------|------|----------------------|
| `BatteryHook` | SystemUI | after × 2 | `onAttachedToWindow`, `onBatteryLevelChanged` |
| `DpiHook` | Todas las apps | before × 1 | `ResourcesImpl.updateConfiguration` |
| `RecentsHook` | Launcher | before/after × 12+ | `TaskView.*`, `RecentsView.*`, `TaskCornerRadius.*` |
| `FlagSecureHook` | android | before × 1 | `WindowState.isSecureLocked` |
| `SettingsHook` | Settings | after/before × 5+ | `DashboardFragment.*`, `PreferenceGroupAdapter.*`, etc. |
| `LauncherHook` | Launcher | after × 1 | `AllAppsPagedView.addApps` |
| `QSPanelHook` | SystemUI | after × 1 | `QSContainerImpl.onFinishInflate` |
| `ScrimHook` | SystemUI | before/after × 2 | `ScrimController.attachViews`, `updateScrims` |
| `NotificationHook` | SystemUI | before/after × 6 | `NotificationBackgroundView.*`, `ExpandableNotificationRow.*`, etc. |
| `SystemUIHook` | SystemUI | resource replacement | Reemplaza drawables, colores, dimensiones |

---

## 4. Componentes UI

### 4.1 MainActivity - Pantalla Principal

**Descripción**: Activity principal con navegación por tabs (bottom navigation).

#### Tabs

| Tab | Layout | Función |
|-----|--------|---------|
| Inicio | `tab_inicio.xml` | Estado del módulo, info del dispositivo |
| Hooks | `tab_hooks.xml` | Gestión de hooks activos |
| Logs | `tab_logs.xml` | Visualización de logs del módulo |
| Ajustes | `tab_settings.xml` | Configuración (próximamente) |

#### Funcionalidades del Tab Inicio

- **Estado de LSPosed**: Verifica si hay hooks activos
- **Estado de Root**: Detecta Magisk/KernelSU/APatch
- **Info del dispositivo**: Modelo, versión Android, kernel, arquitectura
- **Botón reiniciar SystemUI**: Reinicia SystemUI para aplicar cambios

### 4.2 BatteryStyleActivity - Selector de Estilo de Batería

**Descripción**: Permite seleccionar el estilo del icono de batería y personalizar colores.

#### Estilos con Preview en Tiempo Real

```
┌─────────────────────────────────────────────────┐
│  ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌─────────┐  │
│  │ IOS 26  │  │ IOS 17  │  │ OneUI 9 │  │ OneUI 8 │  │
│  │  [75]   │  │  [75]   │  │  [75]   │  │  [75]   │  │
│  └─────────┘  └─────────┘  └─────────┘  └─────────┘  │
│                                                         │
│  [Personalizar Colores]  ← BottomSheet con 6 opciones  │
└─────────────────────────────────────────────────────────┘
```

#### Selector de Colores (BottomSheet)

| Estado | Opciones |
|--------|----------|
| Normal | Fondo Normal, Texto Normal |
| Cargando | Fondo Cargando, Texto Cargando |
| Batería Baja | Fondo Batería Baja, Texto Batería Baja |

### 4.3 DpiActivity - Selector de DPI

**Descripción**: Lista de apps instaladas con opción de cambiar DPI individualmente.

#### Características

- **Lista de apps**: Muestra todas las apps no-sistema
- **DPI actual**: Muestra el DPI configurado o "Default"
- **Dialog de edición**: Input numérico para nuevo DPI
- **Reinicio automático**: Fuerza cierre de la app para aplicar cambios

### 4.4 BypassActivity - Bypass de Seguridad

**Descripción**: Switch para habilitar/deshabilitar el bypass de FLAG_SECURE.

### 4.5 CustomizeSettingsActivity - Personalización de Ajustes

**Descripción**: Editor de perfil para la tarjeta personalizada en Ajustes.

#### Características

- **Preview en tiempo real**: Muestra cambios mientras se editan
- **Selector de avatar**: Permite elegir imagen de galería
- **Avatar por defecto**: Genera inicial del nombre con fondo circular
- **Límite de tamaño**: Avatar redimensionado a 256x256px máximo

### 4.6 BatteryIconView - Vista Personalizada de Batería

**Descripción**: Componente de vista personalizada que dibuja el icono de batería.

```java
public class BatteryIconView extends View {
    public enum Estilo {
        IOS_26,
        ONEUI_9,
        ONEUI_8,
        IOS_17
    }
    
    // Métodos principales
    public void actualizarEstado(int nivel, boolean cargando);
    public void setEstilo(Estilo nuevoEstilo);
    public void setColoresNormal(int colorFondo, int colorTexto);
    public void setColoresCargando(int colorFondo, int colorTexto);
    public void setColoresBateriaBaja(int colorFondo, int colorTexto);
}
```

---

## 5. Gestión de Datos

### 5.1 ModPrefs - ContentProvider Centralizado

**Autoridad**: `com.zxerox.lg_extended.prefs`
**URI**: `content://com.zxerox.lg_extended.prefs/prefs`

#### Operaciones

| Operación | Método | Descripción |
|-----------|--------|-------------|
| Leer | `query()` | Lee una preferencia por key |
| Escribir | `insert()` | Escribe o actualiza una preferencia |

#### Formato de ContentValues

```java
ContentValues values = new ContentValues();
values.put("key", "nombre_preferencia");
values.put("type", "string|int|boolean");
values.put("value", "valor");
contentResolver.insert(PREFS_URI, values);
```

#### Lectura de Preferencias

```java
Cursor c = contentResolver.query(
    PREFS_URI,
    new String[]{"key_name"},    // projection
    "type",                       // selection (tipo de dato)
    new String[]{"default_value"}, // selectionArgs (valor por defecto)
    null
);
if (c != null && c.moveToFirst()) {
    String value = c.getString(0);
    c.close();
}
```

### 5.2 Sistema de Logs

**Clase**: `LogWriter`

#### Características

- **Almacenamiento**: En SharedPreferences vía ModPrefs
- **Formato**: `YYYY-MM-DD HH:mm:ss | NIVEL | mensaje`
- **Niveles**: `OK`, `ERR`, `INFO`
- **Límite**: 200 entradas máximo (rotación automática)

#### Estructura de LogEntry

```java
public static class LogEntry {
    public String timestamp;
    public String level;
    public String message;
}
```

#### Uso en Hooks

```java
// En BatteryHook.java
LogWriter.write(ctx, "OK", "BatteryHook", packageName, true);

// Output: "2024-01-15 10:30:45 | OK | BatteryHook applied in com.android.systemui"
```

### 5.3 Persistencia de Datos

| Dato | Ubicación | Formato |
|------|-----------|---------|
| Preferencias | `shared_prefs/lg_extended_prefs.xml` | XML SharedPreferences |
| Logs | Misma ubicación (via ModPrefs) | String con newlines |
| Avatar | Misma ubicación | Base64 encoded string |

---

## 6. Integración con Root

### 6.1 Dependencia libsu

```gradle
implementation 'com.github.topjohnwu.libsu:core:5.2.2'
```

### 6.2 DeviceInfoProvider

**Descripción**: Obtiene información del dispositivo usando comandos root.

#### Datos Obtenidos

| Dato | Fuente |
|------|--------|
| Modelo | `Build.MANUFACTURER + " " + Build.MODEL` |
| Versión Android | `Build.VERSION.RELEASE` |
| Build Number | `Build.DISPLAY` |
| Arquitectura | `Build.SUPPORTED_ABIS[0]` |
| Versión Kernel | `uname -r` (via root) |
| Root Manager | Detección automática |

#### Detección de Root Manager

```java
// Prioridad de detección:
1. `su -v` → busca "magisk", "kernelsu", "apatch"
2. `/data/adb/ksu` → KernelSU
3. `/data/adb/ap` → APatch
4. `/data/adb/magisk` → Magisk
5. `su --version` → Root genérico
```

### 6.3 RootUtils

```java
// Verificar root
RootUtils.tieneRoot(); // boolean

// Reiniciar SystemUI
RootUtils.reiniciarSystemUI(() -> {
    // Callback después del reinicio
});
// Equivale a: killall com.android.systemui
```

### 6.4 Uso de Root en la App

| Función | Comando Root |
|---------|--------------|
| Reiniciar SystemUI | `killall com.android.systemui` |
| Forzar cierre de app | `am force-stop {package}` |
| Verificar root | `Shell.getShell().isRoot()` |
| Info kernel | `uname -r` |

---

## 7. Configuración del Build

### 7.1 Archivos de Build

```
LG_Extended/
├── build.gradle                    # Build script raíz
├── app/
│   └── build.gradle               # Build script del módulo
├── settings.gradle                 # Configuración de dependencias
├── gradle.properties               # Propiedades de Gradle
└── gradle/
    └── libs.versions.toml          # Catálogo de versiones
```

### 7.2 Configuración del Módulo

```gradle
// app/build.gradle
android {
    namespace 'com.zxerox.lg_extended'
    compileSdk {
        version = release(36) {
            minorApiLevel = 1
        }
    }

    defaultConfig {
        applicationId "com.zxerox.lg_extended"
        minSdk 29
        targetSdk 36
        versionCode 1
        versionName "1.0"
    }

    compileOptions {
        sourceCompatibility JavaVersion.VERSION_11
        targetCompatibility JavaVersion.VERSION_11
    }
}
```

### 7.3 Dependencias Principales

```gradle
dependencies {
    // Xposed API
    compileOnly 'de.robv.android.xposed:api:82'
    
    // Root (libsu)
    implementation 'com.github.topjohnwu.libsu:core:5.2.2'
    
    // Color Picker
    implementation 'com.github.skydoves:colorpickerview:2.3.0'
    
    // AndroidX
    implementation libs.activity.ktx
    implementation libs.appcompat
    implementation libs.constraintlayout
    implementation libs.material
}
```

### 7.4 Configuración Xposed

```xml
<!-- AndroidManifest.xml -->
<meta-data
    android:name="xposedmodule"
    android:value="true" />
<meta-data
    android:name="xposedminversion"
    android:value="82" />
<meta-data
    android:name="xposeddescription"
    android:value="LG Extended - Suite de mods para LG V60" />
```

```properties
# xposed_init
com.zxerox.lg_extended.MainHook
```

### 7.5 Repositorios

```gradle
// settings.gradle
repositories {
    google()
    mavenCentral()
    maven { url 'https://api.xposed.info/' }
    maven { url 'https://jitpack.io' }
}
```

---

## 8. Estructura de Archivos

### 8.1 Árbol del Proyecto

```
LG_Extended/
├── app/
│   ├── build.gradle
│   ├── libs/                          # Xposed API jar
│   └── src/
│       └── main/
│           ├── AndroidManifest.xml
│           ├── assets/
│           │   └── xposed_init        # Entry point para Xposed
│           ├── java/
│           │   └── com/zxerox/lg_extended/
│           │       ├── MainHook.java           # Entry point principal
│           │       ├── hooks/
│           │       │   ├── BatteryHook.java       # Icono de batería
│           │       │   ├── DpiHook.java           # DPI por app
│           │       │   ├── FlagSecureHook.java    # Bypass FLAG_SECURE
│           │       │   ├── RecentsHook.java       # Estilo iOS recientes
│           │       │   ├── SettingsHook.java      # Personalización Ajustes
│           │       │   ├── LauncherHook.java      # Orden automático apps
│           │       │   ├── QSPanelHook.java       # Panel QS MIUI
│           │       │   ├── ScrimHook.java         # Blur en SystemUI
│           │       │   ├── NotificationHook.java  # Notificaciones MIUI
│           │       │   └── SystemUIHook.java      # Reemplazo recursos
│           │       ├── log/
│           │       │   ├── LogAdapter.java     # Adapter RecyclerView
│           │       │   └── LogWriter.java      # Escritura de logs
│           │       ├── prefs/
│           │       │   └── ModPrefs.java       # ContentProvider
│           │       ├── root/
│           │       │   ├── DeviceInfoProvider.java  # Info del dispositivo
│           │       │   └── RootUtils.java      # Utilidades root
│           │       ├── ui/
│           │       │   ├── AppAdapter.java     # Adapter para lista de apps
│           │       │   ├── BatteryStyleActivity.java  # Selector estilo batería
│           │       │   ├── BypassActivity.java # Toggle bypass
│           │       │   ├── CustomizeSettingsActivity.java  # Editor perfil
│           │       │   ├── DpiActivity.java    # Selector DPI
│           │       │   ├── IosStyleActivity.java # Toggle estilo iOS
│           │       │   └── MainActivity.java   # Activity principal
│           │       └── views/
│           │           └── BatteryIconView.java # Vista personalizada batería
│           ├── keepRules/
│           │   └── rules.keep                 # ProGuard/R8 rules
│           └── res/
│               ├── drawable/                   # Iconos y drawables
│               ├── layout/                     # Layouts de activities
│               ├── mipmap-*/                   # Iconos de launcher
│               ├── values/
│               │   ├── colors.xml              # Colores
│               │   ├── strings.xml             # Strings
│               │   └── themes.xml              # Temas
│               ├── values-night/               # Tema oscuro
│               └── xml/
│                   ├── backup_rules.xml
│                   └── data_extraction_rules.xml
├── iconos_ajustes_oneui_muted/        # Iconos SVG para Ajustes
├── xml icons/                         # Iconos adicionales
├── hook iconos.txt                    # Código de referencia (JADX)
├── nuevohook.txt                      # Investigación de hooks
├── build.gradle
├── settings.gradle
├── gradle.properties
└── gradlew / gradlew.bat
```

### 8.2 Archivos de Recursos

#### Layouts Principales

| Archivo | Descripción |
|---------|-------------|
| `activity_main_new.xml` | MainActivity con bottom navigation |
| `activity_battery_style.xml` | Selector de estilo de batería |
| `activity_bypass.xml` | Toggle de bypass |
| `activity_customize_settings.xml` | Editor de perfil |
| `activity_dpi.xml` | Lista de apps para DPI |
| `activity_ios_style.xml` | Toggle estilo iOS |
| `tab_inicio.xml` | Tab de inicio |
| `tab_hooks.xml` | Tab de hooks |
| `tab_logs.xml` | Tab de logs |
| `tab_settings.xml` | Tab de ajustes |

#### Drawables Principales

| Categoría | Archivos |
|-----------|----------|
| Iconos Settings | `ic_accesibilidad.xml`, `ic_bateria.xml`, `ic_bluetooth.xml`, etc. |
| Iconos UI | `ic_home.xml`, `ic_hook.xml`, `ic_log.xml`, `ic_settings.xml` |
| Backgrounds | `bg_card_section.xml`, `bg_main_glass.xml`, `bg_nav_floating.xml` |
| Selectores | `row_selected.xml`, `row_default.xml`, `nav_selector.xml` |

---

## 9. Dependencias

### 9.1 Dependencias Principales

| Dependencia | Versión | Propósito |
|-------------|---------|-----------|
| `de.robv.android.xposed:api` | 82 | Framework Xposed |
| `com.github.topjohnwu.libsu:core` | 5.2.2 | Operaciones root |
| `com.github.skydoves:colorpickerview` | 2.3.0 | Selector de colores |
| `androidx.activity:activity-ktx` | - | Extensiones Activity |
| `androidx.appcompat:appcompat` | - |Compatibilidad hacia atrás |
| `com.google.android.material:material` | - | Componentes Material Design |

### 9.2 Dependencias de Compilación

| Dependencia | Tipo |
|-------------|------|
| `*.jar` en `libs/` | compileOnly |
| Xposed API | compileOnly |

> **Nota**: Xposed API es `compileOnly` porque se proporciona por el framework en runtime.

---

## 10. Guía de Instalación

### 10.1 Requisitos Previos

1. **Dispositivo LG** (preferiblemente LG V60)
2. **Android 10+** (API 29+)
3. **Root** via Magisk, KernelSU o APatch
4. **LSPosed** (o similar framework Xposed)

### 10.2 Pasos de Instalación

#### Paso 1: Instalar LSPosed

```bash
# Si usas Magisk:
# Descargar LSPosed desde GitHub
# Instalar vía Magisk Manager
# Habilitar en /data/adb/lspd/

# Si usas KernelSU:
# LSPosed suele venir preinstalado
```

#### Paso 2: Compilar o Descargar el Módulo

```bash
# Opción A: Compilar desde código fuente
./gradlew assembleRelease

# Opción B: Descargar APK precompilada
# (Descargar desde releases)
```

#### Paso 3: Instalar el Módulo

```bash
# Instalar APK como app normal
adb install app-release.apk
```

#### Paso 4: Habilitar en LSPosed

1. Abrir **LSPosed**
2. Ir a **Modules**
3. Buscar **LG Extended**
4. Habilitar el módulo
5. Seleccionar alcance:
   - ☑️ `com.android.systemui`
   - ☑️ `com.lge.launcher3` / `com.android.launcher3`
   - ☑️ `android`
   - ☑️ `com.android.settings`

#### Paso 5: Reiniciar

```bash
# Reiniciar dispositivo o
adb shell reboot
```

### 10.5 Uso

1. Abrir **LG Extended**
2. Ir a **Hooks** para configurar cada mod
3. Los cambios se aplican en tiempo real (algunos requieren reinicio de SystemUI)

---

## 11. Troubleshooting

### 11.1 Problemas Comunes

| Problema | Causa | Solución |
|----------|-------|----------|
| Módulo no aparece en LSPosed | No instalado correctamente | Reinstalar APK y habilitar en LSPosed |
| Hooks no funcionan | Alcance incorrecto | Verificar que todos los paquetes estén seleccionados |
| Icono batería no cambia | SystemUI no reiniciado | Usar botón "Restart System UI" o reboot |
| DPI no se aplica | App no en alcance | Asegurar que la app esté en el scope de LSPosed |
| Crash al abrir Ajustes | Conflicto con hooks | Deshabilitar hooks temporalmente |

### 11.2 Logs

Los logs del módulo se pueden visualizar en:
- **LG Extended** → Tab **Logs**
- **LSPosed** → Logs
- **Logcat** con filtro: `LG_Extended`

### 11.3 Debug

```bash
# Ver logs del módulo
adb logcat | grep "LG_Extended"

# Ver logs de Xposed
adb logcat | grep "XposedBridge"

# Forzar reinicio de SystemUI
adb shell killall com.android.systemui

# Ver preferencias del módulo
adb shell cat /data/data/com.zxerox.lg_extended/shared_prefs/lg_extended_prefs.xml
```

---

## Apendice A: Mapeo de Paquetes

| Paquete | Descripción | Hooks |
|---------|-------------|-------|
| `com.android.systemui` | SystemUI | BatteryHook, QSPanelHook, ScrimHook, NotificationHook, SystemUIHook |
| `com.lge.systemui` | LG SystemUI | BatteryHook, QSPanelHook, ScrimHook, NotificationHook, SystemUIHook |
| `com.lge.launcher3` | LG Launcher | RecentsHook, LauncherHook |
| `com.android.launcher3` | AOSP Launcher | RecentsHook, LauncherHook |
| `android` | Proceso del sistema | FlagSecureHook |
| `com.android.settings` | Ajustes | SettingsHook |
| `com.zxerox.lg_extended` | Esta app | (Ninguno - excluida) |

---

## Apendice B: Referencia de Preferencias

### Preferencias de Batería

| Key | Tipo | Default | Descripción |
|-----|------|---------|-------------|
| `battery_style` | String | ONEUI_8 | Estilo de batería |
| `battery_color_fondo` | int | #1C1C1E | Color fondo normal |
| `battery_color_texto` | int | Blanco | Color texto normal |
| `battery_color_borde` | int | Blanco | Color borde normal |
| `battery_color_fondo_cargando` | int | #34C759 | Color fondo cargando |
| `battery_color_texto_cargando` | int | Blanco | Color texto cargando |
| `battery_color_borde_cargando` | int | Blanco | Color borde cargando |
| `battery_color_fondo_bajo` | int | #FF3B30 | Color fondo batería baja |
| `battery_color_texto_bajo` | int | Blanco | Color texto batería baja |
| `battery_color_borde_bajo` | int | Blanco | Color borde batería baja |

### Preferencias de Recientes

| Key | Tipo | Default | Descripción |
|-----|------|---------|-------------|
| `recents_enabled` | boolean | false | Estilo iOS recientes |

### Preferencias de Seguridad

| Key | Tipo | Default | Descripción |
|-----|------|---------|-------------|
| `bypass_flag_secure` | boolean | true | Bypass FLAG_SECURE |

### Preferencias de Ajustes

| Key | Tipo | Default | Descripción |
|-----|------|---------|-------------|
| `hook_settings_icons` | boolean | false | Reemplazar iconos ajustes |
| `profile_name` | String | LG V60 User | Nombre perfil |
| `profile_phrase` | String | Stock is a suggestion | Frase perfil |
| `profile_avatar_base64` | String | "" | Avatar Base64 |

### Preferencias del Launcher

| Key | Tipo | Default | Descripción |
|-----|------|---------|-------------|
| `hook_auto_sort_apps` | boolean | false | Ordenar apps alfabéticamente |

### Preferencias de SystemUI (MIUI)

| Key | Tipo | Default | Descripción |
|-----|------|---------|-------------|
| `pref_enable_miui_qs` | boolean | false | Habilitar estilo MIUI QS |
| `pref_hide_notification_arrow` | boolean | false | Ocultar flecha de notificación |
| `pref_separate_notification_cards` | boolean | false | Separar tarjetas de notificación |

### Estado de Hooks Activos (automático)

| Key | Tipo | Descripción |
|-----|------|-------------|
| `hook_active_battery` | boolean | BatteryHook activo |
| `hook_active_dpi` | boolean | DpiHook activo |
| `hook_active_recents` | boolean | RecentsHook activo |
| `hook_active_flagsecure` | boolean | FlagSecureHook activo |
| `hook_active_settings` | boolean | SettingsHook activo |
| `hook_active_launcher` | boolean | LauncherHook activo |
| `hook_active_qspanel` | boolean | QSPanelHook activo |
| `hook_active_scrim` | boolean | ScrimHook activo |
| `hook_active_notification` | boolean | NotificationHook activo |

---

## Apendice C: Créditos y Licencias

### Autor
- **ZxeroX** - Desarrollador principal

### Dependencias con Licencia

| Librería | Licencia |
|----------|----------|
| Xposed Framework | Apache License 2.0 |
| libsu | Apache License 2.0 |
| ColorPickerView | Apache License 2.0 |
| Material Components | Apache License 2.0 |

---

*Documentación generada el: 02 de Agosto, 2026*
*Versión del documento: 2.0*

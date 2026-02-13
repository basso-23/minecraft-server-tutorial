# Servidor Minecraft Fabric 1.20.1 Optimizado para VPS Hostinger

Guía para crear un servidor Minecraft Fabric 1.20.1 optimizado que NO sobrecargue tu VPS de Hostinger.

**Tu VPS:** 2 vCPU | 8 GB RAM | 100 GB NVMe

---

## Paso 1: Crear el Servidor en Pterodactyl

1. Ve al panel: `http://147.93.118.226`
2. **Admin** > **Servers** > **Create New**

### Configuración Recomendada:

| Campo | Valor | Explicación |
|-------|-------|-------------|
| **Server Name** | Mi Servidor Fabric | El nombre que quieras |
| **Server Owner** | Tu usuario | Selecciona tu cuenta |
| **Default Allocation** | 147.93.118.226:25565 | Puerto principal |
| **Memory** | 5120 MB (5GB) | Deja 3GB para el sistema y Wings |
| **Disk** | 20480 MB (20GB) | Suficiente para mundo + mods |
| **CPU Limit** | 150% | **IMPORTANTE: Esto evita que Hostinger te limite** |
| **Block IO Weight** | 500 | Valor por defecto |

### Selección de Nest y Egg:

- **Nest:** Minecraft
- **Egg:** Fabric (si no está, usa "Vanilla" y luego instalamos Fabric manual)

### Variables del Egg (si aplica):

- **Minecraft Version:** `1.20.1`
- **Fabric Loader Version:** `latest` o dejar vacío
- **Server Jar File:** `fabric-server-launch.jar`

3. Click en **Create Server**

---

## Paso 2: Configurar Startup (Flags de Java Optimizados)

Después de crear el servidor, ve a **Admin** > **Servers** > Tu servidor > **Startup**

### Aikar's Flags para 5GB RAM:

En el campo **Startup Command** o en la variable de Java Flags, usa:

```
java -Xms5120M -Xmx5120M -XX:+UseG1GC -XX:+ParallelRefProcEnabled -XX:MaxGCPauseMillis=200 -XX:+UnlockExperimentalVMOptions -XX:+DisableExplicitGC -XX:+AlwaysPreTouch -XX:G1NewSizePercent=30 -XX:G1MaxNewSizePercent=40 -XX:G1HeapRegionSize=8M -XX:G1ReservePercent=20 -XX:G1HeapWastePercent=5 -XX:G1MixedGCCountTarget=4 -XX:InitiatingHeapOccupancyPercent=15 -XX:G1MixedGCLiveThresholdPercent=90 -XX:G1RSetUpdatingPauseTimePercent=5 -XX:SurvivorRatio=32 -XX:+PerfDisableSharedMem -XX:MaxTenuringThreshold=1 -Dusing.aikars.flags=https://mcflags.emc.gs -Daikars.new.flags=true -jar {{SERVER_JARFILE}}
```

> **Nota:** `-Xms` y `-Xmx` deben ser IGUALES (5120M). Esto evita que Java redimensione la memoria constantemente y reduce uso de CPU.

---

## Paso 3: Instalar Mods de Optimización

Accede a los archivos del servidor desde el panel (File Manager) o por SFTP.

### Crear carpeta mods:
Si no existe, crea la carpeta `mods` en la raíz del servidor.

### Mods OBLIGATORIOS de optimización:

Descarga estos mods de [Modrinth](https://modrinth.com) o [CurseForge](https://curseforge.com) (versión Fabric 1.20.1):

| Mod | Función | Descarga |
|-----|---------|----------|
| **Lithium** | Optimización general del servidor | [Modrinth](https://modrinth.com/mod/lithium) |
| **Starlight** | Optimización del motor de iluminación | [Modrinth](https://modrinth.com/mod/starlight) |
| **FerriteCore** | Reduce uso de memoria RAM | [Modrinth](https://modrinth.com/mod/ferrite-core) |
| **Krypton** | Optimización de red/networking | [Modrinth](https://modrinth.com/mod/krypton) |
| **LazyDFU** | Inicio del servidor más rápido | [Modrinth](https://modrinth.com/mod/lazydfu) |
| **C2ME** | Carga de chunks multihilo | [Modrinth](https://modrinth.com/mod/c2me-fabric) |
| **ServerCore** | Optimizaciones adicionales del servidor | [Modrinth](https://modrinth.com/mod/servercore) |

### Mod de API requerido:
| Mod | Función |
|-----|---------|
| **Fabric API** | Requerido por la mayoría de mods | [Modrinth](https://modrinth.com/mod/fabric-api) |

### Sube los mods:
1. En Pterodactyl ve a **Files**
2. Navega a la carpeta `mods`
3. Click en **Upload** y sube todos los `.jar`

---

## Paso 4: Configurar server.properties

Edita el archivo `server.properties` con estos valores optimizados:

```properties
# RENDIMIENTO - Valores optimizados para VPS limitado
view-distance=6
simulation-distance=4
max-tick-time=60000
network-compression-threshold=256
max-players=5
spawn-protection=0
entity-broadcast-range-percentage=50

# GENERACIÓN DE MUNDO
level-type=default
generate-structures=true
spawn-monsters=true
spawn-animals=true
spawn-npcs=true

# RED
server-port=25565
server-ip=
enable-query=false
enable-rcon=false
prevent-proxy-connections=false
online-mode=true

# GAMEPLAY
pvp=true
allow-flight=false
difficulty=normal
gamemode=survival
hardcore=false

# OTROS
motd=Servidor Fabric 1.20.1
enable-command-block=true
allow-nether=true
```

### Explicación de valores críticos:

| Propiedad | Valor | Por qué |
|-----------|-------|---------|
| `view-distance` | 6 | Menos chunks = menos CPU (default es 10) |
| `simulation-distance` | 4 | Solo simula entidades cerca del jugador |
| `max-tick-time` | 60000 | Evita que el servidor crashee por lag |
| `entity-broadcast-range-percentage` | 50 | Reduce tráfico de red |

---

## Paso 5: Configurar ServerCore (Opcional pero Recomendado)

Si instalaste ServerCore, crea/edita `config/servercore.json`:

```json
{
  "activation_range": {
    "enabled": true,
    "villager": 16,
    "monster": 24,
    "animal": 16,
    "misc": 8
  },
  "breeding_cap": {
    "enabled": true,
    "max_per_chunk": 20
  },
  "lobotomize_villagers": {
    "enabled": true,
    "check_interval": 100
  }
}
```

Esto reduce MUCHO el uso de CPU limitando la IA de entidades.

---

## Paso 6: Configurar Reinicios Automáticos

Los reinicios periódicos ayudan a mantener el rendimiento y evitan memory leaks.

### En Pterodactyl:

1. Ve a tu servidor > **Schedules**
2. Click en **Create Schedule**
3. Configura:
   - **Name:** Reinicio Diario
   - **Minute:** `0`
   - **Hour:** `6` (6 AM)
   - **Day of Month:** `*`
   - **Month:** `*`
   - **Day of Week:** `*`
4. Guarda y agrega una tarea:
   - **Action:** Send Power Action
   - **Payload:** `restart`

---

## Paso 7: Monitorear Recursos

### Desde SSH en tu VPS:

Ver uso de CPU y RAM en tiempo real:
```bash
htop
```

Ver solo Docker (donde corre el servidor):
```bash
docker stats
```

### Desde Pterodactyl:

Ve a tu servidor > **Console** - Verás el uso de RAM y CPU en la barra lateral.

---

## Paso 8: Tips para Evitar Limitaciones de Hostinger

### Lo que Hostinger penaliza:
- Uso de CPU al 100% por tiempo prolongado
- Spikes constantes de CPU

### Cómo evitarlo:

1. **CPU Limit en 150%:** Ya configurado en el Paso 1. Esto limita a 1.5 cores de los 2 disponibles.

2. **No pre-generar chunks durante gameplay:**
   ```
   # En consola del servidor, ANTES de que entren jugadores:
   /worldborder center 0 0
   /worldborder set 3000
   ```
   Luego usa un mod como Chunky para pre-generar OFFLINE.

3. **Evitar granjas masivas de mobs:**
   - Las granjas de hierro, raids, etc. causan lag
   - Si las usas, ponles límite de entidades

4. **Limitar explosiones:**
   Si usas TNT, considera el mod `Carpet` con la regla `explosionNoBlockDamage`.

5. **No usar shaders del lado del servidor:**
   Los shaders son solo cliente, no afectan al servidor.

---

## Resumen de Configuración Final

| Recurso | Valor Asignado | Para el Sistema |
|---------|----------------|-----------------|
| RAM | 5GB | 3GB libres |
| CPU | 150% (1.5 cores) | 0.5 cores libres |
| Disco | 20GB | 80GB libres |

### Mods instalados:
- Fabric API
- Lithium
- Starlight
- FerriteCore
- Krypton
- LazyDFU
- C2ME
- ServerCore

---

## Conexión al Servidor

Los jugadores se conectan con:
```
147.93.118.226:25565
```

O si el puerto es el default (25565):
```
147.93.118.226
```

---

## Solución de Problemas

### El servidor usa mucha CPU:
1. Reduce `view-distance` a 4
2. Reduce `simulation-distance` a 3
3. Verifica que no haya granjas de mobs activas

### El servidor se queda sin RAM:
1. Verifica que -Xms y -Xmx sean iguales
2. Quita mods de contenido pesados
3. Reduce el tamaño del mundo con worldborder

### Hostinger suspende el VPS:
1. Reduce CPU Limit a 100% en Pterodactyl
2. Contacta soporte explicando que es un servidor de juegos
3. Considera un VPS dedicado para gaming

### TPS bajos (lag):
En consola escribe `/tps` (necesitas mod Spark para ver TPS detallado)
- 20 TPS = Perfecto
- 15-19 TPS = Aceptable
- Menos de 15 = Lag notable

---

## Comandos Útiles en Consola

```bash
# Ver TPS (si tienes Spark instalado)
/spark tps

# Guardar el mundo
/save-all

# Expulsar a todos (antes de reiniciar)
/kick @a Reinicio del servidor

# Detener servidor
/stop
```

---

**Con esta configuración tu servidor debería correr sin problemas y sin que Hostinger te limite.**

Si necesitas agregar más mods de contenido, hazlo de uno en uno y monitorea el rendimiento.

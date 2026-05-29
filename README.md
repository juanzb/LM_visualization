# Visualización Modular para LogicMachine (App-Vis)

Este proyecto provee un motor de renderizado gráfico de alta tecnología para sistemas LogicMachine. Su arquitectura está diseñada bajo el principio de **"Cero Código, Sólo Configuración"**. El integrador o usuario final **únicamente necesita modificar un objeto JSON** para generar automáticamente toda la interfaz gráfica, la navegación, el control KNX, los roles de seguridad y el ruteo.

Todo el motor lógico ha sido precompilado utilizando Vite, entregándote un paquete altamente optimizado.

---

## 1. Instalación Rápida en Producción

Para instalar esta librería en un servidor LogicMachine, no necesitas tocar el código fuente, solo hacer lo siguiente:

1. **Sube los Archivos Compilados**:
   Toma la carpeta `/dist` generada por Vite y subela por FTP a la LogicMachine (ejemplo: a la ruta `/user/`).

2. **Inyecta el Cargador en LogicMachine**:
   Ve a la sección "Custom JavaScript" de tu visualización en la LogicMachine y pega la estructura del JSON junto con el bootstrapper.

### El Bootstrapper (Cargador)

Este es el único bloque de código que debes incluir al final de tu configuración en LogicMachine para encender el motor:

```javascript
(function startBMS() {
  const globalCss = document.createElement("link");
  globalCss.rel = "stylesheet";
  globalCss.href = "/user/dist/app-vis.css"; // Ruta donde subiste el CSS
  document.head.appendChild(globalCss);

  const compScript = document.createElement("script");
  compScript.type = "module";
  compScript.src = "/user/dist/app-vis.js"; // Ruta donde subiste el JS moderno
  document.body.appendChild(compScript);
})();
```

---

## 2. Guía Pedagógica del JSON

El JSON es el corazón de tu aplicación. Todo lo que dibujes y configures aquí, el motor lo transformará en una interfaz profesional automáticamente. Se divide en dos objetos globales: **`window.AuthData`** (Seguridad Global) y **`window.appData`** (Estructura Visual y Seguridad Granular).

> [!TIP]
> Puedes consultar el archivo `demo/demo-config.js` para ver un ejemplo funcional y completo de cómo se debe estructurar este JSON.

### 2.1 Seguridad Global y Roles (`window.AuthData`)

Aquí defines el diccionario maestro: quiénes son los usuarios, qué rol tienen y qué permisos generales poseen.

```javascript
window.AuthData = {
  roles: {
    // modeProfile: "manipular" permite usar botones y sliders. "ver" bloquea cualquier interacción.
    // showSchedule / showTrends: Habilita o deshabilita los botones de modales globales.
    admin: { modeProfile: "manipular", showSchedule: true, showTrends: true },
    operador: {
      modeProfile: "manipular",
      showSchedule: false,
      showTrends: false,
    },
    invitado: { modeProfile: "ver", showSchedule: false, showTrends: false },
  },
  users: {
    admin: "admin", // El usuario "admin" adquiere el rol "admin"
    pedro_op: "operador", // El usuario "pedro_op" adquiere el rol "operador"
    visitante1: "invitado",
  },
};
```

### 2.2 Seguridad Granular (Por Objeto)

Además de la seguridad global, el motor permite **bloquear o dar permisos específicos a casi cualquier objeto** del JSON (Pisos, Zonas, Sistemas e incluso Tarjetas individuales). Esto se logra agregando dos claves opcionales a cualquier objeto:

- `roles_permission`: `["rol_a", "rol_b"]` -> Si esta clave existe, **solamente** los roles aquí listados podrán ver el objeto. Los demás ni siquiera sabrán que existe.
- `denied_users`: `["usuario_c"]` -> Bloquea el acceso a usuarios específicos, incluso si su rol se los permite.

**Ejemplo de un Sistema bloqueado:**

```javascript
{
  id: "sys_bodega_segura",
  label: "Bodega de Valores",
  roles_permission: ["admin"], // ¡Solo los administradores verán este sistema en el menú!
  denied_users: ["pedro_op"],  // pedro_op tiene prohibido verlo explícitamente
  view_type: "tarjetas_mixtas",
  data: [ ... ]
}
```

### 2.3 Navegación entre Visualizaciones Externas (`url_lm`)

En proyectos inmensos (como un hotel de múltiples torres), es común que cada torre tenga su propio proyecto/visualización en la LogicMachine. Para vincularlas sin que el usuario lo note, puedes usar la clave `url_lm`.
Esta clave se puede agregar a un **Piso (`level`)** o a una **Zona (`zone`)**.

Al hacer clic en un menú que contenga esta clave, el sistema **no dibujará las tarjetas locales**, sino que recargará el navegador enviando al usuario a la URL especificada, arrastrando inteligentemente la navegación en la barra de direcciones.

**Ejemplo:**

```javascript
{
  id: "torre_b",
  label: "Torre B - Administrativa",
  url_lm: "http://192.168.0.175/scada-vis/" // ¡Al hacer clic, el usuario será redirigido aquí!
}
```

---

## 3. La Jerarquía Física (`levels` -> `zones` -> `systems` -> `data`)

El menú de navegación y las páginas se generan leyendo la propiedad `levels` de `window.appData`. La regla de oro es: **Un Piso (`level`) tiene Zonas (`zones`), una Zona tiene Sistemas (`systems`), y un Sistema tiene Tarjetas (`data`)**.

```javascript
levels: [
  {
    id: "piso_1",
    label: "Piso 1 - Áreas Comunes",
    zones: [
      {
        id: "z_recepcion",
        label: "Recepción",
        systems: [
          {
            id: "sys_ilu",
            label: "Iluminación",
            category: "sys_ilu", // Opcional para agrupar sistemas lógicamente
            view_type: "tarjetas_mixtas", // Cómo se dibuja la página
            data: [
              // ---> ¡AQUÍ ADENTRO VAN LAS TARJETAS (CARDS)! <---
            ],
          },
        ],
      },
    ],
  },
];
```

---

## 4. Catálogo de Tarjetas (Cards) - Referencia Completa

El array `data` dentro de tus sistemas es una lista de objetos. **La clave más importante de cada objeto es `card_type`**, ya que le dice al motor qué diseño debe inyectar.

Aquí tienes el objeto **completo con absolutamente todas las claves posibles** para cada tarjeta. Si omites claves opcionales, la tarjeta simplemente se adaptará y ocultará los botones correspondientes.

### A. Iluminación / Dimmers (`card_type: "app-card-ilu"`)

Ideal para focos simples (On/Off) o luces dimeables con integración de sensores.

```javascript
{
  id: "luz_1",
  circuit: "L125",
  card_type: "app-card-ilu",
  name: "Ducto Edificación 1",

  // (Seguridad granular opcional)
  roles_permission: ["admin", "operador"],
  denied_users: ["visitante1"],

  // Control On/Off
  on_off: "1/1/1",       // Dirección KNX de Escritura (1 bit)
  rs_on_off: "1/2/1",    // Dirección KNX de Lectura / Feedback (1 bit)

  // Control Dimmer (Opcional)
  byte: "1/3/1",         // Dirección KNX de Escritura de Brillo (1 byte, 0-100%)
  rs_byte: "1/4/1",      // Dirección KNX de Lectura de Brillo (1 byte, 0-100%)

  // Modales Iframe (Opcional - Activan los botones superiores)
  url_schedule: "/scada-vis/schedulers?id=1&nohol",
  url_trend: "/scada-vis/trends?id=1&mode=day",

  // Sensor de Movimiento (Opcional - Activa el botón de configuración del sensor)
  sensor: {
    name: "SP3-102",
    unidad: "min",           // Etiqueta visual para el usuario
    limite_maximo: 20,       // Límite máximo en el input del UI
    dg_tiempo: "2/1/1",      // KNX Ajuste del tiempo
    dg_bloqueo: "2/1/4",     // KNX Bloqueo del sensor (Lectura y Escritura para control y estado visual)
    url_schedule: "/scada-vis/schedulers?id=2", // Horarios propios del sensor
    url_trend: "/scada-vis/trends?id=2"         // Tendencias propias del sensor
  }
}
```

### B. Climatización / AC (`card_type: "app-card-acc"`)

Un termostato completo con control de modos, velocidad y oscilación.

```javascript
{
  id: "clima1",
  card_type: "app-card-acc",
  name: "Modo Reunión",

  // Control On/Off de la Máquina
  on_off: "4/1/1",
  rs_on_off: "4/1/2",
  temp_actual: "4/1/99", // Lectura de Temperatura Ambiente

  // Modales Iframe Generales
  url_schedule: "/scada-vis/schedulers?id=10",
  url_trend: "/scada-vis/trends?id=10",

  // Control de Temperatura (Setpoint)
  consigna: {
    dg_action: "4/1/3",
    min_limit: 16,
    max_limit: 30,
    url_schedule: "/scada-vis/schedulers?id=11",
    url_trend: "/scada-vis/trends?id=11"
  },
  rs_consigna: "4/1/4", // Feedback del Setpoint

  // Control de Modos (Solo se mostrarán los declarados aquí)
  mode: {
    dg_action: "4/1/5",
    cool: { name: "frío", value: 1 },
    fan: { name: "ventilador", value: 2 },
    heat: { name: "calor", value: 3 },
    humedad: { name: "humedad", value: 4 },
    auto: { name: "auto", value: 5 },
    dry: { name: "dry", value: 6 }
  },
  rs_mode: "4/1/6",

  // Control de Velocidad (Solo se mostrarán las declaradas)
  speed: {
    dg_action: "4/1/7",
    spped_1: { name: "vel 1", value: 1 },
    spped_2: { name: "vel 2", value: 2 },
    spped_3: { name: "vel 3", value: 3 },
    spped_4: { name: "vel 4", value: 4 }
  },
  rs_speed: "4/1/8",

  // Control de Oscilación (Swing)
  swim: "4/1/9",
  rs_swim: "4/1/10"
}
```

### C. Cortinas / Persianas (`card_type: "app-card-curtain"`)

Control de motores de persianas, con posibilidad de slider de posición.

```javascript
{
  id: "cort1",
  card_type: "app-card-curtain",
  name: "Persiana Ventanal",

  // Movimiento Fijo (Arriba / Abajo)
  move: {
    dg_mover: "5/1/1",
    open: 0,  // Byte/Bit enviado a KNX para abrir
    close: 1  // Byte/Bit enviado a KNX para cerrar
  },

  // Detener el motor
  stop: "5/1/2",

  // Slider de Posición Fina (0-100%) - Opcional
  position: "5/1/3",
  rs_position: "5/1/4",

  // Modales
  url_schedule: "/scada-vis/schedulers?id=20",
  url_trend: "/scada-vis/trends?id=20"
}
```

### D. Botones de Escena (`card_type: "app-card-scene"`)

Tarjetas simples de un solo clic para gatillar escenas preconfiguradas.

```javascript
{
  id: "esc1",
  card_type: "app-card-scene",
  name: "Modo Película",

  dg_ejecutar: "2/1/4", // Dirección KNX que activa la escena
  valor: "23",          // El byte numérico que dispara la escena (Default)

  // Opcional: Si necesitas mandar un booleano (0/1) en vez de un byte:
  data_type: "bool"     // Escribe "bool" si la orden es de 1-bit.
}
```

# Especificaciones técnicas — Migración a Firebase
## Sistema de Pre-Nómina y Control Laboral · Grupo LEM

**Versión:** 1.0 · 6 de junio de 2026
**Autor:** Dirección General (Sergio Rountree) con asistencia de Claude
**Sistema de referencia:** `sistema_prenomina_grupo_lem.html` (STORAGE_KEY `lem_app_v16`)

Este documento es el plano completo para programar el backend en Firebase. Toda regla aquí descrita fue extraída y verificada contra el código del prototipo funcional. El prototipo es la fuente de verdad del comportamiento; este documento es la fuente de verdad de la arquitectura.

---

## 1. Objetivo y alcance

**Qué se migra:** el prototipo actual funciona en un solo archivo HTML (React + Tailwind + Babel) con datos en `localStorage` del navegador. Esto significa que los datos viven en UNA sola computadora y no se comparten. La migración lleva los datos a Firebase para que los 5 roles trabajen al mismo tiempo desde cualquier dispositivo, con datos centralizados, seguros y respaldados.

**Qué NO cambia:** la interfaz, los módulos y las reglas de negocio ya están definidos y probados en el prototipo. La migración reemplaza la capa de datos (localStorage → Firestore) y la capa de acceso (login de mentira → Firebase Authentication). El archivo HTML único se conserva como frontend durante toda la migración (el SDK de Firebase se carga por CDN, igual que React).

**Fuera de alcance de esta etapa:** horas extra, deducciones (ISR/IMSS), timbrado de nómina (CFDI), integración con contabilidad externa.

## 2. Arquitectura propuesta

| Componente | Servicio Firebase | Para qué |
|---|---|---|
| Identidad y acceso | **Firebase Authentication** | Login con correo y contraseña; recuperación de contraseña |
| Base de datos | **Cloud Firestore** | Todos los datos (colaboradores, pre-nóminas, etc.) en tiempo real |
| Seguridad de datos | **Security Rules** | Permisos por rol aplicados EN el servidor, no solo en pantalla |
| Publicación de la app | **Firebase Hosting** | URL propia (p. ej. `prenomina.grupolem.mx`) con HTTPS |
| Lógica de servidor | **Cloud Functions** (mínimas) | Crear cuentas de usuario desde el panel Admin; auditoría inmutable |

**Plan de cobro:** el plan gratuito (Spark) cubre Auth, Firestore y Hosting a la escala de Grupo LEM (107 colaboradores, 16 sucursales, capturas semanales quedan muy por debajo de los límites gratuitos: 50,000 lecturas y 20,000 escrituras diarias). Cloud Functions requiere subir al plan Blaze (pago por uso), que a este volumen cuesta ~$0/mes pero pide tarjeta. La misma decisión ya se tomó con la app de inventarios.

**Patrón igual que Inventarios:** esta migración replica el camino ya recorrido con la app de inventarios de Doña Tota (Firebase/JavaScript), por lo que el equipo ya conoce la consola de Firebase.

## 3. Autenticación y cuentas de usuario

### 3.1 Cómo funcionan los correos (decisión clave)

- Firebase **no crea correos**; no es un servidor de correo. Se registra el correo que la persona **ya tiene**.
- El correo es solo el **identificador de acceso**. Puede ser Gmail personal; no se requiere buzón @grupolem.mx.
- Si el correo es real, funciona "olvidé mi contraseña" (Firebase envía el enlace). Con correos inventados, solo el Admin puede restablecer contraseñas.
- **Decisión de dirección:** usar los correos personales reales de gerentes y asesores (ya están capturados en el Excel de colaboradores; 106 de 107 tienen correo).

### 3.2 Modelo de cuentas

Se conserva la separación actual del prototipo:

- **Cuenta de acceso (usuario):** quién puede entrar al sistema y con qué rol. Vive en Firebase Auth + documento en `usuarios/{uid}`.
- **Expediente (colaborador):** los datos laborales de la persona. Vive en `colaboradores/{id}`.
- Un gerente necesita ambos, vinculados por el campo `colaboradorId` del usuario.

### 3.3 Flujo de alta de un usuario (módulo "Usuarios y accesos", solo Admin)

1. Admin captura nombre, correo, rol y (opcional) colaborador vinculado.
2. Una **Cloud Function** (`crearUsuario`) crea la cuenta en Firebase Auth con contraseña temporal y escribe el documento en `usuarios/{uid}` con el rol. *(Crear usuarios desde el cliente no es posible sin cerrar la sesión del Admin; por eso se usa una Function con el Admin SDK.)*
3. El rol se copia también a **custom claims** del token, para que las Security Rules lo lean sin consultas extra.
4. El usuario entra con la contraseña temporal y el sistema le pide cambiarla.

### 3.4 Cuentas iniciales

Las tres cuentas actuales se crean en Auth desde el inicio: director@grupolem.mx (admin), rh@grupolem.mx (rh), contabilidad@grupolem.mx (contabilidad). La contraseña "demo" desaparece: cada cuenta tendrá contraseña propia y segura.

## 4. Modelo de datos en Firestore

Notación: `coleccion/{documento}`. Los nombres de campos replican el prototipo para que el frontend cambie lo mínimo.

### 4.1 Mapa general

| Prototipo (state) | Firestore | Tipo |
|---|---|---|
| colaboradores[] | `colaboradores/{colabId}` | colección |
| (sueldoDiario, bonoTransporte) | `colaboradores/{colabId}/privado/compensacion` | subcolección restringida |
| usuarios[] | `usuarios/{uid}` | colección |
| sucursales[] | `sucursales/{sucId}` | colección |
| prenominas{} | `prenominas/{sucId_weekId}` | colección |
| horarios{} | `horarios/{sucId_weekId}` | colección |
| vacaciones[] | `vacaciones/{vacId}` | colección |
| bajas[] | `bajas/{bajaId}` | colección |
| movimientos[] | `movimientos/{movId}` | colección |
| (montos de mov. de sueldo) | `movimientos/{movId}/privado/montos` | subcolección restringida |
| contratos[] | `contratos/{ctrId}` | colección |
| auditoria[] | `auditoria/{logId}` | colección (solo agregar) |
| notificaciones[] | `notificaciones/{notifId}` | colección |
| SEED (unidades, puestos, motivosBaja, indicadores) | `config/catalogos` | documento único |

### 4.2 `colaboradores/{colabId}` — expediente SIN sueldos

```
noEmp            number   · consecutivo de empleado
usuarioId        string?  · uid de su cuenta de acceso (si tiene)
curp             string   · 18 caracteres; extranjeros sin CURP usan 'XXXX…' (caso real: Mauricio Rodríguez)
rfc              string?  
nss              string?  
nombre           string
apellidoPaterno  string
apellidoMaterno  string?  · NO obligatorio (casos reales: Yolanda Alicia Castillo, Gilberto Ariel Diaz)
fechaNacimiento  string   · 'YYYY-MM-DD'
telefono         string
correo           string?
sucursalId       string
puestoId         string   · referencia al catálogo de puestos
gerenteId        string   · uid responsable
fechaIngreso     string   · 'YYYY-MM-DD' (base para vacaciones por aniversario)
statusIMSS       string   · 'alta' | 'pendiente'
estatus          string   · 'activo' | 'baja'
fechaBajaActual  string?  · si estatus = 'baja'
recontratado     bool?    · más fechaRecontratacion y recontratadoPor
```

**REGLA DE ORO a nivel base de datos:** `sueldoDiario` y `bonoTransporte` NO van en este documento. Van en `colaboradores/{colabId}/privado/compensacion`:

```
sueldoDiario     number   · sueldo POR DÍA (p. ej. 315.04)
bonoTransporte   number   · bono semanal fijo (p. ej. 200)
```

Así, aunque alguien manipule el navegador, el servidor le niega la lectura de montos a RH, Asesor y Gerente. En el prototipo esta regla solo existe en la interfaz; en Firebase queda blindada en el servidor.

### 4.3 `usuarios/{uid}` — cuentas de acceso

```
email         string
rol           string   · 'admin' | 'rh' | 'contabilidad' | 'asesor' | 'gerente'
nombre        string
puestoLabel   string?
colaboradorId string?  · vínculo a su expediente
activo        bool
```
El campo `password` del prototipo desaparece (lo maneja Firebase Auth). El rol se duplica en custom claims.

### 4.4 `sucursales/{sucId}`

```
nombre             string   · 16 sucursales actuales
unId               string   · 'corp' | 'wings' | 'tota' | 'mini'
gerenteId          string?  · uid del gerente asignado (módulo Sucursales)
asesorId           string?  · uid del asesor asignado
plantillaObjetivo  number   · para indicador de vacantes
```
El alcance de gerentes y asesores se deriva SIEMPRE de estos dos campos (fuente única de verdad, igual que el prototipo: `sucursalesDeGerente`/`sucursalesDeAsesor`).

### 4.5 `prenominas/{sucursalId_weekId}` — captura semanal

ID del documento: `{sucursalId}_{weekId}`, donde weekId = `'2026-W23'` (año + número de semana; semana inicia lunes; **domingo es el día 7**, índice 6).

```
sucursalId             string
weekId                 string
semanaInicio           string  · lunes 'YYYY-MM-DD'
semanaFin              string  · domingo
estado                 string  · 'borrador' → 'enviada' → 'aprobada' | 'rechazada' → 'corregida' → … → 'cerrada'
detalle                map     · { colabId: { '0'..'6': { indicador: 'A'|'D'|'FI'|'INC'|'VAC'|'PSGS'|'PCGS'|'BAJA', comentario? } } }
prellenadaDesdeHorario bool
fechaCreacion / fechaEnvio / enviadoPor
motivoRechazo / rechazadoPor (si aplica)
```

VAC se prellena automáticamente desde solicitudes de vacaciones aprobadas que crucen la semana. Una pre-nómina `aprobada` o `cerrada` es de solo lectura para el gerente.

### 4.6 `horarios/{sucursalId_weekId}`

Horario de la semana PRÓXIMA definido por el gerente. `estado: 'borrador' | 'definido'`, `detalle` por colaborador y día. Alimenta el prellenado de la pre-nómina y la **alerta de descansos en fin de semana** (viernes/sábado/domingo) visible para asesor y admin.

### 4.7 Solicitudes con flujo de aprobación

Todas comparten el patrón `estado: 'pendiente' → 'aprobada' | 'rechazada'` + `solicitanteId`, `fechaSolicitud`, `aprobadorId`, `fechaResolucion`, `motivoRechazo?`.

**`vacaciones/{id}`:** colaboradorId, fechaInicio, fechaFin, diasNaturales, motivo. Valida saldo disponible (ver §7).

**`bajas/{id}`:** colaboradorId, fechaBaja, ultimoDiaLaborado, motivo (catálogo de 7 + texto libre en 'Otro'), comentarios, tieneDocumento. Al aprobar: el colaborador pasa a `estatus:'baja'` con `fechaBajaActual` (NUNCA se borra el expediente).

**`movimientos/{id}`:** colaboradorId, tipo `'sucursal' | 'puesto' | 'sueldo'`, detalleAnterior, detalleNuevo, resumen, fechaEfectiva, motivo. **Para tipo 'sueldo', los montos van en `movimientos/{id}/privado/montos`** (regla de oro); el documento público solo dice "cambio de sueldo" sin cifras.

**`contratos/{id}`:** colaboradorId, tipo `'temporal' | 'renovacion' | 'planta'`, fechaInicio, fechaFin (solo temporal/renovación; planta no vence). Alerta para RH cuando faltan ≤ **30 días** (DIAS_AVISO) o ya venció. Se puede registrar contrato directo al dar de alta al colaborador.

### 4.8 `auditoria/{logId}`

```
fecha, usuarioId, modulo, accion, entidad, valorAnterior, valorNuevo
```
En Firestore se vuelve **inmutable**: las reglas permiten crear, nunca editar ni borrar. Los registros que involucren montos no deben guardar cifras en claro en `valorAnterior/valorNuevo` cuando el módulo sea de sueldos (guardar solo "monto modificado").

### 4.9 `config/catalogos` (documento único)

Unidades (4), puestos (10, con `cat`: op/gerencial/asesor/corp y `bonoFI`), motivos de baja (7), indicadores (8) y la **tabla de equivalencias de puestos para importación** (25 entradas, ampliada el 6-jun-2026 con: cocinera, jefa de cocina, mesera, mesero/mesera fines → su puesto consolidado; logistica → Auxiliar Administrativo).

## 5. Matriz de permisos por rol

| Acción | Admin | RH | Contab. | Asesor | Gerente |
|---|---|---|---|---|---|
| Ver montos de sueldo/bono | ✔ | ✖ | ✔ | ✖ | ✖ |
| Configurar todo / catálogos | ✔ | ✖ | ✖ | ✖ | ✖ |
| Crear cuentas de usuario | ✔ | ✖ | ✖ | ✖ | ✖ |
| Alta de colaboradores / importar Excel | ✔ | ✔ | ✔* | ✖ | ✖ |
| Autorizar recontratación | ✔ | ✖ | ✖ | ✖ | ✖ |
| Capturar pre-nómina (sus sucursales) | ✔ | ✖ | ✖ | ✖ | ✔ |
| Aprobar/rechazar pre-nómina | ✔ | ✖ | ✔ | ✖ | ✖ |
| Calcular nómina semanal / export | ✔ | ✖ | ✔ | ✖ | ✖ |
| Aprobar vacaciones (operativos) | ✔ | ✔ | ✖ | ✔ (sus suc.) | ✖ |
| Aprobar vacaciones (asesores/corp) | ✔ | ✖ | ✖ | ✖ | ✖ |
| Aprobar bajas | ✔ | ✔ | ✖ | ✔ (sus suc.) | ✖ |
| Aprobar mov. sucursal/puesto | ✔ | ✔ | ✖ | ✔ (sus suc.) | ✖ |
| Autorizar mov. de sueldo / bonos | ✔ | ✖ | ✔ | ✖ | ✖ |
| Contratos y alertas de vencimiento | ✔ | ✔ | ✖ | ✖ | ✖ |
| Estatus IMSS | ✔ | ✔ | ✔ | ✖ | ✖ |
| Definir horarios (sus sucursales) | ✔ | ✖ | ✖ | ✖ | ✔ |
| Solicitar vacaciones/bajas/movimientos | ✔ | ✔ | ✖ | ✔ | ✔ |
| Ver auditoría | ✔ | ✖ | ✖ | ✖ | ✖ |

\* Contabilidad importa CON columnas de sueldo; RH importa sin ellas (la plantilla que descarga RH ni siquiera trae esas columnas — comportamiento ya implementado).

Gerente y Asesor solo ven documentos cuya `sucursalId` esté en sus sucursales asignadas.

## 6. Security Rules (esqueleto verificado contra la matriz)

```js
rules_version = '2';
service cloud.firestore {
  match /databases/{db}/documents {

    function rol() { return request.auth.token.rol; }
    function esAdmin() { return rol() == 'admin'; }
    function veSueldos() { return rol() in ['admin','contabilidad']; }
    function misSucursalesIncluyen(sucId) {
      return get(/databases/$(db)/documents/sucursales/$(sucId)).data.gerenteId == request.auth.uid
          || get(/databases/$(db)/documents/sucursales/$(sucId)).data.asesorId  == request.auth.uid;
    }

    match /colaboradores/{id} {
      allow read: if rol() in ['admin','rh','contabilidad']
               || (rol() in ['gerente','asesor'] && misSucursalesIncluyen(resource.data.sucursalId));
      allow create, update: if rol() in ['admin','rh','contabilidad'];
      allow delete: if false; // las bajas cambian estatus, nunca borran

      match /privado/compensacion {
        allow read, write: if veSueldos();   // ← REGLA DE ORO
      }
    }

    match /prenominas/{id} {
      allow read: if rol() in ['admin','contabilidad','rh']
               || (rol() in ['gerente','asesor'] && misSucursalesIncluyen(resource.data.sucursalId));
      // Gerente captura y envía solo en sus sucursales y solo si no está aprobada/cerrada
      allow create, update: if (rol() == 'gerente'
                                && misSucursalesIncluyen(request.resource.data.sucursalId)
                                && !(resource.data.estado in ['aprobada','cerrada']))
                            || rol() in ['admin','contabilidad'];
    }

    match /auditoria/{id} {
      allow read: if esAdmin();
      allow create: if request.auth != null;
      allow update, delete: if false;        // inmutable
    }
    // vacaciones, bajas, movimientos, contratos, horarios: mismo patrón,
    // aprobadores según la matriz §5 (validar transición de 'pendiente' en update).
  }
}
```

## 7. Reglas de negocio (verificadas contra `calcularNominaColab`)

### 7.1 Indicadores por día

`A` asistencia · `D` descanso · `FI` falta injustificada (alerta a las 4 FI/30 días) · `INC` incapacidad (requiere folio IMSS) · `VAC` vacaciones (auto desde solicitud aprobada) · `PSGS` permiso sin goce (requiere comentario) · `PCGS` permiso con goce (requiere comentario) · `BAJA` (solo con baja aprobada). Semana de lunes (día 1, índice 0) a **domingo (día 7, índice 6)**.

### 7.2 Cálculo semanal por colaborador

El sueldo es **DIARIO**. Sueldo y bono se muestran **SEPARADOS** en todo desglose.

1. **Días base** = A + VAC + PCGS.
2. **Descanso pagado:** requiere D ≥ 1 en la semana Y días cubiertos (A+VAC+PCGS) ≥ 5. Factor = 1 − (FI/6), mínimo 0. Con < 5 cubiertos: descanso = 0. **Se paga máximo UN día de descanso por semana aunque haya dos D.**
3. **Prima dominical:** si el domingo es `A`, se suman 0.25 días (el día ya cuenta en días base; la prima es el 25% extra). Descanso en domingo = sin prima.
4. **Días pagados** = días base + factor descanso + prima dominical.
5. **Monto sueldo** = días pagados × sueldo diario (redondeo a centavos).
6. **Bono semanal fijo** (`bonoTransporte`, monto propio de cada quien): completo si NO hubo FI ni PSGS; se pierde COMPLETO con una sola FI o PSGS. VAC y PCGS no lo afectan.
7. **No se pagan:** FI, PSGS, INC (la incapacidad la cubre el IMSS).
8. **Total** = monto sueldo + monto bono.

### 7.3 Ejemplos numéricos (verificados contra el código)

Colaboradora con sueldo diario **$315.04** y bono semanal **$200**:

| Semana (L→D) | Cálculo | Sueldo | Bono | Total |
|---|---|---|---|---|
| A A A A A D A | base 6 + descanso 1 + prima 0.25 = 7.25 días | $2,284.04 | $200.00 | **$2,484.04** |
| A A FI A A D A | base 5 + descanso 5/6 + prima 0.25 = 6.0833 | $1,916.49 | $0.00 (FI) | **$1,916.49** |

Colaborador con sueldo diario **$350** y bono **$150**:

| Semana (L→D) | Cálculo | Sueldo | Bono | Total |
|---|---|---|---|---|
| A A VAC VAC A D D | base 5 + descanso 1 (solo se paga 1 D) = 6 | $2,100.00 | $150.00 | **$2,250.00** |
| A A A A D PSGS D | cubiertos 4 < 5 → sin descanso; base 4 | $1,400.00 | $0.00 (PSGS) | **$1,400.00** |

La nómina se calcula por colaborador y por sucursal, con exportación a Excel (solo Admin/Contabilidad).

### 7.4 Vacaciones (LFT reforma 2023, por aniversario)

El año de vacaciones corre de aniversario a aniversario de `fechaIngreso`. Días por años cumplidos: 1→12, 2→14, 3→16, 4→18, 5→20, y de 6 en adelante +2 por cada bloque de 5 años (6-10→22, 11-15→24…). La solicitud valida saldo disponible del periodo vigente.

### 7.5 Recontratación

Al capturar CURP o RFC de un ex-colaborador (`estatus:'baja'`), el sistema lo detecta y **solo el Administrador** puede autorizar la recontratación (marca explícita). El expediente se reactiva conservando historial (`recontratado: true`).

### 7.6 Contratos

Cadena Temporal → Renovación → Planta. Temporal y Renovación vencen (fechaFin); Planta no. Alerta a RH con 30 días de anticipación y al vencer.

## 8. Importación de colaboradores (Excel)

Plantilla real verificada: `importar_colaboradores_FINAL.xlsx`, hoja "Colaboradores", **107 registros, 0 errores** tras la ampliación de equivalencias del 6-jun-2026.

**Columnas (14):** CURP\*, RFC, NSS, Nombre(s)\*, Apellido paterno\*, Apellido materno, Fecha de nacimiento\*, Teléfono\*, Correo, Sucursal\*, Puesto\*, Fecha de ingreso\*, Sueldo Diario†, Bonos†. (\* obligatoria · † solo visible para Admin/Contabilidad)

**Validaciones:** obligatorias presentes; sucursal existe (las 16 coinciden); puesto resuelve por equivalencias o catálogo; duplicados por CURP/RFC contra activos se omiten (y disparan flujo de recontratación si es ex-colaborador). Fechas aceptan serial de Excel, 'YYYY-MM-DD' o 'DD/MM/YYYY'. Importados quedan con `statusIMSS:'pendiente'`.

**Datos reales:** sueldos $250–$1,000 diarios; bonos $0 (29 personas), $15.96, $200 (67), $215.96, $415.96, $615.96; 1 persona sin correo; The Wings D1 es la sucursal más grande (19).

En Firebase, la importación escribe en lote (batched writes): expediente público + subdocumento privado de compensación por cada fila.

## 9. Plan de migración por fases

| Fase | Entregable | Criterio de éxito |
|---|---|---|
| 0. Preparación | Proyecto Firebase creado, plan Blaze, dominios autorizados | Consola lista; mismo proyecto-patrón que Inventarios |
| 1. Autenticación | Login real (Auth) + 3 cuentas iniciales + cambio de contraseña | Entra cada rol con SU contraseña; "demo" eliminado |
| 2. Datos maestros | `colaboradores` (con subcolección privada), `sucursales`, `config/catalogos` + importación de los 107 | Los 107 visibles; RH no puede leer compensación ni con consola del navegador |
| 3. Operación semanal | `prenominas` y `horarios` en tiempo real | Dos gerentes capturan a la vez sin pisarse; Contabilidad aprueba |
| 4. Flujos | `vacaciones`, `bajas`, `movimientos`, `contratos`, `notificaciones` | Solicitud → aprobación entre dos sesiones distintas |
| 5. Nómina y auditoría | Cálculo semanal sobre Firestore + `auditoria` inmutable + export Excel | Totales idénticos a los del prototipo (mismos ejemplos §7.3) |
| 6. Publicación | Hosting con URL propia + reglas endurecidas + respaldo programado | Gerentes entran desde sus sucursales con su cuenta |

Regla de trabajo entre fases: el HTML sigue siendo el archivo único de trabajo; cada fase debe compilar con Babel y poder demostrarse en el navegador. La versión localStorage se conserva como respaldo hasta cerrar la Fase 5.

## 10. Decisiones pendientes (Dirección)

1. **Correo por gerente/asesor:** confirmar la lista de correos personales reales para crear sus cuentas (Fase 1-2).
2. **"Gerente General" → Gerente de Sucursal** quedó como equivalencia provisional (3-jun): ¿se queda así o se crea el puesto?
3. **Dominio propio** para Hosting (`prenomina.grupolem.mx`) o subdominio gratuito `.web.app`.
4. **Semana con dos descansos:** hoy se paga máximo 1 día de descanso (§7.2.2). Confirmar que es el comportamiento deseado.
5. **Quién ejecuta la creación del proyecto Firebase** (mismo dueño/cuenta Google que la app de Inventarios, se recomienda).

## 11. Glosario

**weekId:** identificador de semana `AAAA-Wnn` (p. ej. 2026-W23); semana lunes-domingo. · **Custom claims:** datos (como el rol) sellados dentro del token de sesión, legibles por las Security Rules sin consultar la base. · **Batched write:** escritura en lote, todo-o-nada. · **Spark/Blaze:** planes de Firebase (gratuito / pago por uso). · **Pre-nómina:** captura semanal de indicadores por día, previa al cálculo. · **Regla de oro:** solo Admin y Contabilidad ven montos salariales.

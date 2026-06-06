# Sistema de Pre-Nómina y Control Laboral · Grupo LEM

Aplicación web para la captura semanal de pre-nómina, control laboral y cálculo de nómina de Grupo LEM (4 unidades de negocio, 16 sucursales, ~107 colaboradores).

## Estado

**Prototipo funcional completo** en un solo archivo HTML (React + Tailwind + Babel, datos en localStorage del navegador). En migración hacia Firebase (Auth + Firestore + Hosting) según el documento de especificaciones.

## Archivos

| Archivo | Qué es |
|---|---|
| `sistema_prenomina_grupo_lem.html` | La aplicación completa. Se abre directo en el navegador. Versión interna: `lem_app_v16` |
| `Especificaciones_Firebase_PreNomina_LEM.md` | Plano técnico para programar el backend en Firebase (modelo de datos, permisos, reglas de nómina, plan de fases) |

## Módulos

Dashboard por rol · Pre-nóminas semanales · Horarios · Colaboradores (importar/exportar Excel) · Sucursales · Movimientos internos · Nómina semanal · Bajas · Vacaciones (LFT 2023) · Reportes imprimibles · Usuarios y accesos · Contratos · Auditoría.

## Roles

Administrador, RH, Contabilidad, Asesor y Gerente de sucursal. **Regla de oro:** solo Administrador y Contabilidad ven montos salariales.

## Uso (prototipo)

1. Abrir `sistema_prenomina_grupo_lem.html` en Chrome/Edge.
2. Entrar con una cuenta de demostración (contraseña: `demo`).
3. Los datos se guardan en el navegador (localStorage). Cambiar de computadora o borrar datos del navegador reinicia el sistema.

> **Importante:** la plantilla de colaboradores reales (Excel) NO se versiona en este repositorio por contener datos personales (CURP, NSS, sueldos).

---
Grupo LEM · Chihuahua, México · Proyecto interno

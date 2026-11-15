# EduGest · UD01 · Informe de análisis

| | |
|---|---|
| **Proyecto** | EduGest: gestión académica del IES Serra Gelada |
| **Práctica** | EduGest-1 (RA1: c, d, f, i) |
| **Autor** | Carlos García Durendez (1º ASIR semipresencial) |
| **Fecha** | 2026-11-15 |

---

## 1. Situación actual

Hoy el centro trabaja con **una hoja de cálculo por ciclo** (alumnado, matrículas y notas) y **una hoja de faltas por cada profesor**. Es un sistema basado en ficheros independientes, y tiene los inconvenientes típicos de ese modelo: redundancia, inconsistencia, dificultad de acceso, ausencia de control de integridad, de concurrencia y de seguridad.

### Problema 1 · Datos repetidos e inconsistentes (redundancia)

Los datos personales de un alumno aparecen en la hoja de su ciclo **y** en la hoja de faltas de cada uno de sus profesores (en un grupo de primero, unas seis o siete hojas distintas). Si el alumno cambia de teléfono o de correo, hay que corregirlo en todas. En la práctica alguna se queda sin actualizar y el centro no sabe cuál es el dato bueno. Además, el módulo *0484 Bases de datos* está en DAM y en DAW, y cada hoja lo describe a su manera (nombre, horas), así que nada garantiza que coincidan.

### Problema 2 · No se puede cruzar información sin hacerlo a mano

Preguntas tan normales como «¿qué alumnos de 1DAW superan el 15 % de faltas sin justificar en *Bases de datos*?» o «¿cuántas horas lectivas tiene asignadas cada profesor este curso?» obligan a abrir varias hojas, copiar y pegar, y sumar a mano. Las faltas están repartidas entre las hojas personales de cada docente, y la tutoría no tiene una visión conjunta de su grupo. Esto es lento y provoca errores justo en los procesos más delicados: avisos de pérdida de evaluación continua, actas y estadísticas para la Conselleria.

### Problema 3 · Ni integridad, ni concurrencia, ni seguridad

- **Integridad:** una hoja de cálculo deja escribir un 11 como nota, una convocatoria 7, un grupo que no existe o matricular dos veces al mismo alumno en el mismo módulo y curso. Nada lo impide.
- **Concurrencia:** si dos personas de secretaría abren y guardan la misma hoja a la vez, los cambios de una pisan los de la otra.
- **Seguridad y protección de datos:** quien tiene la hoja lo ve todo (DNI, teléfono, fecha de nacimiento, notas). No hay perfiles ni registro de accesos. Los ficheros se copian a portátiles, pendrives o correos, lo que multiplica el riesgo de una **brecha de datos personales**.

---

## 2. Usuarios del sistema

| Perfil en el centro | Qué necesita hacer (operaciones concretas) | Perfil de usuario del SGBD |
|---|---|---|
| **Secretaría** | Dar de alta alumnos y actualizar sus datos personales · Formalizar matrículas (módulo, curso académico, convocatoria) · Emitir certificados y actas · Asignar o cambiar el grupo de un alumno | **Usuario final** (formularios de la aplicación de gestión). Necesita privilegios de inserción y modificación sobre alumnado y matrículas |
| **Profesorado** | Registrar faltas de asistencia de sus grupos · Justificar faltas · Poner las notas finales de sus módulos · Consultar la lista de alumnos de sus grupos | **Usuario final**. Solo debe ver **sus** grupos y módulos y sin datos de contacto que no necesite (vista restringida) |
| **Tutoría** | Consultar las faltas de todo su grupo y en todos los módulos · Ver el resumen de notas del grupo para la sesión de evaluación · Contactar con el alumnado (correo y teléfono) | **Usuario final** con una vista ampliada limitada a su grupo |
| **Jefatura de estudios** | Crear grupos y asignar tutores · Asignar qué profesor imparte cada módulo a cada grupo y cuántas horas · Controlar que nadie supere las 20 horas lectivas · Obtener estadísticas de absentismo y de aprobados por ciclo | **Usuario avanzado / analista**: lanza consultas e informes sobre todo el centro, además de las operaciones de planificación |
| **Alumnado** | Consultar sus notas · Consultar sus faltas · Revisar que sus datos personales son correctos (derecho de acceso) | **Usuario final** de solo lectura y limitado a **sus propias filas** |
| **Administrador de la base de datos (DBA)** | Instalar y configurar Oracle · Crear usuarios, roles y privilegios · Programar y comprobar copias de seguridad · Vigilar el rendimiento y aplicar parches · Revisar la auditoría de accesos | **DBA** |
| **Equipo de desarrollo** (empresa o departamento de Informática) | Diseñar el esquema · Programar la aplicación y la lógica en PL/SQL (triggers, procedimientos) | **Diseñador** y **programador de aplicaciones** |

---

## 3. Solución propuesta

### 3.1 Tipo de SGBD

| Criterio | Elección | Justificación a partir del caso |
|---|---|---|
| **Modelo de datos** | **Relacional** | Los datos están muy estructurados y relacionados entre sí: un alumno pertenece a un grupo, el grupo a un ciclo y la matrícula une alumno, módulo y curso académico. Hay reglas de unicidad («un alumno no se matricula dos veces del mismo módulo en el mismo curso») y referencias que deben existir siempre. Es exactamente lo que resuelven las claves primarias, alternativas y ajenas. |
| **Garantías** | **Transacciones ACID** | Matrículas y notas son datos con valor legal (actas, títulos). Una promoción de curso que se quede a medias o una nota que se pierda no son aceptables. |
| **Número de usuarios** | **Multiusuario, cliente/servidor** | Decenas de docentes registran faltas al mismo tiempo, sobre todo a primera hora, y secretaría trabaja a la vez. Hace falta control de concurrencia. |
| **Ubicación** | **Centralizada** | Un único centro con unos cientos de alumnos. No se justifica una base de datos distribuida. |
| **Despliegue** | **Local (*on-premise*)** en un servidor del centro o de la Conselleria | Los datos del alumnado (muchos son menores) son sensibles. Un servidor propio, sin exponerlo a Internet, simplifica el control. Si en el futuro se migra a la nube, debe ser con un proveedor de la UE y con un contrato de encargado del tratamiento. |
| **Licencia** | **Edición gratuita** de un SGBD comercial | Un centro público no tiene presupuesto de licencias, y el volumen (miles de filas, no millones) cabe con mucha holgura en los límites de una edición gratuita. |
| **Propósito** | **OLTP** (transaccional) | El uso principal son altas, cambios y consultas del día a día. Las estadísticas son consultas puntuales y no justifican un sistema OLAP. |

### 3.2 Por qué Oracle para la parte relacional

Usaremos **Oracle AI Database 26ai Free**:

- **Coste cero y límites suficientes.** La edición Free es gratuita y sus límites de CPU, memoria y almacenamiento (unos 12 GB de datos de usuario) sobran para la información de un instituto.
- **Integridad declarativa completa.** Claves primarias, alternativas y ajenas, `CHECK`, `NOT NULL` y políticas de borrado. Las reglas que no son declarativas (jefe de departamento, límite de 20 horas...) se pueden programar **dentro del servidor** con **PL/SQL** (triggers y procedimientos), y así no dependen de la aplicación.
- **Seguridad.** Usuarios, roles, privilegios a nivel de objeto, vistas para restringir columnas y filas y auditoría. Es lo que necesita un sistema con datos personales de menores.
- **Es un estándar en la empresa.** Oracle es uno de los SGBD más implantados en grandes organizaciones y en la Administración, y es el sistema de referencia del módulo, por lo que lo que aprendamos es directamente aplicable en el trabajo.
- **Es multimodelo.** Además de tablas, admite JSON. En la UD09 se comparará con una versión documental en **MongoDB** del expediente del alumnado.

Las alternativas libres (PostgreSQL, MariaDB) también serían técnicamente válidas para este caso. La elección de Oracle se basa en la programación en el servidor (PL/SQL), en sus herramientas de seguridad y en su implantación profesional, no en que las otras no sirvan.

---

## 4. Protección de datos

Resumen de las conclusiones de la auditoría de protección de datos (práctica 1.5) aplicadas a EduGest.

### 4.1 Normativa y base legal

- **Reglamento (UE) 2016/679 (RGPD)** y **Ley Orgánica 3/2018 (LOPDGDD)**. La autoridad de control es la **AEPD**.
- El **responsable del tratamiento** es el centro (y la Conselleria de la que depende). La empresa que desarrolle o aloje EduGest sería **encargada del tratamiento** y necesita un contrato (art. 28 RGPD).
- La **base legal** de los datos académicos no es el consentimiento, sino el **cumplimiento de una misión realizada en interés público** (art. 6.1.e RGPD), amparada por la normativa educativa (disposición adicional 23.ª de la Ley Orgánica 2/2006, de Educación).

### 4.2 Inventario de datos personales

| Dato | Interesado | ¿Categoría especial? | Finalidad |
|---|---|---|---|
| NIA, DNI/NIE, nombre, apellidos | Alumno | No | Identificación y gestión académica |
| Fecha de nacimiento | Alumno | No (pero revela si es **menor**) | Gestión académica, mayoría de edad |
| Correo, teléfono, localidad | Alumno | No | Comunicación con el alumno o la familia |
| Matrículas, convocatorias, notas | Alumno | No | Expediente académico, actas |
| Faltas de asistencia | Alumno | No | Control de asistencia, evaluación continua |
| DNI, nombre, apellidos, correo, especialidad, fecha de alta | Profesor | No | Gestión de personal y docencia |
| *Diagnóstico de NEE* (petición nueva) | Alumno | **Sí, datos de salud** (art. 9) | Atención educativa por orientación |
| *Fotografía para el carné* (petición nueva) | Alumno | No (no es biométrico si solo se muestra) | Identificación en el carné |

### 4.3 Peticiones nuevas del centro

| Petición | ¿Es lícita? | Condiciones y cambios en la base de datos |
|---|---|---|
| Diagnósticos de **NEE** | Sí, para la atención educativa, pero es una **categoría especial** | Solo los datos imprescindibles. **Tabla separada** con acceso únicamente para orientación (fragmentación vertical), cifrada y con auditoría de accesos. Nunca visible para el profesorado en general. |
| **Fotografía** del alumnado | Sí, para la finalidad del carné | Informar al alumnado y a las familias. Columna o tabla aparte, accesible solo a secretaría. No usarla para otra cosa (web, redes) sin consentimiento expreso. |
| Venta de correos y teléfonos a una **autoescuela** | **No** | Es una cesión a un tercero con una finalidad incompatible (comercial), sin base legal. Vulnera los principios de **limitación de la finalidad** y de **licitud**. Da igual que el alumno sea mayor de edad. Se rechaza. |
| Guardar a los antiguos alumnos «**para siempre**» | **No**, tal como está planteado | Principio de **limitación del plazo de conservación**. El expediente académico se conserva lo que marque la normativa educativa y de archivos. Los datos de contacto y las faltas, no. Propuesta: suprimir los datos de contacto al terminar los estudios y, pasado el plazo, **anonimizar** (quitar NIA, DNI, nombre y contacto y mantener solo ciclo, curso académico y nota para estadísticas). |

### 4.4 Medidas técnicas que se aplicarán

| # | Medida | Unidad |
|---|---|---|
| 1 | **Minimización:** no crear columnas «por si acaso». Solo los atributos que pide la entrevista | UD02 |
| 2 | **Seudonimización:** identificador interno `id_alumno` como clave. El DNI no se usa como clave ajena en otras tablas | UD03 |
| 3 | **Exactitud:** `CHECK` para notas (0-10), convocatoria (1-4), turno, grado y formato de correo | UD05 |
| 4 | **Vista para el profesorado** sin DNI, teléfono ni fecha de nacimiento, y solo con el alumnado de sus grupos | UD05 |
| 5 | **Roles con mínimo privilegio** (secretaría, profesorado, tutoría, jefatura, alumnado). Nadie trabaja con el usuario propietario `EDUGEST` | UD05 |
| 6 | **Trigger de auditoría** que registre quién y cuándo modifica una nota | UD08 |
| 7 | **Tarea programada** de anonimización de antiguos alumnos al acabar el plazo de conservación | UD08 |
| 8 | **Copias de seguridad cifradas** y probadas periódicamente. El servidor no se expone a Internet | Administración (DBA) |

### 4.5 Procedimiento ante una brecha de seguridad

1. **Detectar y contener:** cortar el acceso (bloquear usuarios o la conexión afectada) y conservar las evidencias (registros de auditoría).
2. **Avisar** al responsable del tratamiento y al **delegado de protección de datos (DPD)** del centro o de la Conselleria.
3. **Evaluar** qué datos y qué personas se han visto afectados y el riesgo que supone para ellas.
4. **Notificar a la AEPD en un máximo de 72 horas** desde que se conoce la brecha (art. 33 RGPD), salvo que sea improbable que suponga un riesgo.
5. Si el riesgo es alto (por ejemplo, datos de salud o de menores), **comunicarlo también a los afectados** sin dilación indebida (art. 34 RGPD).
6. **Documentar** la brecha y las medidas tomadas, aunque no haga falta notificarla, y corregir la causa.

---

## 5. Glosario

| Término | Definición (propia) |
|---|---|
| **Dato** | Un valor suelto sin contexto, como `7,5` o `1DAW`. |
| **Información** | Datos con contexto que sirven para tomar decisiones: «Ana tiene un 7,5 en Bases de datos en 1DAW». |
| **Fichero** | Conjunto de registros guardados en un soporte. Lo maneja directamente la aplicación que lo creó. |
| **Redundancia** | Guardar el mismo dato en varios sitios. Ocupa más espacio y provoca inconsistencias cuando se cambia en uno y no en los demás. |
| **Base de datos** | Conjunto de datos relacionados, organizados y almacenados sin redundancias innecesarias, que comparten varios usuarios y aplicaciones. |
| **SGBD** | Software que crea, mantiene y da acceso a la base de datos y controla la integridad, la seguridad, la concurrencia y la recuperación. Oracle o MySQL son ejemplos. |
| **Independencia de los datos** | Poder cambiar cómo se guardan los datos (nivel físico) o su estructura lógica sin tener que reescribir las aplicaciones que los usan. |
| **Integridad** | Que los datos sean correctos y coherentes con las reglas del negocio. Por ejemplo, que no exista una matrícula de un alumno que no existe. |
| **Transacción** | Conjunto de operaciones que se ejecutan como una unidad: o se hacen todas o ninguna. Cumple las propiedades ACID. |
| **Concurrencia** | Varios usuarios accediendo a los mismos datos a la vez. El SGBD debe evitar que uno estropee el trabajo del otro. |
| **DBA** | Administrador de la base de datos: instala, configura, gestiona usuarios y permisos, hace copias y vigila el rendimiento. |
| **Dato personal** | Cualquier información sobre una persona física identificada o identificable, como el nombre, el DNI, el correo o una foto. |

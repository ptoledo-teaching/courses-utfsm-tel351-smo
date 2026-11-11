# SMO - Seguridad y monitoreo de infraestructura

Este laboratorio utiliza Amazon CloudWatch y AWS CloudTrail para investigar el funcionamiento de una aplicación, reconocer operaciones rechazadas y configurar su monitoreo. Una plantilla de CloudFormation proporciona la infraestructura base. El trabajo consiste en investigar su funcionamiento, revisar el alcance de los permisos y dejar funcionando las consultas, métricas y alertas necesarias para seguir su operación.

El trabajo es individual y se realiza desde AWS Management Console, con la interfaz en inglés. La actividad dura 70 minutos, incluyendo una presentación inicial breve y la evaluación. La limpieza se realiza después del bloque.

## Resultados esperados

Al completar el laboratorio, será posible:

1. Investigar una operación fallida a partir de registros estructurados
2. Corregir una falla de acceso y ajustar los permisos de una función según el principio de mínimo privilegio
3. Construir una consulta de CloudWatch Logs Insights que seleccione registros relevantes
4. Transformar eventos de logs en una métrica y utilizarla para activar una alarma
5. Relacionar una operación, su registro, una métrica y una notificación
6. Reunir señales de funcionamiento en un dashboard
7. Identificar mediante CloudTrail una modificación realizada sobre la infraestructura

## Contexto

La HoloNet permite transmitir comunicaciones holográficas y datos entre estaciones distribuidas en distintos sistemas. Cada estación conserva reportes operacionales que describen el funcionamiento de sus enlaces. El equipo de operaciones utiliza un servicio de consulta para revisar esos reportes sin acceder directamente al almacenamiento.

Los reportes operacionales pueden ser consultados por este servicio. Otros reportes contienen antecedentes restringidos que deben permanecer fuera de su alcance. Ambos tipos de información están almacenados, pero el servicio solo debe tener autorización para leer los primeros. Rechazar el acceso a un reporte restringido es un resultado esperado, mientras que rechazar un reporte operacional impide cumplir la función del sistema.

El equipo de operaciones necesita comprobar que la implementación respete esas responsabilidades y disponer de información para investigar los accesos rechazados. Un rechazo aislado debe quedar registrado, pero no requiere una notificación inmediata. En cambio, tres o más rechazos durante un mismo minuto deben activar una alerta. Las solicitudes exitosas no deben contribuir a esa condición y la ausencia de tráfico no debe generar una alerta ni mantenerla activa indefinidamente.

El sistema utiliza una API HTTP de API Gateway, una Lambda function y un bucket privado de S3. La función intenta leer el reporte solicitado y registra el resultado como un mensaje JSON en CloudWatch Logs. El mensaje lo genera el código de la aplicación, utilizando la respuesta recibida desde S3. CloudWatch conserva el registro y permite consultarlo, pero no decide si la operación está autorizada.

## Interacción y evaluación

Las pruebas se realizan abriendo las URL publicadas en **Outputs** del stack. Cada nueva solicitud produce un registro. Para repetir una prueba, actualizar la pestaña del navegador y comprobar que la respuesta tenga un nuevo `requestId`.

La evaluación comprueba la infraestructura y los resultados de esta misma actividad. No requiere un informe, capturas ni una entrega separada de código. Puede utilizarse durante el bloque para comprobar avances parciales y repetirse después de corregir la configuración. Antes de evaluar cada avance, comprobarlo mediante las solicitudes, los registros y las métricas disponibles.

El sitio de evaluación [lab11.shareddomain.link](https://lab11.shareddomain.link) solicitará el **AWS account ID** y el **RUT normalizado**, sin puntos ni guion y con `k` minúscula. Utilizará el role `TEL351-Evaluator` de los laboratorios anteriores e identificará los recursos del stack `tel351-smo`. Mantener su trust policy y la managed policy `ReadOnlyAccess`, sin agregar permisos de escritura.

Los intentos quedarán registrados en un historial asociado al RUT. Las comprobaciones serán independientes, aunque algunas requieren que existan los recursos relacionados. Por ejemplo, la activación de una alarma necesita que previamente se hayan generado registros y publicado datos en su métrica.

| Aspecto | Puntaje |
| --- | ---: |
| Acceso operacional permitido y acceso restringido rechazado | 5 |
| Permisos del servicio acordes con el principio de mínimo privilegio | 10 |
| Registros estructurados de ambos resultados y retención del log group | 10 |
| Consulta guardada que selecciona los accesos rechazados | 15 |
| Filtro de métricas y publicación de rechazos | 20 |
| Configuración de la alarma y activación por datos reales | 20 |
| Topic SNS asociado y suscripción de correo confirmada | 10 |
| Dashboard con las métricas y la alarma requeridas | 10 |
| **Total** | **100** |

La alarma puede haber regresado a `OK` al evaluar. Su historial debe permitir comprobar que alcanzó `ALARM` como resultado de la métrica. La evaluación de SNS comprueba la configuración y la confirmación de la suscripción, no el contenido de la casilla de correo. La consulta de CloudTrail forma parte de la actividad, pero no tiene puntaje por navegación en la consola.

## Actividad

### 1. Desplegar la infraestructura base

1. Ingresar a AWS Management Console con su identidad administrativa y seleccionar **South America (São Paulo)**, `sa-east-1`
2. Descargar `infrastructure.yaml` desde este repositorio mediante **Download raw file**
3. Abrir **CloudFormation → Stacks → Create stack → With new resources (standard)**
4. Seleccionar **Choose an existing template → Upload a template file**, cargar `infrastructure.yaml` y continuar
5. Utilizar `tel351-smo` como **Stack name**
6. Continuar hasta la revisión, reconocer la creación de recursos IAM y seleccionar **Submit**
7. Esperar hasta que el stack alcance `CREATE_COMPLETE` y abrir **Outputs**

La plantilla crea la API, la función con su execution role, el log group y un bucket privado con los reportes de prueba. El código y los reportes están incluidos en la plantilla; una función auxiliar carga los archivos al desplegar el stack. No es necesario descargar paquetes adicionales, modificar código ni crear estos recursos por separado. Los recursos de monitoreo se configurarán durante la actividad.

Reconocer los siguientes outputs y mantener esta pestaña disponible:

| Output | Uso |
| --- | --- |
| `OperationalUrl` | Solicitar el reporte operacional |
| `RestrictedUrl` | Solicitar el reporte restringido |
| `ReaderFunctionName` | Identificar la función que procesa las solicitudes |
| `ReaderRoleName` | Identificar el execution role de esa función |
| `DataBucketName` | Identificar el bucket que conserva los reportes |
| `LogGroupName` | Identificar el grupo de logs de la aplicación |

Mantener `sa-east-1` para todos los recursos regionales. No modificar ni eliminar recursos de otros laboratorios.

### 2. Investigar el funcionamiento y los permisos

#### 2.1 Reconocer los registros

1. Abrir `OperationalUrl` y `RestrictedUrl` en pestañas diferentes
2. Determinar qué respuesta debería producir cada solicitud según el contexto, comparar con lo observado y conservar ambos `requestId`
3. Abrir **CloudWatch → Logs → Log groups** y seleccionar el grupo indicado en `LogGroupName`
4. Abrir el log stream con actividad reciente y localizar los registros de ambas solicitudes. Si existen varios streams, utilizar **Search all log streams**
5. Examinar `event`, `requestId`, `operation`, `objectKey`, `outcome` y, cuando esté presente, `awsErrorCode`. Identificar qué información permite explicar cada respuesta
6. Volver al log group y configurar **Retention setting** en **1 week (7 days)**

Los registros `storage_access` describen intentos de lectura. `ALLOWED` indica que se obtuvo el objeto y `DENIED`, que S3 rechazó el acceso. Otros registros de Lambda describen el inicio y término de una invocación, pero no reemplazan el resultado de la operación sobre S3.

#### 2.2 Diagnosticar las diferencias

1. Identificar las diferencias entre el comportamiento esperado y el observado. Distinguir un rechazo que cumple los requisitos de otro que impide el funcionamiento del servicio
2. Formular una explicación para cada diferencia e identificar qué dato de la configuración permitiría comprobarla. Utilizar los outputs para localizar los recursos relacionados
3. Contrastar esa explicación con los registros y la configuración antes de realizar cambios
4. Realizar un ajuste a la vez, repetir las solicitudes y comprobar si el resultado coincide con lo previsto. Si no coincide, revisar la explicación antes de continuar modificando recursos

#### 2.3 Revisar el alcance de los permisos

1. Abrir **IAM → Roles**, seleccionar el role indicado en `ReaderRoleName` y revisar sus policies en **Permissions**
2. Identificar las acciones y los recursos autorizados por el conjunto de policies asociadas al role
3. Contrastar esos permisos con las responsabilidades del servicio descritas en el contexto y determinar si cumplen el principio de mínimo privilegio
4. Realizar los ajustes que resulten necesarios a partir de ese análisis, conservando el funcionamiento del servicio y la generación de logs
5. Repetir ambas solicitudes y comprobar los resultados:
   - La solicitud operacional entrega el reporte y genera un registro `ALLOWED`
   - La solicitud restringida no entrega el reporte y genera un registro `DENIED`

El principio de mínimo privilegio consiste en conceder solo las acciones y los recursos necesarios para cumplir una función. Que las solicitudes de prueba produzcan los resultados esperados no demuestra por sí solo que los permisos sean adecuados. La evaluación comprobará tanto las respuestas como el alcance de las policies.

Mantener el bucket privado, sus controles de acceso público y las ACL deshabilitadas. No modificar el código, los archivos de prueba ni el role `TEL351-Evaluator`. No asociar policies administrativas ni conceder acceso a todos los buckets para resolver el problema. Los permisos necesarios para escribir logs deben conservarse.

Antes de continuar, comparar una solicitud rechazada con la métrica `Errors` de la función en **Lambda → Monitor → Metrics**, considerando el intervalo de la prueba y el tiempo de publicación. Determinar si esa métrica permite contar los rechazos observados. Si es necesario, consultar el código proporcionado para entender cómo termina la invocación, sin modificarlo.

### 3. Construir una consulta de Logs Insights

1. Abrir **CloudWatch → Logs Insights** y seleccionar el lenguaje **Logs Insights QL**
2. Seleccionar únicamente el log group del laboratorio y un intervalo que incluya las pruebas realizadas
3. Escribir una consulta que muestre `@timestamp`, `requestId`, `operation`, `objectKey` y `outcome`, conservando solamente los intentos de lectura rechazados por el almacenamiento. Utilizar los registros examinados para determinar las condiciones
4. Ordenar los resultados del más reciente al más antiguo y limitar la salida a 20 registros
5. Ejecutar la consulta mediante **Run query** y comprobar, usando los `requestId`, que incluye una solicitud rechazada y excluye una solicitud exitosa del mismo intervalo. No filtrar por esos identificadores en la consulta guardada: debe servir también para solicitudes posteriores
6. Guardar la consulta mediante **Save**, con el nombre `tel351-smo-denied`

[Logs Insights QL](https://docs.aws.amazon.com/AmazonCloudWatch/latest/logs/CWL_QuerySyntax.html) utiliza instrucciones separadas por `|`. La siguiente referencia permite construir la consulta:

| Instrucción | Uso |
| --- | --- |
| `fields campo1, campo2` | Seleccionar los campos visibles |
| `filter condición` | Conservar los registros que cumplen una condición; utilizar `=` para comparar valores y `and` para exigir varias condiciones |
| `sort campo desc` | Ordenar de mayor a menor según un campo |
| `limit cantidad` | Restringir la cantidad de resultados mostrados |

Escribir los valores de texto entre comillas. Acotar el intervalo y el log group reduce el volumen examinado. `limit` restringe los resultados mostrados, no el volumen de logs leído.

### 4. Crear una métrica de accesos rechazados

1. Regresar al log group y seleccionar **Actions → Create metric filter**
2. Escribir un **Filter pattern** que seleccione los mismos tipos de eventos que la consulta anterior, utilizando la sintaxis de los metric filters
3. En **Test pattern**, probar el patrón con tres registros: una lectura exitosa, una lectura rechazada y un registro de control que conserve `outcome: DENIED`, pero tenga un valor de `event` distinto. Preparar este último modificando una copia del JSON solo en el campo de prueba; no publicarlo en el log group. Pegar cada JSON completo en una sola línea
4. Comprobar que solo coincide la lectura rechazada. Si aparece otra coincidencia, revisar qué condición falta o está mal expresada
5. Configurar la métrica con los siguientes identificadores:

   | Campo | Valor |
   | --- | --- |
   | Filter name | `tel351-smo-denied` |
   | Metric namespace | `TEL351/SMO` |
   | Metric name | `DeniedAccess` |
   | Unit | `Count` |
   | Dimensions | No agregar dimensiones |

6. Determinar **Metric value** y **Default value** para que cada rechazo aporte una unidad y los registros que no coinciden no aumenten el contador
7. Crear el filtro y realizar una nueva solicitud de cada tipo
8. Abrir **CloudWatch → Metrics → All metrics**, localizar `DeniedAccess` en `TEL351/SMO` y contrastar sus datos con los registros generados después de crear el filtro. Si aún no aparece, continuar con la creación del topic SNS y volver a consultar

El filtro utiliza una [sintaxis distinta de Logs Insights QL](https://docs.aws.amazon.com/AmazonCloudWatch/latest/logs/FilterAndPatternSyntax.html). Encerrar la expresión entre `{` y `}`, referirse a cada campo JSON mediante `$.nombre` y utilizar `=` para comparar valores, comillas para los textos y `&&` para exigir varias condiciones. **Metric value** es el valor publicado por cada coincidencia; **Default value** se utiliza cuando llegan registros sin coincidencias.

El filtro procesa los registros recibidos después de su creación, no los anteriores. La aparición de la métrica puede tardar algunos minutos. El valor por omisión no garantiza datos durante períodos sin registros.

### 5. Configurar y comprobar una alerta

#### 5.1 Preparar la notificación

1. Abrir **SNS → Topics → Create topic**, seleccionar **Standard** y utilizar el nombre `tel351-smo-alerts`
2. Crear una suscripción con **Protocol: Email** y una dirección de correo accesible durante la actividad
3. Abrir el mensaje recibido y seleccionar **Confirm subscription**
4. Actualizar la lista de suscripciones y comprobar que ya no indica **Pending confirmation**

El correo de confirmación habilita la suscripción, pero no demuestra que la alarma haya enviado una notificación.

#### 5.2 Crear la alarma

1. Abrir **CloudWatch → Alarms → All alarms → Create alarm**
2. Seleccionar la métrica `DeniedAccess` del namespace `TEL351/SMO`, sin dimensiones
3. Configurar un umbral **Static** y un período de **1 minute**. Elegir **Statistic**, el operador de comparación, el umbral y **Datapoints to alarm** para que un único período con tres o más rechazos sea suficiente para activar la alarma, sin activarla con uno o dos
4. Elegir **Missing data treatment** de acuerdo con el comportamiento requerido cuando no existe tráfico
5. Asociar una notificación para **In alarm** al topic `tel351-smo-alerts` y mantener habilitadas las acciones
6. Utilizar `tel351-smo-denied` como nombre de la alarma y completar su creación

**Statistic** determina cómo se combinan las muestras del período. Comparar las opciones disponibles considerando qué valores publica el filtro y qué cantidad se necesita observar. **Datapoints to alarm** se refiere a períodos que incumplen el umbral, no a solicitudes individuales. **Missing data treatment** define cómo se evalúan los períodos sin muestras; no es lo mismo que recibir un valor cero.

#### 5.3 Comprobar la condición de alerta

1. Esperar que la alarma se encuentre en `OK` y planificar dos pruebas en períodos de un minuto distintos: una que no deba activarla y otra que alcance exactamente la cantidad de rechazos requerida. Incluir solicitudes exitosas para comprobar que no contribuyen al contador
2. Ejecutar la primera prueba y contrastar la cantidad de rechazos con los registros, el gráfico de `DeniedAccess` y el estado de la alarma
3. Ejecutar la segunda prueba y comprobar una transición a `ALARM` provocada por los datos, además de la recepción de la notificación en el correo confirmado
4. Dejar de generar tráfico y comprobar posteriormente el regreso a `OK`

Los períodos corresponden a intervalos de agregación de CloudWatch, no a un minuto contado desde que se abre la API. Iniciar cada prueba al comienzo de un minuto, completarla antes del siguiente y verificar en el gráfico a qué período pertenecen los datos. Si los rechazos quedaron distribuidos en períodos distintos, esa prueba no demuestra que se haya alcanzado el umbral.

La publicación de métricas y la evaluación de alarmas no son inmediatas. Preparar el dashboard y consultar CloudTrail mientras se actualizan, sin generar solicitudes adicionales durante las pruebas controladas. Si la alarma ya estaba en `ALARM`, una nueva solicitud no necesariamente produce otro correo: la notificación está asociada al cambio de estado. Para repetir la comprobación, esperar el regreso a `OK`.

No forzar manualmente el estado de la alarma ni publicar valores artificiales en la métrica. La comprobación debe relacionar una solicitud real con su registro, la métrica y la transición. Si falta un resultado, revisar el punto de ese recorrido en que dejó de aparecer información.

### 6. Revisar el funcionamiento y los cambios

#### 6.1 Preparar un dashboard

1. Abrir **CloudWatch → Dashboards → Create dashboard** y utilizar el nombre `tel351-smo`
2. Agregar un gráfico que permita conocer cuántas solicitudes procesó la función. Seleccionar la métrica correspondiente en `AWS/Lambda` y utilizar únicamente la función indicada en `ReaderFunctionName`
3. Agregar un segundo gráfico que permita conocer cuántos accesos al almacenamiento fueron rechazados
4. Elegir una estadística adecuada para mostrar cantidades totales por minuto y utilizar el mismo período de un minuto en ambos gráficos
5. Agregar un widget de estado para la alarma `tel351-smo-denied`
6. Guardar el dashboard y seleccionar un intervalo que incluya las pruebas

Utilizar las pruebas anteriores para determinar si los gráficos representan lo mismo. Ante una diferencia, relacionarla con los resultados de las solicitudes y comprobar si corresponde al comportamiento esperado o a un problema de configuración. Revisar también que los intervalos y la función seleccionada sean los correctos.

#### 6.2 Consultar un cambio en CloudTrail

1. Abrir **CloudTrail → Event history**, manteniendo `sa-east-1`
2. Seleccionar un intervalo que incluya el cambio de retención realizado en la sección 2
3. Utilizar **Lookup attributes** para localizar los eventos de la identidad utilizada o del servicio `logs.amazonaws.com`
4. Identificar el evento que registra la modificación de retención y comprobar que corresponde al log group del laboratorio, no a otro recurso
5. Examinar **Event record** e identificar la operación, la hora, la identidad y el valor de retención enviado. Contrastar ese valor con la configuración actual

Este evento registra una modificación de configuración, no la lectura del reporte en S3. Event history conserva eventos de administración y no requiere crear un trail para esta consulta. Los eventos pueden tardar en aparecer; ampliar el intervalo y volver a consultar si todavía no se encuentra el cambio.

### 7. Evaluar la infraestructura

1. Abrir [lab11.shareddomain.link](https://lab11.shareddomain.link) e ingresar el AWS account ID y su RUT normalizado
2. Solicitar la evaluación y revisar el resultado de cada comprobación
3. Para cada aspecto incompleto, localizar la evidencia correspondiente en la consola, determinar la causa y comprobar el ajuste antes de repetir la evaluación
4. Comprobar que el último resultado aparece en el historial antes de finalizar

Conservar los recursos y registros hasta que termine la evaluación. No es necesario mantener abierta la pestaña de la API ni generar solicitudes continuamente. Si una prueba aún no cuenta con datos, comprobar primero que se haya ejecutado después de crear el recurso que los utiliza.

Cada evaluación realiza una solicitud operacional y una restringida para comprobar las respuestas y sus registros. Completar las pruebas controladas de la alarma antes de solicitarla, ya que estas solicitudes también generan actividad en el servicio.

## Limpieza

Realizar esta sección después del laboratorio y de completar la evaluación. La limpieza es obligatoria y no forma parte de los 70 minutos.

1. Eliminar la alarma `tel351-smo-denied` en CloudWatch para detener sus acciones
2. Eliminar el dashboard `tel351-smo` y la consulta guardada `tel351-smo-denied`
3. Eliminar el metric filter `tel351-smo-denied` desde el log group
4. Eliminar la suscripción y el topic SNS `tel351-smo-alerts`
5. Abrir el bucket indicado en `DataBucketName` y vaciarlo. Mantener el bucket para que CloudFormation lo elimine junto con el stack
6. Eliminar el stack `tel351-smo` y comprobar que finalice su eliminación
7. Confirmar que no queden la API, la función, su execution role, el bucket ni el log group creados por el stack. Si la eliminación falla, revisar **Events**, resolver la causa y reintentar

Las métricas históricas de CloudWatch no se eliminan manualmente: dejan de recibir datos al retirar sus fuentes y permanecen según la retención del servicio. Tampoco es necesario eliminar los eventos de CloudTrail. Conservar la identidad administrativa y el role `TEL351-Evaluator` para las demás actividades de la asignatura.

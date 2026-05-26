# iFlow 1 — API Onboarding (Flujo Principal)
**SAP Cloud Integration | Agustina Mendoza**

---

## Propósito

Este iFlow es el **núcleo del proceso de onboarding**. Orquesta todas las integraciones necesarias para procesar un nuevo empleado: consulta su información en BambooHR, enriquece sus datos con el libro de bienvenida desde Bookshop, descarga su foto al SFTP corporativo y finalmente crea el registro en SAP S/4HANA mediante un IDoc.

Está expuesto como API a través de **SAP API Management** y protegido con **API Key**, lo que permite que cualquier sistema autorizado lo invoque, no solo el iFlow 2.

---

## Diagrama Principal

![Flow Principal](https://github.com/CodeByAgus/sap-btp-integration-suite-bootcamp/blob/main/docs/Flow2_Principal.jpeg?raw=true)

---

## Requerimientos que cubre

- Recibir la solicitud de onboarding vía HTTP con autenticación por API Key.
- Consultar los datos del empleado desde BambooHR (nombre, foto, ID de libro, campo aleatorio solicitado).
- Consultar el nombre del libro de bienvenida desde Bookshop OData usando el ID almacenado en BambooHR (`address1`).
- Actualizar el nombre del libro obtenido en el campo `address2` del empleado en BambooHR.
- Descargar la foto del empleado y subirla al servidor SFTP corporativo.
- Crear el empleado en SAP S/4HANA mediante un IDoc.
- Publicar la respuesta final en el tópico AMQP de salida.
- Controlar cada excepción posible en cada subproceso de forma independiente.

---

## Descripción del Flujo

```
[HTTPS — API Management]
         │
    [Start] → [Extraer ID y CampoAleatorio]
         │
    [Consulta BambooHR] ──► OC-BambooHR
         │
    [Convertir JSON → XML]
         │
    [Extraer propiedades del empleado]
         │
    [Multicast] ──────────────────────────────────────┐
         │                                             │
    [Subproceso: Photo]                     [Subproceso: Book]
    Descarga foto → SFTP                    Consulta Bookshop OData
         │                                  → Actualiza BambooHR
         └──────────────────┬───────────────┘
                            │
                   [Subproceso: IDoc Sender]
                   Construye IDoc → SAP S/4HANA
                            │
                   [Construir respuesta JSON]
                            │
                   [Publicar en tópico AMQP]
                            │
                          [End]

Exception Subprocess global → mensaje de error estructurado
```

---

## Subproceso: Consulta BambooHR

![Flow BambooHR](https://github.com/CodeByAgus/sap-btp-integration-suite-bootcamp/blob/main/docs/Flow2_Principal.jpeg?raw=true)

**Requerimiento:** obtener los datos del empleado desde BambooHR usando su ID.

El iFlow realiza un GET a `/employees/{id}` mediante el adaptador **Open Connectors**, que abstrae la autenticación y el protocolo hacia BambooHR. La respuesta JSON se convierte a XML para facilitar la extracción de propiedades y el mapeo posterior en los subprocesos.

De la respuesta se extraen y almacenan como propiedades del mensaje los siguientes campos:

| Propiedad interna | Campo BambooHR | Uso posterior |
|---|---|---|
| ID del empleado | `id` | IDoc, respuesta final |
| Nombre | `firstName` / `lastName` | IDoc |
| URL de foto | `photoUrl` | Subproceso Photo |
| ID del libro | `address1` | Subproceso Book |
| Campo aleatorio | dinámico (viene en el request) | Respuesta final |

---

## Subproceso: Photo

![Photo Connection](https://github.com/CodeByAgus/sap-btp-integration-suite-bootcamp/blob/main/docs/Photo_Connection.png?raw=true)

**Requerimiento:** descargar la foto del empleado y almacenarla en el SFTP corporativo.

Un Router evalúa si existe una URL de foto válida en los datos del empleado:

- **Si hay foto** → se descarga desde BambooHR vía HTTP y se deposita en el SFTP corporativo en la ruta `/upload/[participante]/[archivo]`.
- **Si no hay foto** → el flujo continúa sin error (regla de negocio explícita).

El subproceso tiene su propio Exception Subprocess para aislar fallos de conectividad con el SFTP o con la URL de la foto.

---

## Subproceso: Book

![Book Connection](https://github.com/CodeByAgus/sap-btp-integration-suite-bootcamp/blob/main/docs/Book_conecction.png?raw=true)

**Requerimiento:** consultar el nombre del libro de bienvenida y actualizarlo en BambooHR.

El ID del libro favorito del empleado está almacenado de forma no convencional en el campo `address1` de BambooHR. Con ese ID se realiza una consulta OData V2 a la **Bookshop App** (accesible a través del mismo adaptador Open Connectors).

Un Router evalúa el resultado:

- **Si se encontró el libro** → se actualiza el campo `address2` del empleado en BambooHR con el nombre obtenido, usando el adaptador Open Connectors (operación PUT).
- **Si no se encontró el libro** → el campo de respuesta toma el valor `"Libro no encontrado"` y el flujo continúa.

El subproceso tiene su propio Exception Subprocess para aislar errores de la consulta OData o del PUT a BambooHR.

---

## Subproceso: IDoc Sender

![IDoc Connection](https://github.com/CodeByAgus/sap-btp-integration-suite-bootcamp/blob/main/docs/Idoc_conecction.png?raw=true)

**Requerimiento:** crear el empleado en SAP S/4HANA mediante un IDoc de tipo `DEBMAS06`.

Un script Groovy construye el XML del IDoc dinámicamente con los datos del empleado (nombre, apellido, ID) extraídos de las propiedades del mensaje. El número de documento (`DOCNUM`) se genera de forma secuencial con un correlativo interno.

El IDoc resultante se envía vía HTTP al endpoint de SAP S/4HANA. El subproceso tiene su propio Exception Subprocess para capturar errores de comunicación con SAP.

**Estructura del IDoc generado:**

```xml
<DEBMAS06>
  <IDOC BEGIN="1">
    <EDI_DC40 SEGMENT="1">
      <MESTYP>DEBMAS</MESTYP>
      <IDOCTYP>DEBMAS06</IDOCTYP>
      <SNDPOR>PORT_CPI</SNDPOR>
      <SNDPRN>CPI_DEMO</SNDPRN>
      <DOCNUM>11000000000{correlativo}</DOCNUM>
      <RCVPRN>EC5CLNT400</RCVPRN>
    </EDI_DC40>
    <E1KNA1M SEGMENT="1">
      <MSGFN>009</MSGFN>
      <NAME1>Agustina Mendoza, {nombre empleado}, {id empleado}</NAME1>
      <ORT01>BUENOS AIRES</ORT01>
      <LAND1>AR</LAND1>
      <SPRAS>S</SPRAS>
    </E1KNA1M>
  </IDOC>
</DEBMAS06>
```

---

## Trama de Respuesta Final (AMQP)

Al finalizar todos los subprocesos, se construye y publica el siguiente JSON en el tópico `alkemy/2026/final_test/output`:

```json
{
  "participante": "Agustina Mendoza",
  "id_empleado": "<ID del empleado procesado>",
  "CampoAleatorio": "<nombre del campo solicitado en el request>",
  "ValorCampoAleatorio": "<valor de ese campo obtenido desde BambooHR>",
  "urlFotoEmpleado": "<URL de la foto devuelta por BambooHR>"
}
```

---

## Manejo de Errores

Cada subproceso tiene su propio **Exception Subprocess** independiente, lo que permite aislar fallos sin afectar los demás subprocesos del Multicast. Adicionalmente, el Integration Process principal cuenta con un Exception Subprocess global que captura cualquier error no manejado y devuelve un JSON estructurado con el código de error y el mensaje descriptivo.

---

## Buenas Prácticas Aplicadas

- **Parámetros externalizados**: todos los hosts, rutas y nombres de credenciales se configuran como parámetros externos sin valores hardcodeados.
- **Secure parameters / Credential Store**: las credenciales (API Keys, usuario SFTP, contraseña SAP) se almacenan en el Credential Store de CPI, nunca en el iFlow.
- **Exception Subprocesses por subproceso**: aislamiento de errores en cada integración externa.
- **Multicast con subprocesos locales**: Photo y Book se ejecutan en paralelo, reduciendo la latencia total del proceso.
- **Schemas XSD**: la respuesta de BambooHR se valida contra schemas XSD definidos en el proyecto para garantizar la integridad de los datos antes del mapeo.
- **Script Groovy modular**: la construcción del IDoc está encapsulada en un script independiente, facilitando su mantenimiento y prueba aislada.
- **Nombres descriptivos**: todos los artefactos, pasos y subprocesos usan nombres que reflejan su función en el proceso de negocio.

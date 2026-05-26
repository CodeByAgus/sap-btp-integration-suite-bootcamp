# iFlow 2 — Consumidor AMQP
**SAP Cloud Integration | Agustina Mendoza**

---

## Propósito

Este iFlow actúa como **punto de entrada al proceso de onboarding**. Su responsabilidad es escuchar la cola de mensajes AMQP, recibir la solicitud del sistema legacy y delegar la orquestación completa al iFlow principal (API Onboarding) mediante una llamada HTTP autenticada con API Key.

La separación en dos iFlows responde a una decisión de diseño: desacoplar la capa de transporte (AMQP) de la lógica de negocio, facilitando la reutilización y el mantenimiento independiente de cada componente.

---

## Diagrama

![Flow Entrega Final 1](https://github.com/CodeByAgus/sap-btp-integration-suite-bootcamp/blob/bbb128a6dee3a9cddee05a6b43402281aec2b8fa/Src/Flow2%20AMQP.jpeg)

---

## Requerimientos que cubre

- Consumir mensajes desde el tópico `alkemy/2026/final_test/input` a través de una cola personal suscrita a ese tópico.
- Transformar o adaptar el mensaje de ser necesario antes de invocar al iFlow 1.
- Invocar al iFlow de API Onboarding con autenticación por API Key.
- Controlar errores de conectividad o procesamiento mediante manejo de excepciones.

---

## Descripción del Flujo

```
[Cola AMQP — Tópico: alkemy/2026/final_test/input]
         │
    [Start] ──► [ApiKey - Content Modifier] ──► [End] ──HTTP──► iFlow 1
                                                    │
                                       Exception Subprocess
                                    [Error] → notificación de fallo
```

### Pasos principales

**Entrada AMQP** — El adaptador AMQP escucha la cola personal del participante, suscrita al tópico de entrada del curso. Cuando llega un mensaje, el iFlow se activa.

**Preparación del mensaje** — Un Content Modifier agrega el header de autenticación (API Key) requerido por el iFlow 1, externalizado como parámetro seguro en CPI.

**Llamada al iFlow 1** — El mensaje se reenvía vía HTTP al endpoint expuesto por el iFlow de API Onboarding a través de SAP API Management.

**Manejo de errores** — Un Exception Subprocess captura cualquier falla (error de conexión, timeout, respuesta inválida) y genera un mensaje de error estructurado para trazabilidad.

---

## Trama de Entrada (XML)

El mensaje que llega desde el sistema legacy tiene el siguiente formato:

```xml
<?xml version="1.0" encoding="utf-8"?>
<Empresas xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance">
  <Empresa>
    <nombre_empresa>Empresa</nombre_empresa>
    <Empleados>
      <Empleado>ID del Empleado</Empleado>
      <CampoAleatorio>campo aleatorio</CampoAleatorio>
    </Empleados>
  </Empresa>
</Empresas>
```

- `Empleado` → ID del empleado a procesar en BambooHR.
- `CampoAleatorio` → nombre del campo cuyo valor se debe extraer desde BambooHR e incluir en la respuesta final.

---

## Buenas Prácticas Aplicadas

- **Parámetros externalizados**: la API Key y el endpoint del iFlow 1 se configuran como parámetros externos, sin valores hardcodeados en el flujo.
- **Secure parameters**: las credenciales se gestionan mediante el Credential Store de SAP CPI.
- **Exception Subprocess**: manejo explícito de errores con mensaje estructurado.
- **Separación de responsabilidades**: este iFlow solo gestiona la entrada y el enrutamiento; toda la lógica de negocio reside en el iFlow 1.

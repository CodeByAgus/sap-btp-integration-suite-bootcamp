# SAP Integration Suite — Automatización de Onboarding de Empleados
### Accenture by Alkemy — Evaluación Final

---

## Caso de Negocio

**ABC Consulting INC** está modernizando su proceso de onboarding de empleados. Actualmente toda la creación y modificación se realiza de forma manual, lo que genera ineficiencias y errores. El objetivo es automatizar completamente este proceso utilizando **SAP Integration Suite (CPI)** como plataforma de integración central.

Cuando un nuevo empleado ingresa a la empresa:

- Su información base proviene de un sistema legacy que envía datos por mensajería.
- Esa información se enriquece desde **BambooHR**, el sistema de RRHH.
- Como parte del onboarding, cada empleado recibe un **libro de bienvenida**, cuyo ID está almacenado en BambooHR; el nombre del libro se consulta desde la aplicación interna **Bookshop**.
- La **foto del empleado** se almacena en el servidor SFTP corporativo.
- El empleado queda registrado en **SAP S/4HANA** mediante un IDoc.
- El resultado del proceso se publica en una cola de mensajes para consumo asíncrono.

---

## Solución Implementada

La solución consta de **dos iFlows** en SAP Integration Suite que trabajan en conjunto:

```
[Cola AMQP — Sistema Legacy]
         │
         ▼
  ┌──────────────────┐
  │    iFlow 2       │  ← Consume la cola, transforma y delega
  │  AMQP Consumer   │
  └────────┬─────────┘
           │  (HTTP con API Key)
           ▼
  ┌──────────────────────────────────┐
  │           iFlow 1                │
  │       API Onboarding             │
  │                                  │
  │  BambooHR → Bookshop OData       │
  │  → SFTP (foto del empleado)      │
  │  → SAP S/4HANA (IDoc DEBMAS06)   │
  │  → Tópico AMQP (respuesta final) │
  └──────────────────────────────────┘
```

---

## Flujo Funcional

1. El sistema legacy publica un mensaje XML en el tópico AMQP de entrada.
2. El **iFlow 2** consume el mensaje desde su cola personal suscrita al tópico.
3. Se invoca al **iFlow 1** (API Onboarding) con autenticación por API Key.
4. Se consulta la información del empleado en **BambooHR** usando su ID.
5. Se descarga la foto del empleado (URL provista por BambooHR).
6. Se consulta el **nombre del libro de bienvenida** desde Bookshop OData, usando el ID almacenado en BambooHR.
7. Se actualiza el nombre del libro obtenido en el registro del empleado en BambooHR.
8. La foto se sube al **servidor SFTP corporativo**.
9. Se crea el empleado en **SAP S/4HANA** mediante un IDoc `DEBMAS06`.
10. Se publica una trama JSON de respuesta en el **tópico AMQP de salida**.

---

## Reglas de Negocio

- Si no se encuentra el libro → el campo de respuesta indica `"Libro no encontrado"` y el flujo continúa.
- Si no hay foto disponible → el flujo no se interrumpe.
- La trama de entrada incluye el ID del empleado y un campo aleatorio a extraer desde BambooHR.
- Todos los errores son capturados y controlados mediante Exception Subprocesses dedicados.

---

## Estructura del Repositorio

```
/
├── README.md                        ← Este archivo (visión general)
├── Entrega_Final1/                  ← iFlow 2: Consumidor AMQP
│   ├── README_EntregaFinal1.md      ← Documentación detallada
│   └── src/...
├── Entrega_Final2/                  ← iFlow 1: API Onboarding (flujo principal)
│   ├── README_EntregaFinal2.md      ← Documentación detallada
│   └── src/...
└── docs/
    ├── Flow1.jpeg
    ├── Flow2_Principal.jpeg
    ├── Book_conecction.png
    ├── Photo_Connection.png
    └── Idoc_conecction.png
```

---

## Sistemas Integrados

| Sistema | Adaptador | Rol |
|---|---|---|
| Sistema Legacy | AMQP | Origen del mensaje de entrada |
| BambooHR | Open Connectors | Fuente de datos del empleado |
| Bookshop App | OData V2 (Cloud Connector) | Nombre del libro de bienvenida |
| Servidor SFTP | SFTP | Almacenamiento de la foto |
| SAP S/4HANA | HTTP / IDoc | Creación del empleado en SAP |
| Solace | AMQP | Publicación de la respuesta final |

---

## Documentación Detallada

- [iFlow 2 — Consumidor AMQP](./Entrega_Final1/README_EntregaFinal1.md)
- [iFlow 1 — API Onboarding (flujo principal)](./Entrega_Final2/README_EntregaFinal2.md)

---

## Tecnologías

SAP Cloud Integration (CPI) · SAP API Management · SAP Open Connectors · SAP Cloud Connector · Groovy · IDoc DEBMAS06 · OData V2 · AMQP · SFTP · HTTP/S

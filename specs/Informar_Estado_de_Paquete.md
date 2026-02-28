# Feature Specification: Informar estado de paquete

**Created:** 2026-02-27

---

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Sincronización de Novedades para Liquidación Financiera (Priority: P1)

Como **Módulo de Gestión de Paquetes**, quiero centralizar el envío de estados de novedad, evidencias y valores comerciales al Módulo de Finanzas para automatizar el cobro de seguros y el ajuste de pagos por logística inversa.

**Why this priority:**
Es el trigger contable para las excepciones operativas. Sin este informe, el Módulo 3 no podría procesar penalidades por daños o extravíos detectados en bodega, ni liquidar correctamente los retornos (devoluciones), afectando la precisión financiera del sistema.

**Independent Test:**
Registrar una novedad de `Dañado` o `Extraviado` en el Módulo 1 y verificar mediante logs que el Módulo de Finanzas reciba el JSON con el UUID, el tipo de falla y el Valor Declarado correspondiente.

---

## Acceptance Scenarios

### Scenario: Notificación de incidencia económica (Flujo desde Novedad)
* **Given** un paquete que acaba de pasar por el proceso de `Gestionar novedad de paquete` con estado `Dañado` o `Extraviado`.
* **When** el caso de uso es invocado obligatoriamente por la gestión de la novedad en bodega.
* **Then** el sistema envía al Módulo 3 el estado de falla, el **Valor Declarado** y el enlace a la evidencia para aplicar los descuentos o cobros de póliza correspondientes.

### Scenario: Notificación de retorno por devolución
* **Given** un paquete registrado en el sistema con novedad de `Devolución` tras su re-ingreso a bodega.
* **When** el sistema procesa el cambio de estado administrativo.
* **Then** el sistema informa al Módulo 3 para que liquide el porcentaje de pago por logística inversa según la matriz de costos de retorno definida.

---

## Edge Cases

* **Timeout del Módulo de Finanzas:** Si el Módulo 3 no responde, el sistema encola el mensaje en estado `Pendiente` y realiza reintentos automáticos cada 5 minutos hasta confirmar la recepción (ACK).
* **Valor Declarado ausente:** Si el paquete no tiene Valor Declarado registrado en la admisión, el sistema genera una excepción interna y notifica al administrador antes de permitir el cierre financiero del UUID.

---

## Requirements *(mandatory)*

### Functional Requirements
* **FR-001:** El sistema DEBE construir un paquete de datos (Payload) que incluya: UUID, Estado Final de Novedad y Valor Declarado.
* **FR-002:** El sistema DEBE adjuntar la referencia o URL de la evidencia fotográfica capturada en bodega cuando el estado sea `Dañado`.
---

## Key Entities
* **UUID:** Identificador único que garantiza que Finanzas liquide el paquete correcto.
* **Valor Declarado:** Monto económico capturado en la admisión que define el límite de responsabilidad del seguro.
* **Estado Final de Novedad:** Atributo (`Dañado`, `Extraviado`, `Devolución`) que determina la penalidad o el costo de retorno.

---

## Success Criteria *(mandatory)*

### Measurable Outcomes
* **SC-001:** El 100% de las novedades registradas en el Módulo 1 deben generar un registro de comunicación exitosa con el Módulo 3.
* **SC-002:** El tiempo de latencia entre el registro de la novedad y la recepción del dato en el Módulo de Finanzas no debe exceder los 10 segundos en condiciones normales.

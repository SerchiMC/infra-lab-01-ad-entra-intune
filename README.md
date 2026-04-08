# Lab NEXUS — Entorno híbrido (AD + Entra ID + Intune + Conditional Access)

## 1. Introducción

En este laboratorio se ha implementado un entorno híbrido basado en Active Directory on-premises y Microsoft Entra ID, enfocado a la gestión de usuarios y dispositivos en un entorno corporativo.

El objetivo es simular un escenario real de empresa, no solo desplegando la infraestructura, sino también trabajando su uso posterior: sincronización de identidades, gestión mediante Intune y control de acceso a través de Conditional Access.

---

## 2. Arquitectura del entorno

La arquitectura del entorno consta de los siguientes elementos:

- Controlador de dominio (Windows Server 2022): NEX-DC01  
- Equipo cliente (Windows 10/11): NEX-CL01  
- Dominio on-prem: NEXUS.local  
- Tenant de Microsoft: sergiotech26.onmicrosoft.com  
- Servicios utilizados: Active Directory, DNS, DHCP, Entra ID, Azure AD Connect, Intune y Conditional Access  

---

## 3. Configuración on-prem (Active Directory)

Se ha creado el dominio en el controlador de dominio, instalando los servicios necesarios como DNS y DHCP.

![01-nexus-domain-created](01-nexus-domain-created)

Se ha definido la estructura de unidades organizativas (OU), creando una OU principal de usuarios y otras específicas para los distintos departamentos.

![02-nexus-ad-ou-structure](02-nexus-ad-ou-structure)

Se ha configurado un sufijo UPN personalizado para su uso en el entorno híbrido.

![03-nexus-upn-suffix-added](03-nexus-upn-suffix-added)

Se han creado los grupos necesarios para la gestión de usuarios.

![04-nexus-ad-group-membership](04-nexus-ad-group-membership)

---

## 4. Servicios base y GPO

Se ha configurado DHCP con un rango de direcciones IP de 192.168.0.10 a 192.168.0.50, con una concesión de 8 días.

Se une el equipo cliente al dominio y se valida la IP asignada:

![05-nexus-dhcp-client-ip](05-nexus-dhcp-client-ip)

Validación de unión al dominio:

![06-nexus-client-domain-joined](06-nexus-client-domain-joined)  
![07-nexus-domain-user-login](07-nexus-domain-user-login)

Se crea una GPO para mapear la unidad de red V: (Ventas):

![08-gpo-drive-mapping-ventas](08-gpo-drive-mapping-ventas)  
![09-client-drive-mapped](09-client-drive-mapped)

Se despliega una impresora mediante GPO:

![10-gpo-printer-deployment](10-gpo-printer-deployment)  
![11-client-printer-deployed](11-client-printer-deployed)

---

## 5. Integración híbrida (Entra ID)

Se instala Azure AD Connect en el controlador de dominio y se configura la sincronización de identidades.

Validación de usuarios sincronizados:

![12-entra-synced-users](12-entra-synced-users)

Se valida el estado híbrido del equipo:

![13-hybrid-join-status-nex-cl01](13-hybrid-join-status-nex-cl01)

---

## 6. Gestión de dispositivos (Intune)

Debido a problemas en el enrollment automático, se implementa una GPO para forzar la inscripción en Intune.

![14-gpo-intune-created](14-gpo-intune-created)  
![15-gpo-intune-autoenrollment-config](15-gpo-intune-autoenrollment-config)

Validación en Entra ID:

![16-intune-device-enrolled-nex-cl01](16-intune-device-enrolled-nex-cl01)

Validación en Intune:

![17-intune-device-visible](17-intune-device-visible)

---

## 7. Gestión de identidades

Se crea el usuario cgarcia y el grupo GG-Excepciones, utilizado para gestionar exclusiones en políticas de acceso.

![18-ad-user-cgarcia-created](18-ad-user-cgarcia-created)  
![19-ad-group-excepciones-members](19-ad-group-excepciones-members)

Se valida la sincronización con Entra ID:

![20-entra-user-cgarcia-synced](20-entra-user-cgarcia-synced)

---

## 8. Control de acceso y cumplimiento (Conditional Access + Intune)

Se crea una política de cumplimiento basada en el estado del firewall.

![21-intune-compliance-policy-created](21-intune-compliance-policy-created)

Asignación al grupo:

![22-intune-compliance-assigned-ventas](22-intune-compliance-assigned-ventas)

Configuración de Conditional Access:

![23-entra-ca-exclude-groups](23-entra-ca-exclude-groups)  
![24-entra-ca-include-groups](24-entra-ca-include-groups)

Prueba de incumplimiento:

![25-client-firewall-disabled](25-client-firewall-disabled)  
![26-intune-device-noncompliant-firewall](26-intune-device-noncompliant-firewall)

Bloqueo de acceso:

![27-entra-ca-blocked-ventas-user](27-entra-ca-blocked-ventas-user)

Excepción aplicada:

![28-entra-ca-exclusion-working](28-entra-ca-exclusion-working)

---

## 9. Incidencias relevantes y resolución

Durante la implementación del entorno híbrido se identificaron incidencias reales relacionadas con conectividad, sincronización y gestión de dispositivos.

### 1. Conectividad parcial en el servidor (DNS)

**Problema**  
El servidor tenía conectividad (ping) pero no podía acceder a servicios externos.

**Causa**  
Falta de configuración de reenviadores DNS.

**Solución**  
Configuración de forwarders externos:
- 8.8.8.8  
- 1.1.1.1  

**Conclusión**  
La conectividad IP no garantiza resolución DNS funcional.

---

### 2. Hybrid Join sin inscripción en Intune

**Problema**  
El dispositivo estaba correctamente unido:
- AzureAdJoined: YES  
- DomainJoined: YES  
- MDM: None  

**Causa**  
El proceso de auto enrollment no se ejecutó automáticamente.

**Solución**  
Aplicación de GPO de inscripción MDM:
- Configuración del equipo → Plantillas administrativas → Componentes de Windows → MDM  
- Inscripción automática con credenciales de Entra  
- Tipo de credencial: Usuario  

**Resultado**  
El dispositivo se registró correctamente en Intune.

---

### 3. Estado inconsistente de enrolamiento

**Problema**  
El dispositivo desapareció de Entra tras intentar el enrollment.

**Causa**  
Registro en estado inconsistente.

**Solución**
dsregcmd /leave  
- Reinicio del equipo  
- Inicio de sesión con usuario de dominio  
- Reaplicación de GPO  

**Resultado**  
Estado final correcto:
- AzureAdJoined: YES  
- DomainJoined: YES  
- MDM: Microsoft Intune  

---

### 4. Error al asignar licencias (Usage Location)

**Problema**  
No era posible asignar licencias a usuarios sincronizados.

**Causa**  
El usuario no tenía definido el atributo país/región requerido.

**Solución**  
Durante la asignación de licencia se definió el país del usuario.

**Resultado**  
Asignación de licencias completada correctamente.

---

### 5. Conditional Access no aplicado

**Problema**  
El acceso no era bloqueado pese a incumplir la política.

**Causa**  
La política no estaba aplicada a todos los recursos.

**Solución**  
Configuración de:
- Recursos → All cloud apps  

**Resultado**  
El acceso se bloquea correctamente en dispositivos no compliant.

---

## 10. Conclusiones

Este laboratorio ha permitido implementar un entorno híbrido completo combinando Active Directory on-premises con Microsoft Entra ID e Intune, reproduciendo un escenario cercano a un entorno empresarial real.

A nivel técnico, se han trabajado conceptos clave como:
- Gestión de identidades en Active Directory  
- Sincronización de usuarios mediante Entra Connect  
- Unión híbrida de dispositivos (Hybrid Join)  
- Inscripción y gestión de dispositivos en Intune  
- Aplicación de políticas de cumplimiento (compliance)  
- Implementación de Acceso Condicional basado en estado del dispositivo  

Además, el laboratorio ha estado marcado por la resolución de incidencias reales relacionadas con sincronización, enrolamiento de dispositivos, asignación de licencias y comportamiento de políticas de seguridad.

Como resultado final, se ha conseguido:
- Un entorno funcional con identidad híbrida  
- Dispositivos gestionados mediante Intune  
- Políticas de seguridad aplicadas correctamente  
- Control de acceso basado en cumplimiento del dispositivo  

Este laboratorio demuestra no solo la configuración del entorno, sino también la capacidad de diagnóstico y resolución de problemas en escenarios reales.

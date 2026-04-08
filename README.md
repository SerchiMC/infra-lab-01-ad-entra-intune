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

![01-nexus-domain-created](capturas/01-nexus-domain-created.png)

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

# Lab NEXUS — Hybrid Environment (AD + Entra ID + Intune + Conditional Access)

## 1. Introduction

This lab implements a hybrid environment based on on-premises Active Directory and Microsoft Entra ID, focused on user and device management in a corporate scenario.

The objective is to simulate a real enterprise environment, not only by deploying the infrastructure, but also by working on its practical use: identity synchronization, device management with Intune, and access control through Conditional Access.

---

## 2. Environment Architecture

The environment consists of the following components:

- Domain Controller (Windows Server 2022): NEX-DC01  
- Client machine (Windows 10/11): NEX-CL01  
- On-prem domain: NEXUS.local  
- Microsoft tenant: sergiotech26.onmicrosoft.com  
- Services used: Active Directory, DNS, DHCP, Entra ID, Azure AD Connect, Intune and Conditional Access  

---

## 3. On-prem Configuration (Active Directory)

The domain was created on the domain controller, installing required services such as DNS and DHCP.

![01-nexus-domain-created](capturas/01-nexus-domain-created.png)

An Organizational Unit (OU) structure was defined, including a main users OU and additional OUs for different departments.

![02-nexus-ad-ou-structure](capturas/02-nexus-ad-ou-structure.png)

A custom UPN suffix was configured to support hybrid identity.

![03-nexus-upn-suffix-added](capturas/03-nexus-upn-suffix-added.png)

Required groups were created for user management and later use in policy assignment.

![04-nexus-ad-group-membership](capturas/04-nexus-ad-group-membership.png)

---

## 4. Core Services and GPO

DHCP was configured with an IP range from 192.168.0.10 to 192.168.0.50, with an 8-day lease.

The client machine was joined to the domain and validated via IP assignment:

![05-nexus-dhcp-client-ip](capturas/05-nexus-dhcp-client-ip.png)

Domain join validation:

![06-nexus-client-domain-joined](capturas/06-nexus-client-domain-joined.png)  
![07-nexus-domain-user-login](capturas/07-nexus-domain-user-login.png)

A GPO was created to map a network drive (V: - Ventas):

![08-gpo-drive-mapping-ventas](capturas/08-gpo-drive-mapping-ventas.png)  
![09-client-drive-mapped](capturas/09-client-drive-mapped.png)

A shared printer was deployed using GPO:

![10-gpo-printer-deployment](capturas/10-gpo-printer-deployment.png)  
![11-client-printer-deployed](capturas/11-client-printer-deployed.png)

---

## 5. Hybrid Integration (Entra ID)

Azure AD Connect was installed on the domain controller and configured to synchronize identities.

Validation of synced users:

![12-entra-synced-users](capturas/12-entra-synced-users.png)

Hybrid join status was validated on the client machine:

![13-hybrid-join-status-nex-cl01](capturas/13-hybrid-join-status-nex-cl01.png)

---

## 6. Device Management (Intune)

Due to issues with automatic enrollment, a GPO was implemented to force device registration into Intune.

![14-gpo-intune-created](capturas/14-gpo-intune-created.png)  
![15-gpo-intune-autoenrollment-config](capturas/15-gpo-intune-autoenrollment-config.png)

Validation in Entra ID:

![16-intune-device-enrolled-nex-cl01](capturas/16-intune-device-enrolled-nex-cl01.png)

Validation in Intune:

![17-intune-device-visible](capturas/17-intune-device-visible.png)

---

## 7. Identity Management

A new user (cgarcia) and a group (GG-Excepciones) were created for use in Conditional Access scenarios.

![18-ad-user-cgarcia-created](capturas/18-ad-user-cgarcia-created.png)  
![19-ad-group-excepciones-members](capturas/19-ad-group-excepciones-members.png)

User synchronization was validated in Entra ID:

![20-entra-user-cgarcia-synced](capturas/20-entra-user-cgarcia-synced.png)

---

## 8. Access Control and Compliance (Conditional Access + Intune)

A compliance policy was created in Intune, requiring the Windows Firewall to be enabled.

![21-intune-compliance-policy-created](capturas/21-intune-compliance-policy-created.png)

Assignment to group:

![22-intune-compliance-assigned-ventas](capturas/22-intune-compliance-assigned-ventas.png)

Conditional Access configuration:

![23-entra-ca-exclude-groups](capturas/23-entra-ca-exclude-groups.png)  
![24-entra-ca-include-groups](capturas/24-entra-ca-include-groups.png)

Non-compliance test:

![25-client-firewall-disabled](capturas/25-client-firewall-disabled.png)  
![26-intune-device-noncompliant-firewall](capturas/26-intune-device-noncompliant-firewall.png)

Access blocked:

![27-entra-ca-blocked-ventas-user](capturas/27-entra-ca-blocked-ventas-user.png)

Exception applied:

![28-entra-ca-exclusion-working](capturas/28-entra-ca-exclusion-working.png)

---

## 9. Relevant Issues and Resolution

During the implementation of the hybrid environment, several real-world issues were identified related to connectivity, synchronization and device management.

### 1. Partial connectivity on the server (DNS)

**Problem**  
The server had network connectivity (ping) but could not access external services.

**Cause**  
Missing DNS forwarders.

**Solution**  
Configured external forwarders:
- 8.8.8.8  
- 1.1.1.1  

**Conclusion**  
IP connectivity does not guarantee proper DNS resolution.

---

### 2. Hybrid Join without Intune enrollment

**Problem**  
The device was correctly joined:
- AzureAdJoined: YES  
- DomainJoined: YES  
- MDM: None  

**Cause**  
Auto enrollment process did not trigger automatically.

**Solution**  
Applied GPO for MDM enrollment:
- Computer Configuration → Administrative Templates → Windows Components → MDM  
- Enable automatic enrollment using Entra credentials  
- Credential type: User  

**Result**  
Device successfully enrolled in Intune.

---

### 3. Inconsistent enrollment state

**Problem**  
The device disappeared from Entra after enrollment attempt.

**Cause**  
Device registration entered an inconsistent state.

**Solution**
dsregcmd /leave  
- Reboot  
- Domain user login  
- Reapply GPO  

**Result**  
Final state:
- AzureAdJoined: YES  
- DomainJoined: YES  
- MDM: Microsoft Intune  

---

### 4. License assignment error (Usage Location)

**Problem**  
Licenses could not be assigned to synced users.

**Cause**  
Missing "usage location" attribute.

**Solution**  
Defined location during license assignment.

**Result**  
Licenses assigned successfully.

---

### 5. Conditional Access not applied

**Problem**  
Access was not blocked despite policy conditions.

**Cause**  
Policy was not applied to all resources.

**Solution**  
Configured:
- Resources → All cloud apps  

**Result**  
Access correctly blocked on non-compliant devices.

---

## 10. Conclusions

This lab demonstrates the implementation of a complete hybrid environment combining on-prem Active Directory with Microsoft Entra ID and Intune, closely simulating a real enterprise setup.

Key concepts covered include:
- Identity management in Active Directory  
- User synchronization with Entra Connect  
- Hybrid device join  
- Device enrollment and management with Intune  
- Compliance policy enforcement  
- Conditional Access based on device state  

In addition to deployment, the lab involved resolving real-world issues related to synchronization, device enrollment, licensing and security policy behavior.

Final outcome:
- Fully functional hybrid identity environment  
- Devices managed through Intune  
- Security policies correctly enforced  
- Access control based on compliance  

This lab demonstrates not only technical implementation, but also the ability to troubleshoot and resolve issues in a real-world hybrid environment.

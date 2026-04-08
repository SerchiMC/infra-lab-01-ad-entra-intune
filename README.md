# Lab NEXUS — Entorno Híbrido (AD + Entra ID + Intune + Acceso Condicional)

---

## **1. Introducción**
En este laboratorio se implementa un entorno híbrido combinando Active Directory on-prem con Microsoft Entra ID, centrado en la gestión de identidades y dispositivos en un contexto corporativo.

El objetivo no es únicamente desplegar la infraestructura, sino validar su comportamiento en un escenario real: sincronización de identidades, gestión de dispositivos mediante Intune y control de acceso basado en el estado del dispositivo mediante Conditional Access.

Este enfoque permite simular un modelo básico de seguridad tipo Zero Trust, donde el acceso a recursos no depende únicamente de la identidad, sino también del cumplimiento del dispositivo.

---

## **2. Enfoque del laboratorio**
Este laboratorio no está enfocado a la instalación paso a paso, sino a validar el comportamiento de un entorno híbrido en escenarios reales de gestión de identidades, dispositivos y control de acceso.

---

## **3. Qué se ha implementado**
- Entorno híbrido Active Directory + Entra ID
- Hybrid Join de dispositivos
- Gestión de dispositivos con Intune
- Políticas de cumplimiento
- Conditional Access basado en estado del dispositivo
- Validación de escenarios de acceso y exclusión
- Resolución de incidencias reales durante la implementación

---

## **4. Arquitectura del entorno**
La arquitectura del entorno consta de los siguientes elementos:

- Controlador de dominio (Windows Server 2022): NEX-DC01  
- Equipo cliente (Windows 10/11): NEX-CL01  
- Dominio on-prem: NEXUS.local  
- Tenant de Microsoft: sergiotech26.onmicrosoft.com  
- Servicios utilizados: Active Directory, DNS, DHCP, Entra ID, Azure AD Connect, Intune y Conditional Access  

Se trata de un entorno reducido, pero suficiente para reproducir los flujos clave de identidad híbrida y gestión de acceso en escenarios empresariales.

---

## **5. Configuración on-prem (Active Directory)**
Se despliega el dominio en el controlador de dominio, incluyendo los servicios de DNS y DHCP necesarios para la operativa del entorno.

![01-nexus-domain-created](capturas/01-nexus-domain-created.png)

Se define una estructura de Unidades Organizativas (OU) orientada a segmentar usuarios y facilitar la aplicación de políticas por departamento.

![02-nexus-ad-ou-structure](capturas/02-nexus-ad-ou-structure.png)

Se configura un sufijo UPN personalizado alineado con el entorno cloud, lo que permite una integración coherente con Entra ID tras la sincronización.

![03-nexus-upn-suffix-added](capturas/03-nexus-upn-suffix-added.png)

Se crean los grupos de seguridad que servirán como base para la asignación de políticas (GPO, Intune y Conditional Access), replicando un modelo habitual de gestión por grupos.

![04-nexus-ad-group-membership](capturas/04-nexus-ad-group-membership.png)

---

## **6. Servicios base y GPO**
Se configura DHCP en el controlador de dominio, definiendo un rango de direcciones IP desde 192.168.0.10 hasta 192.168.0.50 con una concesión de 8 días, simulando asignación dinámica en red corporativa.

El equipo cliente se une al dominio, validando la correcta asignación de IP y la conectividad con el entorno.

![05-nexus-dhcp-client-ip](capturas/05-nexus-dhcp-client-ip.png)

Se verifica la autenticación contra Active Directory iniciando sesión con un usuario de dominio.

![06-nexus-client-domain-joined](capturas/06-nexus-client-domain-joined.png)  
![07-nexus-domain-user-login](capturas/07-nexus-domain-user-login.png)

Se implementa una GPO de mapeo de unidad de red (V:) asociada al departamento de Ventas, simulando acceso a recursos compartidos controlado por grupo.

![08-gpo-drive-mapping-ventas](capturas/08-gpo-drive-mapping-ventas.png)  
![09-client-drive-mapped](capturas/09-client-drive-mapped.png)

Se configura el despliegue de una impresora mediante GPO, replicando un escenario típico de provisión centralizada de recursos.

![10-gpo-printer-deployment](capturas/10-gpo-printer-deployment.png)

Se valida su correcta aplicación en el cliente tras la actualización de políticas.

![11-client-printer-deployed](capturas/11-client-printer-deployed.png)

---

## **7. Integración híbrida (Entra ID)**
Se instala Azure AD Connect en el controlador de dominio para habilitar la sincronización entre Active Directory y Entra ID.

Se configura la sincronización de identidades, permitiendo que los usuarios on-prem estén disponibles en el entorno cloud.

![12-entra-synced-users](capturas/12-entra-synced-users.png)

Se valida el estado híbrido del dispositivo mediante `dsregcmd /status`, confirmando su unión tanto a Active Directory como a Entra ID (Hybrid Join).

![13-hybrid-join-status-nex-cl01](capturas/13-hybrid-join-status-nex-cl01.png)

---

## **8. Gestión de dispositivos (Intune)**
El proceso de enrollment en Intune no se realiza automáticamente, lo que obliga a intervenir para forzar la inscripción mediante políticas.

Se implementa una GPO de auto enrollment para registrar los dispositivos en Intune al iniciar sesión con un usuario de dominio.

![14-gpo-intune-created](capturas/14-gpo-intune-created.png)  
![15-gpo-intune-autoenrollment-config](capturas/15-gpo-intune-autoenrollment-config.png)

Se valida en Entra ID que el dispositivo se registra correctamente como gestionado vía MDM.

![16-intune-device-enrolled-nex-cl01](capturas/16-intune-device-enrolled-nex-cl01.png)

Se confirma su aparición en Intune como dispositivo gestionado.

![17-intune-device-visible](capturas/17-intune-device-visible.png)

---

## **9. Gestión de identidades**
Se crea el usuario `cgarcia` junto con el grupo `GG-Excepciones`, que se utilizará para validar escenarios de exclusión en políticas de acceso.

![18-ad-user-cgarcia-created](capturas/18-ad-user-cgarcia-created.png)  
![19-ad-group-excepciones-members](capturas/19-ad-group-excepciones-members.png)

Se comprueba su correcta sincronización con Entra ID.

![20-entra-user-cgarcia-synced](capturas/20-entra-user-cgarcia-synced.png)

---

## **10. Control de acceso y cumplimiento (Conditional Access + Intune)**
Se define una política de cumplimiento en Intune basada en el estado del firewall, como indicador simple pero efectivo del estado del dispositivo.

![21-intune-compliance-policy-created](capturas/21-intune-compliance-policy-created.png)

La política se asigna al grupo GG-Ventas, estableciendo un control segmentado por usuarios.

![22-intune-compliance-assigned-ventas](capturas/22-intune-compliance-assigned-ventas.png)

Se configura Conditional Access en Entra ID, incluyendo GG-Ventas y excluyendo GG-Excepciones, simulando control de acceso basado en rol y excepciones.

![23-entra-ca-exclude-groups](capturas/23-entra-ca-exclude-groups.png)  
![24-entra-ca-include-groups](capturas/24-entra-ca-include-groups.png)

Se fuerza un estado de no cumplimiento desactivando el firewall en el cliente.

![25-client-firewall-disabled](capturas/25-client-firewall-disabled.png)

Intune detecta el cambio y marca el dispositivo como no compliant.

![26-intune-device-noncompliant-firewall](capturas/26-intune-device-noncompliant-firewall.png)

El acceso a recursos cloud es bloqueado para usuarios afectados por la política.

![27-entra-ca-blocked-ventas-user](capturas/27-entra-ca-blocked-ventas-user.png)

Se valida que los usuarios excluidos mantienen acceso.

![28-entra-ca-exclusion-working](capturas/28-entra-ca-exclusion-working.png)

---

## **11. Incidencias relevantes y resolución**
Durante la implementación se han identificado incidencias reales relacionadas con sincronización, conectividad y gestión de dispositivos.

**1. Conectividad parcial en el servidor (DNS)**  
**Problema**  
El servidor tenía conectividad (ping) pero no podía acceder a servicios externos.  

**Causa**  
Falta de configuración de reenviadores DNS.  

**Solución**  
Configuración de forwarders externos:  
• 8.8.8.8  
• 1.1.1.1  

**Conclusión**  
La conectividad IP no garantiza resolución DNS funcional.  

---

**2. Hybrid Join sin inscripción en Intune**  
**Problema**  
El dispositivo estaba correctamente unido:  
• AzureAdJoined: YES  
• DomainJoined: YES  
• MDM: None  

**Causa**  
El proceso de auto enrollment no se ejecutó automáticamente.  

**Solución**  
Aplicación de GPO de inscripción MDM:  
• Ruta:  
Configuración del equipo → Plantillas administrativas → Componentes de Windows → MDM  
• Directiva: inscripción automática con credenciales de Entra  
• Tipo: Usuario  

**Resultado**  
El dispositivo se registró correctamente en Intune.  

---

**3. Estado inconsistente de enrolamiento**  
**Problema**  
El dispositivo desapareció de Entra tras intentar el enrollment.  

**Causa**  
Registro en estado inconsistente.  

**Solución**  
dsregcmd /leave  
• Reinicio del equipo  
• Inicio de sesión con usuario de dominio  
• Reaplicación de GPO  

**Resultado**  
Estado final correcto:  
• AzureAdJoined: YES  
• DomainJoined: YES  
• MDM: Microsoft Intune  

---

**4. Error al asignar licencias (Usage Location)**  
**Problema**  
No era posible asignar licencias a usuarios sincronizados.  

**Causa**  
Falta de atributo "país/región" en Active Directory.  

**Solución**  
Configuración desde AD:  
• Usuario → Propiedades → Dirección → País: España  

**Resultado**  
Asignación de licencias completada correctamente.  

---

**5. Conditional Access no aplicado**  
**Problema**  
El acceso no era bloqueado pese a incumplir la política.  

**Causa**  
La política no estaba aplicada a todos los recursos.  

**Solución**  
Configuración de:  
• Recursos → All cloud apps  

**Resultado**  
El acceso se bloquea correctamente en dispositivos no compliant.  

---

## **12. Conclusiones**
Este laboratorio demuestra la implementación de un entorno híbrido funcional donde identidad, dispositivo y acceso están interrelacionados.

Más allá del despliegue técnico, se valida:
- La importancia de la sincronización entre entornos  
- El impacto del estado del dispositivo en el acceso  
- El comportamiento real de políticas de seguridad en escenarios no ideales  

El resultado es un entorno donde el acceso a recursos depende no solo de la identidad, sino también del cumplimiento del dispositivo, alineándose con modelos modernos de seguridad.

Este laboratorio demuestra la capacidad de trabajar con identidades híbridas, gestión de endpoints y control de acceso basado en estado del dispositivo, entendiendo tanto la configuración como el comportamiento real del entorno.
---



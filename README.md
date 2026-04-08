# Lab NEXUS — Entorno Híbrido (AD + Entra ID + Intune + Acceso Condicional)

---

## **1. Introducción**
En este laboratorio he montado un entorno híbrido basado en Active Directory on-prem y Entra ID, enfocado a la gestión de usuarios y dispositivos en un entorno corporativo.
El objetivo es simular un escenario real de empresa, no solo montando la infraestructura, sino también trabajando su uso posterior: sincronización de identidades, gestión con Intune y control de acceso mediante Conditional Access.

---

## **2. Arquitectura del entorno**
La arquitectura del entorno consta de los siguientes elementos:

- Controlador de dominio (Windows Server 2022): NEX-DC01  
- Equipo cliente (Windows 10/11): NEX-CL01  
- Dominio on-prem: NEXUS.local  
- Tenant de Microsoft: sergiotech26.onmicrosoft.com  
- Servicios utilizados: Active Directory, DNS, DHCP, Entra ID, Azure AD Connect, Intune y Conditional Access  

---

## **3. Configuración on-prem (Active Directory)**
Para comenzar, se ha creado el dominio en el controlador de dominio, instalando los servicios necesarios como DNS y DHCP.

![01-nexus-domain-created](capturas/01-nexus-domain-created.png)

A continuación, se ha definido la estructura de unidades organizativas (OU), creando una OU principal de usuarios y otras específicas para los distintos departamentos.

![02-nexus-ad-ou-structure](capturas/02-nexus-ad-ou-structure.png)

Posteriormente, se ha configurado un sufijo UPN personalizado, que se ha utilizado en la creación de los usuarios para su integración con el entorno híbrido.

![03-nexus-upn-suffix-added](capturas/03-nexus-upn-suffix-added.png)

Por último, se han creado los grupos necesarios para la gestión de usuarios, que se utilizarán más adelante para aplicar configuraciones y controlar accesos.

![04-nexus-ad-group-membership](capturas/04-nexus-ad-group-membership.png)

---

## **4. Servicios base y GPO**
Se ha configurado el servicio DHCP en el controlador de dominio, definiendo un rango de direcciones IP desde 192.168.0.10 hasta 192.168.0.50, con una concesión de 8 días.

Una vez configurado, se ha unido el equipo cliente al dominio, comprobando mediante ipconfig que recibe correctamente una dirección IP dentro del rango definido.

![05-nexus-dhcp-client-ip](capturas/05-nexus-dhcp-client-ip.png)

Se valida la correcta inclusión en el dominio iniciando sesión con un usuario de Active Directory.

![06-nexus-client-domain-joined](capturas/06-nexus-client-domain-joined.png)  
![07-nexus-domain-user-login](capturas/07-nexus-domain-user-login.png)

A continuación, se ha creado una GPO para mapear una unidad de red, asignando la unidad “V” con el nombre “Ventas”, y vinculándola al grupo correspondiente. Se verifica su aplicación en el equipo cliente.

![08-gpo-drive-mapping-ventas](capturas/08-gpo-drive-mapping-ventas.png)  
![09-client-drive-mapped](capturas/09-client-drive-mapped.png)

Posteriormente, se ha configurado el despliegue de una impresora mediante GPO. Para ello, se ha instalado previamente en el controlador de dominio y se ha compartido en red, aplicando después la política para su distribución a los usuarios.

![10-gpo-printer-deployment](capturas/10-gpo-printer-deployment.png)

Finalmente, se comprueba en el cliente que la impresora se instala correctamente tras la aplicación de la GPO.

![11-client-printer-deployed](capturas/11-client-printer-deployed.png)

---

## **5. Integración híbrida (Entra ID)**
Para integrar el entorno on-prem con Entra ID, se ha instalado Azure AD Connect en el controlador de dominio, utilizando el agente descargado desde el portal de Entra.

Una vez instalado, se configura la sincronización de identidades entre Active Directory y Entra ID, permitiendo que los usuarios creados en el entorno on-prem aparezcan en la nube.

Tras completar la configuración, se accede a Entra ID para comprobar que la sincronización se ha realizado correctamente.

![12-entra-synced-users](capturas/12-entra-synced-users.png)

Por último, se valida el estado híbrido del equipo cliente iniciando sesión con un usuario del dominio y verificando mediante el comando dsregcmd /status que el dispositivo está correctamente unido tanto a Active Directory como a Entra ID (Hybrid Join).

![13-hybrid-join-status-nex-cl01](capturas/13-hybrid-join-status-nex-cl01.png)

---

## **6. Gestión de dispositivos (Intune)**
Durante la configuración del enrollment en Intune, el proceso no se realizó automáticamente, incidencia que se detalla más adelante en el apartado correspondiente.

Para solucionarlo, se ha creado una GPO de auto enrollment que permite registrar automáticamente los equipos en Intune al iniciar sesión con un usuario del dominio.

![14-gpo-intune-created](capturas/14-gpo-intune-created.png)  
![15-gpo-intune-autoenrollment-config](capturas/15-gpo-intune-autoenrollment-config.png)

Una vez aplicada la GPO, se comprueba en Entra ID que el dispositivo ha sido correctamente inscrito, filtrando por MDM para validar su registro.

![16-intune-device-enrolled-nex-cl01](capturas/16-intune-device-enrolled-nex-cl01.png)

Finalmente, se verifica que el equipo aparece correctamente en Intune como dispositivo gestionado.

![17-intune-device-visible](capturas/17-intune-device-visible.png)

---

## **7. Gestión de identidades**
En este apartado se ha creado un nuevo usuario llamado cgarcia, junto con un grupo denominado GG-Excepciones, que se utilizará posteriormente en la configuración de acceso condicional.

El usuario cgarcia se añade a dicho grupo para aplicar excepciones dentro de las políticas de acceso.

![18-ad-user-cgarcia-created](capturas/18-ad-user-cgarcia-created.png)  
![19-ad-group-excepciones-members](capturas/19-ad-group-excepciones-members.png)

Una vez creado, se comprueba que el usuario se sincroniza correctamente con Entra ID como parte del entorno híbrido.

![20-entra-user-cgarcia-synced](capturas/20-entra-user-cgarcia-synced.png)

---

## **8. Control de acceso y cumplimiento (Conditional Access + Intune)**
Se ha creado una política de cumplimiento en Intune para dispositivos Windows 10/11, utilizando como criterio principal la obligatoriedad de tener el firewall activo, ya que permite validar de forma inmediata el estado del dispositivo.

![21-intune-compliance-policy-created](capturas/21-intune-compliance-policy-created.png)

La política se asigna al grupo GG-Ventas, comprobando posteriormente en Intune que los dispositivos del grupo reciben correctamente dicha configuración.

![22-intune-compliance-assigned-ventas](capturas/22-intune-compliance-assigned-ventas.png)

A continuación, se configura una política de Conditional Access en Entra ID, donde se incluye el grupo GG-Ventas (usuarios a los que se aplica la política) y se excluye el grupo GG-Excepciones, utilizado para gestionar accesos especiales.

![23-entra-ca-exclude-groups](capturas/23-entra-ca-exclude-groups.png)  
![24-entra-ca-include-groups](capturas/24-entra-ca-include-groups.png)

Para validar el funcionamiento, se realiza una prueba en el equipo cliente iniciando sesión con un usuario del grupo Ventas. Se desactiva el firewall de Windows con permisos de administrador para forzar un estado de no cumplimiento.

![25-client-firewall-disabled](capturas/25-client-firewall-disabled.png)

Tras este cambio, Intune detecta que el dispositivo pasa a estado No compliant.

![26-intune-device-noncompliant-firewall](capturas/26-intune-device-noncompliant-firewall.png)

Se intenta acceder a un recurso cloud (portal.office.com) con el usuario cmolina, perteneciente al grupo GG-Ventas, y el acceso es bloqueado al no cumplir las condiciones definidas en la política.

![27-entra-ca-blocked-ventas-user](capturas/27-entra-ca-blocked-ventas-user.png)

Por último, se realiza la misma prueba con el usuario cgarcia, perteneciente al grupo GG-Excepciones, comprobando que el acceso se permite al quedar excluido de la política.

![28-entra-ca-exclusion-working](capturas/28-entra-ca-exclusion-working.png)

---

## **9. Incidencias relevantes y resolución**
Durante la implementación del entorno híbrido se identificaron incidencias reales relacionadas con conectividad, sincronización y gestión de dispositivos.

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

## **10. Conclusiones**
Este laboratorio ha permitido implementar un entorno híbrido completo combinando Active Directory on-premises con Microsoft Entra ID e Intune, reproduciendo un escenario cercano a un entorno empresarial real.

A nivel técnico, se han trabajado conceptos clave como:  
• Gestión de identidades en Active Directory  
• Sincronización de usuarios mediante Entra Connect  
• Unión híbrida de dispositivos (Hybrid Join)  
• Inscripción y gestión de dispositivos en Intune  
• Aplicación de políticas de cumplimiento (compliance)  
• Implementación de Acceso Condicional basado en estado del dispositivo  

Además de la configuración, el laboratorio ha estado marcado por la resolución de incidencias reales, especialmente en:  
• Procesos de sincronización entre entornos  
• Inscripción de dispositivos en MDM  
• Asignación de licencias en entornos híbridos  
• Comportamiento de políticas de seguridad  

Estas situaciones han permitido desarrollar habilidades de diagnóstico y validación, entendiendo que en entornos híbridos los procesos no siempre son inmediatos ni lineales.

Como resultado final, se ha conseguido:  
• Un entorno funcional con identidad híbrida  
• Dispositivos gestionados mediante Intune  
• Políticas de seguridad aplicadas correctamente  
• Control de acceso basado en cumplimiento del dispositivo  

Este laboratorio no solo valida conocimientos técnicos, sino también la capacidad de enfrentarse a problemas reales, analizar su causa y aplicar soluciones de forma estructurada.

---


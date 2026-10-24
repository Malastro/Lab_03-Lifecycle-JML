# Lab_03-Lifecycle-JML

### 1. Descripción del proyecto
   
En el marco de la modernización corporativa de Mariscos Recio S.L., la dirección ejecutiva encabezada por Antonio Recio (Mayorista no limpia pescado) constata que la administración de personal ya no puede gestionarse de manera informal ni resolverse a base de arbitrariedades operativas y despidos a viva voz.

Para garantizar el cumplimiento normativo, la trazabilidad en auditorías de seguridad y la mitigación de vectores de ataque internos, se formaliza un protocolo técnico de Identidad y Gestión de Accesos (IAM) basado en el ciclo de vida JML (Joiner, Mover, Leaver), sincronizado de forma híbrida entre el Directorio Activo local (AD DS) y la nube (Microsoft Entra ID):

### Joiner (Alta e Incorporación Corporativa):

Escenario: Contratación formal de Rosario Parrales como personal de la empresa asignado inicialmente al departamento de transporte.

Operativa técnica: Aprovisionamiento estructurado de la cuenta (rparrales), definición de atributos corporativos estandarizados (Title: Repartidor, Department: Reparto), asignación en la unidad organizativa de clasificación (OU=Reparto) y entrega de permisos mediante el rol base de seguridad (ACL-Reparto) bajo un modelo RBAC.

### Mover (Movilidad Interna y Transición de Rol):

Escenario: Reclasificación laboral y promoción de Rosario Parrales, pasando de repartidor a responsable de mostrador (Pescadero).

Operativa técnica: Reubicación jerárquica del objeto hacia la carpeta departamental (OU=Pescaderia), actualización de metadatos laborales y reasignación de privilegios operativos (ACL-Pescaderia). Se aplica una política estricta de mitigación de Privilege Creep revocando de inmediato la pertenencia a ACL-Reparto para impedir la acumulación residual de permisos no requeridos.

### Leaver (Cese de Funciones y Contención Zero Trust):

Escenario: Extinción del contrato laboral y desvinculación inmediata del empleado de la compañía.

Operativa técnica: Ejecución del protocolo defensivo bajo el principio Zero Trust («Asumir la brecha» y «Acceso mínimo indispensable»). Bloqueo instantáneo de la autenticación (Disable-ADAccount), revocación total de pertenencia a grupos de seguridad departamentales, aislamiento del objeto en la unidad de cuarentena (OU=Bajas) y forzado de sincronización delta hacia Microsoft Entra ID (Cuenta habilitada: No) para invalidar el acceso federado o directo a cualquier recurso corporativo local o en la nube.

### 2. Fases de implementación

## Fase 1: Joiner — Aprovisionamiento de Identidad y RBAC

### 1. Propósito Técnico y Arquitectura
En esta primera fase del ciclo de vida de la identidad (JML), se implementa el aprovisionamiento de un nuevo empleado en la infraestructura híbrida de **Mariscos Recio**. 

Se diseñó una estructura organizativa desacoplada basada en dos pilares fundamentales:
* **Unidades Organizativas (OUs):** Utilizadas exclusivamente para la clasificación administrativa, jerarquía del directorio y aplicación de políticas de grupo (GPOs) (`Mariscos_Recio/Reparto`).
* **Grupos de Seguridad (RBAC):** Se implementa el modelo de Control de Acceso Basado en Roles (*Role-Based Access Control*). El usuario no recibe permisos de forma individual; la autorización se canaliza a través de grupos globales de seguridad dedicados (`ACL-Reparto`), garantizando una gestión centralizada y escalable de los privilegios.

### 2. Aprovisionamiento en Active Directory Domain Services (AD DS)
Se ejecutó un script en PowerShell automatizando la creación de la identidad en la OU correspondiente y la asignación inmediata de su rol departamental:

```powershell
# Definición de parámetros para evitar errores de sintaxis y caracteres residuales
$userParams = @{
    Name                  = "Rosario Parrales"
    GivenName             = "Rosario"
    Surname               = "Parrales"
    DisplayName           = "Rosario Parrales"
    SamAccountName        = "rparrales"
    UserPrincipalName     = "rparrales@malastro.onmicrosoft.com"
    Department            = "Reparto"
    Title                 = "Repartidor"
    Path                  = "OU=Reparto,OU=Mariscos_Recio,DC=montepinar,DC=local"
    AccountPassword       = (ConvertTo-SecureString "PasswordTemporal2026!" -AsPlainText -Force)
    Enabled               = $true
    ChangePasswordAtLogon = $true
}

# Creación de usuario y asignación a rol RBAC
New-ADUser @userParams
Add-ADGroupMember -Identity "ACL-Reparto" -Members "rparrales"
```

![Parrales en AD](01_ParralesAD.png)
![Parrales en Entra ID](02_ParralesEntraID.png)

## Fase 2: Mover — Transición de Rol y Mitigación de Privilege Creep

### 1. Justificación Técnica y Principio de Menor Privilegio
En la gestión del ciclo de vida de la identidad (JML), la fase de movilidad interna representa uno de los vectores críticos de riesgo en auditorías de seguridad e IAM: el **Privilege Creep** (acumulación residual de permisos). 

Cuando un empleado cambia de departamento o función laboral (en este caso, la promoción de Rosario Parrales de *Reparto* a *Pescadería*), el modelo RBAC exige:
1. Reubicar el objeto en su contexto jerárquico dentro del directorio.
2. Otorgar los nuevos accesos operativos correspondientes a su nueva labor.
3. **Revocar de forma explícita e inmediata** los grupos de seguridad del rol anterior, evitando que conserve privilegios innecesarios sobre recursos a los que ya no debe acceder.

### 2. Implementación en PowerShell (AD DS)
Se ejecutó un procedimiento desacoplado para modificar la ubicación organizativa, actualizar los metadatos corporativos y reasignar los grupos de seguridad:

```powershell
# 1. Mover el objeto a su nueva OU mediante su Distinguished Name (DN)
Move-ADObject -Identity "CN=Rosario Parrales,OU=Reparto,OU=Mariscos_Recio,DC=montepinar,DC=local" `
              -TargetPath "OU=Pescaderia,OU=Mariscos_Recio,DC=montepinar,DC=local"

# 2. Actualizar metadatos de puesto y departamento
Set-ADUser -Identity "rparrales" -Department "Pescaderia" -Title "Pescadero"

# 3. Asignación del nuevo rol RBAC
Add-ADGroupMember -Identity "ACL-Pescaderia" -Members "rparrales"

# 4. Revocación del rol previo (Mitigación de Privilege Creep)
Remove-ADGroupMember -Identity "ACL-Reparto" -Members "rparrales" -Confirm:$false
```
![Parrales pescaderia AD](03_ParralesPescaderia.png)
![Parrales pescaderia ID](04_ParralesPescaderiaID.png)

## Fase 3: Leaver — Baja, Revocación de Accesos y Principio Zero Trust

### 1. Justificación de Seguridad y Gestión del Riesgo
La desvinculación de un empleado representa una de las fases más críticas en el ciclo de vida de la identidad (JML). Una ejecución deficiente da lugar a **cuentas huérfanas o cuentas zombi** (*dormant/zombie accounts*), las cuales constituyen vectores de ataque primarios para intrusiones persistentes o amenazas internas.

Bajo los principios de **Zero Trust** (*«Asumir la brecha»* y *«Verificación explícita»*), el proceso de baja corporativa no elimina destructivamente el objeto de forma inmediata, sino que ejecuta una revocación total y contención en cuarentena:
1. **Revocación inmediata de autenticación:** Inhabilitación de la cuenta en el origen de identidad autoritativo para revocar sesiones y bloquear futuros inicios de sesión.
2. **Revocación estricta de privilegios (RBAC):** Retirada de todas las membresías a grupos de seguridad departamentales (`ACL-Pescaderia`).
3. **Aislamiento en OU de cuarentena:** Traslado del objeto a `OU=Bajas` para evitar la aplicación de directivas de grupo operativas (GPOs) y mantener trazabilidad para auditorías forenses y cumplimiento normativo.

### 2. Implementación en PowerShell (AD DS)
Se ejecutó el procedimiento de desaprovisionamiento y aislamiento:

```powershell
# 1. Bloquear inmediatamente la autenticacion
Disable-ADAccount -Identity "rparrales"

# 2. Revocar pertenencia al rol de seguridad departamental
Remove-ADGroupMember -Identity "ACL-Pescaderia" -Members "rparrales" -Confirm:$false

# 3. Trasladar el objeto a la OU de cuarentena/bajas para auditoria
Move-ADObject -Identity "CN=Rosario Parrales,OU=Pescaderia,OU=Mariscos_Recio,DC=montepinar,DC=local" `
              -TargetPath "OU=Bajas,OU=Mariscos_Recio,DC=montepinar,DC=local"

# 4. Registrar metadatos de auditoria interna
Set-ADUser -Identity "rparrales" -Description "Baja efectiva - Bloqueado por auditoria IAM"
```

![Parrales deja la empresa AD](05_ParralesADLeaver.png)
![Parrales deja la empresa ID](06_ParralesIDLeaver.png)

## 3. Conclusiones y Competencias Demostradas

1. **Gobernanza de Identidades Híbridas:** Despliegue y configuración del conector Microsoft Entra Connect, controlando el filtrado de Unidades Organizativas (OUs) autoritativas y forzando ciclos de sincronización delta mediante PowerShell.
2. **Implementación Estricta de RBAC:** Desacoplamiento entre la estructura jerárquica del directorio (OUs para categorización y políticas) y los grupos de seguridad globales para asignación granular de privilegios.
3. **Mitigación Activa de Privilege Creep:** Demostración práctica de cómo un proceso de movilidad interna (Mover) debe incluir la revocación inmediata de pertenencia a grupos departamentales anteriores.
4. **Postura Zero Trust en Procesos de Salida (Leaver):** Aislamiento en OU de cuarentena, revocación completa de membresías y bloqueo de credenciales sincronizado inmediatamente con la nube para neutralizar el riesgo de cuentas huérfanas (*dormant accounts*).

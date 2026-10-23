# Lab_03-Lifecycle-JML

### 1. Descripción del proyecto
   
Enrique Pastor (Concejal de Juventud y Tiempo Libre y Secretario/Administrador en funciones) advierte a la junta que la gestión de personal no puede ser un caos de favores personales ni gritos de "¡Quedas despedido!". Exige formalizar un protocolo técnico de Recursos Humanos sincronizado con el directorio:

- Joiner (Alta / Nuevo empleado): Contratación formal de Coque Calatrava como jardinero/portero o incorporación de un nuevo empleado en Mariscos Recio. Aprovisionamiento automatizado de cuenta, correo corporativo, atributos de puesto y grupos base.

- Mover (Movilidad / Cambio de puesto): Reclasificación interna. Por ejemplo, ascenso o traslado de departamento (Coque pasa de jardinero a consejere). Debe asignarse el nuevo rol y revocarse estrictamente el acceso anterior para evitar Privilege Creep.

- Leaver (Baja / Adiós a la empresa): Cese inmediato de funciones. Bloqueo de cuenta en Active Directory, traslado a OU de archivo, forzado de sincronización hacia la nube y revocación inmediata de tokens de sesión activos (Continuous Access Evaluation / Revoke Sign-in Sessions) para expulsar al usuario de todos sus dispositivos al instante.

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

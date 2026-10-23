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


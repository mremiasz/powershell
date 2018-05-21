#Powershell oraz active directory

Skrypt do tworzenia urzytkownikow z pliku txt w active directory:
plik users.txt to:
Tomasz Nowak
Anna Kowalska

$file = get-content .\users.txt
$file | % { 
 $pass = -join((65..90) + (97..122) | Get-Random -Count 8 | %{[char]$_}) +"2#"
 $login = ($_.split(' '))[1]
 Write-Host $login $pass
 New-ADUser $login -AccountPassword (ConvertTo-SecureString $pass -AsPlainText -Force)
}
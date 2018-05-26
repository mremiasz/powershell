# Powershell oraz active directory
**Do zrobienia i dodania są zadania z drugiej części**
### Skrypt do tworzenia urzytkownikow z pliku txt w active directory:
plik users.txt to:
```
 Tomasz Nowak
 Anna Kowalska
```

```powershell
$file = get-content .\users.txt
$file | % { 
 $pass = -join((65..90) + (97..122) | Get-Random -Count 8 | %{[char]$_}) +"2#"
 $login = ($_.split(' '))[1]
 Write-Host $login $pass
 New-ADUser $login -AccountPassword (ConvertTo-SecureString $pass -AsPlainText -Force)
}
```
### Pierwsza część zaliczenia
**1** Zatrzymaj wszystkie procesy, których nazwa zaczyna się na literę p.
```bash
$ps -e | grep " p" | awk '{ print $1 }' | xargs kill
```
```powershell
Get-Process | ?{$_.Name -match "^p.*$"} | % {$_.kill()}
get-process p* | % {$_.kill()}
```
**2** Znajdź wszytkie procesy, które zajmują więcej niż 1000MB pamięci i ubij je.
```bash
$ps -el | awk '{ if ( $6 > (1024*10)) { print $3 } }' | grep -v PID | xargs kill
```
```powershell
get-process | select {$_.privatememorysize/1mb -ge 100} | % {$_.kill()}
```
**3** Sprawdź ile bajtów zajmują pliki w danej kartotece.
```bash
$tot=0; for file in $( ls )
do
 set -- $( ls -log $file )
 echo $3
 (( tot = $tot + $3 ))
done; echo $tot

$ls -l | awk { tot += $5; print tot; } | tail -1
```
```powershell
$a = Get-ChildItem $directory -recurse | Measure-Object -property length -sum
$a.sum
```
**4** Sprawdź czy ilość pamięci potrzebna procesom się nie zmieniła.
```bash
$while [ true ]
 do
 msize1=$(ps -el|grep application|grep -v grep|awk '{print $6}')
 sleep 10
 msize2=$(ps -el|grep application|grep -v grep|awk '{print $6}')
 expr $msize2 - $msize1
 msize1=$msize2
done
```
```powershell
$Output1=0
$Output2=0
$Processes = (get-wmiobject Win32_PerfFormattedData_PerfProc_Process) 
foreach($Process in $Processes)
{
 $Output1 += $Process.PageFileBytes 
}

$Processes = (get-wmiobject Win32_PerfFormattedData_PerfProc_Process) 
foreach($Process in $Processes)
{
 $Output2 += $Process.PageFileBytes 
}

$output1/1mb
$output2/1mb

$output1 -eq $output2
```
**5** Sprawdź czy dany proces jeszcze działa.
```bash
$processToWatch=$( ps -e | grep application | awk '{ print $1 }'
$while [ true ]
do
 sleep 10
 processToCheck=$(ps -e |grep application |awk '{print $1}' )
 if [ -z "$processToCheck" -or "$processToWatch" != "$processToCheck" ]
  then
   echo "Process application is not running"
  return
  fi
done
```
```powershell
$procSTART = get-process | select-object id,name
While ((Compare-Object $procSTART (get-process | select-object id,name)) -eq $null)
{
	 sleep 2
	 write-host "true"
} 
Compare-Object $procSTART (get-process | select-object id,name)
```
**6** Zamień na duże litery.
```bash
$echo "this is a string" | tr [:lower:] [:upper:]
$echo "this is a string" | tr '[a-z]' '[A.Z]'
```
```powershell
$napis = "ala ma kota"
$napis = $napis.ToUpper()
```
**7** Wstaw słowo ABC po drugiej literce w stringu.
```bash
$echo "string" | sed "s|\(.\)\(.*)|\1ABC\2|"
```
```powershell
$str = "string"
$strMOD = ($str.Substring(0,1))+"ABS"+($str.Substring(1,($str.length -1)))
```
**8** Odczytaj Bloga z adresu "http://blogs.msdn.com/powershell/rss.aspx"
```powershell
$strona = New-Object Net.WebClient
$strona.DownloadString("http://blogs.msdn.com/powershell/rss.aspx")
```
```powershell
$oIE = new-object -com internetexplorer.application
$oIE.navigate2("http://blogs.msdn.com/powershell/rss.aspx")
$oIE.visible = $true
```
### Druga część zaliczenia

```
Set-ExecutionPolicy -ExecutionPolicy Unrestricted
```

DCPROMO jest narzędziem do promowania serwera do kontrolera domeny, czyli zrobić serwer, który nie jest kontrolerem domeny:
```
dcpromo
import-module activedirectory
```

1) Zmien nazwe komputera z maszyny wirtualnej Windows.2008.001, na AD-001

Komputer lokalny:
```
$compName = Get-WmiObject Win32_ComputerSystem
$compName.Rename("AD-001")
RESTART-COMPUTER -force
Rename-Computer -NewName "ad-002" -DomainCredential rem.dom.pl\Administrator -Restart
```
Komputer zdalny:
```
Rename-Computer -ComputerName "Srv01" -NewName "Server001" -LocalCredential Srv01\Admin01 -DomainCredential Domain01\Admin01 -Force -PassThru -Restart
```
2) Na komputerze AD-001:
	- wyłącz pierwszy (Local Area Connection) interfejs sieciowy 
		
		```
		Get-NetAdapter [This cmdlet is part of the NetAdapter module that comes with Windows 8/Server 2012 and later.]
		$1stLAC = Get-WmiObject -Class Win32_NetworkAdapter -filter "Name LIKE '%Network Connection'"
		$1stLAC.disable()
		# $1stLAC.enable() - aby włączyć
		```
	
	- dla drugiego interfejsu sieciowego (Local Area Connection) wyłącz protokół IPv6, oraz ustaw: 
	
		```
		$2ndLAC = Get-WMIObject Win32_NetworkAdapterConfiguration | where{$_.IPEnabled -eq "TRUE"}
		#Get-NetAdapterBinding -ComponentID ms_tcpip6
		#Enable-NetAdapterBinding -Name "Network Connection #2" -ComponentID ms_tcpip6
		```
		
		- adres statyczny ip4 na 192.168.20.2
		- maska sieciowa na 255.255.255.0
		```
		$2ndLAC.EnableStatic("192.168.20.2", "255.255.255.0")
		https://docs.microsoft.com/en-us/powershell/module/nettcpip/set-netipaddress?view=win10-ps
		http://www.freenode-windows.org/resources/all-windows-versions/configuring-ipv6
		```
		
		- brama 192.168.20.2
		```
		$2ndLAC.SetGateways("192.168.20.1")
		```
		
		```
		https://gist.github.com/munim/794bc70b24dd2288adb65789279ac194
		```
		Kalkulator: https://mxtoolbox.com/subnetcalculator.aspx
		
		- adres DNS na 192.168.20.2
		```
		$DNSServer = "192.168.20.2"
		$2ndLAC.SetDNSServerSearchOrder($DNSServer)
		```
		
		https://docs.microsoft.com/en-us/powershell/module/dnsclient/set-dnsclientserveraddress?view=win10-ps
3) Zainstaluj AD oraz DNS na komputerze AD-001 w domenie winadm.local w trybie zgodności Windows 2008 R2.
	```
		dcpromo
		import-module activedirectory
	```

	https://blogs.technet.microsoft.com/uktechnet/2016/06/08/setting-up-active-directory-via-powershell/

	- skonfiguruj odpowiednio Forward Zone i Reversed Zone
	
4) Zmien nazwe komputera z maszyny wirtualnej Windows.2008.002, na Win-002
	```
		$compName = Get-WmiObject Win32_ComputerSystem
		$compName.Rename("Win-002")
		RESTART-COMPUTER -force
	```

	- wyłącz pierwszy (Local Area Connection) interfejs sieciowy 
	```
		$1stLAC = Get-WmiObject -Class Win32_NetworkAdapter -filter "Name LIKE '%Network Connection'"
		$1stLAC.disable()
	```
	
	- dla drugiego interfejsu sieciowego (Local Area Connection 2) wyłącz protokół IPv6, oraz ustaw :
	```
		$2ndLAC = Get-WMIObject Win32_NetworkAdapterConfiguration | where{$_.IPEnabled -eq "TRUE"}
	```
		- adres statyczny ip4 na 192.168.20.10
		- maskę sieciową na 255.255.255.0
		- brama 192.168.20.2
		- adres DNS na 192.168.20.2
		```
			$2ndLAC.EnableStatic("192.168.20.10", "255.255.255.0")
			$2ndLAC.SetGateways("192.168.20.2")
			$2ndLAC.SetDNSServerSearchOrder("192.168.20.2")
		```
5) Podłącz komputer Win-002 do domeny
	```
		Add-Computer -DomainName "winadm.local"
	```
	
6) Utwórz użytkownika Domenowego
	- Jan Kowalski z loginem kowalski z miasta Warszawa
	- Tomek Nowak z działu poczta
	```
 		$pass = -join((65..90) + (97..122) | Get-Random -Count 8 | %{[char]$_}) +"2#"
		$pass = zVSneits2#
 		$GivenName = "Jan"
		$name = “kowalski”
		$city = "Warszawa"
		$surn = "Kowalski"
 		New-ADUser -GivenName $GivenName -name $name -City $city -Surname $surn -AccountPassword (ConvertTo-SecureString $pass -AsPlainText -Force)
		$name = “tnowak”
		$dep = "poczta"
		$GivenName = "Tomek"
		$surn = "Nowak"
		$city = "Łódź"
		New-ADUser -GivenName $GivenName -Name $name -City $city -Surname $surn -Department $dep -AccountPassword (ConvertTo-SecureString $pass -AsPlainText -Force)
		
		#Remove-ADUser -Identity tnowak
		Get-ADUser -Filter * | Select-Object name, surname, city, Department,GivenName
		
	```

7) Powershell:
	- Zmień hasło użytkownika Kowalski
	```	
		Set-AdAccountPassword -Identity kowalski -Reset -NewPassword (ConvertTo-SecureString -AsPlainText "BNcPakYF49#" -Force)
	```
	
	- Wymuś zmianę hasła użytkownika kowalski przy następnym logowaniu
	- Wyświetl wszytkich użytkowników z miasta Warszawa
	- Zablokuj konto użytkownika z Warszawy
	- Wyświetl członków danej grupy (rekurencyjnie) 
	- Wyświetl informację o ostatnim logowaniu na danym komputerze,
	- Wyświetl informację o systemach operacyjnych na komputerach w domenie
8) Zmień GPO, tak aby przy logowaniu użytkownik nie widział opcji RUN w start menu
9) Znajdź wersję instalacyjną msi programu firefox i utwórz politykę zmuszającą do zainstalowania tego oprogramownia w czasie startu systemu na komputerach w domenie. 

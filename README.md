# Common Active Directory Queries
Collection of common queries for Domain Account enumeration

## Domain Queries

### Get All Sites for Current Forest
`([DirectoryServices.ActiveDirectory.Forest]::GetCurrentForest()).Sites | Out-GridView`

### Get Domain Levels and Modes for Current Forest
`([DirectoryServices.ActiveDirectory.Forest]::GetCurrentForest()).Domains`

### Get All Trust Relationships for Current Domain
`([System.DirectoryServices.ActiveDirectory.Domain]::GetCurrentDomain()).GetAllTrustRelationships()`

### Get All Domain Controllers for Current Domain
`([System.DirectoryServices.ActiveDirectory.Domain]::GetCurrentDomain()).FindAllDomainControllers() | Select-Object -Property Name,Forest,Domain,IPAddress,SiteName,OSVersion,Roles | Out-GridView`

### Get Primary Domain Controller for Current Forest
`[System.DirectoryServices.ActiveDirectory.Forest]::GetCurrentForest().RootDomain.PDCRoleOnwer.Name`

### Domain Password Policy (including complexity)

Password Policy Base Settings
`[ADSI]'' | Select-Object -Property minPwdAge,maxPwdAge,minPwdLength,pwdHistoryLength,lockoutThreshold,lockoutDuration,lockoutObservationWindow,pwdProperties`

Password Complexity Value Definitions ("pwdProperties")

* 0 - {"Passwords can be simple and the administrator account cannot be locked out"}
* 1 - {"Passwords must be complex and the administrator account cannot be locked out"}
* 8 - {"Passwords can be simple, and the administrator account can be locked out"}
* 9 - {"Passwords must be complex, and the administrator account can be locked out"}

References:  
https://www.morgantechspace.com/2016/03/find-ad-domain-password-policy-settings-powershell.html  
Carlos Perez - PS Class from DerbyCon 2018

`[ADSI]"LDAP://dc=domain,dc=tld" | Select -Property pwdProperties`

## Account/User Queries

### Get descriptions for all active accounts in AD

#### ADSI
```
$searchUsers = [adsisearcher]"(&(objectCategory=person)(objectClass=user)(!(userAccountControl:1.2.840.113556.1.4.803:=2)))";
$searchUsers.PageSize = 1000;
$users = $searchUsers.FindAll();
$userProps = $users.Properties;
$userProps | Where-Object {$_.description} | foreach {$_.description + ' : ' + $_.samaccountname} | Sort-Object
```

### Get Count of All Active User Accounts with "Password Never Expires" (Using ADSI or Get-ADUser)

#### Get-ADUser
```
$pwdNExp = Get-ADUser -filter * -properties SamAccountName, PasswordNeverExpires | where { $_.passwordNeverExpires -eq "true" } | where {$_.enabled -eq "True"} | Sort-Object -Property SamAccountName;
Add-Content -Value "Total: $($pwdNExp.Count) Accounts`r`n" .\ADUsers-PasswordNeverExpires.txt -Encoding UTF8;
Add-Content -Value $pwdNExp.SamAccountName .\ADUsers-PasswordNeverExpires.txt -Encoding UTF8
```

#### ADSI
Note: "UAC Flags: 2 == Disabled, 66048 == 512 (normal account) + 65536 (Password Never Expires)"
```
$searchUsers = [adsisearcher]"(&(objectCategory=person)(objectClass=user)(userAccountControl:1.2.840.113556.1.4.803:=66048)(!(userAccountControl:1.2.840.113556.1.4.803:=2)))";
$searchUsers.PageSize = 1000;
$usersPwdNExp = $searchUsers.FindAll();
Add-Content -Value "Total: $($usersPwdNExp.Count) Accounts`r`n" .\ADUsers-PasswordNeverExpires.txt -Encoding UTF8;
Add-Content -Value $usersPwdNExp.SamAccountName .\ADUsers-PasswordNeverExpires.txt -Encoding UTF8
```

### Get Count of All Disabled Accounts (Using ADSI or Get-ADUser)

#### Get-ADUser
```
$usersDisabled = Get-ADUser -filter * -properties SamAccountName, Enabled | where {$_.Enabled -ne "True"} | Sort-Object -Property SamAccountName;
Add-Content -Value "Total: $($usersDisabled.Count) Accounts`r`n" .\ADUsers-Disabled.txt -Encoding UTF8;
Add-Content -Value $usersDisabled.SamAccountName .\ADUsers-Disabled.txt -Encoding UTF8
```

#### ADSI
```
$searchUsers = [adsisearcher]"(&(objectCategory=person)(objectClass=user)(userAccountControl:1.2.840.113556.1.4.803:=2))";
$searchUsers.PageSize = 1000;
$usersDisabled = $searchUsers.FindAll();
Add-Content -Value "Total: $($usersDisabled.Count) Accounts`r`n" .\ADUsers-Disabled.txt -Encoding UTF8;
Add-Content -Value $usersDisabled.SamAccountName .\ADUsers-Disabled.txt -Encoding UTF8
```

### Get Count of All Active User Accounts with "Password Not Required" (Using ADSI or Get-ADUser)

#### Get-ADUser
```
$pwdNReq = Get-ADUser -filter * -Properties SamAccountName, PasswordNotRequired | where { $_.passwordnotrequired -eq "true" } | where {$_.enabled -eq "true"} | Sort-Object -Property SamAccountName;
Add-Content -Value "Total: $($pwdNReq.Count) Accounts`r`n" .\ADUsers-PasswordNotRequired.txt -Encoding UTF8;
Add-Content -Value $pwdNReq.SamAccountName .\ADUsers-PasswordNotRequired.txt -Encoding UTF8
```

#### ADSI
```
$searchUsers = [adsisearcher]"(&(objectCategory=person)(objectClass=user)(userAccountControl:1.2.840.113556.1.4.803:=32)(!(userAccountControl:1.2.840.113556.1.4.803:=2)))";
$searchUsers.PageSize = 1000;
$usersIntDomTrst = $searchUsers.FindAll();
Add-Content -Value "Total: $($pwdNReq.Count) Accounts`r`n" .\ADUsers-PasswordNotRequired.txt -Encoding UTF8;
Add-Content -Value $pwdNReq.SamAccountName .\ADUsers-PasswordNotRequired.txt -Encoding UTF8
```

### Get Count of Inter Domain Trust Accounts (Using ADSI or Get-ADUser)

#### Get-ADUser
*Need Get-ADUser syntax

#### ADSI
```
$searchUsers = [adsisearcher]"(&(objectCategory=person)(objectClass=user)(userAccountControl:1.2.840.113556.1.4.803:=2560)(!(userAccountControl:1.2.840.113556.1.4.803:=2)))"
$searchUsers.PageSize = 1000
$usersIntDomTrt = $searchUsers.FindAll()
$usersIntDomTrt.Count
```

### Get Active Accounts with Passwords Older than 1 Year (Using ADSI or Get-ADUser)

#### Get-ADUser
*Need Get-ADUser syntax

#### ADSI
```
$searchUsers = [adsisearcher]"(&(objectCategory=person)(objectClass=user)(userAccountControl:1.2.840.113556.1.4.803:=512)(!(userAccountControl:1.2.840.113556.1.4.803:=2)))"
$searchUsers.PageSize = 1000
$usersPwdGT1Yr = $searchUsers.FindAll()
$pwdLastSetTable = ($usersPwdGT1Yr | Select-Object -ExpandProperty Properties).pwdlastset
$pwdLastSetGT1Yr = @()
$pwdLastSetTable | foreach {$pwdLastSetDate = [datetime]::FromFileTime($_); if ($pwdLastSetDate -lt (Get-Date).adddays(-365)) { $pwdLastSetGT1Yr += $_ }}
$pwdLastSetGT1Yr.Count
```
** $pwdLastSetTable | foreach([datetime]::FromFileTime($_.Item()))

### Get Active Accounts that haven't been logged into in over 90 days
```
$90Days = (get-date).adddays(-90)
$accts90 = Get-ADUser -properties * -filter {(lastlogondate -notlike "*" -OR lastlogondate -le $90days) -AND (passwordlastset -le $90days) -AND (enabled -eq $True) -and (PasswordNeverExpires -eq $false) -and (whencreated -le $90days)} | select-object name, SAMaccountname, passwordExpired, PasswordNeverExpires, logoncount, whenCreated, lastlogondate, PasswordLastSet, lastlogontimestamp
$accts90.Count
```

## Group Queries

### Get a formatted list of "Domain Admins" members  
```
$objADSI = [ADSI]'LDAP://CN=Domain Admins,CN=Users,DC=[domain],DC=com'
$DAmembers = $objADSI.member | ForEach-Object {[adsi]"LDAP://$_"}
$DAmembers.distinguishedname | foreach {(($_.Split('=')[1]).Substring(0)).TrimEnd(',OU')} | Sort-Object
```

## Host Queries

### Find all hostnames that contain a keyword (Ex. "dc01")
```
$compsA = [adsisearcher]"(&(objectCategory=computer))"
$compsA.PageSize = 1000
$compsB = $compsA.FindAll()
$compsC = $compsB.Properties
$compsC.dnshostname | Where-Object {$_ -like '*dc01*'}
```

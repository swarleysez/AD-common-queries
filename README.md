# adsi-queries
Collection of common ADSI queries for Domain Account enumeration



### Get a formatted list of "Domain Admins" members  
```
$objADSI = [ADSI]'LDAP://CN=Domain Admins,CN=Users,DC=[domain],DC=com'
$DAmembers = $objADSI.member | ForEach-Object {[adsi]"LDAP://$_"}
$DAmembers.distinguishedname | foreach {(($_.Split('=')[1]).Substring(0)).TrimEnd(',OU')} | Sort-Object
```

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

### Get Count of All Active User Accounts with "Password Never Expires" (Using ADSI or Get-ADUser)
`$pwdNExp = Get-ADUser -filter * -properties Name, PasswordNeverExpires | where { $_.passwordNeverExpires -eq "true" } | where {$_.enabled -eq "true"}; $pwdNExp.Count`

"UAC Flags: 2 == Disabled, 66048 == 512 (normal account) + 65536 (Password Never Expires)"
```
$searchUsers = [adsisearcher]"(&(objectCategory=person)(objectClass=user)(userAccountControl:1.2.840.113556.1.4.803:=66048)(!(userAccountControl:1.2.840.113556.1.4.803:=2)))"
$searchUsers.PageSize = 1000
$usersPwdNExp = $searchUsers.FindAll()
$usersPwdNExp.Count
```

### Get Count of All Disabled Accounts (Using ADSI or Get-ADUser)
*Need Get-ADUser syntax

```
$searchUsers = [adsisearcher]"(&(objectCategory=person)(objectClass=user)(userAccountControl:1.2.840.113556.1.4.803:=2))"
$searchUsers.PageSize = 1000
$usersDisabled = $searchUsers.FindAll()
$usersDisabled.Count
```

### Get Count of All Active User Accounts with "Password Not Required" (Using ADSI or Get-ADUser)
`$pwdNReq = Get-ADUser -filter * -Properties * | where { $_.passwordnotrequired -eq "true" } | where {$_.enabled -eq "true"}; $pwdNReq.Count`

```
$searchUsers = [adsisearcher]"(&(objectCategory=person)(objectClass=user)(userAccountControl:1.2.840.113556.1.4.803:=32)(!(userAccountControl:1.2.840.113556.1.4.803:=2)))"
$searchUsers.PageSize = 1000
$usersIntDomTrst = $searchUsers.FindAll()
$usersIntDomTrst.Count
```

### Get Count of Inter Domain Trust Accounts (Using ADSI or Get-ADUser)
*Need Get-ADUser syntax

```
$searchUsers = [adsisearcher]"(&(objectCategory=person)(objectClass=user)(userAccountControl:1.2.840.113556.1.4.803:=2560)(!(userAccountControl:1.2.840.113556.1.4.803:=2)))"
$searchUsers.PageSize = 1000
$usersIntDomTrt = $searchUsers.FindAll()
$usersIntDomTrt.Count
```

### Get Active Accounts with Passwords Older than 1 Year (Using ADSI or Get-ADUser)
*Need Get-ADUser syntax

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

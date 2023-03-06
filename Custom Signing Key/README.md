# Why do we need custom signing keys?

Custom Signing keys becomes an important step when we look at configuring claims mapping policies for applications. Application developers can use optional claims in their Azure AD applications to specify which claims they want in tokens sent to their application.

You can use optional claims to:

  1. Select additional claims to include in tokens for your application.
  2. Change the behavior of certain claims that the Microsoft identity platform returns in tokens.
  3. Add and access custom claims for your application.

While creating custom claims mapping policies for applications, you may run into the following error in your application:

    "AADSTS50146: This application is required to be configured with an application-specific signing key. It is either not configured with one, or the key has expired or is not yet valid."
  
To overcome this error, the general guidance is to modify the app manifest file to set "acceptmappedclaims" attribute to TRUE.

However the documentation also shares another warning as below -

    Warning: Do not set acceptMappedClaims property to true for multi-tenant apps, which can allow malicious actors to create claims-mapping policies for your app.

Hence in case of multi-tenant apps, the guideline is to use a custom signing key.

The following portion of this doc provides detailed guidelines on how to create custom signing keys and then also demonstrates how you could create claim mapping policies for an application and try to fetch an access token without hitting the above specfied error.


## The first step here is to have a certificate with public and private key exported 

The following code can be used to create this:

    Param(
    [Parameter(Mandatory=$true)]
    [string]$fqdn,
    [Parameter(Mandatory=$true)]
    [string]$pwd,
    [Parameter(Mandatory=$true)]
    [string]$location
    ) 

    if (!$PSBoundParameters.ContainsKey('location'))
      {
        $location = "."
      } 

    $cert = New-SelfSignedCertificate -certstorelocation cert:\currentuser\my -DnsName $fqdn
    $pwdSecure = ConvertTo-SecureString -String $pwd -Force -AsPlainText
    $path = 'cert:\currentuser\my\' + $cert.Thumbprint
    $cerFile = $location + "\\" + $fqdn + ".cer"
    $pfxFile = $location + "\\" + $fqdn + ".pfx" 

    Export-PfxCertificate -cert $path -FilePath $pfxFile -Password $pwdSecure
    Export-Certificate -cert $path -FilePath $cerFile
    
Once this has been done, go ahead with the rest of the steps -

Required Information: SP OID, App ID, Tenant ID


### 1. Create JSON payload from certificate:

Pre-reqs: 
Certificate exported with and without private key (.pfx and .cer) 
PFX requires PKCS12 format 

Open PowerShell and set variables -

    $pfxFile = "C:\folder\certificate.pfx" 
    #Pfx file location 

    $cerFile = "C:\folder\certificate.cer" 
    #Cer file location 

    $payload = "C:\folder\payload.txt" 
    #File which payload will be exported to (should not already exist, will be created) 

    $pwd="P@SSw0rd" 
    #Password for pfx file 

    $cert=Get-PfxCertificate -FilePath $pfxFile 
    #Gets certificate details 

    #Create base64 of PRIVATE KEY 
    $pfx_cert = get-content $pfxFile -Encoding Byte 
    $base64pfx = [System.Convert]::ToBase64String($pfx_cert) 

    #Create base64 of PUBLIC KEY 
    $cer_cert = get-content $cerFile -Encoding Byte 
    $base64cer = [System.Convert]::ToBase64String($cer_cert)

    Build JSON payload # getting id for the keyCredential object 
    $guid1 = New-Guid 
    $guid2 = New-Guid 

    Get the custom key identifier from the certificate thumbprint: 
    $hasher = [System.Security.Cryptography.HashAlgorithm]::Create('sha256') 
    $hash = $hasher.ComputeHash([System.Text.Encoding]::UTF8.GetBytes($cert.Thumbprint)) 
    $customKeyIdentifier = [System.Convert]::ToBase64String($hash) 

    Get end date and start date for our keycredentials 
    $endDateTime = ($cert.NotAfter).ToUniversalTime().ToString( "yyyy-MM-ddTHH:mm:ssZ" ) 
    $startDateTime = ($cert.NotBefore).ToUniversalTime().ToString( "yyyy-MM-ddTHH:mm:ssZ" ) 

Building our json payload 

    $object = [ordered]@{ 
        keyCredentials = @( [ordered]@{ 
              customKeyIdentifier = $customKeyIdentifier 
              endDateTime = $endDateTime 
              keyId = $guid1 
              startDateTime = $startDateTime 
              type = "X509CertAndPassword" 
              usage = "Sign" 
              key = $base64pfx 
              displayName = "CN=testcertificate.local" }

              , [ordered]@{ 
                customKeyIdentifier = $customKeyIdentifier 
                endDateTime = $endDateTime 
                keyId = $guid2 
                startDateTime = $startDateTime 
                type = "AsymmetricX509Cert" 
                usage = "Verify" 
                key = $base64cer 
                displayName = "testcertificate.local" 
              } ) 

                passwordCredentials = @( [ordered]@{ 
                  customKeyIdentifier = $customKeyIdentifier 
                  keyId = $guid1 
                  endDateTime = $endDateTime 
                  startDateTime = $startDateTime 
                  secretText = $pwd } ) } 

                $json = $object | ConvertTo-Json -Depth 99 

Export JSON payload to txt file for use with Microsoft Graph 
    
    $json |Out-File $payload
    

### 2. Upload custom signing key via Graph:

Pre reqs: SP OID 

Connect to Graph Explorer as user with appropriate permissions on target SP 
Confirm SPOID returns correct 

GET https://graph.microsoft.com/v1.0/serviceprincipals/SPOID 

Set:
Request Header Key 
Content-type Value Application/json 

Paste payload into body, change action to PATCH Run Query, and check for response No content - 204 


### 3. Check current signing key:

Pre reqs: App ID, Tenant ID

Browse to https://login.microsoftonline.com/{tenantID}/discovery/v2.0/keys?appid={appID} 

This should list multiple (x4) keys 


### 4. Create Claims Mapping Policy and assign to SP:

Pre reqs: SP OID 

  - Open PowerShell 

  - Connect to tenant as user with appropriate permissions 

  - Connect-AzureAD 

**Check if SP currently has policy applied**

      Get-AzureADServicePrincipalPolicy -Id SPOID 

**Create a new Claims Mapping Policy and set to variable**

      $pol = New-AzureADPolicy -Definition @('{"ClaimsMappingPolicy":{"Version":1,"IncludeBasicClaimSet":"true"}}') -DisplayName "SigningKeyPolicy" -Type "ClaimsMappingPolicy" 

**Alternatively, use existing Claims Mapping Policy**

      Get-AzureADPolicy 
      List all policies, locate existing policy Id and use with: 
      $pol = Get-AzureADPolicy -Id xxx 

**Assign the policy to the SP**

      Add-AzureADServicePrincipalPolicy -Id SPOID -RefObjectId $pol.Id 

**Confirm the policy applied correctly**

      Get-AzureADServicePrincipalPolicy -Id SPOID 
      

### 5. Check the new signing key:

Pre reqs: App ID, Tenant ID

Browse to https://login.microsoftonline.com/{tenantID}/discovery/v2.0/keys?appid={appID} 

This should now list a single key 


### 6. Request a token and verify signature:

Pre reqs: App ID, Client Secret

  1. Open Postman 
  2. Use the Client Credential with shared secret flow to request a token with the following key:value pairs 
  
        - Grant_type: client_credentials 
        - Client_id: AppID 
        - Scope: AppID/.default 
        - Client_secret: ClientSecret 

**NOTE:** Scope cannot be MS Graph (for example user.read) as these scopes will ALWAYS be signed by AzureAD 

Copy the received access token and paste into https://jwt.ms to decode.

Check the KID against the URL.








## Why do we need custom signing keys?

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

**Step 1: The first step here is to have a certificate with public and private key exported**

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


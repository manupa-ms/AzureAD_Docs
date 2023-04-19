# Claims Trasnformation 

When a user signs in, Azure AD sends an ID token that contains a set of claims about the user. A claim is simply a piece of information, expressed as a key/value pair. For example, email=bob@contoso.com. Claims have an issuer (in this case, Azure AD), which is the entity that authenticates the user and creates the claims. You trust the claims because you trust the issuer.

During the authentication flow, you might want to modify the claims that you get from the IDP.

Here are some examples of claims transformation:

1. Claims normalization, or making claims consistent across users. This is particularly relevant if you are getting claims from multiple IDPs, which might use different claim types for similar information.
2. Add default claim values for claims that aren't present (for example, assigning a user to a default role). In some cases this can simplify authorization logic.
3. Add custom claim types with application-specific information about the user. For example, you might store some information about the user in a database. You could add a custom claim with this information to the authentication ticket. The claim is stored in a cookie, so you only need to get it from the database once per login session. On the other hand, you also want to avoid creating excessively large cookies, so you need to consider the trade-off between cookie size versus database lookups.

## Special claims transformations
You can use the following special claims transformations functions.

|Function |Description|
|-----|--------|
|ExtractMailPrefix()|Removes the domain suffix from either the email address or the user principal name. This function extracts only the first part of the user name. For example, joe_smith instead of joe_smith@contoso.com.       |
|ToLower()  |Converts the characters of the selected attribute into lowercase characters.     |
|ToUpper()  |Converts the characters of the selected attribute into uppercase characters.     |

### In our example here we are going to look at the approach of creating a claims transformation policy for extrarting only username attribute from userprincipalname and passing it in the token via a claims mapping policy to signify the samAccountName.

SAML applications can leverage the claim transformation feature available in https://portal.azure.com. Screenshot below.

![Alt text](https://media.licdn.com/dms/image/C5612AQFl-3AlYOcTyA/article-inline_image-shrink_1000_1488/0/1615392123283?e=1687392000&v=beta&t=rcl1YsjfPZ02jIgEQ2kcuO8ALQ4W6uZqLXfVlk8AHMA)

Same claim transformation functionality is not available for Open ID/OAuth integrated applications as of now via the Azure portal.

We will have to create a custom claim transformation policy and map it to the application using Azure AD PowerShell commands.

Here is the sample script -

    New-AzureADPolicy -Definition @('{"ClaimsMappingPolicy":{"Version":1,"IncludeBasicClaimSet":"true","ClaimsSchema":[{"Source":"user","ID":"userprincipalname"},{"Source":"transformation","ID":"ExtractPrefix","TransformationId":"ExtractThePrefix","JwtClaimType":"username_prefix"}],"ClaimsTransformations":[{"ID":"ExtractThePrefix","TransformationMethod":"ExtractMailPrefix","InputClaims":[{"ClaimTypeReferenceId":"userprincipalname","TransformationClaimType":"mail"}],"OutputClaims":[{"ClaimTypeReferenceId":"ExtractPrefix","TransformationClaimType":"outputClaim"}]}]}}') -DisplayName "Username Prefix Extraction Claims Mapping Policy" -Type "ClaimsMappingPolicy"

#### Note the following parameters:

**ID:** Use the ID element to reference this transformation entry in the TransformationID Claims Schema entry. This value must be unique for each transformation entry within this policy.

**TransformationMethod:** The TransformationMethod element identifies which operation is performed to generate the data for the claim.


Based on the method chosen, a set of inputs and outputs is expected. Define the inputs and outputs by using the InputClaims, InputParameters and OutputClaims elements.


**InputClaims:** Use an InputClaims element to pass the data from a claim schema entry to a transformation. It has three attributes: ClaimTypeReferenceId, TransformationClaimType and TreatAsMultiValue

  - ClaimTypeReferenceId is joined with ID element of the claim schema entry to find the appropriate input claim.
  - TransformationClaimType is used to give a unique name to this input. This name must match one of the expected inputs for the transformation method.
  - TreatAsMultiValue is a Boolean flag indicating if the transform should be applied to all values or just the first. By default, transformations will only be applied to the first element in a multi value claim, by setting this value to true it ensures it's applied to all. ProxyAddresses and groups are two examples for input claims that you would likely want to treat as a multi value.

**InputParameters:** Use an InputParameters element to pass a constant value to a transformation. It has two attributes: Value and ID.

  - Value is the actual constant value to be passed.
  - ID is used to give a unique name to the input. The name must match one of the expected inputs for the transformation method.

**OutputClaims:** Use an OutputClaims element to hold the data generated by a transformation, and tie it to a claim schema entry. It has two attributes: ClaimTypeReferenceId and TransformationClaimType.

  - ClaimTypeReferenceId is joined with the ID of the claim schema entry to find the appropriate output claim.
  - TransformationClaimType is used to give a unique name to the output. The name must match one of the expected outputs for the transformation method.

This will then help create a custom claim mapping policy for the application and the corresponding token obtained from Azure AD, would have a new claim named 'username_prefix'.







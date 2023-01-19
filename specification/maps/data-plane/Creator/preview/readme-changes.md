## Autorest cli for client generation
```
autorest --tag=package-preview-2022-09
         --csharp
         --csharp-sdks-folder=D:\Dev\github\azure-sdk-for-net\sdk
         --verbose
         --public-clients=true
         --generation1-convenience-client
         --use:@autorest/csharp@3.0.0-beta.20230111.3
```

## Observations and changes

1. Initial autorest build fails mostly due to `securityDefinitions:SharedKey:in` schema property having `query` value.  This is of course required by the actual API but fails the schema validation.  In my particular use case I want to use AAD so only rely on the `ClientId` being supplied as a header in all API calls (for which it was missing in a few).  To bypass the schema validation, this was changed to `header`.  It means no `subscription-key` can be used but allows for me to continue.

2. Added `ClientId` property to parameters of all calls where missing

3. In some cases a POST api can return either a `200` or `202`.  It appears the client generated will not add both switch cases and chooses to add only one.  In the cases this effects, the `202` is the important one as it indicates the LRO has started.  In these cases, the `200` response was removed.  See data.json /mapData POST and other POST calls.

These 3 changes were made in all the json schemas in the commit.

## Other specific file changes

### data.json

4. Generated client creates a method to support starting upload of a stream of data; StartUpload and StartUploadAsync.  Each of these makes 2 calls to RestClient.CreateUploadRequest, one directly as a parameter to DataUploadOperation and indirectly via the initial call to RestClient.UploadAsync.  The 2nd call is a problem as each call directly accesses the stream, however after the first call the stream is closed.  This causes an exception every time.

    I Modified the relevant generated code as follows:

    ```
    var request = RestClient.CreateUploadRequest(dataFormat, uploadContent, description).Request;
    var originalResponse = await RestClient.UploadAsync(dataFormat, uploadContent, description, cancellationToken).ConfigureAwait(false);
    return new DataUploadOperation(_clientDiagnostics, _pipeline, request, originalResponse);
    ```

### dwgconversion.json

5. dwgconversion.json `OutputOntology` type had issue generating code due to duplicate type with different values.  Renamed type to avoid schema issue (not ideal).

6. dwgconversion.json `/conversions/operations/{operationId}` GET had missing `clientId` parameter which is needed for AAD.

7. dwgconversion.json used standard LRO Result type for `200` response when quering LRO so misses out on some data returned from actual API, namely `properties:diagnosticPackageLocation` and the `errors` and `warnings` properties containing important details of why a conversion failed.

### mapconfiguration.json

8. `x-ms-pageable` `itemName` `mapConfiguration` needed renaming to plural `mapConfigurations`.


### wayfind.json

9. `x-ms-enum` values for `avoidFeatures` required object array for schema validation.  Did not like string array.


### routeset.json

10. Changed capital "S" of "Sets" in property names to lowercase as used in other schemas, e.g. dataSetId becomes datasetId.

11. Renamed `facilities` property to `facilityDetails` property as returned by api currently.

12. Parameter definition for `Description` changed `x-ms-parameter-location` from `client` to `method`.

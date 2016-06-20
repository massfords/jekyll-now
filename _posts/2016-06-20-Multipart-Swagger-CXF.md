---
published: false
layout: post
title: Multipart Upload with Swagger and CXF
---
Most of my REST services accept simple JSON payloads and are easy to test via Swagger. However, I found that adding support for a multi-part upload service wasn't entirely obvious so I figured I'd make a note of it here.


## JAX-RS Service Interface

In this example we want to upload a password protected zip file and include a password so the server can unzip and process the contents. 

```java
@ApiOperation("Uploads a password protected zip file")
@POST
@Path("/import")
@Consumes(MediaType.MULTIPART_FORM_DATA)
@ApiImplicitParams({
    @ApiImplicitParam(name="password", value = "password to unlock the zip file", 
        dataType = "String", paramType = "form"),
    @ApiImplicitParam(name="file", value = "file", 
        required = true, dataType = "java.io.File", paramType = "form")})
ImportedFiles importZip(@ApiParam(hidden=true)
                      @Multipart(value = "file") InputStream is,
                      @ApiParam(hidden=true)
                      @Multipart(value = "password", required = false) String password);

```

## Swagger's Implicit Params

Notice that the service interface above uses Swagger's `@ApiImplicitParams`. This is because the Multipart annotation is not a standard part of JAX-RS so the swagger-ui project doesn't know how to render the params from the method signature alone.

I'm using swagger-ui version `2.1.8-M1` which will respects the `@ApiParam` hidden attribute and hides the regular params in favor of the implicit params with the additional metadata.

![Swagger_UI.png]({{site.baseurl}}/assets/Swagger_UI.png)

Notice how the file param is modeled as a file upload and the string is modeled as a simple text field.


## Supporting Multiple Parts with JSON

If you have a service that has multiple parts where some of the parts are modeled as POJO's relying on Jackson's unmarshalling then you need to ensure that the client passes these parts with a Content-Type header of application/json or CXF will fail to route your request.

I ran into this problem recently and opted to change the server to accept a plain string for the part and do the unmarshalling myself in code. This was a compromise since it seemed simpler than navigating through the [config of the Content-Type](https://github.com/danialfarid/ng-file-upload/issues/449) in [ng-file-upload](https://github.com/danialfarid/ng-file-upload).

In the two examples below, the service is identical except for the final param changing from a Java POJO that we expect to get in JSON format to a String where it's the same payload but we do the unmarshalling into our payload within the method implementation (not shown).

The only downside here is that the service interface in Java isn't as clean since the use of String for a JSON encoded payload isn't as explicit or descriptive as the proper type. That said, it's unlikely you'll be invoking these service interfaces via its Java interface because the CXF Client Proxy doesn't work with Multipart (that sounds like a nice project to do).

### Example Before String Workaround
```java
@ApiOperation("Uploads a password protected zip file")
@POST
@Path("/import")
@Consumes(MediaType.MULTIPART_FORM_DATA)
@ApiImplicitParams({
    @ApiImplicitParam(name="patterns", 
        value = "patterns for filtering the files in the zip you want to process", 
        dataType = "com.massfords.Patterns", paramType = "form"),
    @ApiImplicitParam(name="password", value = "password to unlock the zip file", 
        dataType = "String", paramType = "form"),
    @ApiImplicitParam(name="file", value = "file", 
        required = true, dataType = "java.io.File", paramType = "form")})
ImportedFiles importZip(@ApiParam(hidden=true)
                      @Multipart(value = "file") InputStream is,
                      @ApiParam(hidden=true)
                      @Multipart(value = "password", required = false) String password,
                      @ApiParam(hidden=true)
                      @Multipart(value = "patterns", required = true) Patterns patterns);

```

### Example After String Workaround
```java
@ApiOperation("Uploads a password protected zip file")
@POST
@Path("/import")
@Consumes(MediaType.MULTIPART_FORM_DATA)
@ApiImplicitParams({
    @ApiImplicitParam(name="patterns", 
        value = "patterns for filtering the files in the zip you want to process", 
        dataType = "String", paramType = "form"),
    @ApiImplicitParam(name="password", value = "password to unlock the zip file", 
        dataType = "String", paramType = "form"),
    @ApiImplicitParam(name="file", value = "file", 
        required = true, dataType = "java.io.File", paramType = "form")})
ImportedFiles importZip(@ApiParam(hidden=true)
                      @Multipart(value = "file") InputStream is,
                      @ApiParam(hidden=true)
                      @Multipart(value = "password", required = false) String password,
                      @ApiParam(hidden=true)
                      @Multipart(value = "patterns", required = true) String patterns);

```



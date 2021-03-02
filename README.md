To build and Run the solution
```
 docker-compose build
  docker-compose up
```

## To reproduce the issue

Getting the URL remotely returns a 500 error: -

```
PS C:\Users\Sam.Pratt> curl http://localhost:8000/swagger/docs/aggregates/aggregates
curl : The remote server returned an error: (500) Internal Server Error.
At line:1 char:1
+ curl http://localhost:8000/swagger/docs/aggregates/aggregates
+ ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
    + CategoryInfo          : InvalidOperation: (System.Net.HttpWebRequest:HttpWebRequest) [Invoke-WebRequest], WebExc
   eption
    + FullyQualifiedErrorId : WebCmdletWebResponseException,Microsoft.PowerShell.Commands.InvokeWebRequestCommand
```

Fails with the Exception below
```
apigateway_1  |       System.Net.Http.HttpRequestException: Cannot assign requested address (localhost:8000)
apigateway_1  |        ---> System.Net.Sockets.SocketException (99): Cannot assign requested address
```

Connecting to the container and retriving the URL via CURL works: -
```
PS C:\Users\Sam.Pratt> docker ps
CONTAINER ID   IMAGE                            COMMAND                  CREATED         STATUS         PORTS                    NAMES
d869fc5cff74   swaggerforocelotbug_fooservice   "dotnet FooService.d…"   4 minutes ago   Up 4 minutes                            swaggerforocelotbug_fooservice_1
10926258ab46   swaggerforocelotbug_barservice   "dotnet BarService.d…"   4 minutes ago   Up 4 minutes                            swaggerforocelotbug_barservice_1
d7e2700c5ac2   swaggerforocelotbug_apigateway   "dotnet ApiGateway.d…"   7 minutes ago   Up 4 minutes   0.0.0.0:8000->5000/tcp   swaggerforocelotbug_apigateway_1
PS C:\Users\Sam.Pratt> docker exec -it d7e2700c5ac2 /bin/bash
root@d7e2700c5ac2:/app# curl http://localhost:5000/swagger/docs/aggregates/aggregates
{
  "openapi": "3.0.1",
  "info": {
    "title": "Aggregates",
    "version": "aggregates"
  },
  "paths": {
    "/api/foobar": {
      "get": {
        "tags": [
          "bar-foo"
        ],
        "summary": "Aggregation of routes: bar, foo",
        "description": "",
        "responses": {
          "200": {
            "description": "Success",
            "content": {
              "application/json": {
                "schema": {
                  "type": "object",
                  "properties": {
                    "foo": {
                      "type": "string"
                    },
                    "bar": {
                      "type": "string"
                    }
                  },
                  "additionalProperties": false
                }
              }
            }
          }
        }
      }
    }
  },
  "components": { }
}root@d7e2700c5ac2:/app#
```

## Observations

The issue appears to be the port mapping in the docker compose file: -

```
version: "3.9"
services:
  apigateway:
    build: .\ApiGateway\
    ports:
      - "8000:5000"
  fooservice:
    build: .\FooService
  barservice:
    build: .\BarService
    
```

As you can see we're mapping 5000 in the container to 8000 externally. However inside the container Kestral is trying to connect to 
8000 locally

```
apigateway_1  |       Connection id "0HM6TD4HNUVBJ", Request id "0HM6TD4HNUVBJ:00000002": An unhandled exception was thrown by the application.
apigateway_1  |       System.Net.Http.HttpRequestException: Cannot assign requested address (localhost:8000)
```

Modifying the Docker compose file to remove the port mapping (as below) seems to fix the issue as below: -

```
version: "3.9"
services:
  apigateway:
    build: .\ApiGateway\
    ports:
      - "5000:5000"
  fooservice:
    build: .\FooService
  barservice:
    build: .\BarService

```

```
docker-compose build
docker-compose up
```

```
PS C:\Users\Sam.Pratt> curl http://localhost:5000/swagger/docs/aggregates/aggregates


StatusCode        : 200
StatusDescription : OK
Content           : {123, 10, 32, 32...}
RawContent        : HTTP/1.1 200 OK
                    Transfer-Encoding: chunked
                    Date: Tue, 02 Mar 2021 10:59:37 GMT
                    Server: Kestrel

                    {
                      "openapi": "3.0.1",
                      "info": {
                        "title": "Aggregates",
                        "version": "aggregates"
                      },
                      "...
Headers           : {[Transfer-Encoding, chunked], [Date, Tue, 02 Mar 2021 10:59:37 GMT], [Server, Kestrel]}
RawContentLength  : 874
```
## XAVI Sample

This provides a sample that shows how to provide a RESTful facade on top
of a soap service using [Xavi](https://github.com/xtracdev/xavi).

### Building the Sample

<pre>
go get github.com/Sirupsen/logrus
go get github.com/xtracdev/xavi
go build
</pre>

### Service Imposter

To simulate a soap service, we'll use [mountebank](http://www.mbtest.org/)

The imposters.json file contains a simple definition to simulate the stock
quote services example from the [SOAP 1.1](http://www.w3.org/TR/2000/NOTE-SOAP-20000508/) spec:

```xml
{
  "port": 4545,
  "protocol": "http",
  "stubs": [
    {
      "responses": [
        {
          "is": {
            "statusCode": 200,
            "body": "<SOAP-ENV:Envelope
  xmlns:SOAP-ENV=\"http://schemas.xmlsoap.org/soap/envelope/\"
  SOAP-ENV:encodingStyle=\"http://schemas.xmlsoap.org/soap/encoding/\">
   <SOAP-ENV:Body>
       <m:GetLastTradePriceResponse xmlns:m=\"Some-URI\">
           <Price>34.5</Price>
       </m:GetLastTradePriceResponse>
   </SOAP-ENV:Body>
</SOAP-ENV:Envelope>\n"
          }
        }
      ],
      "predicates": [
              {
                "and": [
                  {"equals": {"path": "/services/quote/getquote","method": "POST"}},
                  { "contains": { "body": "Envelope" } }
                ]
              }
            ]
    }
  ]
}
```

Start mountebank, and configure it to expose the above endpoint:

<pre>
curl -i -X POST -H 'Content-Type: application/json' -d@imposter.json http://127.0.0.1:2525/imposters
</pre>

Based on the above, if a request is posted to the /services/quotes/getquote endpoint,
and the body contains 'Envelope', a SOAP response is returned.

What if we wanted a simple RESTful endpoint, where we could obtain a quote
for a symbol via a simple get on resource like /quote/symbol?

Doing this with Xavi is simple - we write a plug for the framework, register
it on startup, and provide some configuration with a route .

### Writing the plugin

For this example, you need to have golang 1.7.x.

First, we grab Xavi from github via go get.

<pre>
go get github.com/xtracdev/xavi
</pre>

Next, we create the plugin by creating a wrapper type, a method to instantiate the wrapper, and
implement the Wrap method.  

The details are in the quote package.

### Registering the Plugin

The plugin will be referenced by name in the Xavi route configuration. To make
the plugin available to the configuration interface, it needs to be
registered when the application is started.

Xavi provides a package that serves as an entry point to applications built using
the toolkit. Applications pass the command line arguments and a function to
register the plugins.

Refer to main.go for details.

### Other Plugins

Also of note is the use of some provided Xavi plugins. This sample shows how the 
recovery plugin can be used and extended via a custom wrapper factory method
and registering the plugin in main.go.

This also shows the use of the timing plugin. The order of the plugins listed on
the command line is significant: plugins are layered in order, with the earlier wrapped
inside the later ones. The command line configuration below intentionally places
the timer plugin outside the panic recovery plugin, so that we don't record timings for
panic'd service calls.

### Xavi configuration

Once the plugin and main function are available, build the application, and
use it to configure the servers, backends, routes, and listener. The below
configuration assumes the sample was built using `go build`.

Note that before you run the commands below you need to set the
value of the XAVI_KVSTORE_URL environment variable, e.g.

<pre>
export XAVI_KVSTORE_URL=file:///`pwd`/config
</pre>

<pre>
./xavisample add-server -address localhost -port 4545 -name quotesvr1
./xavisample add-backend -name quote-backend -servers quotesvr1
./xavisample add-route -name quote-route -backends quote-backend -base-uri /quote/ -plugins Quote,SessionId,Timing,Recovery
./xavisample add-listener -name quote-listener -routes quote-route
</pre>

At this point, the listener can be started.

Boot the listener:

<pre>
./xavisample listen -ln quote-listener -address 0.0.0.0:8080
</pre>

The xavi endpoint is now ready to go.

### Trying it out

First, you can directly access the SOAP endpoint:


```xml
curl -X POST -d '
<SOAP-ENV:Envelope
  xmlns:SOAP-ENV="http://schemas.xmlsoap.org/soap/envelope/"
  SOAP-ENV:encodingStyle="http://schemas.xmlsoap.org/soap/encoding/">
   <SOAP-ENV:Body>
       <m:GetLastTradePrice xmlns:m="Some-URI">
           <symbol>DIS</symbol>
       </m:GetLastTradePrice>
   </SOAP-ENV:Body>
</SOAP-ENV:Envelope>
' localhost:4545/services/quote/getquote
```

This should produce the following response:

```xml
<SOAP-ENV:Envelope  
  xmlns:SOAP-ENV="http://schemas.xmlsoap.org/soap/envelope/"  
  SOAP-ENV:encodingStyle="http://schemas.xmlsoap.org/soap/encoding/
">   
  <SOAP-ENV:Body>
    <m:GetLastTradePriceResponse xmlns:m="Some-URI">           
      <Price>34.5</Price>
    </m:GetLastTradePriceResponse>   
  </SOAP-ENV:Body>
</SOAP-ENV:Envelope>
</pre>
```

Using the RESTful facade is much easier:

<pre>
curl localhost:8080/quote/foo
34.5
</pre>

To generate a panic, use this uri

<pre>
curl localhost:8080/quote/XTRAC
</pre>

### Config With Load Balancing and Healthcheck

Montebank Setup:

<pre>
curl -i -X POST -H 'Content-Type: application/json' -d@imposter.json http://127.0.0.1:2525/imposters
curl -i -X POST -H 'Content-Type: application/json' -d@imposter2.json http://127.0.0.1:2525/imposters
</pre>

This is a modification to the above to add in a health check

<pre>
./xavisample add-server -address localhost -port 4545 -name quotesvr1 -health-check http-get -ping-uri /
./xavisample add-server -address localhost -port 5454 -name quotesvr2 -health-check http-get -ping-uri /
./xavisample add-backend -name quote-backend -servers quotesvr1,quotesvr2
./xavisample add-route -name quote-route -backends quote-backend -base-uri /quote/ -plugins Quote,SessionId,Timing,Recovery
./xavisample add-listener -name quote-listener -routes quote-route
</pre>

Run it:

<pre>
./xavisample listen -ln quote-listener -address 0.0.0.0:8080
</pre>


### Cancellation and Timeout

Backend calls made via Xavi can be made with a context that allows cancellation or timeout.
In this sample, random cancellations and timeouts can be injected by setting the
MAYBE_TIMEOUT and MAYBE_CANCEL environment variables to a non-empty string, e.g.

<pre>
MAYBE_CANCEL=1 MAYBE_TIMEOUT=1 ./xavisample listen -ln quote-listener -address 0.0.0.0:8080
</pre>

The plugin author may insert a cancellable context into the call flow to allow explicit cancelling 
of the backend request, cancellation of the request based on a timeout, or a duration based timeout. 

If the context times out or is cancelled while Xavi is invoking the backend service, Xavi will set
the http response code based on the cancellation type, update the timing context for the backend with
an error status, and return. While plugins can wait on the Done channel associated with the context,
any manipulation of the http response should be done when call control returns to the plugin to avoid
a race condition with Xavi's handling of the timeout/cancellation.



### HTTPs Transport

To run the HTTPs transport sample, you'll need to generate your own signing cert and key. The cert and key are 
both embedded in the mountebank configuration, and the cert file is referenced in the xavisample configuration.

Gophers can build `src/crypto/tls/generate_cert.go`

<pre>
cd $GOROOT
cd src/crypto/tls
go build generate_cert.go
</pre>

The generate the cert and key via:

<pre>
$GOROOT/src/crypto/tls/generate_cert -ca -host `hostname`
</pre>

Set up mb using imposters-https.json (after editing it to use your key and cert):

<pre>
curl -i -X POST -H 'Content-Type: application/json' -d@imposter-https.json http://127.0.0.1:2525/imposters
</pre>
 
Next then configure xavisample thusly:

NOTE: you will have to generate your own key and pem since the hostname on the cert needs to match your
set up. You'll also need to change the hostname for the server to match your hostname and tls key.

<pre>
export XAVI_KVSTORE_URL=file:///`pwd`/config
</pre>
 
<pre>
./xavisample add-server -address `hostname` -port 4443 -name quotesvr1
./xavisample add-backend -name quote-backend -servers quotesvr1 -cacert-path ./cert.pem -tls-only=true
./xavisample add-route -name quote-route -backends quote-backend -base-uri /quote/ -plugins Quote,SessionId,Timing,Recovery
./xavisample add-listener -name quote-listener -routes quote-route
</pre>

Boot the listener:

<pre>
./xavisample listen -ln quote-listener -address 0.0.0.0:8080
</pre>

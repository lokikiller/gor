[![Build Status](https://travis-ci.org/buger/gor.png?branch=master)](https://travis-ci.org/buger/gor)

## About

Gor is a simple http traffic replication tool written in Go.
Its main goal is to replay traffic from production servers to staging and dev environments.


Now you can test your code on real user sessions in an automated and repeatable fashion.
**No more falling down in production!**

Here is basic worlkflow: The listener server catches http traffic and sends it to the replay server or saves to file.The replay server forwards traffic to a given address.


![Diagram](http://i.imgur.com/9mqj2SK.png)


## Examples

### Capture traffic from port
```bash
# Run on servers where you want to catch traffic. You can run it on each `web` machine.
sudo gor --input-raw :80 --output-tcp replay.local:28020

# Replay server (replay.local).
gor --input-tcp replay.local:28020 --output-http http://staging.com
```

### Using 1 Gor instance for both listening and replaying
It's recommended to use separate server for replaying traffic, but if you have enough CPU resources you can use single Gor instance.

```
sudo gor --input-raw :80 --output-http "http://staging.com"
```

## Advanced use

### Rate limiting
Both replay and listener support rate limiting. It can be useful if you want
forward only part of production traffic and not overload your staging
environment. You can specify your desired requests per second using the
"|" operator after the server address:

#### Limiting replay
```
# staging.server will not get more than 10 requests per second
gor --input-tcp :28020 --output-http "http://staging.com|10"
```

#### Limiting listener
```
# replay server will not get more than 10 requests per second
# useful for high-load environments
gor --input-raw :80 --output-tcp "replay.local:28020|10"
```

### Forward to multiple addresses

You can forward traffic to multiple endpoints. Just add multiple --output-* arguments.
```
gor --input-tcp :28020 --output-http "http://staging.com"  --output-http "http://dev.com"
```

#### Splitting traffic
By default it will send same traffic to all outputs, but you have options to equally split it:

```
gor --input-tcp :28020 --output-http "http://staging.com"  --output-http "http://dev.com" --split-output true
```

### Saving requests to file
You can save requests to file, and replay them later:
```
# write to file
gor --input-raw :80 --output-file requests.gor

# read from file
gor --input-file requests.gor --output-http "http://staging.com"
```

**Note:** Replay will preserve the original time differences between requests.

### Injecting headers

Additional headers can be injected/overwritten into requests during replay. This may be useful if the hostname that staging responds to differs from production, you need to identify requests generated by Gor, or enable feature flagged functionality in an application:

```
gor --input-raw :80 --output-http "http://staging.server" \
    --output-http-header "Host: staging.server" \
    --output-http-header "User-Agent: Replayed by Gor" -header " \
    --output-http-header "Enable-Feature-X: true"
```

### Basic Auth

If your development or staging environment is protected by Basic Authentication then those credentials can be injected in during the replay:

```
gor --input-raw :80 --output-http "http://user:pass@staging .com"
```

Note: This will overwrite any Authorization headers in the original request.

## Stats 
### ElasticSearch 
For deep response analyze based on url, cookie, user-agent and etc. you can export response metadata to ElasticSearch. See [ELASTICSEARCH.md](ELASTICSEARCH.md) for more details.

```
gor --input-tcp :80 --output-http "http://staging.com" --output-http-elasticsearch "es_host:api_port/index_name"
```


## Additional help

Feel free to ask question directly by email or by creating github issue.

## Latest releases (including binaries)

https://github.com/buger/gor/releases

## Command line reference
`gor -h` output:
```
  -cpuprofile="": write cpu profile to file
  -memprofile="": write memory profile to this file
  
  -input-dummy=[]: Used for testing outputs. Emits 'Get /' request every 1s

  -input-file=[]: Read requests from file: 
    gor --input-file ./requests.gor --output-http staging.com

  -input-raw=[]: Capture traffic from given port (use RAW sockets and require *sudo* access):
    # Capture traffic from 8080 port
    gor --input-raw :8080 --output-http staging.com

  -input-tcp=[]: Used for internal communication between Gor instances. Example: 
    # Receive requests from other Gor instances on 28020 port, and redirect output to staging
    gor --input-tcp :28020 --output-http staging.com

  -output-dummy=[]: Used for testing inputs. Just prints data coming from inputs.

  -output-file=[]: Write incoming requests to file: 
    gor --input-raw :80 --output-file ./requests.gor

  -output-http=[]: Forwards incoming requests to given http address.
    # Redirect all incoming requests to staging.com address 
    gor --input-raw :80 --output-http http://staging.com

  -output-http-elasticsearch="": Send request and response stats to ElasticSearch:
    gor --input-raw :8080 --output-http staging.com --output-http-elasticsearch 'es_host:api_port/index_name'

  -output-http-header=[]: Inject additional headers to http reqest:
    gor --input-raw :8080 --output-http staging.com --output-http-header 'User-Agent: Gor'
    
  -output-tcp=[]: Used for internal communication between Gor instances. Example: 
    # Listen for requests on 80 port and forward them to other Gor instance on 28020 port
    gor --input-raw :80 --output-tcp replay.local:28020
  -split-output=false: By default each output gets same traffic. If set to `true` it splits traffic equally among all outputs.
```

## Building from source
1. Setup standard Go environment http://golang.org/doc/code.html and ensure that $GOPATH environment variable properly set.
2. `go get github.com/buger/gor`.
3. `cd $GOPATH/src/github.com/buger/gor`
4. `go build ./bin/gor.go` to get binary, or `go run ./bin/gor.go` to build and run (useful for development)

## FAQ

### What OS are supported?
For now only Linux based. *BSD (including MacOS is not supported yet, check https://github.com/buger/gor/issues/22 for details)

### Why does the `--input-raw` requires sudo or root access?
Listener works by sniffing traffic from a given port. It's accessible
only by using sudo or root access.

### I'm getting 'too many open files' error
Typical linux shell has a small open files soft limit at 1024. You can easily raise that when you do this before starting your gor replay process:
  
  ulimit -n 64000

More about ulimit: http://blog.thecodingmachine.com/content/solving-too-many-open-files-exception-red5-or-any-other-application

## Contributing

1. Fork it
2. Create your feature branch (git checkout -b my-new-feature)
3. Commit your changes (git commit -am 'Added some feature')
4. Push to the branch (git push origin my-new-feature)
5. Create new Pull Request

## Companies using Gor

* http://granify.com
* [GOV.UK](https://www.gov.uk) ([Government Digital Service](http://digital.cabinetoffice.gov.uk/))
* To add your company drop me a line to github.com/buger or leonsbox@gmail.com

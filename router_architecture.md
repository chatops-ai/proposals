Due to DialogFlows single webhook fufillment ability, a router that parses the users responses to the correct action is neccesary. We have chosen Go for the router as it has a great community and collection of libraries surronding the creation of microservices and lightweight, high performace HTTP servers. In particular, the web framework [`gin`](https://github.com/gin-gonic/gin) fufills all requirements:

- Configurable middleware
- High performance routing (using the lightweight [`httprouter`](https://github.com/julienschmidt/httprouter) package
- Defined templating for `HandlerFunc`'s for webhook requests
- A robust `context.Context` that can parse payloads from DialogFlow's service

The router will consist of a handful of HTTP REST endpoints. `/v1/parse` being the most important; as it will handle incoming payloads from DialogFlow. The others:

- `/v1/query` for `key:value` requests from the router's `store` 
- `/v1/healthz` as a healthcheck (to ensure we always have a healthcheck without needing authentication)
- `/v1/yaml` for any `.yaml` files (for example, most plugins will have a `config.yaml` file)

The router can be broken down into the following parts: 

1. The server's, or gin's `http.ListenAndServer`, of which includes the endpoint configuration 
1. Parsing incoming JSON from `/v1/query` and actually routing it to a plugin, or provide a fallback
1. Storing and parsing of `yaml` coming from `/v1/yaml`; correctly mapping settings to their plugins
1. Persisting data using a `store`. In this case, defining a `store` interface would be a good start, allowing drop in backends.

So, it's a lot. The centerpiece of the whole idea - which is why the design and implementation details are of such importance. Each of these parts is further broken down below:

**WIP**

## Plugin Architecture

In order to allow room for growth and development with trained chatbots, we should provide a defined solution for creating `HttpFunc` handlers that can fufill any request sent to it from the router. The router also must be notified of the plugins existence, and configured to connect with its service. 

For configuration, I mirror my proposal of using `/v1/yaml` to ingest `yaml` files and store them in the `store` after processing them. When the code needs to get credentials or variables defined in the `config.yaml` file, it can be retrieved from the `store` and used as a global constant (?). 

The `yaml` must provide, at minimum, the following information for the router:

- Routes -> `HandlerFunc(s)` name, corresponds with API actions
- Attributes -> All entities involved with the routes
- Credentials in the form of upper case strings with the `_` separator
- Author
- Website
- Version

Example:

```Yaml
Version: v1.2.9
Website: chatops.ai
Author: Alec Cunningham
Credentials:
  // These are stored, encrypted (dependent on the `store` backend)
  // as KEY:VALUE pairs for retrieval when the router 
  // initializes its connection with the service
  DATABASE_USER: Alec
  DATABASE_PASS: gunter123!
  SERVICE_TOKEN: fkladfj98032hbui3f8
Routes:
  // We name space via the API version and plugin name
  // to avoid issues with different services + same action
  - /$VERSION/jenkins/start
  - /$VERSION/jenkins/stop
  - /$VERSION/jenkins/build
  - /$VERSION/jenkins/jobs
Attributes:
  // Note, any attribute values you recieve will
  // be from the DialogFlow payload 
  ID: [start, stop, build, jobs]
  BRANCH: [start, stop, jobs]
  JOB_PARAM: [start, jobs]
```

Following the definitions from the `config.yaml`, the router should be a singular file named after its service, like `jenkins.go`. In it it need to, at minimum, fufill this interface:

```Go
type Plugin interface {
  Connect(string)      (Client, error)
  Close(string)
  HealthCheck(string)  *Status
  Trigger(string)      *Action
}
```

Most plugins will inherently have other methods that are similair, specifically in the case of CI providers:

```Go
type BasicCI interface {
  // plugin fufills Plugin interface
  plugin              types.Plugin
  // ByID retrieves an entity by unique ID
  ByID(int)           (Build, error)
  // ByBranch retrieves an entity (Job, Build, Image) corresponding to a branch
  ByBranch(string)    (Build, error)
  // JobHistory returns a list of Build(s) for a given job
  JobHistory(string)  ([]Build, error)
}
```

**WIP**

# Workshop 04: Deploy and Configure MCP Servers

## Creating Our First MCP Server

We'll create our first MCP server using Quarkus. This server will act as an intermediary to provide weather information by consuming it from an external service using a REST Client.

### 1. Create the Quarkus MCP Server Project

Open a terminal, navigate to the main workshop directory and run the following command to generate the project with the necessary extensions:

```bash
quarkus create app dev.langchain4j.quarkus.workshop:weather-mcp-server:1.0-SNAPSHOT \
  -x quarkus-mcp-server-sse \
  -x quarkus-rest-client-jackson
```

This command creates a new Quarkus application called `weather-mcp-server` in version `1.0-SNAPSHOT` and adds two key extensions:

- `quarkus-mcp-server-sse`: to create MCP servers using Server-Sent Events.
- `quarkus-rest-client-jackson`: to make REST calls and deserialize JSON responses.

Once the project is created, it will be ready to add the logic that queries an external weather service and exposes it as an MCP endpoint.

### 2. Create the Weather REST Client

Create the file `src/main/java/dev/langchain4j/quarkus/workshop/WeatherClient.java` with the following content. This will be our REST client to consume the remote weather API:

```java
package dev.langchain4j.quarkus.workshop;

import jakarta.ws.rs.GET;
import jakarta.ws.rs.Path;
import org.eclipse.microprofile.rest.client.inject.RegisterRestClient;
import org.jboss.resteasy.reactive.RestQuery;

@Path("/v1/forecast")
@RegisterRestClient(configKey="weatherclient")
public interface WeatherClient {

    @GET
    String getForecast(
            @RestQuery double latitude,
            @RestQuery double longitude,
            @RestQuery int forecastDays,
            @RestQuery String hourly
    );

}
```

- This REST client will be used to call the external weather API.
- `@RegisterRestClient(configKey="weatherclient")` allows injecting and configuring the client using the `application.properties` file.
- The parameters (`latitude`, `longitude`, etc.) will be included in the URL as query parameters.



Create the file `src/main/java/dev/langchain4j/quarkus/workshop/Weather.java` with the following content:

```java
package dev.langchain4j.quarkus.workshop;

import org.eclipse.microprofile.rest.client.inject.RestClient;

import io.quarkiverse.mcp.server.Tool;
import io.quarkiverse.mcp.server.ToolArg;

public class Weather {

    @RestClient
    WeatherClient weatherClient;

    @Tool(description = "Get weather forecast for a location.")
    String getForecast(
            @ToolArg(description = "Latitude of the location") double latitude,
            @ToolArg(description = "Longitude of the location") double longitude) {

        return weatherClient.getForecast(
                latitude,
                longitude,
                16,
                "temperature_2m,snowfall,rain,precipitation,precipitation_probability");
    }
}
```

This class defines an MCP tool, `getForecast`, that will be accessible via the MCP protocol and can be called remotely by clients. It uses the `WeatherClient` REST client to retrieve weather forecasts for a given latitude and longitude.



## Configure the `application.properties` File

We'll create the `src/main/resources/application.properties` file that contains all the necessary configuration for our MCP (Model Context Protocol) server, as well as for the REST client that connects to the external weather service. This file is essential for Quarkus to properly initialize the services and clients.

Copy and paste the following content into `src/main/resources/application.properties`:

```properties
# Port where the MCP server will run.
# We configure it to 8081 to avoid conflicts if you have other services running on port 8080 (like the client).
quarkus.http.port=8081

# MCP server configuration.
# Assign a descriptive name to your MCP service:
quarkus.mcp.server.server-info.name=Weather Service

# Enable logging of request and response traffic to the MCP service,
# useful for debugging and monitoring what data passes through the server:
quarkus.mcp.server.traffic-logging.enabled=true
quarkus.mcp.server.traffic-logging.text-limit=100  # Limits the amount of text shown in logs for each message

# REST client configuration (WeatherClient).
# This section allows monitoring and customizing external HTTP calls.
# Enable logs for each request and response:
quarkus.rest-client.logging.scope=request-response

# Allows the REST client to automatically follow HTTP redirects:
quarkus.rest-client.follow-redirects=true

# Limits the size of HTTP body logs (useful if responses are very large):
quarkus.rest-client.logging.body-limit=50

# Associates the "weatherclient" identifier to the external weather API endpoint (Open-Meteo):
quarkus.rest-client."weatherclient".uri=https://api.open-meteo.com/
```

### Explanation of each property:

- **quarkus.http.port**: Changes the default port of the Quarkus HTTP server.
- **quarkus.mcp.server.server-info.name**: Custom name that will be exposed as the MCP service identification.
- **quarkus.mcp.server.traffic-logging.enabled**: Enables logging of MCP operations, useful for development and testing.
- **quarkus.mcp.server.traffic-logging.text-limit**: Limits how much content from each message is logged, avoiding very extensive logs.
- **quarkus.rest-client.logging.scope**: Sets the level of detail for logs (request and response).
- **quarkus.rest-client.follow-redirects**: The HTTP client will automatically follow redirects from the target server.
- **quarkus.rest-client.logging.body-limit**: Limits the amount of request/response body data that will be shown in logs.
- **quarkus.rest-client."weatherclient".uri**: Base URI of the Open-Meteo API service that we will consume from our `WeatherClient` client.

> **Important:** The name `"weatherclient"` must match the `configKey` you defined in your REST client interface, so that Quarkus associates this configuration correctly.

With this file properly configured, your services and clients will work transparently following Quarkus and cloud development best practices.

## Add MCP Client Dependency

Quarkus LangChain4j supports MCP with equally minimal work. To use it, we need to add a new MCP client dependency. 

To add the necessary extension for the MCP client in your Quarkus project, run the following command in the root of your main project:

```bash
./mvnw quarkus:add-extension -Dextensions="io.quarkiverse.langchain4j:quarkus-langchain4j-mcp"
```

This will automatically add the dependency to the `pom.xml` file without needing to edit it manually.

This dependency enables your main application to connect to and use MCP servers as clients, allowing you to leverage the MCP tools and capabilities from your Quarkus application.

## Deploy MCP Server to OpenShift

To deploy the MCP Server to OpenShift, we need to add the OpenShift extension and then build and deploy the application.

### 1. Add OpenShift Extension

First, add the OpenShift extension to your MCP Server project. Navigate to the `weather-mcp-server` directory and run:

```bash
./mvnw quarkus:add-extension -Dextensions="io.quarkus:quarkus-openshift"
```

This will add the necessary OpenShift dependencies to your `pom.xml` file.

### 2. Build and Deploy to OpenShift

Once the extension is added, you can build and deploy the application to OpenShift in a single command:

```bash
./mvnw install -Dquarkus.openshift.deploy=true
```

This command will:
- Build the application
- Create the container image
- Deploy it to your OpenShift cluster
- Create the necessary Kubernetes resources (Deployment, Service, Route, etc.)

**Note:** Make sure you are logged into your OpenShift cluster (`oc login`) and have the appropriate permissions to deploy applications in your target namespace.

After deployment, your MCP Server will be accessible via the OpenShift Route that is automatically created. You can check the deployment status with:

```bash
oc get pods -n <your-namespace>
oc get route -n <your-namespace>
```

---


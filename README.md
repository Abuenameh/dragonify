
> Note
> 
> This is an updated README for [casse-boubou/dragonify](https://github.com/casse-boubou/dragonify), a fork of the original [tjhorner/dragonify](https://github.com/tjhorner/dragonify)

Dragonify is a utility for TrueNAS SCALE that enhances inter-app communication by automatically managing Docker networks. It allows containers to communicate with each other via DNS, restoring and extending the networking functionality available in previous TrueNAS versions. This updated version provides granular control over network creation and container connections, improving both flexibility and security.

It's a stop-gap until inter-app networking is properly implemented.

> **Warning**
>
> Dragonify introduces functionality that is unsupported by iXsystems. If you are having problems with your TrueNAS installation or its apps, please try stopping Dragonify and restarting all apps to see if the problem persists.

### How It Works

Dragonify listens to Docker events to automatically manage networks and container connections:

- **Automatic Network Management**: It can create, manage, and delete Docker bridge networks. Networks created by Dragonify are labeled for easy identification and can be automatically removed when they are no longer in use.
- **DNS Aliasing**: When a container is connected to a network, Dragonify assigns it a DNS alias in the format `{service}.{project}.svc.cluster.local`, allowing other containers on the same network to resolve its address by name.
- **Flexible Configuration**: You can use environment variables and container labels to customize how Dragonify behaves, from creating multiple isolated networks to controlling which containers get connected.

### Configuration

You can control Dragonify's behavior using a combination of environment variables and Docker labels on your application containers.

#### Environment Variables

These variables are set on the `dragonify` container itself.

- `LOG_LEVEL`
    - **Description**: Sets the verbosity of the application's logs.
    - **Values**: `info` (default), `debug`.
    - **Example**: `LOG_LEVEL: debug`
- `CONNECT_ALL`
    - **Description**: Controls whether all TrueNAS-managed `ix-` apps should be automatically connected to the default `apps-internal` network. By default, this is enabled to maintain backward compatibility.
    - **Values**: `true` (default), `false`.
    - **Example**: `CONNECT_ALL: "false"`
- `CUSTOMS_NETWORKS`
    - **Description**: A comma-separated list of Docker networks that Dragonify should create on startup. This is useful for pre-defining networks you plan to use across multiple applications.
    - **Values**: A string of network names, e.g., `media-net,home-automation-net`.
    - **Example**: `CUSTOMS_NETWORKS: apps-internal-custom,app-external`

#### Container Label

To connect an application container to one or more specific networks, you use a Docker label.

- `tj.horner.dragonify.networks`
    - **Description**: A comma-separated list of networks to which the container should be connected. If a specified network does not exist, Dragonify will create it automatically.
    - **Applies to**: Any application container you want Dragonify to manage.
    - **Example**:

```yaml
  labels:
    - "tj.horner.dragonify.networks=media-net,downloads"
```

### Secure Installation with Docker Socket Proxy

For a more secure setup, it is highly recommended to use a socket proxy instead of giving Dragonify direct access to the Docker daemon. This approach limits Dragonify to only the permissions it absolutely needs, following the principle of least privilege.

This example uses `swiftwave-org/docker-socket-proxy`, which offers granular read/write control for each Docker API endpoint.

#### `docker-compose.yml`

```yaml
services:
  # This container securely proxies the Docker socket
  socket-proxy:
    image: ghcr.io/swiftwave-org/docker-socket-proxy:latest
    container_name: dragonify-socket-proxy
    restart: always
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
    # Granting the exact permissions Dragonify needs
    environment:
      ## Grant READ permissions for listing entities ##
      - CONTAINERS_READ=1
      - NETWORKS_READ=1
      - EVENTS_READ=1
      - NETWORKS_READ=1
      - NETWORKS_WRITE=1

  # The Dragonify container, now without direct socket access
  dragonify:
    image: ghcr.io/casse-boubou/dragonify:main
    container_name: dragonify
    restart: always
    # Dragonify now depends on the proxy being available
    depends_on:
      - socket-proxy
    # Configure Dragonify to use the proxy's HTTP endpoint
    environment:
      # DOCKER_HOST points to the proxy service
      - DOCKER_HOST=tcp://socket-proxy:2375
      # --- User-configurable options ---
      - LOG_LEVEL=info
      - CONNECT_ALL="false"
      # Optionally pre-define custom networks
      # - CUSTOMS_NETWORKS=media-net,utility-net
```

### Example Application Using the Proxy

```yaml
services:
  # Your application container
  my-app:
    image: some-app-image:latest
    # ... other container settings ...
    labels:
      # Tell Dragonify to connect this container to the 'media-net'
      - "tj.horner.dragonify.networks=media-net"
```

## 2. Permissions Deep Dive and Justification

A deeper analysis of the `index.ts` code confirms that the permissions requested in the secure `docker-compose.yml` example are accurate and provide the least privilege necessary.

Here is a function-by-function breakdown of the Docker API calls and the corresponding permissions required:

| Function                            | Required Permissions              |
| ----------------------------------- | --------------------------------- |
| setUpNetwork()                      | NETWORKS_READ=1, NETWORKS_WRITE=1 |
| connectContainerToAppsNetwork()     | NETWORKS_READ=1, NETWORKS_WRITE=1 |
| connectAllContainersToAppsNetwork() | CONTAINERS_READ=1                 |
| connectNewContainerToAppsNetwork()  | CONTAINERS_READ=1                 |
| removeEmptyCreatedNetwork()         | NETWORKS_READ=1, NETWORKS_WRITE=1 |
| getEventStream()                    | EVENTS_READ=1                     |

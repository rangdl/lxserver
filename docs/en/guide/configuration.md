# LX Music Sync Server Architecture and Configuration Guide

LX Music Sync Server adheres to the design principle of **zero configuration and out-of-the-box use**, while providing a highly flexible **multi-level configuration injection engine and environment variable reading mechanism** to meet the needs of advanced deployment scenarios. When the Node.js backend service process starts, the system will follow strict hierarchical rules to merge and rewrite various configuration items.

## Configuration Loading Hierarchy and Execution Priority

LX Music Sync Server has built a unified basic model skeleton (located in `src/defaultConfig.ts`). During the initialization of the service instance, the configuration parser will streamingly distribute parameters to the **service-side underlying environment** (such as network binding, WebDAV settings, etc.) and the **front-end running environment** (injected into the browser execution sandbox `/js/config.js`) respectively.

The loading and merging of configurations follow the priority sequence from high to low below. High-priority options will **hardly override** the corresponding keys of low-priority ones:

1. **Runtime Environment Variables (Environment Variables)**: Has the highest priority. For example, mounting `PORT=9527 DISABLE_TELEMETRY=true npm start` at the execution level. Since its intention is the most clear, the system will prioritize trusting and adopting it.
2. **Explicit Custom Configuration File Path (Custom Config File)**: Specify a specific JSON physical mapping file to be read through the environment variable `CONFIG_PATH=/data/custom/my-config.json`.
3. **Global Default Entry Configuration (Global Config.js)**: Based on the Node.js module resolution mechanism, placed in the `config.js` file in the project's root directory.
4. **System-level Default Constants (Default Consts)**: Default guarantee values declared in `src/defaultConfig.ts`.

---

## Core Configuration Parameter Dictionary

The following list an array of environment variable (ENV) parameters that affect critical service behaviors:

### I. Network Communication and Underlying Service Configuration

This module manages the Node.js listening process and the basic settings of the network stack.

| Environment Variable Mapping Key (ENV) | System Default Value | Data Type | Scope and Applicable Scenarios |
| :-------------------- | :------------ | :------- | :-------------------------------------------------------------------------------------------------------------------------------------------------- |
| `PORT` | `9527` | Integer | **Service listening port**. It is recommended to avoid using other high-frequency ports in the host (such as 80, 443, 3306). |
| `BIND_IP` | `0.0.0.0` | String | **Scope of service binding IP interfaces**. Set to `127.0.0.1` to accept only local Lookback calls; set to `0.0.0.0` means listening to all internal and external available network adapters of the host simultaneously. |
| `PROXY_HEADER` | `x-real-ip` | String | **Reverse proxy remote IP penetration identifier**. When the system runs behind reverse proxies or load balancers such as Nginx, it is used to extract the true client source IP address to ensure accurate traceability of equipment audit logs. |
| `DISABLE_TELEMETRY` | `false` | Boolean | **System telemetry feedback circuit breaker**. Set to `true` will completely block anonymous state probe packets between the system and external nodes, and disable all system-level new version updates or announcement distributions. |

### II. Persistence and Account Sandbox Management Strategy

This module involves monitoring the status of connected clients and isolation specifications at the physical storage level.

| Environment Variable Mapping Key (ENV) | System Default Value | Data Type | Scope and Applicable Scenarios |
| :-------------------- | :--------- | :------- | :------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `FRONTEND_PASSWORD` | `123456` | String | **Control Panel Root-level encrypted access credential**. Used to verify credentials entering `\` (the global scope of the control panel). To prevent unauthorized external network access, it is recommended to re-authorize and change it immediately upon the first setup. |
| `MAX_SNAPSHOT_NUM` | `10` | Integer | **Time snapshot retention threshold setting**. The maximum allowed length of the historical archive snapshot queue retained by the system. Early histories exceeding this queue limit will be cyclically discarded by the underlying timed GC task. |
| `USER_ENABLE_PATH` | `true` | Boolean | **Account-exclusive storage sandbox isolation system (Critical)**. After this state is started, the underlying data system will partition multiple discrete and parallel volumes according to active users in the `/data` directory. Ensure that preference files of different distribution devices and multi-users do not have data unauthorized access. |
| `USER_ENABLE_ROOT` | `false` | Boolean | **Root directory flattening access override parameter**. When `true`, the above multi-user sandbox volume partitioning operation will become invalid, and data reading and writing will directly pierce and write into the system register in a reduced-dimension manner. |

### III. WebDAV Integrated Online Automated Cloud Disaster Recovery Configuration

The underlying periodic polling asynchronous daemon of the service will only be fully awakened if the following environment variable group is authorized (especially the `WEBDAV_URL` link effectively takes effect):

| Environment Variable Mapping Key (ENV) | System Default Value | Data Type | Scope and Applicable Scenarios |
| :------------------- | :--------- | :------- | :------------------------------------------------------------------------------------------------------------- |
| `WEBDAV_URL` | `''` | String | Various complete URIs with standard WebDAV protocol gateway interfaces (including HTTPS declaration), for example: `https://dav.jianguoyun.com/dav/Sync`. |
| `WEBDAV_USERNAME` | `''` | String | Authorization identification name used for WebDAV service node handshake authentication. |
| `WEBDAV_PASSWORD` | `''` | String | Remote WebDAV gateway access key (highly recommended to use an independent application-specific authorization password to reduce secondary leakage risks). |
| `SYNC_INTERVAL` | `60` | Integer | Cold shrinking timed parameters (unit: minutes) that trigger full thermal backup and pull comparison synchronization flow periods. |

> 🔖 **Stateful Resurrection Mechanism**: If the service first undergoes cold start initialization and directory gathering and detects that this group of variables is complete and legal. The Node.js backend service will pull cloud archive points, thereby overwriting the data state of the current instance. This creates technical feasibility for friction-less landing for large-scale clusters, disaster reconstruction, and smooth off-site machine migration.

### IV. Web-side Composite Media Playback Space Protection Logic

| Environment Variable Mapping Key (ENV) | System Default Value | Data Type | Scope and Applicable Scenarios |
| :------------------------ | :--------- | :------- | :------------------------------------------------------------------------------------------------------------------ |
| `ENABLE_WEBPLAYER_AUTH` | `false` | Boolean | Whether to establish a separate entry-blocking defense wall for the derived browser access interface (the application entity rendered under the `/music` path) and refuse direct face-to-face from visitors. |
| `WEBPLAYER_PASSWORD` | `123456` | String | If the upper-level authentication mode takes effect, it is the separate password dictionary for verification. This gives administrators the ability to decouple keys for different levels of the audience layer and the backend control panel. |

### V. (Advanced Feature) Silent Preset Accounts in CLI Environment

With the pre-declaration strategy at the operating system level, users can statically write accounts into the data persistence layer within the server initialization startup sequence without skipping graphical interface configuration:

Based on the prefix regex extraction mechanism: Adopt the naming rule of `LX_USER_<target signature string>=<password string>` to write into environment variables to achieve authorized interception and building file execution.

#### Example of environment variable dispatch startup:

```bash
# Execute this system declaration, and the accompanying script task will land these three entity records into the data system for authorized issuance.
export LX_USER_foo="mypassword123"
export LX_USER_bar="mypassword321"
export LX_USER_hello="12345"
npm run start
```

*(Note: After the successful accompanying system control operation mentioned above, this memory object will be converted into entity data and permanently archived to the mounted `<DATA_PATH>/users.json` file for continuous function verification.)*

---

When using Docker environments to orchestrate services, it is recommended that you directly convert the contents of this configuration file mapping manual into an `environment` array in `docker-compose.yml`, or append `-e [KEY]=[VALUE]` to the container parameter adjustment command to achieve system feature definitions and smooth startup.

# Server API Reference

LX Sync Server provides a variety of RESTful API interfaces for automatically obtaining and controlling sync server data and statuses.

## Overview
To ensure security, some APIs require the use of the global password of the management terminal or the device password existing in the request body itself for signature authentication.
Unless otherwise specified, all interfaces **use JSON as the request body and response body** type (`Content-Type: application/json`).

If you have customized a client management console function based on your own needs, or want to use data in other services and develop plugin programs, please refer to the following specifications.

---

## Console Administrator API Series
The following series of APIs basically need to pass a Header for management rights verification:
- `x-frontend-auth`: The `frontend.password` set in your `defaultConfig.ts` or environment variables (default is `123456`).

### 1. Service Status (/api/status)

Get the overall memory consumption of the synchronization server, device online status, and uptime summary status.

- **URL:** `/api/status`
- **Method:** `GET`
- **Header Auth:** Requires passing `x-frontend-auth: <Admin Password>`

**Successful Response (200 OK):**
```json
{
  "users": 2, // Number of users registered in the system
  "devices": 1, // Number of devices currently connected online via WebSocket synchronization
  "uptime": 12435.5, // Continuous running seconds of the node
  "memory": 45367823 // Current memory occupied by RSS in bytes
}
```

### 2. Account System Overview (/api/users)

Used for reading and writing configurations in the `users.json` file.

#### `GET /api/users`
Display all independent device accounts and passwords currently existing in the system (**Requires Admin Auth header**).

**Successful Response (200 OK):**
```json
[
  { "name": "user1", "password": "123" },
  { "name": "user2", "password": "321" }
]
```

#### `POST /api/users`
Quickly add a device synchronization registered user (**Requires Admin Auth header**).

**Request Body:**
```json
{
  "name": "newuser",
  "password": "newpassword"
}
```

#### `DELETE /api/users`
Revoke some synchronization device accounts and their backup information (**Requires Admin Auth header**).

**Request Body:**
```json
{
  "names": ["newuser"], // Accepts batch username array
  "deleteData": true // Whether to clean up the user-specific backup data and database history files together
}
```

### 3. Data Layer Acquisition (/api/data)

Conduct review and acquisition for some users' data and list information.

#### 3.1 Get corresponding user real-time playlist status `/api/data`
**URL Parameter**: `?user={username}`
**Auth Requirement**: Returns the current user's all source data JSON Array after `x-frontend-auth` verifies the identity.

#### 3.2 Historical Snapshot Node List `/api/data/snapshots`
**URL Parameter**: `?user={username}`
**Description**: Get the user's synchronization playlist historical snapshot information record points.

#### 3.3 Pull Single Snapshot `/api/data/snapshot`
**URL Parameter**: `?user={username}&id={snapshot_id}`
**Description**: Pass the Snapshot ID to request the complete snapshot data at that time point from the server (not for direct restoration and application, only for read-only acquisition).

#### 3.4 Issue Snapshot Restoration `/api/data/restore-snapshot`
**Request Method**: `POST`
**URL Parameter**: `?user={username}`
**Request Body**: `{"id": "snapshot_id"}`
**Description**: Command the server side to actively erase the user list and roll back the data to cover the designated Snapshot node.

---

## User-state API Series (Regular Client Linkage API)

Most of the following APIs serve Web players or other clients with the ability to synchronize business-level accounts under their own names. They no longer require `x-frontend-auth` but instead require:
- `x-user-name`: Synchronization account name
- `x-user-password`: Synchronization account password matches

### Batch Exclude Songs from Designated List (/api/music/user/list/remove)
This is an active single deletion flow interface from the synchronization list.

- **URL:** `/api/music/user/list/remove`
- **Method:** `POST`
- **Required Header:** `x-user-name` & `x-user-password`

**Request Body:**
```json
{
  "listId": "default", // The designated playlist ID you expect to operate under this user's name or "default" (My Favorites)
  "songIds": [ // Song IDs to be excluded
    "kg_xxxx",
    "kw_yyyy"
  ]
}
```

> This operation will cause the server side to issue a synchronization hot update notification. Other clients connected under the same `x-user-name` will also exclude these invalid songs from their own lists due to Sync Update.

---

## Web Player Core Music API Series

Mainly used to provide data support for online players, aggregating major music platforms embedded in the original version. Note: Under secure configuration, it may rely on the persistent interception of Cookie (`lx_player_session`) or exempt signatures based on settings.

### 1. Get Basic Configuration `GET /api/music/config`
Returns the global Web player configuration overview of the current server side, such as whether the `player.enableAuth` firewall protection mechanism is turned on. Any visitor can get it as read-only.

### 2. Authentication Process
Only when the system has enabled `player.enableAuth` will requests be required to hold a Session Cookie to release the following core music requests.
* `POST /api/music/auth`: Verify password (`{"password": "password"}`) and obtain a long-term effective HTTP Only Session Token Cookie.
* `GET /api/music/auth/verify`: Only query whether the Token survives under its own browser currently.
* `POST /api/music/auth/logout`: Actively discard and log out of its own current Session.

### 3. Multi-source Aggregated Music Search `/api/music/search`

Conduct major category retrieval.

- **Method:** `GET`
- **URL Query Parameters:**
  - `name`: `(Required)` Search keyword
  - `source`: Designated source, such as `kw`, `kg`, `tx`, `wy`, etc.
  - `limit`: Number of returns (default 20)
  - `page`: Pagination (default 1)

**Response Content:** Aggregated JSON object based on different sources.

### 4. Get Music Playback Direct Link `/api/music/url`

Pass the `songInfo` data to the target for processing and then generate a media data physical link. This interface supports being taken over and extended by custom source scripts.

- **URL:** `/api/music/url`
- **Method:** `POST`

**Request Body:**
```json
{
  "quality": "128k", // Or 320k, flac
  "songInfo": {
    "source": "kw",
    "songmid": "xxxx"
    // And other information brought out in search results, etc.
  }
}
```

### 5. Get Music Lyric Information `/api/music/lyric`

Same parameters as the playback direct link, the input is completely consistent with the previous one. The `musicSdk` strips the song into native text and returns it as a JSON with dynamic strings based on the timestamp `\n[00:xxx]`.

### 6. Get Various Hot Search Information `/api/music/hotSearch`

Obtain real-time hot search term lists of various platforms. Supports URL parameter `source`.
Supports setting cache (Cache-Control: 300s).

### 7. Get Selected Song Comments `/api/music/comment`

- **URL:** `/api/music/comment`
- **Method:** `POST`

**Request Body:**
```json
{
  "songInfo": { "source": "tx", "songmid": "xxxx" },
  "type": "hot",  // hot to get hot comments, getting new comments when not hot
  "page": 1,
  "limit": 20
}
```

# Ladder PLC Editor

Test on: [LADDER-EDITOR](https://html-preview.github.io/?url=https://github.com/hiperiondev/ladder-editor/blob/main/lader_editor.html)

The Ladder PLC Editor is a web-based application designed for creating, editing, and simulating ladder logic diagrams for Programmable Logic Controller (PLC) programming. It leverages WebSockets for real-time communication with a server, enabling seamless program management, uniform network dimensions, and dynamic visual feedback through cell state updates.

The code was created entirely with AI, so I don't consider it my own creation and therefore release it without any restrictions for the use of all mankind.

### Features
- Grid redimension
- Row autocomplete
- Symbol identification (Blink symbol if click in Symbol Values table)
- Easy change symbols in code (add or modify)
- Save and load in JSON format
- Undo/Redo function
- WebSocket for comunnication with external runtime    
    

### Known issues
- Does not always scale correctly on mobile devices

## WebSocket Integration Overview

The editor connects to a server at `ws://<IP>/ws` when <IP> is from webeditor was loaded, using WebSockets, facilitating real-time data exchange for program persistence and simulation. Key functionalities include:

- **Saving and Loading Programs**: Store and retrieve ladder logic configurations via the server or locally as JSON files.
- **Uniform Network Dimensions**: Ensure all networks maintain consistent sizes based on a server flag.
- **Cell State Visualization**: Highlight active cells in yellow during simulation, reflecting server updates.
- **Connection Status Indicator**: Display the connection and server status using a color-coded indicator (red, yellow, green).

Below, we explore each feature, detailing their implementation and interaction with WebSockets.

## WebSocket Connection and Status Management

The connection status is tracked via the `wsStatus` variable, which can be:

- **Disconnected**: No connection to the server (`wsStatus = "disconnected"`).
- **Connected, Not Running**: Connected, but the server is not simulating (`wsStatus = "connected_not_running"`).
- **Connected, Running**: Connected, and the server is simulating (`wsStatus = "connected_running"`).

The `updateIndicator` function reflects this status in the user interface by changing the color of a circular indicator (`#ws-indicator`):

| Status                  | Indicator Color | Description                              |
|-------------------------|-----------------|------------------------------------------|
| Disconnected            | Red             | No server connection.                    |
| Connected, Not Running  | Yellow          | Connected, server not simulating.        |
| Connected, Running      | Green           | Connected, server actively simulating.   |

### Connection Lifecycle

- **On Open (`ws.onopen`)**: The editor sends a `{ action: "get_flag" }` message to retrieve server configurations, such as the `sameDimensions` flag, and sets `wsStatus` to "connected_not_running".
- **On Message (`ws.onmessage`)**: Processes incoming messages to update flags, load data, or apply cell states.
- **On Close (`ws.onclose`) or Error (`ws.onerror`)**: Sets `wsStatus` to "disconnected", updates the indicator, and clears active cell highlights.

## Saving and Loading Programs

### Saving Programs

The `saveNetworks` function handles program saving, adapting to the connection status:

- **Connected Mode (`wsStatus !== "disconnected"`)**:
  - Collects data from all networks, including `id`, `rows`, `cols`, and `networkData`.
  - Serializes the data into JSON and sends it to the server with `{ action: "save", data: [...] }`.
  - Displays a toast notification: "Saved to server."
- **Disconnected Mode**:
  - Serializes network data into a formatted JSON string.
  - Creates a `Blob` with type `application/json` and generates a downloadable file (`ladder_networks.json`) using a temporary URL.
  - Triggers an automatic download and revokes the URL.

### Loading Programs

The `initiateLoad` function manages program loading:

- **Connected Mode (`wsStatus !== "disconnected"`)**:
  - Sends `{ action: "load" }` to the server, requesting saved network data.
  - The server responds with a message processed in `ws.onmessage` under `data.action === "load_response"`.
  - The response contains an array of network objects, which are validated and used to reconstruct the editor’s state:
    - Clears existing networks and UI elements.
    - Creates new `Network` instances with provided `rows`, `cols`, and `id`.
    - Validates `networkData` using `validateNetworkData` to ensure symbol and data integrity.
    - Renders networks and updates the symbol values table.
    - Saves the state to history for undo/redo functionality.
  - Shows a toast: "Loaded from server."
- **Disconnected Mode**:
  - Triggers a file input (`#loadFile`) to allow users to upload a JSON file.
  - The `loadNetworks` function reads the file using `FileReader`:
    - Parses the JSON content, ensuring it’s an array of network objects.
    - Validates dimensions (`rows`, `cols`) and enforces limits (`MAX_ROWS`, `MAX_COLS`).
    - Reconstructs networks similarly to server loading, with validation via `validateNetworkData`.
    - Updates the UI and history.

### Data Validation

The `validateNetworkData` function ensures loaded data is compatible:

- Checks each cell’s `symbol` against the `symbols` object, defaulting to "NOP" if invalid.
- Converts `bar` to a boolean.
- Validates `data` arrays, ensuring entries match the symbol’s `dataEntries` and use allowed types, defaulting to the first allowed type and value "0" if invalid.

This robust validation prevents errors from malformed data, whether loaded from the server or a file.

## Uniform Network Dimensions

The editor supports uniform network dimensions, controlled by the `sameDimensionsFlag` boolean, received from the server in `ws.onmessage` when `data.flag === "sameDimensions"`.

### Implementation

- **Adding Networks (`addNetwork`)**:
  - If `sameDimensionsFlag` is `true` and networks exist, new networks adopt the `rows` and `cols` of the first network (`networks[0]`).
  - Otherwise, defaults to 8x8 dimensions.
  - Creates a new `Network` instance, renders it, and updates the UI.

- **Resizing Networks (`resizeNetwork`)**:
  - Retrieves new dimensions from input fields (`rows-${id}`, `cols-${id}`).
  - Validates inputs (positive numbers, within `MAX_ROWS` and `MAX_COLS`).
  - If `sameDimensionsFlag` is `true`, applies the new dimensions to all networks using `network.resize(newRows, newCols)`.
  - If `false`, only the specified network is resized.
  - The `resize` method:
    - Checks for symbol truncation if new dimensions are smaller.
    - Creates a new `networkData` array, copying valid cells and marking multi-cell symbols’ additional rows as "occupied".
    - Updates input fields and re-renders the network.

### Behavior

When `sameDimensionsFlag` is enabled, the editor ensures all networks maintain identical dimensions, simplifying program design for applications requiring consistent layouts. This is particularly useful in PLC programming, where uniform network sizes can align with hardware constraints or design standards.

## Visual Feedback via Cell State Updates

The editor provides real-time visual feedback during simulation by highlighting active cells, controlled by server updates.

### Implementation

- **Server Updates**:
  - When the server is running (`data.status === "running"` in `ws.onmessage`), it sends `cell_states`, an array of objects with `networkId`, `row`, `col`, and `state` (0 or 1).
  - The `applyCellStates` function processes these updates:
    - Calls `setAllCellsDefault` to remove the `active` class from all cells, resetting highlights.
    - Iterates through `cell_states`, applying the `active` class to cells where `state === 1`.
    - Selects cells using the selector `#network-${cs.networkId} td[data-row="${cs.row}"][data-col="${cs.col}"]`.

- **Styling**:
  - The CSS rule `td.active { background-color: yellow; }` highlights active cells in yellow, indicating active elements like closed contacts or energized coils.

- **Reset on Status Change**:
  - If the server stops running (`data.status === "not_running"`) or the connection is lost (`ws.onclose`, `ws.onerror`), `setAllCellsDefault` clears all `active` classes, ensuring the UI reflects the simulation’s state.

### User Experience

This feature enhances the simulation experience by visually representing the ladder logic’s execution, allowing users to monitor program behavior in real time. The yellow highlight is intuitive, aligning with common conventions in PLC programming interfaces.

## Additional Interactions

While not directly tied to WebSockets, the editor includes a blinking feature for cell selection:

- Clicking a row in the symbol values table (`#symbolValues`) toggles a `blinking` class on the corresponding cell and row, using animations defined in CSS (`backgroundBlink`, `strokeBlink`, `fillBlink`).
- This is client-side and unrelated to WebSocket updates, serving as a navigation aid rather than a simulation feature.

## WebSocket Event Handlers

The following table summarizes the WebSocket event handlers and their roles:

| Event       | Function                              | Description                                                                 |
|-------------|---------------------------------------|-----------------------------------------------------------------------------|
| `onopen`    | Sends `{ action: "get_flag" }`        | Initiates connection, requests server flags, sets status to "connected_not_running". |
| `onmessage` | Processes flags, loads, or states     | Handles `sameDimensions` flag, `load_response`, and status/cell state updates. |
| `onclose`   | Resets status and UI                  | Sets `wsStatus` to "disconnected", updates indicator, clears cell highlights. |
| `onerror`   | Resets status and UI                  | Same as `onclose`, handling connection errors.                              |

## Key Functions

The following functions are central to WebSocket integration:

| Function                | Description                                                                 |
|-------------------------|-----------------------------------------------------------------------------|
| `saveNetworks`          | Saves network data to the server or locally based on connection status.      |
| `initiateLoad`          | Requests data from the server or triggers local file upload.                 |
| `addNetwork`            | Creates new networks, respecting `sameDimensionsFlag` for uniform sizes.     |
| `resizeNetwork`         | Resizes one or all networks based on `sameDimensionsFlag`.                   |
| `applyCellStates`       | Updates cell highlights based on server-provided state updates.              |
| `validateNetworkData`   | Ensures loaded data is valid, correcting symbols and data entries.           |

# WebSocket Communication in Ladder Editor

## WebSocket Connection Lifecycle

1. **Connection Establishment**:
   - The client establishes a WebSocket connection.
   - Upon connection, the client sends a `{ action: "get_flag" }` message to request the "sameDimensions" flag.
   - The connection status is tracked as "disconnected", "connected_not_running", or "connected_running", with a visual indicator (red, yellow, green).

2. **Message Handling**:
   - The client processes incoming messages to update flags, load networks, or apply cell states during simulation.

3. **Connection Closure**:
   - On connection close or error, the client sets the status to "disconnected", updates the indicator to red, and resets all cells to their default state.

## Client to Server Messages

### 1. Get Flag
Sent when the WebSocket connection opens to request the "sameDimensions" flag, which likely indicates if the server expects networks with consistent dimensions.

**JSON Structure**:
```json
{
  "action": "get_flag"
}
```

### 2. Save Networks
Sent to save the current network configurations to the server. The `data` field contains an array of network objects.

**JSON Structure**:
```json
{
  "action": "save",
  "data": [network objects]
}
```

**Example**:
```json
{
  "action": "save",
  "data": [
    {
      "id": 0,
      "rows": 2,
      "cols": 2,
      "networkData": [
        [
          {
            "symbol": "NO",
            "bar": false,
            "data": [
              {
                "name": "value",
                "type": "I",
                "value": "0.0"
              }
            ]
          },
          {
            "symbol": "CONN",
            "bar": false,
            "data": []
          }
        ],
        [
          {
            "symbol": "NOP",
            "bar": true,
            "data": []
          },
          {
            "symbol": "Q",
            "bar": false,
            "data": [
              {
                "name": "value",
                "type": "Q",
                "value": "0.0"
              }
            ]
          }
        ]
      ]
    }
  ]
}
```

### 3. Load Networks
Sent to request network configurations from the server.

**JSON Structure**:
```json
{
  "action": "load"
}
```

## Server to Client Messages

### 1. Flag Response
Response to the "get_flag" action, providing the value of the "sameDimensions" flag.

**JSON Structure**:
```json
{
  "flag": "sameDimensions",
  "value": boolean
}
```

**Example**:
```json
{
  "flag": "sameDimensions",
  "value": true
}
```

### 2. Load Response
Response to the "load" action, providing an array of network objects. The structure mirrors the save message's `data` field.

**JSON Structure**:
```json
{
  "action": "load_response",
  "data": [network objects]
}
```

**Example**:
```json
{
  "action": "load_response",
  "data": [
    {
      "id": 0,
      "rows": 2,
      "cols": 2,
      "networkData": [
        [
          {
            "symbol": "NO",
            "bar": false,
            "data": [
              {
                "name": "value",
                "type": "I",
                "value": "0.0"
              }
            ]
          },
          {
            "symbol": "CONN",
            "bar": false,
            "data": []
          }
        ],
        [
          {
            "symbol": "NOP",
            "bar": true,
            "data": []
          },
          {
            "symbol": "Q",
            "bar": false,
            "data": [
              {
                "name": "value",
                "type": "Q",
                "value": "0.0"
              }
            ]
          }
        ]
      ]
    }
  ]
}
```

### 3. Status Updates
The server sends status updates to indicate whether the simulation is running or not.

#### Running Status
When the simulation is running, the server sends cell state updates to reflect the active/inactive status of cells.

**JSON Structure**:
```json
{
  "status": "running",
  "cell_states": [
    {
      "networkId": number,
      "row": number,
      "col": number,
      "state": number
    },
    ...
  ]
}
```

**Example**:
```json
{
  "status": "running",
  "cell_states": [
    {
      "networkId": 0,
      "row": 0,
      "col": 0,
      "state": 1
    },
    {
      "networkId": 0,
      "row": 0,
      "col": 1,
      "state": 0
    },
    {
      "networkId": 0,
      "row": 1,
      "col": 0,
      "state": 0
    },
    {
      "networkId": 0,
      "row": 1,
      "col": 1,
      "state": 1
    }
  ]
}
```

#### Not Running Status
When the simulation is not running, the server sends a simple status message.

**JSON Structure**:
```json
{
  "status": "not_running"
}
```

## Data Structures

### Network Object
Represents a single network in the ladder logic editor.

| Field         | Type       | Description                                      |
|---------------|------------|--------------------------------------------------|
| `id`          | Number     | Unique identifier for the network.               |
| `rows`        | Number     | Number of rows (default 8, max 100).             |
| `cols`        | Number     | Number of columns (default 8, max 100).          |
| `networkData` | Array      | 2D array of cell objects representing the grid.  |

Each cell in `networkData` is an object with:

| Field    | Type       | Description                                      |
|----------|------------|--------------------------------------------------|
| `symbol` | String     | Symbol type (e.g., "NOP", "CONN", "NO", "Q").    |
| `bar`    | Boolean    | Indicates presence of a vertical bar.            |
| `data`   | Array      | Array of data entries for the symbol.            |

Each data entry in `data` is an object with:

| Field    | Type       | Description                                      |
|----------|------------|--------------------------------------------------|
| `name`   | String     | Name of the data entry (e.g., "value").          |
| `type`   | String     | Data type (e.g., "I", "Q", "M").                 |
| `value`  | String     | Value of the data entry (e.g., "0.0").           |

### Cell State
Represents the state of a cell during simulation.

| Field       | Type       | Description                                      |
|-------------|------------|--------------------------------------------------|
| `networkId` | Number     | Identifier of the network.                       |
| `row`       | Number     | Row index of the cell.                           |
| `col`       | Number     | Column index of the cell.                        |
| `state`     | Number     | State of the cell (e.g., 1 for active, 0 for inactive). |

## Supported Symbols and Data Types
The editor supports various ladder logic symbols, each with specific data entry requirements. Examples include:

| Symbol | Cells | Data Entries (Name, Allowed Types)         | Pins (Left, Right)       |
|--------|-------|--------------------------------------------|--------------------------|
| `NOP`  | 1     | []                                         | [false, false]           |
| `CONN` | 1     | []                                         | [true, true]             |
| `NO`   | 1     | [value, I,Q,M,D,T,C,IW,K,QW]               | [true, true]             |
| `Q`    | 1     | [value, Q]                                 | [true, true]             |

Supported data types include: NONE, M, Q, I, Cd, Cr, Td, Tr, IW, QW, C, T, D, CSTR, REAL, MS, 10MS, 100MS, SEC, MIN, K.

## Additional Notes
- The client validates network data before sending, ensuring `rows` and `cols` are within limits (max 100) and data entries match allowed types.
- The server controls the simulation state, sending "running" or "not_running" updates without explicit client requests to start/stop.
- The maximum number of networks is 10, and file uploads for loading networks are limited to 10MB.

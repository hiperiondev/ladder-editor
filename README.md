# ladder-editor
Ladder diagram editor in HTML/Javascript

Test on: [LADDER-EDITOR](https://html-preview.github.io/?url=https://github.com/hiperiondev/ladder-editor/blob/main/lader_editor.html)

This is a complete ladder diagram editor tailored to my needs. The code was created entirely with AI, so I don't consider it my own creation and therefore release it without any restrictions for the use of all mankind.

Any contribution will be welcome.

Features:

    Grid redimension
    Row autocomplete
    Symbol identification (Blink symbol if click in Symbol Values table)
    Easy change symbols in code (add or modify)
    Save and load in JSON format
    Undo/Redo function
    WebSocket for comunnication with external runtime    
    

Known issues:

    Does not always scale correctly on mobile devices

--------------------------------
The expected WebSocket message format is a JSON object like:
```json
{
  "status": "running",
  "cell_states": [
    { "networkId": 0, "row": 2, "col": 3, "state": 1 },
    { "networkId": 0, "row": 4, "col": 5, "state": 0 },
    ...
  ]
}
```
or:
```json
{ "status": "not_running" }
```
If the message lacks a status field or has an unrecognized status, it is ignored.

How WebSocket Communicates with the External Process

The WebSocket implementation is designed to be a client that listens for messages from the external process. Here’s how the communication flow works:

    Connection Initiation:
        When the editor loads, it attempts to connect to ws://localhost:8080.
        If the external process (a WebSocket server) is running, the connection is established, triggering the onopen event.
        The editor does not send an initial message; it waits for the server to send data.
        
    Receiving Messages:
        The external process sends JSON messages to the editor via the WebSocket connection.
        Messages indicate:
            The status of the process ("running" or "not_running").
            When running, an optional cell_states array specifying which cells are active.
        The editor processes these messages in the onmessage handler, updating the UI accordingly.
        
    No Outgoing Messages:
        The current implementation is one-way: the editor only receives messages and does not send data to the external process.
        This suggests the external process independently manages the ladder logic program’s execution and state, possibly based on a previously saved configuration (e.g., the JSON file from saveNetworks()).
        
    State Visualization:
        Active cells (state 1) are highlighted yellow in the grid.
        The WebSocket status indicator reflects the connection and process state, providing immediate feedback to the user.
        
    Handling Disconnection:
        If the external process stops or the connection fails, the onclose or onerror handlers reset the UI to a disconnected state, clearing all highlights.

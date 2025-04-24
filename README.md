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

| Action | Method | Endpoint |
| :--- | :--- | :--- |
| Create instance | `POST` | `/instance/create` |
| Get QR code | `GET` | `/instance/connect/{instance}` |
| Get pairing code | `POST` | `/instance/connect/{instance}` |
| Check status | `GET` | `/instance/connectionState/{instance}` |
| Send text | `POST` | `/message/sendText/{instance}` |
| Send poll | `POST` | `/message/sendPoll/{instance}` |
| Disconnect | `DELETE` | `/instance/logout/{instance}` |
| Delete instance | `DELETE` | `/instance/delete/{instance}` |
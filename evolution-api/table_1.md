| Scenario | What Happens |
| :--- | :--- |
| **Both intact** | Auto-reconnects on restart. You don’t even notice. |
| **Volume wiped, DB intact** | Evolution knows the instance exists but lost the session keys. You need to re-scan QR, but you don't have to recreate the instance from scratch. |
| **Both wiped** | Everything’s gone. Full recreate + re-scan QR. |
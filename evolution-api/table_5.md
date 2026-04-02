| Variable | What It Does |
| :--- | :--- |
| `AUTHENTICATION_API_KEY` | API key for authenticating all requests |
| `SERVER_URL` | Public URL of your Evolution instance |
| `WEBHOOK_GLOBAL_URL` | Where Evolution sends webhook events |
| `DATABASE_ENABLED` | Enable external database persistence (`true` or `false`) |
| `DATABASE_PROVIDER` | Database type — `postgresql` or `mongodb` |
| `DATABASE_CONNECTION_URI` | Full connection string (use `?schema=evolution`) |
| `DATABASE_SAVE_DATA_INSTANCE` | The one flag you should set to `true` |
| `CACHE_LOCAL_ENABLED` | In-memory caching — keep it simple |
| `DEL_INSTANCE` | Set to `false` so instances survive disconnects |
| `LOG_LEVEL` | `ERROR`,`WARN` for production, `INFO` for debugging |
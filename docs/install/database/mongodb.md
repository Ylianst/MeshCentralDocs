# This section will go into how to configure MongoDB as a database backend.

Following [the schema](https://github.com/Ylianst/MeshCentral/blob/master/meshcentral-config-schema.json) we make the following changes to our `config.json`.<br>
Some required keys have been omitted to focus on database configuration. Do not remove these from your config.

---

### MeshCentral Cheatsheet:

MongoDB is configured using the MongoDB connection string.

```json
{
  "$schema": "https://raw.githubusercontent.com/Ylianst/MeshCentral/master/meshcentral-config-schema.json",
  "__comment__": "Omitted these keys to focus on the database",
  "settings": {
    "mongoDb": "mongodb://localhost:27017/meshcentral"
  },
  "domains": {
    "": {
      "__comment__": "Omitted these keys to focus on the database",
    }
  },
  "_letsencrypt": {
    "__comment__": "Omitted these keys to focus on the database",
  }
}
```
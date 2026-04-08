[< Home](../README.md) | [Reference](../README.md#reference)

# Protocols

Protocols define the interface contract for component implementations. Each protocol has a unique ID and a JSON spec describing its methods.

## Protocol Table

| Protocol | ID | Methods |
|----------|----|---------|
| Installable | `cc26e497-ac49-4eb5-a99b-08cbf215fb73` | `install(self)` |
| Service | `e1b65387-6305-4883-88e7-115c0020f5fc` | `start(self)`, `stop(self)`, `status(self)` |
| Event Handler | `8795affb-6274-41ea-879d-d31b362c8695` | `on-event(self, event)` |
| Query | `6701cfb9-247a-471e-a038-cd6eb2db0cf0` | `run(self, input)` |

## Specs

### Installable

```json
{
  "name": "Installable",
  "protocol-id": "cc26e497-ac49-4eb5-a99b-08cbf215fb73",
  "spec": {
    "methods": {
      "install": {
        "args": ["self"]
      }
    }
  }
}
```

Called once when a component is first installed into a workspace. Sets up pages, databases, subscriptions, and push connectors.

### Service

```json
{
  "name": "Service",
  "protocol-id": "e1b65387-6305-4883-88e7-115c0020f5fc",
  "spec": {
    "methods": {
      "start": {
        "args": ["self"]
      },
      "stop": {
        "args": ["self"]
      },
      "status": {
        "args": ["self"]
      }
    }
  }
}
```

Long-running service lifecycle. `start` begins processing, `stop` halts it, `status` reports current state.

### Event Handler

```json
{
  "name": "Event Handler",
  "protocol-id": "8795affb-6274-41ea-879d-d31b362c8695",
  "spec": {
    "methods": {
      "on-event": {
        "args": ["self", "event"]
      }
    }
  }
}
```

Receives events from page subscriptions. The `event` argument is a map with event data. Coordinators dispatch to sibling handlers via `run-page`.

### Query

```json
{
  "name": "Query",
  "protocol-id": "6701cfb9-247a-471e-a038-cd6eb2db0cf0",
  "spec": {
    "methods": {
      "run": {
        "args": ["self", "input"]
      }
    }
  }
}
```

Stateless query execution. The `input` argument is a map of query parameters; returns results.

## Usage Examples

### Install a component

```bash
qo run "Component Installs/My App" -p "Installable" -m "install"
```

### Run a query

```bash
qo run "Component Installs/My App/Queries/list-data" -p "Query" -m "run" -a '[{"limit": "10"}]'
```

### Trigger an event handler

```bash
qo run "Component Installs/My App/Events/Coordinator" -p "Event Handler" -m "on-event" -a '[{"type": "refresh"}]'
```

### Start a service

```bash
qo run "Component Installs/My App" -p "Service" -m "start"
```

### Check service status

```bash
qo run "Component Installs/My App" -p "Service" -m "status"
```

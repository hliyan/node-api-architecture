# node-api-architecture

# goals

- rather than using microservices+monrepo, build a modular monolith
- modularity = carefully controlling and minimising imports
- a logic layer of pure functions: no I/O operations, minimal inputs, single purpose, returns output
- only the top-level "event handler" layer can perform I/O operations
- event handler layer eager loads all data required for logic functions into a read-only context, at the beginning of the event
- only the primary outputs are sent/written in the main event handler
- secondary effects are not written in-line, but triggered via event listeners
- centrally declared schema / metadata defintions shared across all modules

# example directory structure

```
/types
  trip
  driver
/generated
  /graphql
    trip
    driver
  /typescript
    trip
    driver
/data
  connect
  query
  insert
  update
  delete
/utils
  validate
  emit
  emitError
  listen
  log
/engine
  /trip
    /events
      onNewTripRequest
      onAcceptTrip
      onCancepTrip
    /logic
      newTrip
      acceptTrip
      cancelTrip
    /mutations
      insertTrip
      updateTrip
    /queries
      findTripById
      findTripsByPassenger
      findTripsByDriver
  /driver
    /events
      onDriverOnline
    /logic
      driverOnline
    /mutations
      updateDriver
    /queries
      findDriverById
      findDriversByLocation
```

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
- CQRS - query (read-only) layer is shared across all modules, but writes/mutations are private to modules and can only be invoked by module event handlers

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
  /driver
    /events
      onDriverOnline
    /logic
      driverOnline
    /mutations
      updateDriver
/queries
  findTripById
  findTripsByPassenger
  findTripsByDriver  
  findDriverById
  findDriversByLocation
```

# type definitions

## /types/trip.js
- rather than defining typescript types, define a generic schema that can be used for all purposes, including, if necessary, generating typescript type definitions, graphql type definitions and validations that cannot be achieved through typescript

```javascript
const trip = {
  label: 'trip',
  desc: 'a trip planned or taken by a vehicle',
  fields: {
    id: {
      label: 'Trip ID',
      type: 'string',
      primary: true,
      desc: 'unique, system generated id for a trip'
    },
    passengerId: {
      label: 'Passenger ID',
      type: 'string',
      refers: 'Passenger',
      required: true,
      desc: 'id of the passenger who requested the trip'
    },
    date: {
      label: 'Trip Date',
      type: 'date',
      required: true,
      desc: 'date and time on which the trip was requested'
    },
    stops: {
      label: 'Stops',
      type: 'array',
      of: 'stop'
    }
  },
  functions: {
    validateStops: (trip, context) => {
      if (trip.stops.length < 2) throw new Error('a trip must have at least a start and a destination');
      if (distance(trip.stops[0], trip.stops[trip.stops.length - 1]) > context.configs.maxTripDistance)
        throw new Error('trip too long');
    }
  }
};

const stop = {
  label: 'Stop',
  description: 'a stop within a trip',
  fields: {
    lat: {},
    long: {}
  }
};
```

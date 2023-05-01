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
/types // type definitions are shared across modules
  trip
  driver
  /generated
    /graphql
      trip
      driver
    /typescript
      trip
      driver
/queries // queries are shared across modules
  findTripById
  findTripsByPassenger
  findTripsByDriver  
  findDriverById
  findDriversByLocation
/engine // an application 'engine' is a collection of modules
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
/utils
  validate
  emit
  emitError
  listen
  log
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

# event handlers

## app/routes/trip.js
```javascript
import {onNewTripRequest} from 'engine/trip/event/onNewTripRequest';
import {sendAsHttp} from 'utils';

express.post('/trips', (req, res) => {
  let e = await onNewTripRequest({
    passengerId: req.passengerId,
    stops: req.stops
  });
  sendAsHttp(res, e);
});
```

## engine/trip/event/onNewTripRequest.js

```javascript

// shared imports
import {context} from 'utils';
import {queries} from 'queries';
import {emit, emitError} from 'utils';

// imports within module - not to be imported from other modules
import {newTrip} from '../logic/newTrip';
import {insertTrip} from '../mutations/insertTrip';

const onNewTripRequest = async ({passengerId, stops}, context) => {
  try {
    // input block - eager load all contextual data needed for the entire lifetime of the event
    context.create();
    context.passengers = { [passengerId]: await query.findPassengerById(passengerId) };
    context.trips = await query.findTripsByPassengerId(passengerId);
    context.date = new Date();

    // processing block - validations, calculations and generation of object(s) to be saved to db
    let newTrip = newTrip({
      passengerId,
      stops
    }, context);
    
    // primary output block (e.g. db save)
    insertTrip(newTrip);
    
    // all other side effects via event listeners (audit trail, notifications, updates to other entities)
    return emit(events.NEW_TRIP, {trip: newTrip}); 
  } catch (error) {
    return emitError(events.NEW_TRIP, {error}); 
  }
};
```

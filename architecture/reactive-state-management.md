# Reactive State Management

This document explains the patterns and best practices we put in place to manage state within our Angular application. Angular refers to new Angular, not our legacy AngularJs projects (referred to as AngularJs).

Angular state management is implemented using classes (State class) that extend the RxoState abstract class. This class provides all the features needed to implement state mutations and accessors for listening for state changes and events emitted during mutation events. RxoState requires that inheriting classes implement a mutate method. The State class defines the logic of handling mutation requests.

## Overview

* The State class defines the mutations available
* Components, services, and other structures can use the `observe` and `observeProperty` methods to listen for changes on the state and update themselves as required based on each new state when notified.
* The `peek` method is available in State classes to synchronously access the current state.
* When a state requires updating, for example the user requested page 2 of a list, the component providing access to change the page calls the `mutate` method of the associated State class to request an update to the state.
* The State class accepts the mutation request and initiates any logic required to perform the mutation.
* Mutations can be as simple as passing a boolean value (i.e changing from light mode to dark mode), or as complicated as accepting an object defining the action for the state to make and the data payload required for that action.
* The scope of a State class can be small or large. It can be scoped to a few components (or even one) that implement a feature, or scoped to the entire application. Generally, most State classes will be tied to a feature. For example, the InProgressListState in Check, which handles the state of the list of in progress checks. It accepts delete, add, list, more, filter, etc.
* A State class to manage the application state, might contain the user, org, jwt, last page location, etc.
* The primary effect of a State class is to remove logic that updates the state of a feature out of components (or other structures) and into itself. The means that there is only one place where changes of state occur, instead of being scattered around various constructs. The means that structures, like components, can become very simple. Components are reduced to only two purposes, presenting the current state of the feature/app data to the user and to accept input from the user, that is later conveyed to update the state of the feature/app.

![image](https://github.com/mitchgibson/articles/assets/6553496/b30d24e5-fbf0-44a0-8b4a-fb5913cc2796)

## Creating a State class

To create a State class:

* Define the data model that your state will hold
* Extend `RxoState`
  ```typescript
  npm i rxo-state
  ```
* Call super() in the constructor and pass in the initial value for the state.
* Define your mutation types
* Implement the mutate method
* Emit events (optional)

### Define the data model

The shape of the data held can be anything from a simple primitive value, like a count integer, to a complex class or json object. Naming convention of the model data type is fairly loose. But I do recommend suffixing it “Model” or “Data”.

Simple model example

Here is an example of a very simple model. This model is just a boolean value. In this case, defining the type is entirely optional, since it’s just assigning the boolean type to a custom symbol.

export type VisibleModel = boolean;

Complex model example

Here is an example of a complex json object model. This model holds information for loading and error states, and the data when the model is in a successful state.

export type WidgetStateModel = {
  loading: boolean;
  error: Error | undefined;
  data: {
    uuid: string;
    name: string;
    description: string;
  }[];
};

Extend the RxoState abstract class

The RxoState class provides functionality to manage the state and provides common interface for all state classes. This maintains a consistent pattern and provides some powerful methods for interacting with the State class.

In this example, Angular’s Injectable decorator is used to make the state available for dependency injection. This is the same pattern as with our services. The important distinctions here are the use of the term "State" in the class name, the extension of RxoState, and the passing of the model definition to RxoState. The passing of WidgetStateModel provides typings when using methods like observe and peek.

You may notice the use of provideIn: ‘root' passed into the Injectable decorator. This is not required, but is most often the case. provideIn: ‘root' makes the class a singleton within Angular, meaning only one instance of the state can exist. It is rare, but there may be cases where you don’t want this set, for example, if the state is scoped to a set of components that may be reused in different parts of the app and you don’t want all instances of the set of components to share the same state.

@Injectable({
  providedIn: 'root',
})
export class WidgetState extends RxoState<WidgetStateModel> {}

Call super()

The RxoState requires that you provide it an initial state. To do this, pass an initial value that conforms to your Model’s type into the super method of your State class’s constructor. The State class’s constructor is also an opportunity for any injections you may need, for example an api service used to fetch data from a server.

export class WidgetState extends RxoState<WidgetStateModel> {
  constructor(private widgetApi: WidgetApiService) {
    super({
      loading: false,
      error: undefined,
      data: [],
    });
  }
}

Define Mutation Types

Before you get to writing the mutate method, you will want to define at least one of your mutation types. If you want to have the ability to add more mutations later, I recommend defining these types as a json object with an action property that defines the action you want to perform to mutate the state.

It is not usually necessary to export mutation types because they will get exposed when using the mutate method. But if you want to expose them for use in other structures, they can be exported.

Mutation Types should be defined in the same file as the State class. There may be cases where it should be defined elsewhere, so this is not a hard-set rule.

This example defines three mutation types, one for each action: 

ListMutation - request the state to load a list of widgets

DeleteMutation - requests the state remove an item with uuid matching provided uuid

AddMutation - request the state to add an item to the list

type ListMutation = {
  action: 'list';
};

type DeleteMutation = {
  action: 'delete';
  uuid: string;
};

type AddMutation = {
  action: 'add';
  item: {
    uuid: string;
    name: string;
    description: string;
  };
};

type Mutation = ListMutation | DeleteMutation | AddMutation;

As a final step, we create the Mutation type that can be of any of our defined Mutation Types. Defining mutations this way, allows us to guard against using an incorrect Mutation definition. For example, you will get type errors if you passed this object to the mutate method because the add action requires an item, not a uuid:

{
  action: 'add',
  uuid: 'abc-123'
}

Implement the mutate method

The mutate method is used to by other objects to request the State class to make a change its state in some way. We pass a Mutation to the mutate method to instruct how we want the state to change. The State class validates the request and mutates its state according to the request. It then notifies any listeners of the change of state by calling the next method.

The state can also emit any extra events needed during this process. Events can be emitted with the emit method and can be listened for with the listen method.

In this example, we would put the logic for each type of mutation in their own handler function. Handler functions accept the mutation request and updates the state as necessary. Inside the handler functions, the next method is called with the updated state that was achieve from the logic in the handler. This updates the internal state data to the new value and notifies any subscribers of the updated state.

public async mutate(mutation:Mutation): Promise<void> {
  switch (mutation.action) {
    case 'list':
      this.list();
      break;
    case 'delete':
      this.remove(mutation);
      break;
    case 'add':
      this.add(mutation);
      break;
  }
}

List handler

private async list(): Promise<void> {
  this.next({
    loading: true,
    error: undefined,
    data: [],
  });
  try {
    const data = await this.widgetApi.list();
    this.next({
      loading: false,
      error: undefined,
      data,
    });
  } catch(err) {
    this.next({
      loading: false,
      error: err,
      data: [],
    });
  }
}

Delete handler

private async remove(mutation: DeleteMutation):  Promise<void>  {
  try {
    await this.widgetApi.delete(mutation.uuid);
    this.next({
      ...this.peek(),
      data: this.peek().data?.filter((item) => item.uuid !== mutation.uuid),
    });
  } catch(err) {
    console.error(err);
  }
}

The use of ...this.peek() is a convenient way to preserve the existing state values. Only the data property is set to a new value.

Add handler

private async add(mutation: AddMutation): Promise<void> {
  try {
    await this.widgetApi.add(mutation.item);
    this.next({
      loading: false,
      error: undefined,
      data: await this.widgetApi.list(),
    });
  } catch(err) {
    this.next({
      ...this.peek(),
      error: err,
    });
  }
}

Emitting Events

The State class can emit events of any type whenever it needs to. When creating an event, it’s good practice to define the event name as a const in the same file as the state.

export const DELETE_SUCCESS = 'delete-success';
export const DELETE_FAIL = 'delete-fail';

For example, we can emit a delete event. This event could get picked up by a global toast service to display a toast informing the user that the item was deleted. We could also add a failed delete event for a failure toast.

private async remove(mutation: DeleteMutation):  Promise<void>  {
  try {
    await this.widgetApi.delete(mutation.uuid);
    this.next({
      ...this.peek(),
      data: this.peek().data?.filter((item) => item.uuid !== mutation.uuid),
    });

    // emit delete success event
    this.emit(DELETE_SUCCESS, mutation.uuid);

  } catch(err) {
    // emit delete fail event
    this.emit(DELETE_SUCCESS, mutation.uuid);
  }
}

The Complete State Class

The complete state class might look like this:

import { Injectable } from '@angular/core';
import { RxoState } from '@itc-fe/core';
import { WidgetApiService } from './widget.api.service';

type ListMutation = {
  action: 'list';
};

type DeleteMutation = {
  action: 'delete';
  uuid: string;
};

type AddMutation = {
  action: 'add';
  item: {
    uuid: string;
    name: string;
    description: string;
  };
};

type Mutation = ListMutation | DeleteMutation | AddMutation;

export type WidgetStateModel = {
  loading: boolean;
  error: Error | undefined;
  data: {
    uuid: string;
    name: string;
    description: string;
  }[];
};

export const DELETE_SUCCESS = 'delete-success';
export const DELETE_FAIL = 'delete-fail';

@Injectable({
  providedIn: 'root',
})
export class WidgetState extends RxoState<WidgetStateModel> {
  constructor(private widgetApi: WidgetApiService) {
    super({
      loading: false,
      error: undefined,
      data: [],
    });
  }

  public async mutate(mutation:Mutation): Promise<void> {
    switch (mutation.action) {
      case 'list':
        this.list();
        break;
      case 'delete':
        this.remove(mutation);
        break;
      case 'add':
        this.add(mutation);
        break;
    }
  }

  private async list(): Promise<void> {
    this.next({
      loading: true,
      error: undefined,
      data: [],
    });
    try {
      const data = await this.widgetApi.list();
      this.next({
        loading: false,
        error: undefined,
        data,
      });
    } catch(err) {
      this.next({
        loading: false,
        error: err,
        data: [],
      });
    }
  }

  private async remove(mutation: DeleteMutation):  Promise<void>  {
    try {
      await this.widgetApi.delete(mutation.uuid);
      this.next({
        ...this.peek(),
        data: this.peek().data?.filter((item) => item.uuid !== mutation.uuid),
      });

      // emit delete success event
      this.emit(DELETE_SUCCESS, mutation.uuid);

    } catch(err) {
      // emit delete fail event
      this.emit(DELETE_SUCCESS, mutation.uuid);
    }
  }

  private async add(mutation: AddMutation): Promise<void> {
    try {
      await this.widgetApi.add(mutation.item);
      this.next({
        loading: false,
        error: undefined,
        data: await this.widgetApi.list(),
      });
    } catch(err) {
      this.next({
        ...this.peek(),
        error: err,
      });
    }
  }
}


Using the State Class

Here are some example of how to use the various features of the state class.

observe - subscribe to changes to the state

observeProperty - subscribe to changes when a specific property on the state changes

listen - subscribe to emitted events

peek - get the current value of the state (synchronously)

Using observe()

The observe method returns an observable that will execute the provided callback whenever the state is updated. The new state value is returned in the callback’s params.

Here is an example of its usage inside an Angular component.

import { Component } from "@angular/core";
import { WidgetState, WidgetStateModel } from "./widget.state";
import { Subscription } from "rxjs";

@Component({
  selector: 'app-widget-list',
  templateUrl: './widget.component.html',
  standalone: true,
  imports: []
})
export class WidgetComponent {
  private subscription: Subscription = 
    this.state.observe().subscribe((newState: WidgetStateModel) => {
      ...
    });
  constructor(private state:WidgetState) {}
}

The subscribe method on the observable returns a Subscription object. The Subscription can be used to unsubscribe from the observable after the component has been destroyed. It is important that subscriptions are unsubscribe from in components and services that are not intended to exist for the lifetime of the application.

If you don’t unsubscribe, every time the same component is create, a new subscription will be created and the old one is still running. So you will get a lot of unnecessary subscription callbacks executed.

This will negatively impact performance and could be a source of unexpected behaviour.

Using observe() with Angular’s “async” pipe

Another convenient way to observe a state’s changes, one that doesn’t require unsubscribing, is the use of Angular’s async pipe. 

The async pipe is used in the Angular template and performs the subscribe and unsubscribe for you. Here is an example.

<div *ngIf=(state.observe() | async).loading">Loading...</div>

In this example, the async pipe is applied to state.observe(). It subscribes for changes and whenever a new value is received, it returns the value. We then target the loading property for the final value of the *ngIf. The effect is, when the state’s loading value is true, “Loading…” will be displayed, otherwise it will not.

Using observeProperty()

The observeProperty method is a convenience method to target a specific property when the state’s model is in object form. It can be used the same way as the observe method, except it takes a string parameter in dot notation that defines the path of the object within the state.

For example, to target the error property:

this.state.observeProperty('error').subscribe((err) => { ... });

This will return changes only when the error property has changed.

You could also observe a specific object in an array on the state:

this.state.observeProperty('data[5].name').subscribe((name:string) => { ... });

This will observe the name property on the data property of state at index 5.

Using listen()

The listen method is used by passing the name of the event you want to listen. It returns an observable that you can subscribe to execute a callback whenever an event occurs:

this.deleteSuccess:Subscription = 
  this.state.listen(DELETE_SUCCESS).subscribe((eventData) => {
    ...
  });

The event data can be nothing or whatever is passed into the emit method when called from the State class when the event emitted.

Remember to unsubscribe from your observable when using in structures that are not expected to live the lifetime of your application. This will prevent unexpected behaviour and performance issues.

Using peek()

The peek method is used to observe the State class’s value at that moment in time (synchronously). 

const curState = this.state.peek();

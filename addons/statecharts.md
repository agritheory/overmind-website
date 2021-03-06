# Statechart

{% hint style="info" %}
Before you dive into statecharts it can be a good idea to explore [**statemachines**](../core/defining-state.md#statemachines). These are lower level and more flexible and can in most situations be exactly what you need.
{% endhint %}

Just like [OPERATORS](../core/going-functional.md) is a declarative abstraction over plain actions, **statecharts** is a declarative abstraction over an Overmind configuration of **state** and **actions**. That means you will define your charts by:

```typescript
const configWithStatechart = statechart(config, chart)
```

There are several benefits to using statecharts:

1. You will have a declarative description of what actions should be available in certain states of the application
2. Less bugs because an invalid action will not be executed if called
3. You will be able to implement and test an interaction flow without building the user interface for it
4. Your state definition is cleaned up as your **isLoading** types of state is no longer needed
5. You have a tool to do “top down” implementation instead of “bottom up”

You can basically think of a statechart as a way of limiting what actions are available to be executed in certain states of the application. This concept is very old and was originally used to design machines where the user was exposed to all points of interaction, all buttons and switches, at any time. Statecharts would help make sure that at certain states certain buttons and switches would not operate.

A simple example of this is a Walkman. When the Walkman is in a **playing** state you should not be able to hit the **eject** button. On the web this might seem unnecessary as points of interaction is dynamic. We simply hide and/or disable buttons. But this is the exact problem. It is fragile. It is fragile because the UI implementation itself is all you depend on to prevent logic from running when it should not. A statechart is a much more resiliant way to ensure what logic can actually run in any given state.

In Overmind we talk about these statechart states as **transition states**.

## Get up and running

Install the separate package:

```text
npm install overmind-statechart
```

## Defining a statechart

Let us imagine that we have a login flow. This login flow has 4 different **transition states**:

1. **LOGIN**. We are at the point where the user inserts a username and password
2. **AUTHENTICATING**. The user has submitted
3. **AUTHENTICATED**. The user has successfully logged in
4. **ERROR**. Something wrong happened

Let us do this properly and design this flow “top down”:

{% tabs %}
{% tab title="overmind/login/index.js" %}
```typescript
import { statechart } from 'overmind-statechart'
import * as actions from './actions'
import { state } from './state'

const config = {
  state,
  actions
}

const loginChart = {
  initial: 'LOGIN',
  states: {
    LOGIN: {
      on: {
        changeUsername: null,
        changePassword: null,
        login: 'AUTHENTICATING'
      }
    },
    AUTHENTICATING: {
      on: {
        resolveUser: 'AUTHENTICATED',
        rejectUser: 'ERROR'
      }
    },
    AUTHENTICATED: {
      on: {
        logout: 'LOGIN'
      }
    },
    ERROR: {
      on: {
        tryAgain: 'LOGIN'
      }
    }
  }
}

export default statechart(config, loginChart)
```
{% endtab %}
{% endtabs %}

As you can see we have defined what transition states our login flow can be in and what actions we want available to us in each transition state. If the action points to **null** it means we stay in the same transition state. If it points to an other transition state, the execution of that action will cause that transition to occur.

Since our initial state is **LOGIN**, a call to actions defined in the other transition states would simply be ignored.

{% hint style="info" %}
You might expect actions to throw an error if they are called, but not allowed to do so. This is not the case with statecharts. During development you will get a warning when this happens, but in production absolutely nothing happens. Hitting a submit button multiple times might be perfectly okay, but after the first submit the chart moves to a new state, preventing any further execution of logic on the following submits.
{% endhint %}

## Transitions

If you are familiar with the concept of statemachines you might ask the question: _“Where are the transitions?”_. In Overmind we use actions to define transitions instead of having explicit transition types. That means you think about statecharts in Overmind as:

```text
TRANSITION STATE -> ACTION -> NEW TRANSITION STATE
```

as opposed to:

```text
TRANSITION STATE -> TRANSITION TYPE -> { NEW TRANSITION STATE, ACTION }
```

This approach has three benefits:

1. It is more explicit in the definition that a transition state configures what actions are available
2. When typing your application the actions already has a typed input, which would not be possible with a generic **transition** action
3. It is simpler concept both in code and for your brain

What to take notice of is that the **action** causing the transition is run before the transition actually happens. That means the action runs in the context of the current transition state and any synchronous calls to another action will obey its rules. If the action does something asynchronous, like doing an HTTP request, the transition will be performed and the asynchronous logic will run in the context of the new transition state.

```typescript
const myTransitionAction = async ({ actions }) => {
  // I am still in the current transition state
  actions.someOtherAction()

  await Promise.resolve()

  // I am in the new transition state
  actions.someOtherAction()
}
```

## Nested statecharts

With a more complicated UI we can create nested statecharts. An example of this would be a workspace UI with different tabs. You only want to allow certain actions when the related tab is active. Let us explore an example:

{% tabs %}
{% tab title="overmind/dashboard/index.js" %}
```typescript
import { statechart } from 'overmind-statechart'
import * as actions from './actions'
import { state } from './state'

const config = {
  state,
  actions
}

const issuesChart = {
  initial: 'LOADING',
  states: {
    LOADING: {
      entry: 'fetchIssues',
      exit: 'abortFetchIssues',
      on: {
        resolveIssues: 'LIST',
        rejectIssues: 'ERROR'
      }
    },
    LIST: {
      on: {
        toggleIssueCompleted: null
      }
    },
    ERROR: {
      on: {
        retry: 'LOADING'
      }
    },
  }
}

const projectsChart = {
  initial: 'LOADING',
  states: {
    LOADING: {
      entry: 'fetchProjects',
      exit: 'abortFetchProjects',
      on: {
        resolveIssues: 'LIST',
        rejectIssues: 'ERROR'
      }
    },
    LIST: {
      on: {
        expandAttendees: null
      }
    },
    ERROR: {
      on: {
        retry: 'LOADING'
      }
    },
  }
}

const dashboardChart = {
  initial: 'ISSUES',
  states: {
    ISSUES: {
      on: {
        openProjects: 'PROJECTS'
      },
      chart: issuesChart
    },
    PROJECTS: {
      on: {
        openIssues: 'ISSUES'
      },
      chart: projectsChart
    }
  }
}

export default statechart(config, dashboardChart)
```
{% endtab %}
{% endtabs %}

What to take notice of in this example is that all chart states has its own **chart** property, which allows them to be nested. The nested charts has access to the same actions and state as the parent chart.

In this example we also took advantage of the **entry** and **exit** hooks of a transition state. These also points to actions. When a transition is made into the transition state, the **entry** will run. This behavior is nested. When an **exit** hook exists and a transition is made away from the transition state, it will also run. This behavior is also nested of course.

## Parallel statecharts

It is also possible to define your charts in a parallel manner. You do this by simply using an object of keys where the key represents an ID of the chart. The **chart** property on a transition state allows the same. Either a single chart or an object of multiple charts where the key represents an ID of the chart.

```typescript
export default statechart(config, {
  issues: issuesChart,
  projects: projectsChart
})
```

## Conditions

In our chart above we let the user log in even though there is no **username** or **password**. That seems a bit silly. In statecharts you can define conditions. These conditions receives the state of the configuration and returns true or false.

{% tabs %}
{% tab title="overmind/login/index.js" %}
```typescript
import { statechart } from 'overmind-statechart'
import * as actions from './actions'
import { state } from './state'

const config = {
  state,
  actions
}

const loginChart = {
  initial: 'LOGIN',
  states: {
    LOGIN: {
      on: {
        changeUsername: null,
        changePassword: null,
        login: {
          target: 'AUTHENTICATING',
          condition: state => Boolean(state.username && state.password)
        }
      }
    },
    ...
  }
}

export default statechart(config, loginChart)
```
{% endtab %}
{% endtabs %}

Now the **login** action can only be executed when there is a username and password inserted, causing a transition to the new transition state.

## State

Our initial state defined for this configuration is:

{% tabs %}
{% tab title="overmind/login/state.js" %}
```typescript
export const state = {
  username: '',
  password: '',
  user: null,
  authenticationError: null
}
```
{% endtab %}
{% endtabs %}

As you can see we have no state indicating that we have received an error, like **hasError**. We do not have **isLoggingIn** either. There is no reason, because we have our transition states. That means the configuration is populated with some additional state by the statechart. It will actually look like this:

```typescript
{
  username: '',
  password: '',
  user: null,
  authenticationError: null,
  states: [['CHART', 'LOGIN']],
  actions: {
    changeUsername: true,
    changePassword: true,
    login: false,
    logout: false,
    tryAgain: false
  }
}
```

The **states** state is the current transition states. It is defined as an array of arrays. This indicates that we can have parallel and nested charts. The **CHART** symbol in the array indicates that you have defined an immediate chart. If you rather defined parallel charts you would define your own ids.

The **actions** state is a derived state. That means it automatically updates based on the current state of the chart. This is helpful for your UI implementation. It can use it to disable buttons etc. to help the user understand when certain actions are possible.

### Identifying states <a id="statecharts-identifying-states"></a>

There is also a third derived state called **matches**. This derived state returns a function that allows you to figure out what state you are in. This is also the API you use in your components to identify the state of your application:[EDIT ON GITHUB](https://github.com/cerebral/overmind/edit/next/packages/overmind-website/examples/guide/statecharts/matches.ts.ts)

```typescript
state.login.matches({
  LOGIN: true
})
```

You can also do more complex matches related to parallel and nested charts:[EDIT ON GITHUB](https://github.com/cerebral/overmind/edit/next/packages/overmind-website/examples/guide/statecharts/matches_multiple.ts.ts)

```typescript
// Nested
const isSearching = state.dashboard.matches({
  LIST: {
    search: {
      SEARCHING: true
    }
  }
})

// Parallel
const isDownloadingAndUploading = state.files.matches({
  download: {
    LOADING: true
  },
  upload: {
    LOADING: true
  }
})

// Complex match
const isOnlyDownloading = state.files.matches({
  download: {
    LOADING: true
  },
  upload: {
    LOADING: false
  }
})
```

### Actions <a id="statecharts-actions"></a>

Our actions are defined something like:

{% tabs %}
{% tab title="overmind/login/actions.js" %}
```typescript
export const changeUsername = ({ state }, username) => {
  state.login.username = username
}

export const changePassword = ({ state }, password) => {
  state.login.password = password
}

export const login = ({ state, actions, effects }) => {
  try {
    const user = await effects.api.login(state.username, state.password)
    actions.login.resolveUser(user)
  } catch (error) {
    actions.login.rejectUser(error)
  }
}

export const resolveUser = ({ state }, user) => {
  state.login.user = user
}

export const rejectUser = ({ state }, error) => {
  state.login.authenticationError = error.message
}

export const logout = ({ effects }) => {
  effects.api.logout()
}

export const tryAgain = () => {}
```
{% endtab %}
{% endtabs %}

What to take notice of here is that with traditional Overmind we would most likely just set the **user** or the **authenticationError** directly in the **login** action. That is not the case with statcharts because our actions are the triggers for transitions. That means whenever we want to deal with transitions we create an action for it, even completely empty actions like **tryAgain**. This simplifies our chart definition and also we avoid having a generic **transition** action that would not be typed in TypeScript.

Now these two charts would operate individually. This is also the case for the **chart** property on the states of a chart.

## Devtools

The Overmind devtools understands statecharts. That means you are able to get an overview of available statecharts and even manipulate them directly in the devtools.

![](../.gitbook/assets/statecharts.png)

You will see what transition states and actions are available, and active, within each of them. You can click any active action to select it and click again to execute, or insert at payload at the top before execution.

## Typescript

To type a statechart you use the **Statechart** type:

{% tabs %}
{% tab title="overmind/someNamespace/index.ts" %}
```typescript
import { Statechart, statechart } from 'overmind-statechart'
import * as actions from './actions'
import { state } from './state'

const config = {
  state,
  actions
}

const someChart: Statechart<typeof config, {
  FOO: void
  BAR: void
}> = {
  initial: 'FOO',
  states: {
    FOO: {},
    BAR: {}
  }
}

export default statechart(config, someChart)
```
{% endtab %}
{% endtabs %}

The **void** type just defines that there are no nested charts. All the states and points of inserting an action name is now typed. Also the **condition** callback is typed. Even the **matches** API is typed correctly.

### Nested chart

{% tabs %}
{% tab title="overmind/someNamespace/index.ts" %}
```typescript
import { Statechart, statechart } from 'overmind-statechart'
import * as actions from './actions'
import { state } from './state'

const config = {
  state,
  actions
}

const someNestedChart: Statechart<typeof config, {
  NESTED_FOO: void
  NESTED_BAR: void
}> = {
  initial: 'NESTED_FOO',
  states: {
    NESTED_FOO: {},
    NESTED_BAR: {}
  }
}

const someChart: Statechart<typeof config, {
  FOO: typeof someNestedChart
  BAR: void
}> = {
  initial: 'FOO',
  states: {
    FOO: {
      chart: someNestedChart
    },
    BAR: {}
  }
}

export default statechart(config, someChart)
```
{% endtab %}
{% endtabs %}

## Summary

The point of statecharts in Overmind is to give you an abstraction over your configuration that ensures the actions can only be run in certain states. Just like operators you can choose where you want to use it. Maybe only one namespace needs a statechart, or maybe you prefer using it on all of them. The devtools has its own visualizer for the charts, which allows you to implement and test them without implementing any UI.w

## API

### statechart

The factory function you use to wrap an Overmind configuration. You add one or multiple charts to the configuration, where the key is the _id_ of the chart.

```typescript
import { Statechart, statechart } from 'overmind/config'
import * as actions from './actions'
import { state } from './state'

const config = {
  actions,
  state
}

const chart: Statechart<typeof config, {}> = {}

export default statechart(config, chart)
```

### initial

Define the initial state of the chart. When a parent chart enters a transition state, any nested chart will move to its initial transition state.

```typescript
import { Statechart, statechart } from 'overmind/config'
import * as actions from './actions'
import { state } from './state'

const config = {
  actions,
  state
}

const chart: Statechart<typeof config, {
  STATE_A: void
}> = {
  initial: 'STATE_A'
}

export default statechart(config, chart)
```

### states

Defines the transition states of the chart. The chart can only be in one of these states at any point in time.

```typescript
import { Statechart, statechart } from 'overmind/config'
import * as actions from './actions'
import { state } from './state'

const config = {
  actions,
  state
}

const chart: Statechart<typeof config, {
  STATE_A: void
  STATE_B: void
}> = {
  initial: 'STATE_A',
  states: {
    STATE_A: {},
    STATE_B: {}
  }
}

export default statechart(config, chart)
```

### entry

When a transition state is entered you can optionally run an action. It also runs if it is the initial state.

```typescript
import { Statechart, statechart } from 'overmind/config'
import * as actions from './actions'
import { state } from './state'

const config = {
  actions,
  state
}

const chart: Statechart<typeof config, {
  STATE_A: void
  STATE_B: void
}> = {
  initial: 'STATE_A',
  states: {
    STATE_A: {
      entry: 'someActionName'
    },
    STATE_B: {}
  }
}

export default statechart(config, chart)
```

{% hint style="info" %}
If you want to transition to a new state using an **entry**, you are free to call the action causing that transition from the **entry** action.
{% endhint %}

### exit

When a transition state is changed, any exit defined in current transition state will be run first. Nested charts in a transition state with an exit defined will run before parents.

```typescript
import { Statechart, statechart } from 'overmind/config'
import * as actions from './actions'
import { state } from './state'

const config = {
  actions,
  state
}

const chart: Statechart<typeof config, {
  STATE_A: void
  STATE_B: void
}> = {
  initial: 'STATE_A',
  states: {
    STATE_A: {
      entry: 'someActionName',
      exit: 'someOtherActionName'
    },
    STATE_B: {}
  }
}

export default statechart(config, chart)
```

### on

Unlike traditional statecharts Overmind uses its actions as transition types. This keeps a cleaner chart definition and when using Typescript the actions will have correct typing related to their payload. The actions defined are the only actions allowed to run. They can optionally lead to a new transition state, even conditionally lead to a new transition state.

```typescript
import { Statechart, statechart } from 'overmind/config'
import * as actions from './actions'
import { state } from './state'

const config = {
  actions,
  state
}

const chart: Statechart<typeof config, {
  STATE_A: void
  STATE_B: void
}> = {
  initial: 'STATE_A',
  states: {
    STATE_A: {
      on: {
        // Allow execution, but stay on this transition state
        someAction: null,

        // Move to new transition state when executed
        someOtherAction: 'STATE_B',

        // Conditionally move to a new transition state
        someConditionalAction: {
          target: 'STATE_B',
          condition: state => state.isTrue
        }
      }
    },
    STATE_B: {}
  }
}

export default statechart(config, chart)
```

### nested

A nested statechart will operate within its parent transition state. The means when the parent transition state is entered or exited any defined **entry** and **exit** actions will be run. When the parent enters its transition state the **initial** state of the child statechart\(s\) will be activated.

```typescript
import { Statechart, statechart } from 'overmind/config'
import * as actions from './actions'
import { state } from './state'

const config = {
  actions,
  state
}

const nestedChart: Statechart<typeof config, {
  FOO: void
  BAR: void
}> = {
  initial: 'FOO',
  states: {
    FOO: {
      on: {
        transitionToBar: 'BAR'
      }
    },
    BAR: {
      on: {
        transitionToFoo: 'FOO'
      }
    }
  }
}

const chart: Statechart<typeof config, {
  STATE_A: typeof nestedChart
  STATE_B: void
}> = {
  initial: 'STATE_A',
  states: {
    STATE_A: {
      on: {
        transitionToStateB: 'STATE_B'
      },
      chart: nestedChart
    },
    STATE_B: {
      on: {
        transitionToStateA: 'STATE_A'
      }
    }
  }
}

export default statechart(config, chart)
```

### parallel

Multiple statecharts will run in parallel. Either for the factory configuration or nested charts. You can add the same chart multiple times behind different ids.

```typescript
import { Statechart, statechart } from 'overmind/config'
import * as actions from './actions'
import { state } from './state'

const config = {
  actions,
  state
}

const chart: Statechart<typeof config, {
  STATE_A: void
  STATE_B: void
}> = {
  initial: 'STATE_A',
  states: {
    STATE_A: {
      on: {
        transitionToStateB: 'STATE_B'
      }
    },
    STATE_B: {
      on: {
        transitionToStateA: 'STATE_A'
      }
    }
  }
}

export default statechart(config, {
  chartA: chart,
  chartB: chart
})
```

{% hint style="info" %}
Also the nested **chart** property of charts can contain parallel charts
{% endhint %}

### matches

The matches API is used in your components to identify what state your charts are in. It is accessed on the **state**.

```typescript
// Given that you have added statecharts to the root configuration
state.matches({
  STATE_A: true
})

// Nested chart
state.matches({
  STATE_A: {
    FOO: true
  }
})

// Parallel
state.matches({
  chartA: {
    STATE_A: true
  },
  chartB: {
    STATE_B: true
  }
})

// Negative check
state.matches({
  chartA: {
    STATE_A: true
  },
  chartB: {
    STATE_B: false
  }
})
```


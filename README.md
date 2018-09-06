# learnreasonreact

1. 
The component template is created through ReasonReact.statelessComponent("TheComponentName"). The string being passed is for debugging purposes (the equivalent of ReactJS' displayName).
	```javascript
  let component = ReasonReact.statelessComponent("Greeting");
  ```

2. 
In reason react instead of creating a component class you create a component as below :
```javascript
	/* still in Greeting.re */
let component = ReasonReact.statelessComponent("Greeting");
let make = (~name, _children) => {
  ...component, /* spread the template's other defaults into here  */
  render: _self => <div> {ReasonReact.string(name)} </div>
};
```
Here you pass name as a labeled prop. You pass _children as the last argument of the make function. The last prop must be children. If you don't use it, simply ignore it by naming it _ or _children. Names starting with underscore don't trigger compiler warnings if they're unused. 

3. 
The make function is what's called by ReasonReact's JSX.
```javascript
ReasonReact.element(Greeting.make(~name="John", [||])) /* the `make` function in the module `Greeting` */
/* equivalent to <Greeting name="John" /> */
```

4. 
Note: do not inline let component into the make function body like the following!
```javascript
let make = _children => {
  ...(ReasonReact.statelessComponent("Greeting")),
  render: self => blabla
}
```
Since make is called at every JSX invocation, you'd be accidentally creating a fresh new component every time.

5. 
The prop name cannot be ref nor key. Those are reserved, just like in ReactJS.

6. 
```javascript
<Foo name="Reason" age={this.props.age} />
```
This is a source of bugs, because this.props.age might be accidentally changed to a nullable number while Foo doesn't expect it to be so, or vice-versa; it might be nullable before, and now it's not and Foo is left with a useless null check somewhere in the render.
```javascript
Instead use optionals 
switch (ageFromProps) {
| None => <Foo name="Reason" />
| Some(nonNullableAge) => <Foo name="Reason" age=nonNullableAge />
}
Or
<Foo name="Reason" age=?ageFromProps />
```

7. 
You might have seen the render: (self) => ... part in make. The concept of JavaScript this doesn't exist in ReasonReact (but can exist in Reason, since it has an optional object system); the this equivalent is called self. It's a record that contains state, handle and send, which we pass around to the lifecycle events, render and a few others, when they need the bag of information. These concepts will be explained later on.

8. 
Reason comes with the JSX syntax! ReasonReact transforms it from an agnostic function call into a ReasonReact-specific call through a macro. To take advantage of ReasonReact JSX, put {"reason": {"react-jsx": 2} in your bsconfig.json (schema here).

9. 
```javascript 
<div foo={bar}> {child1} {child2} </div>
```
Transforms into
```javascript
ReactDOMRe.createElement("div", ~props=ReactDOMRe.props(~foo=bar, ()), [|child1, child2|]);
```
which compiles to the JS code:
```javascript
React.createElement('div', {foo: bar}, child1, child2)
```
Note that ReactDOMRe.createElement is intended for internal use by the JSX transform. For escape-hatch scenarios, use ReasonReact.createDomElement instead, as outlined in the children section.

10. 
```javascript
<MyReasonComponent key={a} ref={b} foo={bar} baz={qux}> {child1} {child2} </MyReasonComponent>
transforms to
ReasonReact.element(
  ~key=a,
  ~ref=b,
  MyReasonComponent.make(~foo=bar, ~baz=qux, [|child1, child2|])
);
```

11. 
```javascript
let theChildren = [| <div />, <div /> |];
<MyReasonComponent> theChildren </MyReasonComponent>
Translates to:
let theChildren = [| <div />, <div /> |];
ReasonReact.element(
  MyReasonComponent.make([|theChildren|])
);

let theChildren = [| <div />, <div /> |];
<MyReasonComponent> ...theChildren </MyReasonComponent>
This simply passes theChildren without array wrapping. It becomes:
let theChildren = [| <div />, <div /> |];
ReasonReact.element(
  MyReasonComponent.make(theChildren)
);
```

12. 
In ReactJS, you can easily do: `<div> hello </div>, <div> {1} </div>, <div> {null} </div>`, etc. In Reason, the type system restricts you from passing arbitrary data like so; you can only return ReasonReact.reactElement from render.
Fortunately, we special-case a few special elements of the type ReasonReact.reactElement:
- ReasonReact.null: This is your null equivalent for render's return value. Akin to return null in ReactJS render.
- ReasonReact.string: Takes a string and converts it to a reactElement. You'd use 
  `<div> {ReasonReact.string(string_of_int(10))} </div>` to display an int.
- ReasonReact.array: Takes an array and converts it to a reactElement.

13. 
If you're just forwarding a callback prop onto your child, you'd do exactly the same thing you'd have done in ReactJS:
```javascript
let component = ...;

let make = (~name, ~onClick, _children) => {
  ...component,
  render: (self) => <button onClick=onClick />
};
```
No surprise here. Since Reason's JSX has punning syntax, that button will format into `<button onClick />`.
Similarly, to pre-process a value before sending it back to the component's owner:
let component = ...;
```javascript
let make = (~name, ~onClick, _children) => {
  let click = (event) => onClick(name); /* pass the name string up to the owner */
  {
    ...component,
    render: (self) => <button onClick=click />
  }
};
```

14. 
To access state, send and the other items in self from a callback, you need to wrap the callback in an extra layer called self.handle:
```javascript
let component = ...;
let make = (~name, ~onClick, _children) => {
  let click = (event, self) => {
    onClick(event);
    Js.log(self.state);
  };
  {
    ...component,
    initialState: ...,
    render: (self) => <button onClick={self.handle(click)} />
  }
};
```
Note how your click callback now takes the extra argument self. Formally, self.handle expects a callback that
- accepts the single payload you'd normally directly pass to e.g. handleClick,
- plus the argument self,
- returns "nothing" (aka, (), aka, unit).

Note 2: sometimes you might be forwarding handle to some helper functions. Pass the whole self instead and annotate it. This avoids a complex self record type behavior. See Record Field send/handle Not Found.
In reality, self.handle is just a regular function accepting two arguments, the first being the callback in question, and the second one being the payload that's intended to be passed to the callback.
Get it? Through Reason's natural language-level currying, we usually only ask you to pass the first argument. This returns a new function that takes in the second argument and executes the function body. The second argument being passed by the caller, aka the component you're rendering!
self.handle(click) return a function => onClick calls this function and passes “event” as the payload => this function internally calls the click function and passes this payload along with “self”

15. 
Sometimes, the component you're calling is from JavaScript (using the ReasonReact<->ReactJS interop), and its callback prop asks you to pass a callback that receives more than one argument. In ReactJS, it'd look like:
```javascript
handleSubmit: function(username, password, event) {
  this.setState(...)
}
...
<MyForm onUserClickedSubmit={this.handleSubmit} />
```
You cannot write such handleSubmit in ReasonReact, as handle expects to wrap around a function that only takes one argument. Here's the workaround:
```javascript
let handleSubmitEscapeHatch = (username, password, event) =>
  self.handle(
    (tupleOfThreeItems, self) => doSomething(tupleOfThreeItems, self),
    (username, password, event),
  );
...
<MyForm onUserClickedSubmit=(handleSubmitEscapeHatch) />
```
Basically, you write a normal callback that:
- takes those many arguments from the JS component callback prop,
- packs them into a tuple and call self.handle,
- pass to handle the usual function that expects a single argument,
- finish calling self.handle by passing the tuple directly yourself.

16. 
You can't update state in self.handle; you need to use self.send instead. See the next section.
To declare a stateful ReasonReact component, instead of ReasonReact.statelessComponent("MyComponentName"), use ReasonReact.reducerComponent("MyComponentName").

```javascript
/* State declaration */
type state = {
  count: int,
  show: bool,
};

/* Action declaration */
type action =
  | Click
  | Toggle;

/* Component template declaration.
   Needs to be **after** state and action declarations! */
let component = ReasonReact.reducerComponent("Example");

/* greeting and children are props. `children` isn't used, therefore ignored.
   We ignore it by prepending it with an underscore */
let make = (~greeting, _children) => {
  /* spread the other default fields of component here and override a few */
  ...component,

  initialState: () => {count: 0, show: true},

  /* State transitions */
  reducer: (action, state) =>
    switch (action) {
    | Click => ReasonReact.Update({...state, count: state.count + 1})
    | Toggle => ReasonReact.Update({...state, show: ! state.show})
    },

  render: self => {
    let message =
      "You've clicked this " ++ string_of_int(self.state.count) ++ " times(s)";
    <div>
      <button onClick=(_event => self.send(Click))>
        (ReasonReact.string(message))
      </button>
      <button onClick=(_event => self.send(Toggle))>
        (ReasonReact.string("Toggle greeting"))
      </button>
      (
        self.state.show ?
          ReasonReact.string(greeting) : ReasonReact.null
      )
    </div>;
  },
};
```

17. 
ReactJS' getInitialState is called initialState in ReasonReact. It takes unit and returns the state type. The state type could be anything! An int, a string, a ref or the common record type, which you should declare right before the reducerComponent call:
```javascript
type state = {count: int, show: bool};

let component = ReasonReact.reducerComponent("Example");

let make = (~onClick, _children) => {
  ...component,
  initialState: () => {count: 0, show: true},
  /* ... other fields */
};
```
Since the props are just the arguments on make, feel free to read into them to initialize your state based on them.


18. 
In ReasonReact, you'd gather all these state-setting handlers into a single place, the component's reducer! Please refer to the first snippet of code on this page.
Note: if you ever see mentions of self.reduce, this is the old API. The new API is called self.send. The old API's docs are here.
A few things:
- There's a user-defined type called action, named so by convention. It's a variant of all the possible state transitions in your component. In state machine terminology, this'd be a "token".
- A "reducer"! This pattern-matches on the possible actions and specify what state update each action corresponds to. In state machine terminology, this'd be a "state transition".
- In render, instead of self.handle (which doesn't allow state updates), you'd use self.send. send takes an action.

19. 
Notice the return value of reducer? The ReasonReact.Update part. Instead of returning a bare new state, we ask you to return the state wrapped in this "update" variant. Here are its possible values:
- ReasonReact.NoUpdate: don't do a state update.
- ReasonReact.Update state: update the state.
- ReasonReact.SideEffects(self => unit): no state update, but trigger a side-effect, e.g. 
  `ReasonReact.SideEffects(_self => Js.log("hello!"))`.
- ReasonReact.UpdateWithSideEffects(state, self => unit): update the state, then trigger a side-effect.

20. 
Please read through all these points, if you want to fully take advantage of reducer and avoid future ReactJS Fiber race condition problems.
- The action type's variants can carry a payload: onClick={data => self.send(Click(data.foo))}.
- Don't pass the whole event into the action variant's payload. ReactJS events are pooled; by the time you intercept the action in the reducer, the event's already recycled.
- reducer must be pure! Aka don't do side-effects in them directly. You'll thank us when we enable the upcoming concurrent React (Fiber). Use SideEffects or UpdateWithSideEffects to enqueue a side-effect. The side-effect (the callback) will be executed after the state setting, but before the next render.
- If you need to do e.g. ReactEventRe.BlablaEvent.preventDefault(event), do it in self.send, before returning the action type. Again, reducer must be pure.
- Feel free to trigger another action in SideEffects and UpdateWithSideEffects, e.g. UpdateWithSideEffects(newState, (self) => self.send(Click)).
- If your state only holds instance variables, it also means (by the convention in the instance variables section) that your component only contains self.handle, no self.send. You still needs to specify a reducer like so: reducer: ((), _state) => ReasonReact.NoUpdate. Otherwise you'll get a variable cannot be generalized type error.
Cram as much as possible into reducer. Keep your actual callback handlers (the self.send(Foo) part) dumb and small. This makes all your state updates & side-effects (which itself should mostly only be inside ReasonReact.SideEffects and ReasonReact.UpdateWithSideEffects) much easier to scan through. Also more ReactJS fiber async-mode resilient.

21. 
ReasonReact supports the familiar ReactJS lifecycle events.
```javascript
didMount: self => unit

willReceiveProps: self => state

shouldUpdate: oldAndNewSelf => bool

willUpdate: oldAndNewSelf => unit

didUpdate: oldAndNewSelf => unit

willUnmount: self => unit
```
Note:
- We've dropped the component prefix from all these.
- willReceiveProps asks, for the return type, to be state, not update state (i.e. not NoUpdate/Update/SideEffects/UpdateWithSideEffects). We presume you'd always want to update the state in this lifecycle. If not, simply return the previous state exposed in the lifecycle argument.
- didUpdate, willUnmount and willUpdate don't allow you to return a new state to be updated, to prevent infinite loops.
- willMount is unsupported. Use didMount instead.
- didUpdate, willUpdate and shouldUpdate take in a oldAndNewSelf record, of type {oldSelf: self, newSelf: self}. These two fields are the equivalent of ReactJS' componentDidUpdate's prevProps/prevState/ in conjunction with props/state. Likewise for willUpdate and shouldUpdate.
If you need to update state in a lifecycle event, simply send an action to reducer and handle it correspondingly: self.send(DidMountUpdate).

22. 
One pattern that's sometimes used in ReactJS is accessing a lifecyle event's `prevProps(componentDidUpdate)`, 
`nextProps (componentWillUpdate)`, and so on. ReasonReact doesn't automatically keep copies of previous props for you. We provide the retainedProps API for this purpose:
```javascript
type retainedProps = {message: string};

let component = ReasonReact.statelessComponentWithRetainedProps("RetainedPropsExample");

let make = (~message, _children) => {
  ...component,
  retainedProps: {message: message},
  didUpdate: ({oldSelf, newSelf}) =>
    if (oldSelf.retainedProps.message !== newSelf.retainedProps.message) {
      /* do whatever sneaky imperative things here */
      Js.log("props `message` changed!")
    },
  render: (_self) => ...
};
```
We expose `ReasonReact.statelessComponentWithRetainedProps` and `ReasonReact.reducerComponentWithRetainedProps`. Both work like their ordinary non-retained-props counterpart, and require you to specify a new field, retainedProps (of whatever type you'd like) in your component's spec in make.

23. 
Traditional ReactJS componentWillReceiveProps takes in a nextProps. We don't have nextProps, since those are simply the labeled arguments in make, available to you in the scope. To access the current props, however, you'd use the above retainedProps API:
```javascript
type state = {someToggle: bool};

let component = ReasonReact.reducerComponentWithRetainedProps("MyComponent");

let make = (~name, _children) => {
  ...component,
  initialState: () => {someToggle: false},
  /* just like state, the retainedProps field can return anything! Here it retained the `name` prop's value */
  retainedProps: name,
  willReceiveProps: (self) => {
    if (self.retainedProps === name) {
      ...
      /* previous ReactJS logic would be: if (props.name === nextProps.name) */
    };
    ...
  }
};
```

24. 
A common pattern in ReactJS is to attach extra variables onto a component's spec:
```javascript
const Greeting = React.createClass({
  intervalId: null,
  componentDidMount: () => this.intervalId = setInterval(...),
  render: ...
});
```
In reality, this is nothing but a thinly veiled way to mutate a component's "state", without triggering a re-render. ReasonReact asks you to correctly put these instance variables into your component's state, into Reason refs.
```javascript
type state = {
  someRandomState: option(string),
  intervalId: ref(option(int))
};

let component = ...; /* remember, `component` needs to be close to `make`, and after `state` type declaration! */

let make = (_children) => {
  ...component,
  initialState: () => {someRandomState: Some("hello"), intervalId: ref(None)},
  didMount: ({state}) => {
    /* mutate the value here */
    state.intervalId := Some(Js.Global.setInterval(...));
    /* no extra state update needed */
    ReasonReact.NoUpdate
  },
  render: ...
};
```
All your instance variables (subscriptions, refs, etc.) must be in state fields marked as a ref. Don't directly use a mutable field on the state record, use an immutable field pointing to a Reason ref. Why such constraint? To prepare for concurrent React which needs to manage side-effects & mutations more formally. More details here if you're ever interested.

25. 
A ReasonReact ref would be just another instance variable. You'd type it as ReasonReact.reactRef if it's attached to a custom component, and Dom.element if it's attached to a React DOM element.
```javascript
type state = {
  isOpen: bool,
  mySectionRef: ref(option(ReasonReact.reactRef))
};

let setSectionRef = (theRef, {ReasonReact.state}) => {
  state.mySectionRef := Js.Nullable.toOption(theRef);
  /* wondering about Js.Nullable.toOption? See the note below */
};

let component = ReasonReact.reducerComponent("MyPanel");

let make = (~className="", _children) => {
  ...component,
  initialState: () => {isOpen: false, mySectionRef: ref(None)},
  reducer: ...,
  render: (self) => <Section1 ref={self.handle(setSectionRef)} />
};
```
Attaching to a React DOM element looks the same: state.mySectionRef = {myDivRef: Js.Nullable.toOption(theRef)}.
Note how ReactJS refs can be null. Which is why theRef and myDivRef are converted from a JS nullable to an OCaml option (Some/None). When you use the ref, you'll be forced to handle the null case through a switch, which prevents subtle errors!
You must follow the instanceVars convention in the previous section for ref.
ReasonReact ref only accept callbacks. The string ref from ReactJS is deprecated.
We also expose an escape hatch ReasonReact.refToJsObj of type ReasonReact.reactRef => Js.t {..}, which turns your ref into a JS object you can freely use; this is only used to access ReactJS component class methods.
```javascript
let handleClick = (event, self) =>
  switch (self.state.mySectionRef^) {
  | None => ()
  | Some(r) => ReasonReact.refToJsObj(r)##someMethod(1, 2, 3) /* I solemnly swear that I am up to no good */
  };
```

26. 
You can reuse the same bsb setup (that you might have seen here)! Aka, put a bsconfig.json at the root of your ReactJS project:
```javascript
{
  "name": "my-project-name",
  "reason": {"react-jsx" : 2},
  "sources": [
    "my_source_folder"
  ],
  "package-specs": [{
    "module": "commonjs",
    "in-source": true
  }],
  "suffix": ".bs.js",
  "namespace": true,
  "bs-dependencies": [
    "reason-react"
  ],
  "refmt": 3
}
```
This will build Reason files in my_source_folder (e.g. reasonComponent.re) and output the JS files (e.g. reasonComponent.bs.js) alongside them.
Then add bs-platform to your package.json `(npm install --save-dev bs-platform or yarn add --dev bs-platform)`:
```javascript
"scripts": {
  "start": "bsb -make-world -w"
},
"devDependencies": {
  "bs-platform": "^2.1.0"
},
"dependencies": {
  "react": "^15.4.2",
  "react-dom": "^15.4.2",
  "reason-react": "^0.3.1"
}
...
```
Running npm start (or alias it to your favorite command) starts the bsb build watcher. You don't have to touch your existing JavaScript build configuration!

27. 
Easy! Since other Reason components only need you to expose a make function, fake one up:
```javascript
[@bs.module] external myJSReactClass: ReasonReact.reactClass = "./myJSReactClass";
let make = (~className, ~type_, ~value=?, children) =>
  ReasonReact.wrapJsForReason(
    ~reactClass=myJSReactClass,
    ~props=jsProps(
      ~className,
      ~type_,
      ~value=Js.Nullable.fromOption(value),
    ),
    children,
  );
```
ReasonReact.wrapJsForReason is the helper we expose for this purpose. It takes in:
- The reactClass you want to wrap
- The props js object you'd create through the generated jsProps function from the jsPropstype you've declared above (with values properly converted from Reason data structures to JS)
- The mandatory children you'd forward to the JS side.
props is mandatory. If you don't have any to pass, pass ~props=Js.Obj.empty() instead.
Note: if your app successfully compiles, and you see the error "element type is invalid..." in your console, you might be hitting this mistake.

28. 
We expose a helper for the other direction, ReasonReact.wrapReasonForJs:
```javascript
let component = ...;
let make ...;

[@bs.deriving abstract]
type jsProps = {
  name: string,
  age: Js.nullable(int),
};

let jsComponent =
  ReasonReact.wrapReasonForJs(~component, jsProps =>
    make(
      ~name=jsProps |. name,
      ~age=?Js.Nullable.toOption(jsProps |. age),
      [||],
    )
  );
```
The function takes in:
- The labeled reason component you've created
- A function that, given the JS props, asks you to call make while passing in the correctly converted parameters (bs.deriving abstract above generates a field accessor for every record field you've declared).
You'd assign the whole thing to the name jsComponent. The JS side can then import it:
`var MyReasonComponent = require('./myReasonComponent.bs').jsComponent;`
// make sure you're passing the correct data types!
`<MyReasonComponent name="John" />`
Note: if you'd rather use a default import on the JS side, you can export such default from BuckleScript/ReasonReact:
`let default = ReasonReact.wrapReasonForJs(...)`
and then import it on the JS side with:
`import MyReasonComponent from './myReasonComponent.bs';`
BuckleScript default exports only works when the JS side uses ES6 import/exports. More info here.

29. 
ReasonReact events map cleanly to ReactJS synthetic events. More info in the inline docs.
If you're accessing fields on your event object, like event.target.value, you'd use a combination of a ReactDOMRe helper and BuckleScript's ## object access FFI:
ReactDOMRe.domElementToObj(ReactEventRe.Form.target(event))##value;

30. 
Since CSS-in-JS is all the rage right now, we'll recommend our official pick soon. In the meantime, for inline styles, there's the ReactDOMRe.Style.make API:
```javascript
<div style=(ReactDOMRe.Style.make(~color="#444444", ~fontSize="68px", ()))/>
```
It's a labeled (typed!) function call that maps to the familiar style object {color: '#444444', fontSize: '68px'}. Note that make returns an opaque ReactDOMRe.style type that you can't read into. We also expose a ReactDOMRe.Style.combine that takes in two styles and combine them.

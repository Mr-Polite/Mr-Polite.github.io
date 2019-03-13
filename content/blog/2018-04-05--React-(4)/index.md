---
title: "React (4)"
date: "2018-04-05T09:00:00.009Z"
category: "development"
---
## FYI
My notes are overwritten on the content copied from https://reactjs.org/docs.

## Lifecycle
It's about usage of resources. 

### 1. Mounting
It is when an element is rendered to the DOM for the first time.

### 2. Unmounting
It is when the DOM produced by the element is removed. 

## Lifecycle hooks (methods for mounting and unmounting)
### 1. `componentDidMount()`
This method runs after the component output has been rendered to the DOM.

### 2. `componentWillUnmount()`
This method runs before the component output will be removed.

## State

### Using `setState()`
do not do:
```
this.state.sth = "something";
```

do:
```
this.setState({sth: "something"});
```

### `this.props` and `this.state` may be updated asynchronously.
So you should avoid using `this.state` and `this.props` at the same time because it is dangerous. Instead, use a function with previous state as the first argument:
```
this.setState((prevState, props) => ({
  counter: prevState.counter + props.increment
}));
```


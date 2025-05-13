I was working on a [bug](https://github.com/aidenybai/react-scan/issues/355) reported on React Scan, which was related to forwardRef. So, I decided to dive deep into how forwardRef works internally—and here's what I learned.

*React version: [19.1.0](https://github.com/facebook/react/releases/tag/v19.1.0)*

## What does forwardRef do?

According to [react.dev](https://react.dev/reference/react/forwardRef), forwardRef lets your component expose a DOM node to parent component with a ref.
```
const SomeComponent = forwardRef(render)
```

It's useful when you want to pass a ref to a DOM node inside a functional component.

But we're not here to talk about how to use it—let’s dive into the internals and see what we can find!

Let’s say we have a component called with `forwardRef`

```
const Text = forwardRef((props, ref) => {
  return <p ref={ref}>Hello World!</p>
})
```

After compilation, the JSX is transformed into `React.createElement`. This is what the code looks like:

```
const Text = forwardRef((props, ref) => {
  return React.createElement("p", {
    ref: ref,
    children: "Hello World!"
  });
});
```

Now, let’s look at what `forwardRef` function returns and assigns to the `Text` variable. The following is the actual source code of `forwardRef` in React codebase.

```
export function forwardRef<Props, ElementType: React$ElementType>(
  render: (
    props: Props,
    ref: React$RefSetter<React$ElementRef<ElementType>>,
  ) => React$Node,
) {

  const elementType = {
    $$typeof: REACT_FORWARD_REF_TYPE,
    render,
  };

  return elementType
}
```

Based on the source code, `forwardRef` returns a object named `elementType`, with the following properties:
* $$typeof: a symbol (REACT_FORWARD_REF_TYPE) that tells React this is a forwardRef element.
* render: the component function that will be called during rendering.

`REACT_FORWARD_REF_TYPE` is just a symbol to tell what element it is. `render` will be the Component it will render.

This object is assigned to the type property in the Fiber node. The type in a Fiber node defines what the element is. It can be:
* A string (e.g. `<p>`) for DOM elements
* A function or class for functional/class components
* An object (like the one returned by `forwardRef`) for special elements

## What happens during Render phase?

As we know, React elements are internally converted to a Fiber tree. During the conversion, if it is a forwardRef (by checking if `typeof type === 'object’` and `$$typeof === REACT_FORWARD_REF_TYPE`), React will create a special Fiber with `ForwardRef` tag.

In the render phase, React will specially handle it when the tag is `ForwardRef`. It will call `renderWithHooks` with the ref

```
renderWithHooks(
  current,
  workInProgress,
  render, // renamed to Component in the function
  propsWithoutRef,
  ref, <-----
  renderLanes,
);
```

Inside the function, it calls the inner component with the outer ref like
```
Component(propsWithoutRef, ref)
```

Now it is successfully passed to its child
```
(propsWithoutRef, ref) => {
  return React.createElement("p", {
    ref: ref,
    children: "Hello World!"
  });
};
```

## What is the difference without forwardRef?

Now let’s look at the difference between handling `ForwardRef` tag and `FunctionComponent` tag, you will understand why we have to call `forwardRef` to pass the ref to the child.

When it is a functional component, during the render phase, React will render the component by calling the same function `renderWithHooks` but with different arguments.

```jsx
renderWithHooks(
  current,
  workInProgress,
  Component,
  nextProps,
  context, <----- not ref!
  renderLanes,
);
```

The context might be undefined. Hence that’s why, when it renders the component

```jsx
Component(nextProps, undefined)
```

It loses the ref

```jsx

(nextProps) => {
  return React.createElement("p", {
    ref: ref, <--- no reference
    children: "Hello World!"
  });
};
```

## Thoughts

I'm not sure what the original reasoning was for not treating ref as a prop, but starting from React 19, `forwardRef` is no longer necessary. You can now pass ref directly as a prop, even in functional components, without needing to wrap them in `forwardRef`.

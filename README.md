This project aims to illustrate several caveats of using TypeScript and how to overcome them. Since TypeScript eventually just runs JavaScript, insufficient typing may work but can introduce subtle bugs. In addition, lacking compiler support can lead to dead code accumulating over time. 

Each additional commit deals with another problem in the code base, and will subsequently be explained in more detail. Use GitHub's or your IDE's diff editor to view the changes commit-by-commit.

## Commits

### Add return type to prop mappers of the `Login` component

The functions mapping state and dispatch to `LoginProps` do not have explicitly defined return types. This prevents the compiler from notifying us of a piece of deprecated code:
1. An easy solution is adding `Partial<LoginProps>` as a return type to both methods. This would would cause a compiler error on `environment` being mapped in `mapStateToProps` as it is not defined in `LoginProps`. While easy, it is still insufficient because it allows mapping dispatch props in the state mapper and vice versa.
2. The better solution is to split the definition of `LoginProps` into separate types, joining them together afterwards. `LoginStateProps` and `LoginDispatchProps` can then serve as specific return types for the mappers.
3. Realising that `environment` is now injected via context, we can proceed to delete it from the redux state altogether!

### Fix type of `LoginProps.user`

After removing the obsolete `environment` prop, it becomes obvious that `mapStateToProps` lacks the mapping for the `user` props. The compiler does not pick up on this because of a common misunderstanding around TypeScript's optional parameters: `user?: User`.
1. The `Login` component asks whether a user is currently logged in to toggle between login and logout. It does so by making the prop optional by adding a `?` to the prop name. Unlike in languages like Kotlin, though, this does not affect nullability.
2. Instead, it means that you do not need to specify the prop, similar to how you allowed to omit parameters in JavaScript. Hence, the compiler does not give us an error in the mapper.
3. Let's change `user` to a proper nullable type: `user: User | undefined`. This will trigger the desired compile error. Note that you might still pass `undefined` to the prop.

### Add types to mock data in tests

When testing components, test data tends to be more loosely typed in order to reduce clutter. This might lead to similar problems as mentioned above.
1. Adding `LoginProps` as type of `defaultProps` reveals that here, too, we still mapped `environment`.
2. Moreover, adding `Partial<LoginProps>` to the parameter of `renderWithProps` reveals that the object passed in one test does not match the structure of `Login`'s props at all.
3. Changing the value passed in the test to `{ user: { name: ... } }` fixes the compile errors.

A later commit will deal with small changes to reduce clutter in tests and util methods.

### Use type guards to reduce the need for casting

[Type Guards](https://www.typescriptlang.org/docs/handbook/advanced-types.html#type-guards-and-differentiating-types) are a way to help with determining the exact type information of an object without the need for explicit casting.
1. To reduce clutter in the `Products` component, we will add type guards to determine the type of a generic product:
    - `const isBulk = (prod: Product): prod is BulkProduct => prod.type === ProductType.BULK;`
    - `const isUnit = (prod: Product): prod is UnitProduct => "weight" in prod;`

    Note that we use two different ways to determine the type, this is purely for illustration. The important bit is the return type `prod is SomeProduct`, "guaranteeing" the compiler that we know the exact type.
2. As the compiler now knows the type, when rendering `Products` we can shorten the type check and do not have to cast anymore: 
    
    `if (isBulk(product)) label = renderBulk(product)`

### Add type to props of `Product` component

The product component's props are currently not typed at all, hence we do not have any compiler support. This might seem contrived, but can easily happen if a previously locally used components without strong typing (as they are only used internally) is made public to be used in multiple places. Frequently, I see `any` types when external library components are integrated due to their complex types. 

Adding state and dispatch props as we did for `Login` immediately shows several comiler errors:
1. `mapStateToProps` wrongly sets `cart` instead of `products`.
2. `randomProp` in `App.tsx` is another example for obsolete code.
3. `App.tsx` tries to override `onLogout`. This could interfere with the dispatch prop and is now flagged by the compiler.
4. `title` is supposed to be an "OwnProp" to be passed when creating the component. We have to merge `{ title: string }` to the `ProductProps` which will cause the compiler to flag the missing prop in `App.tsx`.

### Make util method parameters more flexible

`renderPriceWithCurrency` accepts a `Product` as a parameter and renders some of its attributes. While neat in the code, it does introduce quite some boilerplate in the test and narrows down the use of the method to only `Product`.
1. Let's change the parameter type to `{ price: number, currency: string }`.
2. Miraculously, this does not result in compile errors. `Product`, while not an explicit sub type, does match all attributes of this type.
3. To avoid defining the types of the attributes twice, we can use `Pick`: `Pick<Product, 'price' | 'currency'>`. This will result in the same type but will stay in sync in case we change `currency` to an enum, for example.
4. Next, delete the mock `Product` in the test and instead pass an object with just those two attributes: `renderPriceWithCurrency({ price: 6, currency: 'SGD' })`. This make the test more robust against changes to `Product`.
5. In the future, any type with `price` and `currency` can also use this method without having to overload or copy it. Imaging a type `Invoice`, for example.

### Derive prop types from Redux mapper methods

There is a lot of opinion involved in structuring Redux applications. According to gospel, the non-connected component can easily be used without redux (which comes in handy when testing them). But in order to properly type the props, we already split them into state, dispatch, and own props, which does not mean anything to the component without Redux.

If, on the hand, we go the other way and tie in Redux even closer, we can reduce the amount of redundancy and code in general. See `Products` as an example:
1. Copy the content of `index.tsx` into `Products.tsx`. Delete `index.tsx`. 
2. Remove `ProductsStateProps` and `ProductsDispatchProps`.
3. Instead, define `ProductsProps` as `ReturnType<typeof mapStateToProps> & ReturnType<typeof mapDispatchToProps> & { title: string }`.
4. Note the reduced redundancy: The properties and types of all props are only defined once by the mapper methods and do not need to be explicitly defined.

## Further reading
* [TypeScript Advanced Types](https://www.typescriptlang.org/docs/handbook/advanced-types.html) explains basic ways to create new types from existing ones, for example Union Types, Literal Types, and Type Guards.
* [TypeScript Utility Types](https://www.typescriptlang.org/docs/handbook/utility-types.html) are another way to modify existing types. Examples are `Pick`, `Partial`, and `ReturnType`.
* [This blog post on how to set up a redux application](https://levelup.gitconnected.com/set-up-a-typescript-react-redux-project-35d65f14b869) loosely follows the [Ducks pattern](https://redux.js.org/style-guide/style-guide#structure-files-as-feature-folders-or-ducks). By bundling redux code by feature (user, product...) instead of splitting it by type (action, reducer...), ducks makes it easier to keep redux strongly typed.


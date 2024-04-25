# InversifyJS

## 📦 Installation

You can get the latest release and the type definitions using your preferred package manager:

```sh
> npm install inversify reflect-metadata --save
> yarn add inversify reflect-metadata
> pnpm add inversify reflect-metadata
```

> ❕**Hint!** If you want to use a more type-safe version of reflect-metadata, try [`@abraham/reflection`](https://www.npmjs.com/package/@abraham/reflection)

The InversifyJS type definitions are included in the inversify npm package.

> :warning: **Important!** InversifyJS requires TypeScript >= 4.4 and the `experimentalDecorators`, `emitDecoratorMetadata`, `types` and `lib`
> compilation options in your `tsconfig.json` file.

```json
{
  "compilerOptions": {
    "target": "es5",
    "lib": ["es6"],
    "types": ["reflect-metadata"],
    "module": "commonjs",
    "moduleResolution": "node",
    "experimentalDecorators": true,
    "emitDecoratorMetadata": true
  }
}
```

InversifyJS requires a modern JavaScript engine with support for:

- [Reflect metadata](https://rbuckton.github.io/reflect-metadata/)
- [Map](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Map)
- [Promise](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Promise) (Only required if using [provider injection](https://github.com/inversify/InversifyJS/blob/master/wiki/provider_injection.md))
- [Proxy](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Proxy) (Only required if using [activation handlers](https://github.com/inversify/InversifyJS/blob/master/wiki/activation_handler.md))

If your environment doesn't support one of these you will need to import a shim or polyfill.

> :warning: **The `reflect-metadata` polyfill should be imported only once in your entire application** because the Reflect object is meant to be a global singleton. More details about this can be found [here](https://github.com/inversify/InversifyJS/issues/262#issuecomment-227593844).

Check out the [Environment support and polyfills](https://github.com/inversify/InversifyJS/blob/master/wiki/environment.md)
page in the wiki and the [Basic example](https://github.com/inversify/inversify-basic-example) to learn more.

## The Basics

Let’s take a look at the basic usage and APIs of InversifyJS with TypeScript:

### Step 1: Declare your interfaces and types

Our goal is to write code that adheres to the [dependency inversion principle](https://en.wikipedia.org/wiki/Dependency_inversion_principle).
This means that we should "depend upon Abstractions and do not depend upon concretions".
Let's start by declaring some interfaces (abstractions).

```ts
// file interfaces.ts

export interface Warrior {
  fight(): string;
  sneak(): string;
}

export interface Weapon {
  hit(): string;
}

export interface ThrowableWeapon {
  throw(): string;
}
```

InversifyJS needs to use the type as identifiers at runtime. We use symbols as identifiers but you can also use classes and or string literals.

PLEASE MAKE SURE TO PLACE THIS TYPES DECLARATION IN A SEPARATE FILE. (see bug #1455)

```ts
// file types.ts

const TYPES = {
  Warrior: Symbol.for("Warrior"),
  Weapon: Symbol.for("Weapon"),
  ThrowableWeapon: Symbol.for("ThrowableWeapon"),
};

export { TYPES };
```

> **Note**: It is recommended to use Symbols but InversifyJS also support the usage of Classes and string literals (please refer to the features section to learn more).

### Step 2: Declare dependencies using the `@injectable` & `@inject` decorators

Let's continue by declaring some classes (concretions). The classes are implementations of the interfaces that we just declared. All the classes must be annotated with the `@injectable` decorator.

When a class has a dependency on an interface we also need to use the `@inject` decorator to define an identifier for the interface that will be available at runtime. In this case we will use the Symbols `Symbol.for("Weapon")` and `Symbol.for("ThrowableWeapon")` as runtime identifiers.

```ts
// file entities.ts

import { injectable, inject } from "inversify";
import "reflect-metadata";
import { Weapon, ThrowableWeapon, Warrior } from "./interfaces";
import { TYPES } from "./types";

@injectable()
class Katana implements Weapon {
  public hit() {
    return "cut!";
  }
}

@injectable()
class Shuriken implements ThrowableWeapon {
  public throw() {
    return "hit!";
  }
}

@injectable()
class Ninja implements Warrior {
  private _katana: Weapon;
  private _shuriken: ThrowableWeapon;

  public constructor(
    @inject(TYPES.Weapon) katana: Weapon,
    @inject(TYPES.ThrowableWeapon) shuriken: ThrowableWeapon,
  ) {
    this._katana = katana;
    this._shuriken = shuriken;
  }

  public fight() {
    return this._katana.hit();
  }
  public sneak() {
    return this._shuriken.throw();
  }
}

export { Ninja, Katana, Shuriken };
```

If you prefer it you can use property injection instead of constructor injection so you don't have to declare the class constructor:

```ts
@injectable()
class Ninja implements Warrior {
  @inject(TYPES.Weapon) private _katana: Weapon;
  @inject(TYPES.ThrowableWeapon) private _shuriken: ThrowableWeapon;
  public fight() {
    return this._katana.hit();
  }
  public sneak() {
    return this._shuriken.throw();
  }
}
```

### Step 3: Create and configure a Container

We recommend to do this in a file named `inversify.config.ts`. This is the only place in which there is some coupling.
In the rest of your application your classes should be free of references to other classes.

```ts
// file inversify.config.ts

import { Container } from "inversify";
import { TYPES } from "./types";
import { Warrior, Weapon, ThrowableWeapon } from "./interfaces";
import { Ninja, Katana, Shuriken } from "./entities";

const myContainer = new Container();
myContainer.bind<Warrior>(TYPES.Warrior).to(Ninja);
myContainer.bind<Weapon>(TYPES.Weapon).to(Katana);
myContainer.bind<ThrowableWeapon>(TYPES.ThrowableWeapon).to(Shuriken);

export { myContainer };
```

### Step 4: Resolve dependencies

You can use the method `get<T>` from the `Container` class to resolve a dependency.
Remember that you should do this only in your [composition root](http://blog.ploeh.dk/2011/07/28/CompositionRoot/)
to avoid the [service locator anti-pattern](http://blog.ploeh.dk/2010/02/03/ServiceLocatorisanAnti-Pattern/).

```ts
import { myContainer } from "./inversify.config";
import { TYPES } from "./types";
import { Warrior } from "./interfaces";

const ninja = myContainer.get<Warrior>(TYPES.Warrior);

expect(ninja.fight()).eql("cut!"); // true
expect(ninja.sneak()).eql("hit!"); // true
```

As we can see the `Katana` and `Shuriken` were successfully resolved and injected into `Ninja`.

InversifyJS supports ES5 and ES6 and can work without TypeScript.
Head to the [**JavaScript example**](https://github.com/inversify/InversifyJS/blob/master/wiki/basic_js_example.md) to learn more!

## 🚀 The InversifyJS Features and API

Let's take a look to the InversifyJS features!

- [Support for classes](https://github.com/inversify/InversifyJS/blob/master/wiki/classes_as_id.md)
- [Support for Symbols](https://github.com/inversify/InversifyJS/blob/master/wiki/symbols_as_id.md)
- [Container API](https://github.com/inversify/InversifyJS/blob/master/wiki/container_api.md)
- [Declaring container modules](https://github.com/inversify/InversifyJS/blob/master/wiki/container_modules.md)
- [Container snapshots](https://github.com/inversify/InversifyJS/blob/master/wiki/container_snapshots.md)
- [Controlling the scope of the dependencies](https://github.com/inversify/InversifyJS/blob/master/wiki/scope.md)
- [Declaring optional dependencies](https://github.com/inversify/InversifyJS/blob/master/wiki/optional_dependencies.md)
- [Injecting a constant or dynamic value](https://github.com/inversify/InversifyJS/blob/master/wiki/value_injection.md)
- [Injecting a class constructor](https://github.com/inversify/InversifyJS/blob/master/wiki/constructor_injection.md)
- [Injecting a Factory](https://github.com/inversify/InversifyJS/blob/master/wiki/factory_injection.md)
- [Auto factory](https://github.com/inversify/InversifyJS/blob/master/wiki/auto_factory.md)
- [Auto named factory](https://github.com/inversify/InversifyJS/blob/master/wiki/auto_named_factory.md)
- [Injecting a Provider (asynchronous Factory)](https://github.com/inversify/InversifyJS/blob/master/wiki/provider_injection.md)
- [Activation handler](https://github.com/inversify/InversifyJS/blob/master/wiki/activation_handler.md)
- [Deactivation handler](https://github.com/inversify/InversifyJS/blob/master/wiki/deactivation_handler.md)
- [Post Construct decorator](https://github.com/inversify/InversifyJS/blob/master/wiki/post_construct.md)
- [Middleware](https://github.com/inversify/InversifyJS/blob/master/wiki/middleware.md)
- [Multi-injection](https://github.com/inversify/InversifyJS/blob/master/wiki/multi_injection.md)
- [Tagged bindings](https://github.com/inversify/InversifyJS/blob/master/wiki/tagged_bindings.md)
- [Create your own tag decorators](https://github.com/inversify/InversifyJS/blob/master/wiki/custom_tag_decorators.md)
- [Named bindings](https://github.com/inversify/InversifyJS/blob/master/wiki/named_bindings.md)
- [Default target](https://github.com/inversify/InversifyJS/blob/master/wiki/default_targets.md)
- [Support for hierarchical DI systems](https://github.com/inversify/InversifyJS/blob/master/wiki/hierarchical_di.md)
- [Contextual bindings & @targetName](https://github.com/inversify/InversifyJS/blob/master/wiki/contextual_bindings.md)
- [Property injection](https://github.com/inversify/InversifyJS/blob/master/wiki/property_injection.md)
- [Circular dependencies](https://github.com/inversify/InversifyJS/blob/master/wiki/circular_dependencies.md)
- [Inheritance](https://github.com/inversify/InversifyJS/blob/master/wiki/inheritance.md)

Please refer to the [wiki](https://github.com/inversify/InversifyJS/blob/master/wiki/readme.md) for additional details.
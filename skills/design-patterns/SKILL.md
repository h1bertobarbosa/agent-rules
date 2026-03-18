---
name: design-patterns
description: Help choose and implement design patterns with trade-offs, code templates, and integration guidance for real codebases.
metadata:
  audience: engineers
  scope: architecture-and-refactoring
---

## What I do

- Identify when a design pattern is useful (or unnecessary) for the current problem.
- Map requirements to suitable patterns and explain trade-offs.
- Provide implementation plans and practical code scaffolding.
- Refactor existing code toward clearer responsibilities and lower coupling.
- Keep solutions aligned with the project style, language, and architecture.

## When to use me

Use this skill when the task involves architecture, extensibility, object collaboration, or recurring code smells such as:

- Condition-heavy object creation
- Tight coupling between modules
- Feature changes causing edits in many places
- Difficulty testing logic due to hidden dependencies

## Workflow

1. Clarify constraints: language, framework, existing architecture, performance, and delivery risk.
2. Diagnose the core problem and classify it: creational, structural, behavioral, or mixed.
3. Propose 1-2 viable patterns, with a recommended option and why.
4. Define roles (interfaces, collaborators, lifecycle boundaries).
5. Implement incrementally with minimal blast radius.
6. Validate with tests and usage examples.
7. Document migration notes and follow-up refactors.

## Decision checklist

- Is this solving a recurring problem or over-engineering a one-off?
- Does it improve testability and readability?
- Are dependencies inverted where needed?
- Is the resulting API simpler for callers?
- Are performance and memory implications acceptable?

## Pattern selection guide

- **Creational**: Abstract Factory, Factory Method, Builder, Prototype, Singleton
  - Prefer for object construction complexity, variant families, or creation policies.
- **Structural**: Adapter, Bridge, Composite, Decorator, Facade, Flyweight, Proxy
  - Prefer for interface compatibility, composition, and subsystem boundaries.
- **Behavioral**: Chain of Responsibility, Command, Iterator, Mediator, Memento, Observer, State, Strategy, Template Method, Visitor
  - Prefer for communication flow, algorithm swaps, and responsibility distribution.

## Output expectations

When applying this skill, produce:

- A brief rationale for chosen pattern(s)
- A concrete integration plan tied to repository files
- Minimal, production-oriented code changes
- Tests or verification steps
- Risks and rollback notes when refactoring existing modules

## Guardrails

- Prefer the simplest design that meets current requirements.
- Do not introduce a pattern if composition, functions, or small refactors are enough.
- Preserve existing public contracts unless the task explicitly allows breaking changes.
- Keep naming explicit and role-based (for example, `Factory`, `Strategy`, `Handler`).

## Example usage

/\*\*

- Abstract Factory Design Pattern
-
- Intent: Lets you produce families of related objects without specifying their
- concrete classes.
  \*/

/\*\*

- The Abstract Factory interface declares a set of methods that return
- different abstract products. These products are called a family and are
- related by a high-level theme or concept. Products of one family are usually
- able to collaborate among themselves. A family of products may have several
- variants, but the products of one variant are incompatible with products of
- another.
  \*/
  interface AbstractFactory {
  createProductA(): AbstractProductA;

      createProductB(): AbstractProductB;

  }

/\*\*

- Concrete Factories produce a family of products that belong to a single
- variant. The factory guarantees that resulting products are compatible. Note
- that signatures of the Concrete Factory's methods return an abstract product,
- while inside the method a concrete product is instantiated.
  \*/
  class ConcreteFactory1 implements AbstractFactory {
  public createProductA(): AbstractProductA {
  return new ConcreteProductA1();
  }

      public createProductB(): AbstractProductB {
          return new ConcreteProductB1();
      }

  }

/\*\*

- Each Concrete Factory has a corresponding product variant.
  \*/
  class ConcreteFactory2 implements AbstractFactory {
  public createProductA(): AbstractProductA {
  return new ConcreteProductA2();
  }

      public createProductB(): AbstractProductB {
          return new ConcreteProductB2();
      }

  }

/\*\*

- Each distinct product of a product family should have a base interface. All
- variants of the product must implement this interface.
  \*/
  interface AbstractProductA {
  usefulFunctionA(): string;
  }

/\*\*

- These Concrete Products are created by corresponding Concrete Factories.
  \*/
  class ConcreteProductA1 implements AbstractProductA {
  public usefulFunctionA(): string {
  return 'The result of the product A1.';
  }
  }

class ConcreteProductA2 implements AbstractProductA {
public usefulFunctionA(): string {
return 'The result of the product A2.';
}
}

/\*\*

- Here's the the base interface of another product. All products can interact
- with each other, but proper interaction is possible only between products of
- the same concrete variant.
  _/
  interface AbstractProductB {
  /\*\*
  _ Product B is able to do its own thing...
  \*/
  usefulFunctionB(): string;

      /**
       * ...but it also can collaborate with the ProductA.
       *
       * The Abstract Factory makes sure that all products it creates are of the
       * same variant and thus, compatible.
       */
      anotherUsefulFunctionB(collaborator: AbstractProductA): string;

  }

/\*\*

- These Concrete Products are created by corresponding Concrete Factories.
  \*/
  class ConcreteProductB1 implements AbstractProductB {

      public usefulFunctionB(): string {
          return 'The result of the product B1.';
      }

      /**
       * The variant, Product B1, is only able to work correctly with the variant,
       * Product A1. Nevertheless, it accepts any instance of AbstractProductA as
       * an argument.
       */
      public anotherUsefulFunctionB(collaborator: AbstractProductA): string {
          const result = collaborator.usefulFunctionA();
          return `The result of the B1 collaborating with the (${result})`;
      }

  }

class ConcreteProductB2 implements AbstractProductB {

    public usefulFunctionB(): string {
        return 'The result of the product B2.';
    }

    /**
     * The variant, Product B2, is only able to work correctly with the variant,
     * Product A2. Nevertheless, it accepts any instance of AbstractProductA as
     * an argument.
     */
    public anotherUsefulFunctionB(collaborator: AbstractProductA): string {
        const result = collaborator.usefulFunctionA();
        return `The result of the B2 collaborating with the (${result})`;
    }

}

/\*\*

- The client code works with factories and products only through abstract
- types: AbstractFactory and AbstractProduct. This lets you pass any factory or
- product subclass to the client code without breaking it.
  \*/
  function clientCode(factory: AbstractFactory) {
  const productA = factory.createProductA();
  const productB = factory.createProductB();

      console.log(productB.usefulFunctionB());
      console.log(productB.anotherUsefulFunctionB(productA));

  }

/\*\*

- The client code can work with any concrete factory class.
  \*/
  console.log('Client: Testing client code with the first factory type...');
  clientCode(new ConcreteFactory1());

console.log('');

console.log('Client: Testing the same client code with the second factory type...');
clientCode(new ConcreteFactory2());
/\*\*

- Adapter Design Pattern
-
- Intent: Provides a unified interface that allows objects with incompatible
- interfaces to collaborate.
  \*/

/\*\*

- The Target defines the domain-specific interface used by the client code.
  \*/
  class Target {
  public request(): string {
  return 'Target: The default target\'s behavior.';
  }
  }

/\*\*

- The Adaptee contains some useful behavior, but its interface is incompatible
- with the existing client code. The Adaptee needs some adaptation before the
- client code can use it.
  \*/
  class Adaptee {
  public specificRequest(): string {
  return '.eetpadA eht fo roivaheb laicepS';
  }
  }

/\*\*

- The Adapter makes the Adaptee's interface compatible with the Target's
- interface.
  \*/
  class Adapter extends Target {
  private adaptee: Adaptee;

      constructor(adaptee: Adaptee) {
          super();
          this.adaptee = adaptee;
      }

      public request(): string {
          const result = this.adaptee.specificRequest().split('').reverse().join('');
          return `Adapter: (TRANSLATED) ${result}`;
      }

  }

/\*\*

- The client code supports all classes that follow the Target interface.
  \*/
  function clientCode(target: Target) {
  console.log(target.request());
  }

console.log('Client: I can work just fine with the Target objects:');
const target = new Target();
clientCode(target);

console.log('');

const adaptee = new Adaptee();
console.log('Client: The Adaptee class has a weird interface. See, I don\'t understand it:');
console.log(`Adaptee: ${adaptee.specificRequest()}`);

console.log('');

console.log('Client: But I can work with it via the Adapter:');
const adapter = new Adapter(adaptee);
clientCode(adapter);
/\*\*

- Bridge Design Pattern
-
- Intent: Lets you split a large class or a set of closely related classes into
- two separate hierarchies—abstraction and implementation—which can be
- developed independently of each other.
-
-               A
-            /     \                        A         N
-          Aa      Ab        ===>        /     \     / \
-         / \     /  \                 Aa(N) Ab(N)  1   2
-       Aa1 Aa2  Ab1 Ab2
  \*/

/\*\*

- The Abstraction defines the interface for the "control" part of the two class
- hierarchies. It maintains a reference to an object of the Implementation
- hierarchy and delegates all of the real work to this object.
  \*/
  class Abstraction {
  protected implementation: Implementation;

      constructor(implementation: Implementation) {
          this.implementation = implementation;
      }

      public operation(): string {
          const result = this.implementation.operationImplementation();
          return `Abstraction: Base operation with:\n${result}`;
      }

  }

/\*\*

- You can extend the Abstraction without changing the Implementation classes.
  \*/
  class ExtendedAbstraction extends Abstraction {
  public operation(): string {
  const result = this.implementation.operationImplementation();
  return `ExtendedAbstraction: Extended operation with:\n${result}`;
  }
  }

/\*\*

- The Implementation defines the interface for all implementation classes. It
- doesn't have to match the Abstraction's interface. In fact, the two
- interfaces can be entirely different. Typically the Implementation interface
- provides only primitive operations, while the Abstraction defines higher-
- level operations based on those primitives.
  \*/
  interface Implementation {
  operationImplementation(): string;
  }

/\*\*

- Each Concrete Implementation corresponds to a specific platform and
- implements the Implementation interface using that platform's API.
  \*/
  class ConcreteImplementationA implements Implementation {
  public operationImplementation(): string {
  return 'ConcreteImplementationA: Here\'s the result on the platform A.';
  }
  }

class ConcreteImplementationB implements Implementation {
public operationImplementation(): string {
return 'ConcreteImplementationB: Here\'s the result on the platform B.';
}
}

/\*\*

- Except for the initialization phase, where an Abstraction object gets linked
- with a specific Implementation object, the client code should only depend on
- the Abstraction class. This way the client code can support any abstraction-
- implementation combination.
  \*/
  function clientCode(abstraction: Abstraction) {
  // ..

      console.log(abstraction.operation());

      // ..

  }

/\*\*

- The client code should be able to work with any pre-configured abstraction-
- implementation combination.
  \*/
  let implementation = new ConcreteImplementationA();
  let abstraction = new Abstraction(implementation);
  clientCode(abstraction);

console.log('');

implementation = new ConcreteImplementationB();
abstraction = new ExtendedAbstraction(implementation);
clientCode(abstraction);
/\*\*

- Builder Design Pattern
-
- Intent: Lets you construct complex objects step by step. The pattern allows
- you to produce different types and representations of an object using the
- same construction code.
  \*/

/\*\*

- The Builder interface specifies methods for creating the different parts of
- the Product objects.
  \*/
  interface Builder {
  producePartA(): void;
  producePartB(): void;
  producePartC(): void;
  }

/\*\*

- The Concrete Builder classes follow the Builder interface and provide
- specific implementations of the building steps. Your program may have several
- variations of Builders, implemented differently.
  \*/
  class ConcreteBuilder1 implements Builder {
  private product: Product1;

      /**
       * A fresh builder instance should contain a blank product object, which is
       * used in further assembly.
       */
      constructor() {
          this.reset();
      }

      public reset(): void {
          this.product = new Product1();
      }

      /**
       * All production steps work with the same product instance.
       */
      public producePartA(): void {
          this.product.parts.push('PartA1');
      }

      public producePartB(): void {
          this.product.parts.push('PartB1');
      }

      public producePartC(): void {
          this.product.parts.push('PartC1');
      }

      /**
       * Concrete Builders are supposed to provide their own methods for
       * retrieving results. That's because various types of builders may create
       * entirely different products that don't follow the same interface.
       * Therefore, such methods cannot be declared in the base Builder interface
       * (at least in a statically typed programming language).
       *
       * Usually, after returning the end result to the client, a builder instance
       * is expected to be ready to start producing another product. That's why
       * it's a usual practice to call the reset method at the end of the
       * `getProduct` method body. However, this behavior is not mandatory, and
       * you can make your builders wait for an explicit reset call from the
       * client code before disposing of the previous result.
       */
      public getProduct(): Product1 {
          const result = this.product;
          this.reset();
          return result;
      }

  }

/\*\*

- It makes sense to use the Builder pattern only when your products are quite
- complex and require extensive configuration.
-
- Unlike in other creational patterns, different concrete builders can produce
- unrelated products. In other words, results of various builders may not
- always follow the same interface.
  \*/
  class Product1 {
  public parts: string[] = [];

      public listParts(): void {
          console.log(`Product parts: ${this.parts.join(', ')}\n`);
      }

  }

/\*\*

- The Director is only responsible for executing the building steps in a
- particular sequence. It is helpful when producing products according to a
- specific order or configuration. Strictly speaking, the Director class is
- optional, since the client can control builders directly.
  \*/
  class Director {
  private builder: Builder;

      /**
       * The Director works with any builder instance that the client code passes
       * to it. This way, the client code may alter the final type of the newly
       * assembled product.
       */
      public setBuilder(builder: Builder): void {
          this.builder = builder;
      }

      /**
       * The Director can construct several product variations using the same
       * building steps.
       */
      public buildMinimalViableProduct(): void {
          this.builder.producePartA();
      }

      public buildFullFeaturedProduct(): void {
          this.builder.producePartA();
          this.builder.producePartB();
          this.builder.producePartC();
      }

  }

/\*\*

- The client code creates a builder object, passes it to the director and then
- initiates the construction process. The end result is retrieved from the
- builder object.
  \*/
  function clientCode(director: Director) {
  const builder = new ConcreteBuilder1();
  director.setBuilder(builder);

      console.log('Standard basic product:');
      director.buildMinimalViableProduct();
      builder.getProduct().listParts();

      console.log('Standard full featured product:');
      director.buildFullFeaturedProduct();
      builder.getProduct().listParts();

      // Remember, the Builder pattern can be used without a Director class.
      console.log('Custom product:');
      builder.producePartA();
      builder.producePartC();
      builder.getProduct().listParts();

  }

const director = new Director();
clientCode(director);
/\*\*

- Chain of Responsibility Design Pattern
-
- Intent: Lets you pass requests along a chain of handlers. Upon receiving a
- request, each handler decides either to process the request or to pass it to
- the next handler in the chain.
  \*/

/\*\*

- The Handler interface declares a method for building the chain of handlers.
- It also declares a method for executing a request.
  \*/
  interface Handler {
  setNext(handler: Handler): Handler;

      handle(request: string): string;

  }

/\*\*

- The default chaining behavior can be implemented inside a base handler class.
  \*/
  abstract class AbstractHandler implements Handler
  {
  private nextHandler: Handler;

      public setNext(handler: Handler): Handler {
          this.nextHandler = handler;
          // Returning a handler from here will let us link handlers in a
          // convenient way like this:
          // monkey.setNext(squirrel).setNext(dog);
          return handler;
      }

      public handle(request: string): string {
          if (this.nextHandler) {
              return this.nextHandler.handle(request);
          }

          return null;
      }

  }

/\*\*

- All Concrete Handlers either handle a request or pass it to the next handler
- in the chain.
  \*/
  class MonkeyHandler extends AbstractHandler {
  public handle(request: string): string {
  if (request === 'Banana') {
  return `Monkey: I'll eat the ${request}.`;
  }
  return super.handle(request);

      }

  }

class SquirrelHandler extends AbstractHandler {
public handle(request: string): string {
if (request === 'Nut') {
return `Squirrel: I'll eat the ${request}.`;
}
return super.handle(request);
}
}

class DogHandler extends AbstractHandler {
public handle(request: string): string {
if (request === 'MeatBall') {
return `Dog: I'll eat the ${request}.`;
}
return super.handle(request);
}
}

/\*\*

- The client code is usually suited to work with a single handler. In most
- cases, it is not even aware that the handler is part of a chain.
  \*/
  function clientCode(handler: Handler) {
  const foods = ['Nut', 'Banana', 'Cup of coffee'];

      for (const food of foods) {
          console.log(`Client: Who wants a ${food}?`);

          const result = handler.handle(food);
          if (result) {
              console.log(`  ${result}`);
          } else {
              console.log(`  ${food} was left untouched.`);
          }
      }

  }

/\*\*

- The other part of the client code constructs the actual chain.
  \*/
  const monkey = new MonkeyHandler();
  const squirrel = new SquirrelHandler();
  const dog = new DogHandler();

monkey.setNext(squirrel).setNext(dog);

/\*\*

- The client should be able to send a request to any handler, not just the
- first one in the chain.
  \*/
  console.log('Chain: Monkey > Squirrel > Dog\n');
  clientCode(monkey);
  console.log('');

console.log('Subchain: Squirrel > Dog\n');
clientCode(squirrel);
/\*\*

- Command Design Pattern
-
- Intent: Turns a request into a stand-alone object that contains all
- information about the request. This transformation lets you parameterize
- methods with different requests, delay or queue a request's execution, and
- support undoable operations.
  \*/

/\*\*

- The Command interface declares a method for executing a command.
  \*/
  interface Command {
  execute(): void;
  }

/\*\*

- Some commands can implement simple operations on their own.
  \*/
  class SimpleCommand implements Command {
  private payload: string;

      constructor(payload: string) {
          this.payload = payload;
      }

      public execute(): void {
          console.log(`SimpleCommand: See, I can do simple things like printing (${this.payload})`);
      }

  }

/\*\*

- However, some commands can delegate more complex operations to other objects,
- called "receivers."
  \*/
  class ComplexCommand implements Command {
  private receiver: Receiver;

      /**
       * Context data, required for launching the receiver's methods.
       */
      private a: string;

      private b: string;

      /**
       * Complex commands can accept one or several receiver objects along with
       * any context data via the constructor.
       */
      constructor(receiver: Receiver, a: string, b: string) {
          this.receiver = receiver;
          this.a = a;
          this.b = b;
      }

      /**
       * Commands can delegate to any methods of a receiver.
       */
      public execute(): void {
          console.log('ComplexCommand: Complex stuff should be done by a receiver object.');
          this.receiver.doSomething(this.a);
          this.receiver.doSomethingElse(this.b);
      }

  }

/\*\*

- The Receiver classes contain some important business logic. They know how to
- perform all kinds of operations, associated with carrying out a request. In
- fact, any class may serve as a Receiver.
  \*/
  class Receiver {
  public doSomething(a: string): void {
  console.log(`Receiver: Working on (${a}.)`);
  }

      public doSomethingElse(b: string): void {
          console.log(`Receiver: Also working on (${b}.)`);
      }

  }

/\*\*

- The Invoker is associated with one or several commands. It sends a request to
- the command.
  \*/
  class Invoker {
  private onStart: Command;

      private onFinish: Command;

      /**
       * Initialize commands.
       */
      public setOnStart(command: Command): void {
          this.onStart = command;
      }

      public setOnFinish(command: Command): void {
          this.onFinish = command;
      }

      /**
       * The Invoker does not depend on concrete command or receiver classes. The
       * Invoker passes a request to a receiver indirectly, by executing a
       * command.
       */
      public doSomethingImportant(): void {
          console.log('Invoker: Does anybody want something done before I begin?');
          if (this.isCommand(this.onStart)) {
              this.onStart.execute();
          }

          console.log('Invoker: ...doing something really important...');

          console.log('Invoker: Does anybody want something done after I finish?');
          if (this.isCommand(this.onFinish)) {
              this.onFinish.execute();
          }
      }

      private isCommand(object): object is Command {
          return object.execute !== undefined;
      }

  }

/\*\*

- The client code can parameterize an invoker with any commands.
  \*/
  const invoker = new Invoker();
  invoker.setOnStart(new SimpleCommand('Say Hi!'));
  const receiver = new Receiver();
  invoker.setOnFinish(new ComplexCommand(receiver, 'Send email', 'Save report'));

invoker.doSomethingImportant();
/\*\*

- Composite Design Pattern
-
- Intent: Lets you compose objects into tree structures and then work with
- these structures as if they were individual objects.
  \*/

/\*\*

- The base Component class declares common operations for both simple and
- complex objects of a composition.
  \*/
  abstract class Component {
  protected parent: Component;

      /**
       * Optionally, the base Component can declare an interface for setting and
       * accessing a parent of the component in a tree structure. It can also
       * provide some default implementation for these methods.
       */
      public setParent(parent: Component) {
          this.parent = parent;
      }

      public getParent(): Component {
          return this.parent;
      }

      /**
       * In some cases, it would be beneficial to define the child-management
       * operations right in the base Component class. This way, you won't need to
       * expose any concrete component classes to the client code, even during the
       * object tree assembly. The downside is that these methods will be empty
       * for the leaf-level components.
       */
      public add(component: Component): void { }

      public remove(component: Component): void { }

      /**
       * You can provide a method that lets the client code figure out whether a
       * component can bear children.
       */
      public isComposite(): boolean {
          return false;
      }

      /**
       * The base Component may implement some default behavior or leave it to
       * concrete classes (by declaring the method containing the behavior as
       * "abstract").
       */
      public abstract operation(): string;

  }

/\*\*

- The Leaf class represents the end objects of a composition. A leaf can't have
- any children.
-
- Usually, it's the Leaf objects that do the actual work, whereas Composite
- objects only delegate to their sub-components.
  \*/
  class Leaf extends Component {
  public operation(): string {
  return 'Leaf';
  }
  }

/\*\*

- The Composite class represents the complex components that may have children.
- Usually, the Composite objects delegate the actual work to their children and
- then "sum-up" the result.
  \*/
  class Composite extends Component {
  protected children: Component[] = [];

      /**
       * A composite object can add or remove other components (both simple or
       * complex) to or from its child list.
       */
      public add(component: Component): void {
          this.children.push(component);
          component.setParent(this);
      }

      public remove(component: Component): void {
          const componentIndex = this.children.indexOf(component);
          this.children.splice(componentIndex, 1);

          component.setParent(null);
      }

      public isComposite(): boolean {
          return true;
      }

      /**
       * The Composite executes its primary logic in a particular way. It
       * traverses recursively through all its children, collecting and summing
       * their results. Since the composite's children pass these calls to their
       * children and so forth, the whole object tree is traversed as a result.
       */
      public operation(): string {
          const results = [];
          for (const child of this.children) {
              results.push(child.operation());
          }

          return `Branch(${results.join('+')})`;
      }

  }

/\*\*

- The client code works with all of the components via the base interface.
  \*/
  function clientCode(component: Component) {
  // ...

      console.log(`RESULT: ${component.operation()}`);

      // ...

  }

/\*\*

- This way the client code can support the simple leaf components...
  \*/
  const simple = new Leaf();
  console.log('Client: I\'ve got a simple component:');
  clientCode(simple);
  console.log('');

/\*\*

- ...as well as the complex composites.
  \*/
  const tree = new Composite();
  const branch1 = new Composite();
  branch1.add(new Leaf());
  branch1.add(new Leaf());
  const branch2 = new Composite();
  branch2.add(new Leaf());
  tree.add(branch1);
  tree.add(branch2);
  console.log('Client: Now I\'ve got a composite tree:');
  clientCode(tree);
  console.log('');

/\*\*

- Thanks to the fact that the child-management operations are declared in the
- base Component class, the client code can work with any component, simple or
- complex, without depending on their concrete classes.
  \*/
  function clientCode2(component1: Component, component2: Component) {
  // ...

      if (component1.isComposite()) {
          component1.add(component2);
      }
      console.log(`RESULT: ${component1.operation()}`);

      // ...

  }

console.log('Client: I don\'t need to check the components classes even when managing the tree:');
clientCode2(tree, simple);
/\*\*

- Decorator Design Pattern
-
- Intent: Lets you attach new behaviors to objects by placing these objects
- inside special wrapper objects that contain the behaviors.
  \*/

/\*\*

- The base Component interface defines operations that can be altered by
- decorators.
  \*/
  interface Component {
  operation(): string;
  }

/\*\*

- Concrete Components provide default implementations of the operations. There
- might be several variations of these classes.
  \*/
  class ConcreteComponent implements Component {
  public operation(): string {
  return 'ConcreteComponent';
  }
  }

/\*\*

- The base Decorator class follows the same interface as the other components.
- The primary purpose of this class is to define the wrapping interface for all
- concrete decorators. The default implementation of the wrapping code might
- include a field for storing a wrapped component and the means to initialize
- it.
  \*/
  class Decorator implements Component {
  protected component: Component;

      constructor(component: Component) {
          this.component = component;
      }

      /**
       * The Decorator delegates all work to the wrapped component.
       */
      public operation(): string {
          return this.component.operation();
      }

  }

/\*\*

- Concrete Decorators call the wrapped object and alter its result in some way.
  _/
  class ConcreteDecoratorA extends Decorator {
  /\*\*
  _ Decorators may call parent implementation of the operation, instead of
  _ calling the wrapped object directly. This approach simplifies extension
  _ of decorator classes.
  \*/
  public operation(): string {
  return `ConcreteDecoratorA(${super.operation()})`;
  }
  }

/\*\*

- Decorators can execute their behavior either before or after the call to a
- wrapped object.
  \*/
  class ConcreteDecoratorB extends Decorator {
  public operation(): string {
  return `ConcreteDecoratorB(${super.operation()})`;
  }
  }

/\*\*

- The client code works with all objects using the Component interface. This
- way it can stay independent of the concrete classes of components it works
- with.
  \*/
  function clientCode(component: Component) {
  // ...

      console.log(`RESULT: ${component.operation()}`);

      // ...

  }

/\*\*

- This way the client code can support both simple components...
  \*/
  const simple = new ConcreteComponent();
  console.log('Client: I\'ve got a simple component:');
  clientCode(simple);
  console.log('');

/\*\*

- ...as well as decorated ones.
-
- Note how decorators can wrap not only simple components but the other
- decorators as well.
  \*/
  const decorator1 = new ConcreteDecoratorA(simple);
  const decorator2 = new ConcreteDecoratorB(decorator1);
  console.log('Client: Now I\'ve got a decorated component:');
  clientCode(decorator2);
  /\*\*
- Facade Design Pattern
-
- Intent: Provides a simplified interface to a library, a framework, or any
- other complex set of classes.
  \*/

/\*\*

- The Facade class provides a simple interface to the complex logic of one or
- several subsystems. The Facade delegates the client requests to the
- appropriate objects within the subsystem. The Facade is also responsible for
- managing their lifecycle. All of this shields the client from the undesired
- complexity of the subsystem.
  \*/
  class Facade {
  protected subsystem1: Subsystem1;

      protected subsystem2: Subsystem2;

      /**
       * Depending on your application's needs, you can provide the Facade with
       * existing subsystem objects or force the Facade to create them on its own.
       */
      constructor(subsystem1: Subsystem1 = null, subsystem2: Subsystem2 = null) {
          this.subsystem1 = subsystem1 || new Subsystem1();
          this.subsystem2 = subsystem2 || new Subsystem2();
      }

      /**
       * The Facade's methods are convenient shortcuts to the sophisticated
       * functionality of the subsystems. However, clients get only to a fraction
       * of a subsystem's capabilities.
       */
      public operation(): string {
          let result = 'Facade initializes subsystems:\n';
          result += this.subsystem1.operation1();
          result += this.subsystem2.operation1();
          result += 'Facade orders subsystems to perform the action:\n';
          result += this.subsystem1.operationN();
          result += this.subsystem2.operationZ();

          return result;
      }

  }

/\*\*

- The Subsystem can accept requests either from the facade or client directly.
- In any case, to the Subsystem, the Facade is yet another client, and it's not
- a part of the Subsystem.
  \*/
  class Subsystem1 {
  public operation1(): string {
  return 'Subsystem1: Ready!\n';
  }

      // ...

      public operationN(): string {
          return 'Subsystem1: Go!\n';
      }

  }

/\*\*

- Some facades can work with multiple subsystems at the same time.
  \*/
  class Subsystem2 {
  public operation1(): string {
  return 'Subsystem2: Get ready!\n';
  }

      // ...

      public operationZ(): string {
          return 'Subsystem2: Fire!';
      }

  }

/\*\*

- The client code works with complex subsystems through a simple interface
- provided by the Facade. When a facade manages the lifecycle of the subsystem,
- the client might not even know about the existence of the subsystem. This
- approach lets you keep the complexity under control.
  \*/
  function clientCode(facade: Facade) {
  // ...

      console.log(facade.operation());

      // ...

  }

/\*\*

- The client code may have some of the subsystem's objects already created. In
- this case, it might be worthwhile to initialize the Facade with these objects
- instead of letting the Facade create new instances.
  \*/
  const subsystem1 = new Subsystem1();
  const subsystem2 = new Subsystem2();
  const facade = new Facade(subsystem1, subsystem2);
  clientCode(facade);
  /\*\*
- Factory Method Design Pattern
-
- Intent: Provides an interface for creating objects in a superclass, but
- allows subclasses to alter the type of objects that will be created.
  \*/

/\*\*

- The Creator class declares the factory method that is supposed to return an
- object of a Product class. The Creator's subclasses usually provide the
- implementation of this method.
  _/
  abstract class Creator {
  /\*\*
  _ Note that the Creator may also provide some default implementation of the
  _ factory method.
  _/
  public abstract factoryMethod(): Product;

      /**
       * Also note that, despite its name, the Creator's primary responsibility is
       * not creating products. Usually, it contains some core business logic that
       * relies on Product objects, returned by the factory method. Subclasses can
       * indirectly change that business logic by overriding the factory method
       * and returning a different type of product from it.
       */
      public someOperation(): string {
          // Call the factory method to create a Product object.
          const product = this.factoryMethod();
          // Now, use the product.
          return `Creator: The same creator's code has just worked with ${product.operation()}`;
      }

  }

/\*\*

- Concrete Creators override the factory method in order to change the
- resulting product's type.
  _/
  class ConcreteCreator1 extends Creator {
  /\*\*
  _ Note that the signature of the method still uses the abstract product
  _ type, even though the concrete product is actually returned from the
  _ method. This way the Creator can stay independent of concrete product
  _ classes.
  _/
  public factoryMethod(): Product {
  return new ConcreteProduct1();
  }
  }

class ConcreteCreator2 extends Creator {
public factoryMethod(): Product {
return new ConcreteProduct2();
}
}

/\*\*

- The Product interface declares the operations that all concrete products must
- implement.
  \*/
  interface Product {
  operation(): string;
  }

/\*\*

- Concrete Products provide various implementations of the Product interface.
  \*/
  class ConcreteProduct1 implements Product {
  public operation(): string {
  return '{Result of the ConcreteProduct1}';
  }
  }

class ConcreteProduct2 implements Product {
public operation(): string {
return '{Result of the ConcreteProduct2}';
}
}

/\*\*

- The client code works with an instance of a concrete creator, albeit through
- its base interface. As long as the client keeps working with the creator via
- the base interface, you can pass it any creator's subclass.
  \*/
  function clientCode(creator: Creator) {
  // ...
  console.log('Client: I\'m not aware of the creator\'s class, but it still works.');
  console.log(creator.someOperation());
  // ...
  }

/\*\*

- The Application picks a creator's type depending on the configuration or
- environment.
  \*/
  console.log('App: Launched with the ConcreteCreator1.');
  clientCode(new ConcreteCreator1());
  console.log('');

console.log('App: Launched with the ConcreteCreator2.');
clientCode(new ConcreteCreator2());
/\*\*

- Flyweight Design Pattern
-
- Intent: Lets you fit more objects into the available amount of RAM by sharing
- common parts of state between multiple objects, instead of keeping all of the
- data in each object.
  \*/

/\*\*

- The Flyweight stores a common portion of the state (also called intrinsic
- state) that belongs to multiple real business entities. The Flyweight accepts
- the rest of the state (extrinsic state, unique for each entity) via its
- method parameters.
  \*/
  class Flyweight {
  private sharedState: any;

      constructor(sharedState: any) {
          this.sharedState = sharedState;
      }

      public operation(uniqueState): void {
          const s = JSON.stringify(this.sharedState);
          const u = JSON.stringify(uniqueState);
          console.log(`Flyweight: Displaying shared (${s}) and unique (${u}) state.`);
      }

  }

/\*\*

- The Flyweight Factory creates and manages the Flyweight objects. It ensures
- that flyweights are shared correctly. When the client requests a flyweight,
- the factory either returns an existing instance or creates a new one, if it
- doesn't exist yet.
  \*/
  class FlyweightFactory {
  private flyweights: {[key: string]: Flyweight} = <any>{};

      constructor(initialFlyweights: string[][]) {
          for (const state of initialFlyweights) {
              this.flyweights[this.getKey(state)] = new Flyweight(state);
          }
      }

      /**
       * Returns a Flyweight's string hash for a given state.
       */
      private getKey(state: string[]): string {
          return state.join('_');
      }

      /**
       * Returns an existing Flyweight with a given state or creates a new one.
       */
      public getFlyweight(sharedState: string[]): Flyweight {
          const key = this.getKey(sharedState);

          if (!(key in this.flyweights)) {
              console.log('FlyweightFactory: Can\'t find a flyweight, creating new one.');
              this.flyweights[key] = new Flyweight(sharedState);
          } else {
              console.log('FlyweightFactory: Reusing existing flyweight.');
          }

          return this.flyweights[key];
      }

      public listFlyweights(): void {
          const count = Object.keys(this.flyweights).length;
          console.log(`\nFlyweightFactory: I have ${count} flyweights:`);
          for (const key in this.flyweights) {
              console.log(key);
          }
      }

  }

/\*\*

- The client code usually creates a bunch of pre-populated flyweights in the
- initialization stage of the application.
  \*/
  const factory = new FlyweightFactory([
  ['Chevrolet', 'Camaro2018', 'pink'],
  ['Mercedes Benz', 'C300', 'black'],
  ['Mercedes Benz', 'C500', 'red'],
  ['BMW', 'M5', 'red'],
  ['BMW', 'X6', 'white'],
  // ...
  ]);
  factory.listFlyweights();

// ...

function addCarToPoliceDatabase(
ff: FlyweightFactory, plates: string, owner: string,
brand: string, model: string, color: string,
) {
console.log('\nClient: Adding a car to database.');
const flyweight = ff.getFlyweight([brand, model, color]);

    // The client code either stores or calculates extrinsic state and passes it
    // to the flyweight's methods.
    flyweight.operation([plates, owner]);

}

addCarToPoliceDatabase(factory, 'CL234IR', 'James Doe', 'BMW', 'M5', 'red');

addCarToPoliceDatabase(factory, 'CL234IR', 'James Doe', 'BMW', 'X1', 'red');

factory.listFlyweights();
/\*\*

- Iterator Design Pattern
-
- Intent: Lets you traverse elements of a collection without exposing its
- underlying representation (list, stack, tree, etc.).
  \*/

interface Iterator<T> {
// Return the current element.
current(): T;

    // Return the current element and move forward to next element.
    next(): T;

    // Return the key of the current element.
    key(): number;

    // Checks if current position is valid.
    valid(): boolean;

    // Rewind the Iterator to the first element.
    rewind(): void;

}

interface Aggregator {
// Retrieve an external iterator.
getIterator(): Iterator<string>;
}

/\*\*

- Concrete Iterators implement various traversal algorithms. These classes
- store the current traversal position at all times.
  \*/

class AlphabeticalOrderIterator implements Iterator<string> {
private collection: WordsCollection;

    /**
     * Stores the current traversal position. An iterator may have a lot of
     * other fields for storing iteration state, especially when it is supposed
     * to work with a particular kind of collection.
     */
    private position: number = 0;

    /**
     * This variable indicates the traversal direction.
     */
    private reverse: boolean = false;

    constructor(collection: WordsCollection, reverse: boolean = false) {
        this.collection = collection;
        this.reverse = reverse;

        if (reverse) {
            this.position = collection.getCount() - 1;
        }
    }

    public rewind() {
        this.position = this.reverse ?
            this.collection.getCount() - 1 :
            0;
    }

    public current(): string {
        return this.collection.getItems()[this.position];
    }

    public key(): number {
        return this.position;
    }

    public next(): string {
        const item = this.collection.getItems()[this.position];
        this.position += this.reverse ? -1 : 1;
        return item;
    }

    public valid(): boolean {
        if (this.reverse) {
            return this.position >= 0;
        }

        return this.position < this.collection.getCount();
    }

}

/\*\*

- Concrete Collections provide one or several methods for retrieving fresh
- iterator instances, compatible with the collection class.
  \*/
  class WordsCollection implements Aggregator {
  private items: string[] = [];

      public getItems(): string[] {
          return this.items;
      }

      public getCount(): number {
          return this.items.length;
      }

      public addItem(item: string): void {
          this.items.push(item);
      }

      public getIterator(): Iterator<string> {
          return new AlphabeticalOrderIterator(this);
      }

      public getReverseIterator(): Iterator<string> {
          return new AlphabeticalOrderIterator(this, true);
      }

  }

/\*\*

- The client code may or may not know about the Concrete Iterator or Collection
- classes, depending on the level of indirection you want to keep in your
- program.
  \*/
  const collection = new WordsCollection();
  collection.addItem('First');
  collection.addItem('Second');
  collection.addItem('Third');

const iterator = collection.getIterator();

console.log('Straight traversal:');
while (iterator.valid()) {
console.log(iterator.next());
}

console.log('');
console.log('Reverse traversal:');
const reverseIterator = collection.getReverseIterator();
while (reverseIterator.valid()) {
console.log(reverseIterator.next());
}
/\*\*

- Mediator Design Pattern
-
- Intent: Lets you reduce chaotic dependencies between objects. The pattern
- restricts direct communications between the objects and forces them to
- collaborate only via a mediator object.
  \*/

/\*\*

- The Mediator interface declares a method used by components to notify the
- mediator about various events. The Mediator may react to these events and
- pass the execution to other components.
  \*/
  interface Mediator {
  notify(sender: object, event: string): void;
  }

/\*\*

- Concrete Mediators implement cooperative behavior by coordinating several
- components.
  \*/
  class ConcreteMediator implements Mediator {
  private component1: Component1;

      private component2: Component2;

      constructor(c1: Component1, c2: Component2) {
          this.component1 = c1;
          this.component1.setMediator(this);
          this.component2 = c2;
          this.component2.setMediator(this);
      }

      public notify(sender: object, event: string): void {
          if (event === 'A') {
              console.log('Mediator reacts on A and triggers following operations:');
              this.component2.doC();
          }

          if (event === 'D') {
              console.log('Mediator reacts on D and triggers following operations:');
              this.component1.doB();
              this.component2.doC();
          }
      }

  }

/\*\*

- The Base Component provides the basic functionality of storing a mediator's
- instance inside component objects.
  \*/
  class BaseComponent {
  protected mediator: Mediator;

      constructor(mediator: Mediator = null) {
          this.mediator = mediator;
      }

      public setMediator(mediator: Mediator): void {
          this.mediator = mediator;
      }

  }

/\*\*

- Concrete Components implement various functionality. They don't depend on
- other components. They also don't depend on any concrete mediator classes.
  \*/
  class Component1 extends BaseComponent {
  public doA(): void {
  console.log('Component 1 does A.');
  this.mediator.notify(this, 'A');
  }

      public doB(): void {
          console.log('Component 1 does B.');
          this.mediator.notify(this, 'B');
      }

  }

class Component2 extends BaseComponent {
public doC(): void {
console.log('Component 2 does C.');
this.mediator.notify(this, 'C');
}

    public doD(): void {
        console.log('Component 2 does D.');
        this.mediator.notify(this, 'D');
    }

}

/\*\*

- The client code.
  \*/
  const c1 = new Component1();
  const c2 = new Component2();
  const mediator = new ConcreteMediator(c1, c2);

console.log('Client triggers operation A.');
c1.doA();

console.log('');
console.log('Client triggers operation D.');
c2.doD();
/\*\*

- Memento Design Pattern
-
- Intent: Lets you save and restore the previous state of an object without
- revealing the details of its implementation.
  \*/

/\*\*

- The Originator holds some important state that may change over time. It also
- defines a method for saving the state inside a memento and another method for
- restoring the state from it.
  _/
  class Originator {
  /\*\*
  _ For the sake of simplicity, the originator's state is stored inside a
  _ single variable.
  _/
  private state: string;

      constructor(state: string) {
          this.state = state;
          console.log(`Originator: My initial state is: ${state}`);
      }

      /**
       * The Originator's business logic may affect its internal state. Therefore,
       * the client should backup the state before launching methods of the
       * business logic via the save() method.
       */
      public doSomething(): void {
          console.log('Originator: I\'m doing something important.');
          this.state = this.generateRandomString(30);
          console.log(`Originator: and my state has changed to: ${this.state}`);
      }

      private generateRandomString(length: number = 10): string {
          const charSet = 'abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ';

          return Array
              .apply(null, { length })
              .map(() => charSet.charAt(Math.floor(Math.random() * charSet.length)))
              .join('');
      }

      /**
       * Saves the current state inside a memento.
       */
      public save(): Memento {
          return new ConcreteMemento(this.state);
      }

      /**
       * Restores the Originator's state from a memento object.
       */
      public restore(memento: Memento): void {
          this.state = memento.getState();
          console.log(`Originator: My state has changed to: ${this.state}`);
      }

  }

/\*\*

- The Memento interface provides a way to retrieve the memento's metadata, such
- as creation date or name. However, it doesn't expose the Originator's state.
  \*/
  interface Memento {
  getState(): string;

      getName(): string;

      getDate(): string;

  }

/\*\*

- The Concrete Memento contains the infrastructure for storing the Originator's
- state.
  \*/
  class ConcreteMemento implements Memento {
  private state: string;

      private date: string;

      constructor(state: string) {
          this.state = state;
          this.date = new Date().toISOString().slice(0, 19).replace('T', ' ');
      }

      /**
       * The Originator uses this method when restoring its state.
       */
      public getState(): string {
          return this.state;
      }

      /**
       * The rest of the methods are used by the Caretaker to display metadata.
       */
      public getName(): string {
          return `${this.date} / (${this.state.substr(0, 9)}...)`;
      }

      public getDate(): string {
          return this.date;
      }

  }

/\*\*

- The Caretaker doesn't depend on the Concrete Memento class. Therefore, it
- doesn't have access to the originator's state, stored inside the memento. It
- works with all mementos via the base Memento interface.
  \*/
  class Caretaker {
  private mementos: Memento[] = [];

      private originator: Originator;

      constructor(originator: Originator) {
          this.originator = originator;
      }

      public backup(): void {
          console.log('\nCaretaker: Saving Originator\'s state...');
          this.mementos.push(this.originator.save());
      }

      public undo(): void {
          if (!this.mementos.length) {
              return;
          }
          const memento = this.mementos.pop();

          console.log(`Caretaker: Restoring state to: ${memento.getName()}`);
          this.originator.restore(memento);
      }

      public showHistory(): void {
          console.log('Caretaker: Here\'s the list of mementos:');
          for (const memento of this.mementos) {
              console.log(memento.getName());
          }
      }

  }

/\*\*

- Client code.
  \*/
  const originator = new Originator('Super-duper-super-puper-super.');
  const caretaker = new Caretaker(originator);

caretaker.backup();
originator.doSomething();

caretaker.backup();
originator.doSomething();

caretaker.backup();
originator.doSomething();

console.log('');
caretaker.showHistory();

console.log('\nClient: Now, let\'s rollback!\n');
caretaker.undo();

console.log('\nClient: Once more!\n');
caretaker.undo();
/\*\*

- Observer Design Pattern
-
- Intent: Lets you define a subscription mechanism to notify multiple objects
- about any events that happen to the object they're observing.
-
- Note that there's a lot of different terms with similar meaning associated
- with this pattern. Just remember that the Subject is also called the
- Publisher and the Observer is often called the Subscriber and vice versa.
- Also the verbs "observe", "listen" or "track" usually mean the same thing.
  \*/

/\*\*

- The Subject interface declares a set of methods for managing subscribers.
  \*/
  interface Subject {
  // Attach an observer to the subject.
  attach(observer: Observer): void;

      // Detach an observer from the subject.
      detach(observer: Observer): void;

      // Notify all observers about an event.
      notify(): void;

  }

/\*\*

- The Subject owns some important state and notifies observers when the state
- changes.
  _/
  class ConcreteSubject implements Subject {
  /\*\*
  _ @type {number} For the sake of simplicity, the Subject's state, essential
  _ to all subscribers, is stored in this variable.
  _/
  public state: number;

      /**
       * @type {Observer[]} List of subscribers. In real life, the list of
       * subscribers can be stored more comprehensively (categorized by event
       * type, etc.).
       */
      private observers: Observer[] = [];

      /**
       * The subscription management methods.
       */
      public attach(observer: Observer): void {
          const isExist = this.observers.includes(observer);
          if (isExist) {
              return console.log('Subject: Observer has been attached already.');
          }

          console.log('Subject: Attached an observer.');
          this.observers.push(observer);
      }

      public detach(observer: Observer): void {
          const observerIndex = this.observers.indexOf(observer);
          if (observerIndex === -1) {
              return console.log('Subject: Nonexistent observer.');
          }

          this.observers.splice(observerIndex, 1);
          console.log('Subject: Detached an observer.');
      }

      /**
       * Trigger an update in each subscriber.
       */
      public notify(): void {
          console.log('Subject: Notifying observers...');
          for (const observer of this.observers) {
              observer.update(this);
          }
      }

      /**
       * Usually, the subscription logic is only a fraction of what a Subject can
       * really do. Subjects commonly hold some important business logic, that
       * triggers a notification method whenever something important is about to
       * happen (or after it).
       */
      public someBusinessLogic(): void {
          console.log('\nSubject: I\'m doing something important.');
          this.state = Math.floor(Math.random() * (10 + 1));

          console.log(`Subject: My state has just changed to: ${this.state}`);
          this.notify();
      }

  }

/\*\*

- The Observer interface declares the update method, used by subjects.
  \*/
  interface Observer {
  // Receive update from subject.
  update(subject: Subject): void;
  }

/\*\*

- Concrete Observers react to the updates issued by the Subject they had been
- attached to.
  \*/
  class ConcreteObserverA implements Observer {
  public update(subject: Subject): void {
  if (subject instanceof ConcreteSubject && subject.state < 3) {
  console.log('ConcreteObserverA: Reacted to the event.');
  }
  }
  }

class ConcreteObserverB implements Observer {
public update(subject: Subject): void {
if (subject instanceof ConcreteSubject && (subject.state === 0 || subject.state >= 2)) {
console.log('ConcreteObserverB: Reacted to the event.');
}
}
}

/\*\*

- The client code.
  \*/

const subject = new ConcreteSubject();

const observer1 = new ConcreteObserverA();
subject.attach(observer1);

const observer2 = new ConcreteObserverB();
subject.attach(observer2);

subject.someBusinessLogic();
subject.someBusinessLogic();

subject.detach(observer2);

subject.someBusinessLogic();
/\*\*

- Prototype Design Pattern
-
- Intent: Lets you copy existing objects without making your code dependent on
- their classes.
  \*/

/\*\*

- The example class that has cloning ability. We'll see how the values of field
- with different types will be cloned.
  \*/
  class Prototype {
  public primitive: any;
  public component: object;
  public circularReference: ComponentWithBackReference;

      public clone(): this {
          const clone = Object.create(this);

          clone.component = Object.create(this.component);

          // Cloning an object that has a nested object with backreference
          // requires special treatment. After the cloning is completed, the
          // nested object should point to the cloned object, instead of the
          // original object. Spread operator can be handy for this case.
          clone.circularReference = {
              ...this.circularReference,
              prototype: { ...this },
          };

          return clone;
      }

  }

class ComponentWithBackReference {
public prototype;

    constructor(prototype: Prototype) {
        this.prototype = prototype;
    }

}

/\*\*

- The client code.
  \*/
  function clientCode() {
  const p1 = new Prototype();
  p1.primitive = 245;
  p1.component = new Date();
  p1.circularReference = new ComponentWithBackReference(p1);

      const p2 = p1.clone();
      if (p1.primitive === p2.primitive) {
          console.log('Primitive field values have been carried over to a clone. Yay!');
      } else {
          console.log('Primitive field values have not been copied. Booo!');
      }
      if (p1.component === p2.component) {
          console.log('Simple component has not been cloned. Booo!');
      } else {
          console.log('Simple component has been cloned. Yay!');
      }

      if (p1.circularReference === p2.circularReference) {
          console.log('Component with back reference has not been cloned. Booo!');
      } else {
          console.log('Component with back reference has been cloned. Yay!');
      }

      if (p1.circularReference.prototype === p2.circularReference.prototype) {
          console.log('Component with back reference is linked to original object. Booo!');
      } else {
          console.log('Component with back reference is linked to the clone. Yay!');
      }

  }

clientCode();
/\*\*

- Proxy Design Pattern
-
- Intent: Provide a surrogate or placeholder for another object to control
- access to the original object or to add other responsibilities.
  \*/

/\*\*

- The Subject interface declares common operations for both RealSubject and the
- Proxy. As long as the client works with RealSubject using this interface,
- you'll be able to pass it a proxy instead of a real subject.
  \*/
  interface Subject {
  request(): void;
  }

/\*\*

- The RealSubject contains some core business logic. Usually, RealSubjects are
- capable of doing some useful work which may also be very slow or sensitive -
- e.g. correcting input data. A Proxy can solve these issues without any
- changes to the RealSubject's code.
  \*/
  class RealSubject implements Subject {
  public request(): void {
  console.log('RealSubject: Handling request.');
  }
  }

/\*\*

- The Proxy has an interface identical to the RealSubject.
  \*/
  class Proxy implements Subject {
  private realSubject: RealSubject;

      /**
       * The Proxy maintains a reference to an object of the RealSubject class. It
       * can be either lazy-loaded or passed to the Proxy by the client.
       */
      constructor(realSubject: RealSubject) {
          this.realSubject = realSubject;
      }

      /**
       * The most common applications of the Proxy pattern are lazy loading,
       * caching, controlling the access, logging, etc. A Proxy can perform one of
       * these things and then, depending on the result, pass the execution to the
       * same method in a linked RealSubject object.
       */
      public request(): void {
          if (this.checkAccess()) {
              this.realSubject.request();
              this.logAccess();
          }
      }

      private checkAccess(): boolean {
          // Some real checks should go here.
          console.log('Proxy: Checking access prior to firing a real request.');

          return true;
      }

      private logAccess(): void {
          console.log('Proxy: Logging the time of request.');
      }

  }

/\*\*

- The client code is supposed to work with all objects (both subjects and
- proxies) via the Subject interface in order to support both real subjects and
- proxies. In real life, however, clients mostly work with their real subjects
- directly. In this case, to implement the pattern more easily, you can extend
- your proxy from the real subject's class.
  \*/
  function clientCode(subject: Subject) {
  // ...

      subject.request();

      // ...

  }

console.log('Client: Executing the client code with a real subject:');
const realSubject = new RealSubject();
clientCode(realSubject);

console.log('');

console.log('Client: Executing the same client code with a proxy:');
const proxy = new Proxy(realSubject);
clientCode(proxy);
/\*\*

- Singleton Design Pattern
-
- Intent: Lets you ensure that a class has only one instance, while providing a
- global access point to this instance.
  \*/

/\*\*

- The Singleton class defines the `getInstance` method that lets clients access
- the unique singleton instance.
  \*/
  class Singleton {
  private static instance: Singleton;

      /**
       * The Singleton's constructor should always be private to prevent direct
       * construction calls with the `new` operator.
       */
      private constructor() { }

      /**
       * The static method that controls the access to the singleton instance.
       *
       * This implementation let you subclass the Singleton class while keeping
       * just one instance of each subclass around.
       */
      public static getInstance(): Singleton {
          if (!Singleton.instance) {
              Singleton.instance = new Singleton();
          }

          return Singleton.instance;
      }

      /**
       * Finally, any singleton should define some business logic, which can be
       * executed on its instance.
       */
      public someBusinessLogic() {
          // ...
      }

  }

/\*\*

- The client code.
  \*/
  function clientCode() {
  const s1 = Singleton.getInstance();
  const s2 = Singleton.getInstance();

      if (s1 === s2) {
          console.log('Singleton works, both variables contain the same instance.');
      } else {
          console.log('Singleton failed, variables contain different instances.');
      }

  }

clientCode();
/\*\*

- State Design Pattern
-
- Intent: Lets an object alter its behavior when its internal state changes. It
- appears as if the object changed its class.
  \*/

/\*\*

- The Context defines the interface of interest to clients. It also maintains a
- reference to an instance of a State subclass, which represents the current
- state of the Context.
  _/
  class Context {
  /\*\*
  _ @type {State} A reference to the current state of the Context.
  \*/
  private state: State;

      constructor(state: State) {
          this.transitionTo(state);
      }

      /**
       * The Context allows changing the State object at runtime.
       */
      public transitionTo(state: State): void {
          console.log(`Context: Transition to ${(<any>state).constructor.name}.`);
          this.state = state;
          this.state.setContext(this);
      }

      /**
       * The Context delegates part of its behavior to the current State object.
       */
      public request1(): void {
          this.state.handle1();
      }

      public request2(): void {
          this.state.handle2();
      }

  }

/\*\*

- The base State class declares methods that all Concrete State should
- implement and also provides a backreference to the Context object, associated
- with the State. This backreference can be used by States to transition the
- Context to another State.
  \*/
  abstract class State {
  protected context: Context;

      public setContext(context: Context) {
          this.context = context;
      }

      public abstract handle1(): void;

      public abstract handle2(): void;

  }

/\*\*

- Concrete States implement various behaviors, associated with a state of the
- Context.
  \*/
  class ConcreteStateA extends State {
  public handle1(): void {
  console.log('ConcreteStateA handles request1.');
  console.log('ConcreteStateA wants to change the state of the context.');
  this.context.transitionTo(new ConcreteStateB());
  }

      public handle2(): void {
          console.log('ConcreteStateA handles request2.');
      }

  }

class ConcreteStateB extends State {
public handle1(): void {
console.log('ConcreteStateB handles request1.');
}

    public handle2(): void {
        console.log('ConcreteStateB handles request2.');
        console.log('ConcreteStateB wants to change the state of the context.');
        this.context.transitionTo(new ConcreteStateA());
    }

}

/\*\*

- The client code.
  \*/
  const context = new Context(new ConcreteStateA());
  context.request1();
  context.request2();
  /\*\*
- Strategy Design Pattern
-
- Intent: Lets you define a family of algorithms, put each of them into a
- separate class, and make their objects interchangeable.
  \*/

/\*\*

- The Context defines the interface of interest to clients.
  _/
  class Context {
  /\*\*
  _ @type {Strategy} The Context maintains a reference to one of the Strategy
  _ objects. The Context does not know the concrete class of a strategy. It
  _ should work with all strategies via the Strategy interface.
  \*/
  private strategy: Strategy;

      /**
       * Usually, the Context accepts a strategy through the constructor, but also
       * provides a setter to change it at runtime.
       */
      constructor(strategy: Strategy) {
          this.strategy = strategy;
      }

      /**
       * Usually, the Context allows replacing a Strategy object at runtime.
       */
      public setStrategy(strategy: Strategy) {
          this.strategy = strategy;
      }

      /**
       * The Context delegates some work to the Strategy object instead of
       * implementing multiple versions of the algorithm on its own.
       */
      public doSomeBusinessLogic(): void {
          // ...

          console.log('Context: Sorting data using the strategy (not sure how it\'ll do it)');
          const result = this.strategy.doAlgorithm(['a', 'b', 'c', 'd', 'e']);
          console.log(result.join(','));

          // ...
      }

  }

/\*\*

- The Strategy interface declares operations common to all supported versions
- of some algorithm.
-
- The Context uses this interface to call the algorithm defined by Concrete
- Strategies.
  \*/
  interface Strategy {
  doAlgorithm(data: string[]): string[];
  }

/\*\*

- Concrete Strategies implement the algorithm while following the base Strategy
- interface. The interface makes them interchangeable in the Context.
  \*/
  class ConcreteStrategyA implements Strategy {
  public doAlgorithm(data: string[]): string[] {
  return data.sort();
  }
  }

class ConcreteStrategyB implements Strategy {
public doAlgorithm(data: string[]): string[] {
return data.reverse();
}
}

/\*\*

- The client code picks a concrete strategy and passes it to the context. The
- client should be aware of the differences between strategies in order to make
- the right choice.
  \*/
  const context = new Context(new ConcreteStrategyA());
  console.log('Client: Strategy is set to normal sorting.');
  context.doSomeBusinessLogic();

console.log('');

console.log('Client: Strategy is set to reverse sorting.');
context.setStrategy(new ConcreteStrategyB());
context.doSomeBusinessLogic();
/\*\*

- Template Method Design Pattern
-
- Intent: Defines the skeleton of an algorithm in the superclass but lets
- subclasses override specific steps of the algorithm without changing its
- structure.
  \*/

/\*\*

- The Abstract Class defines a template method that contains a skeleton of some
- algorithm, composed of calls to (usually) abstract primitive operations.
-
- Concrete subclasses should implement these operations, but leave the template
- method itself intact.
  _/
  abstract class AbstractClass {
  /\*\*
  _ The template method defines the skeleton of an algorithm.
  \*/
  public templateMethod(): void {
  this.baseOperation1();
  this.requiredOperations1();
  this.baseOperation2();
  this.hook1();
  this.requiredOperation2();
  this.baseOperation3();
  this.hook2();
  }

      /**
       * These operations already have implementations.
       */
      protected baseOperation1(): void {
          console.log('AbstractClass says: I am doing the bulk of the work');
      }

      protected baseOperation2(): void {
          console.log('AbstractClass says: But I let subclasses override some operations');
      }

      protected baseOperation3(): void {
          console.log('AbstractClass says: But I am doing the bulk of the work anyway');
      }

      /**
       * These operations have to be implemented in subclasses.
       */
      protected abstract requiredOperations1(): void;

      protected abstract requiredOperation2(): void;

      /**
       * These are "hooks." Subclasses may override them, but it's not mandatory
       * since the hooks already have default (but empty) implementation. Hooks
       * provide additional extension points in some crucial places of the
       * algorithm.
       */
      protected hook1(): void { }

      protected hook2(): void { }

  }

/\*\*

- Concrete classes have to implement all abstract operations of the base class.
- They can also override some operations with a default implementation.
  \*/
  class ConcreteClass1 extends AbstractClass {
  protected requiredOperations1(): void {
  console.log('ConcreteClass1 says: Implemented Operation1');
  }

      protected requiredOperation2(): void {
          console.log('ConcreteClass1 says: Implemented Operation2');
      }

  }

/\*\*

- Usually, concrete classes override only a fraction of base class' operations.
  \*/
  class ConcreteClass2 extends AbstractClass {
  protected requiredOperations1(): void {
  console.log('ConcreteClass2 says: Implemented Operation1');
  }

      protected requiredOperation2(): void {
          console.log('ConcreteClass2 says: Implemented Operation2');
      }

      protected hook1(): void {
          console.log('ConcreteClass2 says: Overridden Hook1');
      }

  }

/\*\*

- The client code calls the template method to execute the algorithm. Client
- code does not have to know the concrete class of an object it works with, as
- long as it works with objects through the interface of their base class.
  \*/
  function clientCode(abstractClass: AbstractClass) {
  // ...
  abstractClass.templateMethod();
  // ...
  }

console.log('Same client code can work with different subclasses:');
clientCode(new ConcreteClass1());
console.log('');

console.log('Same client code can work with different subclasses:');
clientCode(new ConcreteClass2());
/\*\*

- Visitor Design Pattern
-
- Intent: Lets you separate algorithms from the objects on which they operate.
  \*/

/\*\*

- The Component interface declares an `accept` method that should take the base
- visitor interface as an argument.
  \*/
  interface Component {
  accept(visitor: Visitor): void;
  }

/\*\*

- Each Concrete Component must implement the `accept` method in such a way that
- it calls the visitor's method corresponding to the component's class.
  _/
  class ConcreteComponentA implements Component {
  /\*\*
  _ Note that we're calling `visitConcreteComponentA`, which matches the
  _ current class name. This way we let the visitor know the class of the
  _ component it works with.
  \*/
  public accept(visitor: Visitor): void {
  visitor.visitConcreteComponentA(this);
  }

      /**
       * Concrete Components may have special methods that don't exist in their
       * base class or interface. The Visitor is still able to use these methods
       * since it's aware of the component's concrete class.
       */
      public exclusiveMethodOfConcreteComponentA(): string {
          return 'A';
      }

  }

class ConcreteComponentB implements Component {
/\*\*
_ Same here: visitConcreteComponentB => ConcreteComponentB
_/
public accept(visitor: Visitor): void {
visitor.visitConcreteComponentB(this);
}

    public specialMethodOfConcreteComponentB(): string {
        return 'B';
    }

}

/\*\*

- The Visitor Interface declares a set of visiting methods that correspond to
- component classes. The signature of a visiting method allows the visitor to
- identify the exact class of the component that it's dealing with.
  \*/
  interface Visitor {
  visitConcreteComponentA(element: ConcreteComponentA): void;

      visitConcreteComponentB(element: ConcreteComponentB): void;

  }

/\*\*

- Concrete Visitors implement several versions of the same algorithm, which can
- work with all concrete component classes.
-
- You can experience the biggest benefit of the Visitor pattern when using it
- with a complex object structure, such as a Composite tree. In this case, it
- might be helpful to store some intermediate state of the algorithm while
- executing visitor's methods over various objects of the structure.
  \*/
  class ConcreteVisitor1 implements Visitor {
  public visitConcreteComponentA(element: ConcreteComponentA): void {
  console.log(`${element.exclusiveMethodOfConcreteComponentA()} + ConcreteVisitor1`);
  }

      public visitConcreteComponentB(element: ConcreteComponentB): void {
          console.log(`${element.specialMethodOfConcreteComponentB()} + ConcreteVisitor1`);
      }

  }

class ConcreteVisitor2 implements Visitor {
public visitConcreteComponentA(element: ConcreteComponentA): void {
console.log(`${element.exclusiveMethodOfConcreteComponentA()} + ConcreteVisitor2`);
}

    public visitConcreteComponentB(element: ConcreteComponentB): void {
        console.log(`${element.specialMethodOfConcreteComponentB()} + ConcreteVisitor2`);
    }

}

/\*\*

- The client code can run visitor operations over any set of elements without
- figuring out their concrete classes. The accept operation directs a call to
- the appropriate operation in the visitor object.
  \*/
  function clientCode(components: Component[], visitor: Visitor) {
  // ...
  for (const component of components) {
  component.accept(visitor);
  }
  // ...
  }

const components = [
new ConcreteComponentA(),
new ConcreteComponentB(),
];

console.log('The client code works with all visitors via the base Visitor interface:');
const visitor1 = new ConcreteVisitor1();
clientCode(components, visitor1);
console.log('');

console.log('It allows the same client code to work with different types of visitors:');
const visitor2 = new ConcreteVisitor2();
clientCode(components, visitor2);

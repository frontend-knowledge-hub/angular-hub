# Dynamic Rendering Components in Angular

In most cases, Angular components are declared via templates. However, there are situations where you need to create components dynamically — triggered by a click, an event, or based on configuration or conditions. For example:

- Modals and dialogs
- Toasts/snackbars
- Dynamically configurable UI blocks
- Lazily loaded elements

In this guide, we’ll cover three approaches to programmatic component creation:

1. Using the `NgComponentOutlet` directive — a declarative yet flexible method
2. Via `ViewContainerRef#createComponent()` — manually creating a component inside a template
3. Using `createComponent()` and `ApplicationRef` — for injecting components directly into the DOM (e.g., for a snackbar)

## Table of Contents

- [Creating a Test Component](#creating-a-test-component)
- [Dynamic Insertion with `NgComponentOutlet`](#dynamic-insertion-with-ngcomponentoutlet)
- [Using `ViewContainerRef`](#using-viewcontainerref)
- [Angular v20: bindings during component creation](#angular-v20-bindings-during-component-creation)
- [Using `createComponent` and `ApplicationRef`](#using-createcomponent-and-applicationref)
- [Summary](#summary)

## Creating a Test Component

For demonstration, we’ll create a `LoggerComponent` — a simple component that renders text and reacts to clicks.

```ts
export const LOGGER_MESSAGE = new InjectionToken<string>('LOGGER_MESSAGE');

@Component({
  selector: 'app-logger',
  imports: [],
  template: `<p>{{ text() }}</p>`,
  styleUrl: './logger.component.scss',
})
export class LoggerComponent {
  public text = input<string>();
  public interaction = output<void>();

  private logMessage = inject(LOGGER_MESSAGE, { optional: true });

  constructor() {
    if (this.logMessage) {
      console.log(this.logMessage);
    }
  }

  @HostListener('click')
  onClick() {
    this.interaction.emit();
  }
}
```

## Dynamic Insertion with `NgComponentOutlet`

`NgComponentOutlet` is a directive that allows dynamically embedding a component into the view:

```ts
@Component({
  imports: [NgComponentOutlet],
  template: `<div class="message-panel">
    <ng-container
      *ngComponentOutlet="
        component;
        inputs: inputs;
        injector: componentInjector
      "
    ></ng-container>
  </div>`,
  ...
})
export class MessagePanelComponent implements AfterViewInit {
  protected renderedComponent = viewChild<NgComponentOutlet<LoggerComponent>>(NgComponentOutlet);

  protected component = LoggerComponent;
  protected inputs = {
    text: 'This is a message from the message panel component.',
  };
  protected componentInjector: Injector;

  private injector = inject(Injector);
  private loggerMessage = 'LoggerComponent initialized with message from MessagePanelComponent';

  constructor() {
    this.componentInjector = Injector.create({
      providers: [{ provide: LOGGER_MESSAGE, useValue: this.loggerMessage }],
      parent: this.injector,
    });
  }

  ngAfterViewInit() {
    // Subscribe to the `interaction` event from the `LoggerComponent`
    this.renderedComponent()?.componentInstance?.interaction.subscribe(() => {
      console.log('Component click occurred');
    });
  }
}
```

The `ngComponentOutlet` directive accepts the following inputs to control component creation and configuration:

- `ngComponentOutletInputs`: Optional inputs object bound to the component
- `ngComponentOutletInjector`: Optional custom injector (defaults to the view container’s injector)
- `ngComponentOutletContent`: Optional projectable content nodes
- `ngComponentOutletNgModule`: Optional NgModule reference for dynamic module loading

While inputs can be provided declaratively using `ngComponentOutletInputs`, outputs (such as `@Output` events) must be handled imperatively by accessing the `componentInstance` property of the directive.
This is demonstrated in the `ngAfterViewInit()` method, where the interaction event is subscribed to manually.

> ⚠️ Angular automatically destroys components rendered via `*ngComponentOutlet` when the directive is removed from the view (e.g., using `*ngIf`) or when any of the bound inputs such as `component`, `inputs`, or `injector` change.
> This means no manual cleanup (like calling `destroy()`) is needed when replacing or removing dynamically inserted components.

## Using `ViewContainerRef`

In Angular, each template element has an associated **view container**, which allows dynamically adding views. In code, this is represented by `ViewContainerRef`.

Components and directives can inject `ViewContainerRef` to manage their view container.

### Inserting a Component into an Internal View Container

```ts
@Component({
  template: `<button (click)="showMessage()">Click to see a message</button> `,
  ...
})
export class DynamicMessagePanelComponent {
  private injector = inject(Injector);
  private destroyRef = inject(DestroyRef);
  private viewContainer = inject(ViewContainerRef);

  protected showMessage(): void {
    const injector = Injector.create({
      providers: [
        {
          provide: LOGGER_MESSAGE,
          useValue: 'Dynamic Message from DynamicMessagePanelComponent',
        },
      ],
      parent: this.injector,
    });

    const componentRef = this.viewContainer.createComponent(LoggerComponent, {
      injector,
    });
    componentRef.setInput('text', 'This is a dynamic message from the DynamicMessagePanelComponent.');
    componentRef.instance.interaction.subscribe(() => {
      console.log('Component click occurred');
    });
    this.destroyRef.onDestroy(() => componentRef.destroy()); // destroy the component when the view is destroyed
  }
}
```

Calling `createComponent()` returns a `ComponentRef`, representing the created component. Inputs are set via `setInput()`, and outputs can be subscribed to through the `instance`.

> ⚠️ Components created via `viewContainerRef.createComponent()` are **not destroyed automatically**. If you're rendering temporary elements (e.g., snackbars, dialogs), make sure to manually clean them up using one of the following:
>
> - `componentRef.destroy()` — destroy a specific component
> - `viewContainerRef.remove(index)` — remove by index
> - `viewContainerRef.clear()` — remove all components in the container

### Inserting a Component at a Specific Template Location

To render a dynamic component at a specific place in the template, you can use an `ng-container`:

```ts
@Component({
  ...
  template: ` <button (click)="showMessage()">Click to see a message</button>
    <div class="message-container">
      <ng-container #messages></ng-container>
    </div>`,
})
export class DynamicMessagePanelComponent {
  ...

  private viewContainer = viewChild.required('messages', {
    read: ViewContainerRef,
  });

  protected showMessage(): void {
    ...

    const componentRef = this.viewContainer().createComponent(LoggerComponent, {
      injector,
    });

    ...
  }
}
```

## Angular v20: Bindings During Component Creation

As of Angular 20, inputs and outputs can be passed directly when calling `createComponent()`. This removes the need for `setInput()` and manual output subscriptions — everything can be declared inline.

```ts
import { Component, Injector, ViewContainerRef, inject, inputBinding, outputBinding, signal } from '@angular/core';

...

@Component({
  /* ... */
})
export class DynamicMessagePanelComponent {
  private injector = inject(Injector);
  private viewContainer = viewChild.required('messages', {
    read: ViewContainerRef,
  });
  private text = signal('This is a dynamic message from the DynamicMessagePanelComponent.');

  protected showMessage(): void {
    const injector = Injector.create({
      providers: [
        {
          provide: LOGGER_MESSAGE,
          useValue: 'Dynamic Message from DynamicMessagePanelComponent',
        },
      ],
      parent: this.injector,
    });

    // 👇 BEFORE Angular v20
    /*
    const componentRef = this.viewContainer().createComponent(LoggerComponent, {
      injector,
    });
    componentRef.setInput('text', this.text());
    componentRef.instance.interaction.subscribe(() => {
      console.log('Component click occurred');
    });
    */

    // ✅ AFTER Angular v20
    const componentRef = this.viewContainer().createComponent(LoggerComponent, {
      injector,
      bindings: [
        inputBinding('text', this.text),
        outputBinding('interaction', () => {
          console.log('Component click occurred');
        }),
      ],
    });
  }
}
```

> 💡 Note: `this.text` is passed as a signal, not `this.text()` — this is important for reactivity.

## Using `createComponent` and `ApplicationRef`

Let’s look at a more advanced method using `createComponent()` and `ApplicationRef`. This is useful when a component should exist outside the usual Angular template structure, such as:

- Modal dialogs
- Toast notifications
- Tooltips
- Full-screen overlays

Below is an example of building a `SnackBar` service.

**SnackBarComponent**

```ts
@Component({
  template: `
    <div class="snack-bar-container">
      <div class="snack-bar">
        <p class="snack-bar-message">{{ message() }}</p>

        <button type="button" (click)="close.emit()">&times;</button>
      </div>
    </div>
  `,
  ...
})
export class SnackBarComponent {
  public message = input.required<string>();
  public close = output<void>();
}
```

**SnackBarService**

```ts
@Injectable({
  providedIn: 'root',
})
export class SnackBarService {
  private appRef = inject(ApplicationRef);
  private document = inject(DOCUMENT);

  constructor() {}

  public show(message: string): void {
    const component = this.showSnackBar(message);
    this.watchClose(component);
  }

  private showSnackBar(message: string): ComponentRef<SnackBarComponent> {
    const component = createComponent(SnackBarComponent, {
      environmentInjector: this.appRef.injector,
    });
    this.appRef.attachView(component.hostView);
    component.setInput('message', message);
    this.document.body.appendChild(component.location.nativeElement);
    return component;
  }

  private watchClose(component: ComponentRef<SnackBarComponent>): void {
    const sub = component.instance.close.subscribe(() => {
      this.appRef.detachView(component.hostView);
      component.destroy();
      sub.unsubscribe();
    });
  }
}
```

The key method is `showSnackBar()`:

- `createComponent()` — creates the component and requires an `environmentInjector` since it’s detached from any template
- `attachView()` — attaches the component view to Angular’s change detection
- `appendChild()` — manually appends the DOM node to the document

> ⚠️ When creating components outside the Angular view tree using `createComponent()` and `ApplicationRef.attachView()`, you are entirely responsible for their **cleanup**. Once the component is no longer needed, you must explicitly perform two actions:
>
> 1.  Call `applicationRef.detachView(componentRef.hostView)` to remove the component's view from Angular's change detection.
> 2.  Then, call `componentRef.destroy()` to clean up the component and free up resources.

**SnackBar Usage Example**

```ts
@Component({
  template: ` <button (click)="openSnackBar()">Open Snack Bar</button> `,
})
export class SnackBarPageComponent {
  private snackBarService = inject(SnackBarService);

  protected openSnackBar(): void {
    this.snackBarService.show('This is a test message for the snack bar.');
  }
}
```

> 💡 Use this approach only when a component can’t be tied to a `ViewContainerRef` — e.g., it needs to be globally rendered outside the current component structure.

## Summary

Programmatic component rendering in Angular is a powerful tool for dynamically managing the UI beyond static templates. It’s especially useful for creating modals, notifications, plugin-like UIs, or any behavior that can't be predefined at compile time.

Key takeaways:

- `NgComponentOutlet` — great when the component is known, but its config is dynamic
- `ViewContainerRef` — the foundation for most dynamic rendering use cases
- `createComponent` + `ApplicationRef` — ideal for components outside the regular DOM structure (overlays, snackbars, etc.)

With Angular 20, the new `bindings` support makes dynamic component creation even cleaner and more expressive.

Choose your method based on context and the control you need — Angular gives you all the tools for flexible and maintainable dynamic rendering.

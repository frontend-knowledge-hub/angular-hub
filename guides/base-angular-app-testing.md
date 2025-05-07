# Basic Angular Application Unit Testing

Testing is a crucial part of the application development process, but unfortunately, many developers either neglect it or ignore it completely. Well-designed tests help keep the application stable, make the code cleaner and more understandable, and allow changes and scaling without fear of breaking things.
In this short guide, we’ll cover the fundamental principles of unit testing in Angular.

## Getting Started

By default, Angular projects use the Jasmine testing framework. While many modern frontend teams prefer [Jest](https://jestjs.io/) or the rapidly growing [Vitest](https://vitest.dev/), in this guide we’ll focus on the built-in tools.

## Testing Components

Let's start by creating a simple counter component with three actions: increment, decrement, and reset the counter value.

TypeScript file:

```ts
@Component({ ... })
export class CounterComponent {
  protected count = signal(0);

  protected increment(): void {
    this.count.update((value) => value + 1);
  }

  protected decrement(): void {
    this.count.update((value) => value - 1);
  }

  protected reset(): void {
    this.count.set(0);
  }
}
```

HTML file:

```html
<div class="counter">
  <h1 class="text-center" data-testid="count">{{count()}}</h1>

  <div class="counter-controls">
    <button (click)="increment()" data-testid="increment">Increment</button>
    <button (click)="decrement()" data-testid="decrement">Decrement</button>
    <button (click)="reset()" data-testid="reset">Reset</button>
  </div>
</div>
```

Notice that we added `data-testid` attributes to important UI elements — these will help us during testing.

Now let’s look at the `counter.component.spec.ts` file:

```ts
import { ComponentFixture, TestBed } from '@angular/core/testing';

import { CounterComponent } from './counter.component';

describe('CounterComponent', () => {
  let component: CounterComponent;
  let fixture: ComponentFixture<CounterComponent>;

  beforeEach(async () => {
    await TestBed.configureTestingModule({
      imports: [CounterComponent],
    }).compileComponents();

    fixture = TestBed.createComponent(CounterComponent);
    component = fixture.componentInstance;
    fixture.detectChanges();
  });

  it('should create', () => {
    expect(component).toBeTruthy();
  });
});
```

The first thing to notice is the `TestBed` object from Angular's testing library.
`TestBed` is a key utility that allows setting up the testing environment and provides methods for creating components and services inside tests.

Here's what happens step-by-step:

- We configure the testing module and import the component under test.
- We call `compileComponents()` to compile the component's template and styles.
- We create the component with `TestBed.createComponent`.
- We access the instance via `fixture.componentInstance`.
- Finally, we trigger data binding with `fixture.detectChanges()`.

This setup happens in a `beforeEach`, ensuring a fresh instance for each test.

Now let’s add tests. First, we’ll check that the counter starts at zero:

```ts
import { ComponentFixture, TestBed } from '@angular/core/testing';
import { By } from '@angular/platform-browser';

import { CounterComponent } from './counter.component';

describe('CounterComponent', () => {
  ...

  it('should show default count', () => {
    const countDe = fixture.debugElement.query(By.css('[data-testid="count"]'));
    expect(countDe.nativeElement.textContent).toBe('0');
  });
});

```

To interact with the component’s DOM, we use the `debugElement` property of the `fixture` object, which gives us access to the root element. To locate a specific element, we call the `query` method. This method expects a predicate function that returns `true` for matching nodes in the `debugElement` tree. The `By.css` helper makes it easy to create such predicates based on CSS selectors.

Keep in mind that `query` returns a `DebugElement`, not a raw DOM element. To access the actual DOM node, we use the `nativeElement` property. From there, we can read properties like `textContent` to assert the expected values.

Now, let’s move on and see how to test user interactions. In our example, there are three buttons: `Increment`, `Decrement`, and `Reset`. We'll start by testing the first two buttons:

```ts
describe('CounterComponent', () => {
  ...

  it('should increment count', () => {
    const incrementButton = fixture.debugElement.query(By.css('[data-testid="increment"]'));
    const countDe = fixture.debugElement.query(By.css('[data-testid="count"]'));
    incrementButton.triggerEventHandler('click', null);
    fixture.detectChanges();
    expect(countDe.nativeElement.textContent).toBe('1');
  });

  it('should decrement count', () => {
    const incrementButton = fixture.debugElement.query(By.css('[data-testid="decrement"]'));
    const countDe = fixture.debugElement.query(By.css('[data-testid="count"]'));
    incrementButton.triggerEventHandler('click', null);
    fixture.detectChanges();
    expect(countDe.nativeElement.textContent).toBe('-1');
  });
});
```

Here, we use the same approach for accessing UI elements.
To simulate user interactions, we call the `triggerEventHandler` method, which triggers all event handlers attached to a specific event type — in our case, a _click_ event. The second argument is a fake event object. Since we don’t need any event details for this test, we simply pass `null`. After triggering the event, we call `fixture.detectChanges()` to update the view, and then we can verify that the counter shows the expected values: `1` or `-1`.

You might notice that writing tests this way leads to a lot of repetitive boilerplate code. To make tests cleaner and more maintainable, it's a good idea to create a small utilities file for common testing tasks. For example, it could include helper functions like:

```ts
import { DebugElement } from '@angular/core';
import { ComponentFixture } from '@angular/core/testing';
import { By } from '@angular/platform-browser';

export const findByTestId = <T>(fixture: ComponentFixture<T>, testId: string): DebugElement => {
  return fixture.debugElement.query(By.css(`[data-testid="${testId}"]`));
};

export const click = <T>(fixture: ComponentFixture<T>, testId: string, eventObj?: any): void => {
  const button = findByTestId(fixture, testId);
  button.triggerEventHandler('click', eventObj);
};
```

Using these utilities, the `Reset` button test becomes much cleaner:

```ts
describe('CounterComponent', () => {
  ...

  it('should reset count', () => {
    const countDe = findByTestId(fixture, 'count');
    click(fixture, 'increment', null);
    click(fixture, 'increment', null);
    click(fixture, 'increment', null);
    fixture.detectChanges();

    expect(countDe.nativeElement.textContent).toBe('3');
    click(fixture, 'reset', null);
    fixture.detectChanges();

    expect(countDe.nativeElement.textContent).toBe('0');
  });
});
```

Now we have confidence that our `CounterComponent` works correctly.

## Testing Services

Let's create a simple `CounterService`:

```ts
import { Injectable } from '@angular/core';
import { BehaviorSubject, Observable } from 'rxjs';

@Injectable({
  providedIn: 'root',
})
export class CounterService {
  private count = new BehaviorSubject<number>(0);

  constructor() {}

  public getCount(): Observable<number> {
    return this.count.asObservable();
  }

  public increment(): void {
    this.count.next(this.count.value + 1);
  }

  public decrement(): void {
    this.count.next(this.count.value - 1);
  }

  public reset(): void {
    this.count.next(0);
  }
}
```

Now, here's the basic test file:

```ts
import { TestBed } from '@angular/core/testing';

import { CounterService } from './counter.service';

describe('CounterService', () => {
  let service: CounterService;

  beforeEach(() => {
    TestBed.configureTestingModule({});
    service = TestBed.inject(CounterService);
  });

  it('should be created', () => {
    expect(service).toBeTruthy();
  });

  it('should be default count', (done) => {
    service.getCount().subscribe((count) => {
      expect(count).toBe(0);
      done();
    });
  });

  it('should increment count', (done) => {
    service.increment();

    service.getCount().subscribe((count) => {
      expect(count).toBe(1);
      done();
    });
  });

  it('should decrement count', (done) => {
    service.decrement();

    service.getCount().subscribe((count) => {
      expect(count).toBe(-1);
      done();
    });
  });

  it('should reset count', (done) => {
    service.reset();

    service.getCount().subscribe((count) => {
      expect(count).toBe(0);
      done();
    });
  });
});
```

Testing services in Angular is very similar to testing plain JavaScript classes, with the addition of dependency injection. If a service depends on another, like a `LoggerService`, you can mock it:

```ts
const mockLoggerService = {
  log: jasmine.createSpy('log'),
};

describe('CounterService', () => {
  let service: CounterService;

  beforeEach(() => {
    TestBed.configureTestingModule({
      providers: [
        {
          provide: LoggerService,
          useValue: mockLoggerService,
        },
      ],
    });
    service = TestBed.inject(CounterService);
  });

  ...
});
```

Also, notice that we call `done()` inside the `subscription` to notify Jasmine that the test has finished.

## Testing a Directive

Although Angular generates a test file for a directive automatically, in practice directives are often tested together with the components that use them.

Here’s an example directive that changes the background color depending on the counter value:

```ts
import { Directive, effect, HostBinding, input } from '@angular/core';

@Directive({
  selector: '[appHighlight]',
})
export class HighlightDirective {
  public count = input<number>(0);

  @HostBinding('style.backgroundColor') backgroundColor: string = 'transparent';

  constructor() {
    effect(() => {
      const inEven = this.count() % 2 === 0;
      this.backgroundColor = inEven ? 'lightblue' : 'coral';
    });
  }
}
```

Applied to the `CounterComponent`:

```ts
@Component({
  selector: 'app-counter',
  imports: [HighlightDirective],
  ...
})
export class CounterComponent {
  ...
}
```

Updated component template:

```html
<div class="counter">
  <h1 class="text-center" data-testid="count" appHighlight [count]="count()">{{count()}}</h1>
  ...
</div>
```

And here are the updated tests:

```ts
describe('CounterComponent', () => {
  ...

  beforeEach(async () => {
    TestBed.configureTestingModule({
      imports: [CounterComponent, HighlightDirective], // import the directive
    }).compileComponents();

    fixture = TestBed.createComponent(CounterComponent);
    component = fixture.componentInstance;
    fixture.detectChanges();
  });

  ...

  it('should set lightblue background color when count is even', () => {
    const countDe = findByTestId(fixture, 'count');

    expect(countDe.nativeElement.style.backgroundColor).toBe('lightblue');

    click(fixture, 'increment', null);
    click(fixture, 'increment', null);
    fixture.detectChanges();
    expect(countDe.nativeElement.style.backgroundColor).toBe('lightblue');
  });

  it('should set coral background color when count is odd', () => {
    const countDe = findByTestId(fixture, 'count');

    click(fixture, 'increment', null);
    fixture.detectChanges();
    expect(countDe.nativeElement.style.backgroundColor).toBe('coral');

    click(fixture, 'increment', null);
    click(fixture, 'increment', null);
    fixture.detectChanges();
    expect(countDe.nativeElement.style.backgroundColor).toBe('coral');
  });
});
```

By simulating user clicks and checking the element’s `backgroundColor` style, we can verify that the directive behaves correctly.

## Testing a Pipe

To demonstrate pipe testing, let’s create a simple `CapitalizePipe` that capitalizes the first letter of a string:

```ts
import { Pipe, PipeTransform } from '@angular/core';

@Pipe({
  name: 'capitalize',
})
export class CapitalizePipe implements PipeTransform {
  transform(value: string): string {
    if (!value) {
      return '';
    }
    return value.charAt(0).toUpperCase() + value.slice(1);
  }
}
```

Pipes can be tested in isolation since they don’t interact with the view — they simply return transformed values. However, isolated testing alone doesn’t guarantee that everything works as expected in the template.
That’s why I recommend a combined approach:

- test the logic in isolation
- test the pipe inside a component to verify how it behaves in the view.

Let’s start with an isolated unit test:

```ts
import { CapitalizePipe } from './capitalize.pipe';

describe('CapitalizePipe', () => {
  it('create an instance', () => {
    const pipe = new CapitalizePipe();
    expect(pipe).toBeTruthy();
  });

  it('should capitalize the first letter of a string', () => {
    const pipe = new CapitalizePipe();
    expect(pipe.transform('hello')).toBe('Hello');
  });
});
```

As you can see, testing a pipe in isolation is just like testing any regular JavaScript class that implements the `PipeTransform` interface.

If the pipe relies on dependency injection (for example, a logger service), you can pass a mock manually:

```ts
const mockLogger = {
  log: jasmine.createSpy('log'),
} as LoggerService;

describe('CapitalizePipe', () => {
  ...

  it('should capitalize the first letter of a string', () => {
    const pipe = new CapitalizePipe(mockLogger);
    expect(pipe.transform('hello')).toBe('Hello');
  });
});
```

Now let’s look at how to test the pipe in a component. We'll create a simple "dumb" component that accepts a string input and renders it using the `CapitalizePipe`:

```ts
import { Component, input } from '@angular/core';
import { CapitalizePipe } from '../capitalize.pipe';

@Component({
  selector: 'app-dumb-component',
  imports: [CapitalizePipe],
  ...
})
export class DumbComponent {
  public text = input<string>('');
}
```

```html
<h1 data-testid="text">{{ text() | capitalize }}</h1>
```

Here’s the test file for this component:

```ts
import { ComponentFixture, TestBed } from '@angular/core/testing';

import { DumbComponent } from './dumb-component.component';
import { By } from '@angular/platform-browser';

describe('DumbComponent', () => {
  let component: DumbComponent;
  let fixture: ComponentFixture<DumbComponent>;

  beforeEach(async () => {
    await TestBed.configureTestingModule({
      imports: [DumbComponent, CapitalizePipe],
    }).compileComponents();

    fixture = TestBed.createComponent(DumbComponent);
    component = fixture.componentInstance;
    fixture.detectChanges();
  });

  it('should create', () => {
    expect(component).toBeTruthy();
  });

  it('should capitalize input text', () => {
    const textDe = findByTestId(fixture, 'text');
    fixture.componentRef.setInput('text', 'hello world');
    fixture.detectChanges();
    expect(textDe.nativeElement.textContent).toBe('Hello world');
  });
});
```

The most interesting part is the last test case. We locate the element in the template using a test ID, then update the component’s input using `componentRef.setInput`. After triggering change detection, we assert that the rendered output matches the expected capitalized text. This gives us confidence that the pipe works correctly both in logic and in the view.

## Summary

In this guide, we covered the basics of Angular testing. Even simple tests help catch bugs early and make your application more reliable.

If you want to learn more, check out the [official Angular testing documentation](https://angular.dev/guide/testing), the [Testing Angular online book](https://testing-angular.com/), and explore libraries like [Spectator](https://www.npmjs.com/package/@ngneat/spectator) that simplify Angular testing.

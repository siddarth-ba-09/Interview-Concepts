# Angular — Complete Fundamentals Guide

> Angular is a full-featured TypeScript framework by Google. As someone who has worked on Angular, this serves as a comprehensive reference covering all fundamentals.

---

## Table of Contents

1. [Angular Architecture Overview](#1-angular-architecture-overview)
2. [Modules (NgModule & Standalone)](#2-modules-ngmodule--standalone)
3. [Components](#3-components)
4. [Templates & Data Binding](#4-templates--data-binding)
5. [Directives](#5-directives)
6. [Pipes](#6-pipes)
7. [Services & Dependency Injection](#7-services--dependency-injection)
8. [Component Lifecycle Hooks](#8-component-lifecycle-hooks)
9. [Routing & Navigation](#9-routing--navigation)
10. [Forms — Template-Driven](#10-forms--template-driven)
11. [Forms — Reactive](#11-forms--reactive)
12. [HttpClient & HTTP Interceptors](#12-httpclient--http-interceptors)
13. [RxJS & Observables in Angular](#13-rxjs--observables-in-angular)
14. [Route Guards](#14-route-guards)
15. [Lazy Loading](#15-lazy-loading)
16. [State Management](#16-state-management)
17. [Performance Optimization](#17-performance-optimization)
18. [Angular CLI](#18-angular-cli)
19. [Miscellaneous Concepts](#19-miscellaneous-concepts)
20. [Interview Q&A](#20-interview-qa)

---

## 1. Angular Architecture Overview

```
Angular Application
├── NgModules (or Standalone Components)
│   ├── Components  — UI elements (template + class + styles)
│   ├── Directives  — modify DOM behavior/appearance
│   ├── Pipes       — transform data in templates
│   └── Services    — business logic, data, shared state
│
├── Routing         — maps URLs to components
├── Forms           — template-driven or reactive
├── HttpClient      — HTTP communication
└── DI Container    — manages service instances
```

### Angular vs React vs Vue

| Feature | Angular | React | Vue |
|---------|---------|-------|-----|
| Type | Full Framework | Library | Progressive Framework |
| Language | TypeScript (required) | JS/JSX | JS/TS |
| Data Binding | Two-way | One-way | Two-way |
| Forms | Built-in (template + reactive) | External (RHF) | Built-in |
| HTTP | Built-in HttpClient | External (fetch/axios) | External |
| Router | Built-in | React Router | Vue Router |
| State | Services / NgRx | Redux / Zustand | Pinia / Vuex |
| Learning Curve | Steep | Moderate | Low |

---

## 2. Modules (NgModule & Standalone)

### NgModule (Traditional Approach)
```typescript
import { NgModule } from "@angular/core";
import { BrowserModule } from "@angular/platform-browser";
import { HttpClientModule } from "@angular/common/http";
import { ReactiveFormsModule } from "@angular/forms";
import { AppRoutingModule } from "./app-routing.module";
import { AppComponent } from "./app.component";
import { UserListComponent } from "./users/user-list.component";
import { UserService } from "./services/user.service";
import { AuthGuard } from "./guards/auth.guard";

@NgModule({
  declarations: [      // components, directives, pipes belonging to this module
    AppComponent,
    UserListComponent
  ],
  imports: [           // other modules to import
    BrowserModule,
    HttpClientModule,
    ReactiveFormsModule,
    AppRoutingModule
  ],
  providers: [         // services (available module-wide)
    UserService,
    AuthGuard
  ],
  exports: [           // make available to importing modules
    UserListComponent
  ],
  bootstrap: [AppComponent] // only in root AppModule
})
export class AppModule {}
```

### Standalone Components (Angular 14+, preferred in Angular 17+)
```typescript
// No NgModule needed — component is self-contained
@Component({
  selector: "app-root",
  standalone: true,
  imports: [CommonModule, RouterOutlet, HttpClientModule, ReactiveFormsModule],
  template: `<router-outlet />`
})
export class AppComponent {}

// main.ts
import { bootstrapApplication } from "@angular/platform-browser";
import { provideRouter } from "@angular/router";
import { provideHttpClient } from "@angular/common/http";
import { routes } from "./app.routes";

bootstrapApplication(AppComponent, {
  providers: [
    provideRouter(routes),
    provideHttpClient()
  ]
});
```

### Feature Modules
```typescript
// users/users.module.ts
@NgModule({
  declarations: [UserListComponent, UserDetailComponent, UserFormComponent],
  imports: [CommonModule, UsersRoutingModule, ReactiveFormsModule],
  providers: [UserService]
})
export class UsersModule {}
```

---

## 3. Components

```typescript
import { Component, Input, Output, EventEmitter, OnInit, ChangeDetectionStrategy } from "@angular/core";

@Component({
  selector: "app-user-card",     // <app-user-card> in templates
  templateUrl: "./user-card.component.html",
  styleUrls: ["./user-card.component.scss"],
  // Or inline:
  // template: `<div>{{ user.name }}</div>`,
  // styles: [`div { color: red; }`],
  changeDetection: ChangeDetectionStrategy.OnPush,  // performance optimization
  standalone: true,
  imports: [CommonModule]
})
export class UserCardComponent implements OnInit {
  @Input() user!: User;          // input from parent
  @Input() showEmail = false;
  @Output() selected = new EventEmitter<number>(); // output to parent

  title = "User Card";

  constructor(private userService: UserService) {}

  ngOnInit(): void {
    console.log("Component initialized, user:", this.user);
  }

  onSelect(): void {
    this.selected.emit(this.user.id);
  }
}
```

### Component Communication

```typescript
// Parent → Child: @Input()
// Child → Parent: @Output() + EventEmitter
// Sibling/Distant: Service with Subject/BehaviorSubject
// ViewChild: parent accessing child directly

// ─── @Input() with setter ───
private _user!: User;
@Input() set user(value: User) {
  this._user = value;
  this.processUser(value); // run logic when input changes
}
get user(): User { return this._user; }

// ─── @Input with transform (Angular 16+) ───
@Input({ transform: booleanAttribute }) disabled = false;
@Input({ transform: numberAttribute }) count = 0;

// ─── @ViewChild ───
@ViewChild("myInput") inputEl!: ElementRef<HTMLInputElement>;
@ViewChild(ChildComponent) childComp!: ChildComponent;
@ViewChildren(ItemComponent) items!: QueryList<ItemComponent>;

ngAfterViewInit(): void {
  this.inputEl.nativeElement.focus();
}
```

### Component Selectors
```typescript
selector: "app-user-card"    // element: <app-user-card>
selector: "[appHighlight]"   // attribute: <div appHighlight>
selector: ".user-card"       // class: <div class="user-card">
```

---

## 4. Templates & Data Binding

### Interpolation (One-Way: Component → View)
```html
<h1>{{ user.name }}</h1>
<p>{{ 1 + 1 }}</p>
<p>{{ user?.address?.city }}</p>
<p>{{ getDisplayName() }}</p>
```

### Property Binding (Component → DOM property)
```html
<img [src]="user.avatarUrl" [alt]="user.name">
<button [disabled]="isLoading">Submit</button>
<app-card [title]="cardTitle" [data]="items">
<div [ngClass]="{ active: isActive, disabled: !isEnabled }">
<div [ngStyle]="{ color: textColor, fontSize: fontSize + 'px' }">
<input [value]="name">
```

### Event Binding (DOM → Component)
```html
<button (click)="handleClick($event)">Click</button>
<input (input)="onInput($event)" (blur)="onBlur()">
<form (ngSubmit)="onSubmit()">
<div (mouseenter)="onHover()" (mouseleave)="onLeave()">
<input (keydown.enter)="onEnter()">
```

### Two-Way Binding (requires FormsModule)
```html
<input [(ngModel)]="username">
<!-- Equivalent to: -->
<input [ngModel]="username" (ngModelChange)="username = $event">
```

### Template Reference Variables
```html
<input #nameInput type="text">
<button (click)="greet(nameInput.value)">Greet</button>

<app-child #child></app-child>
<button (click)="child.someMethod()">Call Child Method</button>
```

### Attribute Binding
```html
<td [attr.colspan]="colSpan">...</td>
<button [attr.aria-label]="'Close ' + title">×</button>
```

### Class & Style Binding
```html
<div [class.active]="isActive">       <!-- single class -->
<div [class]="'btn btn-' + variant">  <!-- set entire class string -->
<div [ngClass]="classObject">         <!-- multiple classes -->

<div [style.color]="textColor">       <!-- single style -->
<div [style.font-size.px]="fontSize"> <!-- with unit -->
<div [ngStyle]="styleObject">         <!-- multiple styles -->
```

---

## 5. Directives

### Structural Directives (change DOM structure)

```html
<!-- *ngIf -->
<div *ngIf="isLoggedIn; else loginBlock">Welcome back!</div>
<ng-template #loginBlock><p>Please log in</p></ng-template>

<!-- Newer syntax (Angular 17+) -->
@if (isLoggedIn) {
  <div>Welcome back!</div>
} @else {
  <p>Please log in</p>
}

<!-- *ngFor -->
<li *ngFor="let user of users; let i = index; trackBy: trackById">
  {{ i + 1 }}. {{ user.name }}
</li>

<!-- @for (Angular 17+) -->
@for (user of users; track user.id) {
  <li>{{ user.name }}</li>
} @empty {
  <li>No users found</li>
}

<!-- *ngSwitch -->
<div [ngSwitch]="status">
  <p *ngSwitchCase="'active'">Active</p>
  <p *ngSwitchCase="'inactive'">Inactive</p>
  <p *ngSwitchDefault>Unknown</p>
</div>

<!-- @switch (Angular 17+) -->
@switch (status) {
  @case ('active') { <p>Active</p> }
  @case ('inactive') { <p>Inactive</p> }
  @default { <p>Unknown</p> }
}

<!-- ng-container — logical grouping, no DOM output -->
<ng-container *ngIf="user">
  <h2>{{ user.name }}</h2>
  <p>{{ user.email }}</p>
</ng-container>

<!-- ng-template — define reusable template blocks -->
<ng-template #loading>
  <div class="spinner">Loading...</div>
</ng-template>
<ng-container *ngTemplateOutlet="loading"></ng-container>
```

### Attribute Directives

```typescript
// Custom attribute directive
@Directive({
  selector: "[appHighlight]",
  standalone: true
})
export class HighlightDirective {
  @Input("appHighlight") highlightColor = "yellow";
  @Input() defaultColor = "transparent";

  constructor(private el: ElementRef, private renderer: Renderer2) {}

  @HostListener("mouseenter") onMouseEnter() {
    this.renderer.setStyle(this.el.nativeElement, "backgroundColor", this.highlightColor);
  }

  @HostListener("mouseleave") onMouseLeave() {
    this.renderer.setStyle(this.el.nativeElement, "backgroundColor", this.defaultColor);
  }
}

// Usage
<p [appHighlight]="'lightblue'" [defaultColor]="'white'">Hover me</p>
```

### Custom Structural Directive
```typescript
@Directive({ selector: "[appUnless]", standalone: true })
export class UnlessDirective {
  private hasView = false;

  constructor(
    private viewContainer: ViewContainerRef,
    private templateRef: TemplateRef<any>
  ) {}

  @Input() set appUnless(condition: boolean) {
    if (!condition && !this.hasView) {
      this.viewContainer.createEmbeddedView(this.templateRef);
      this.hasView = true;
    } else if (condition && this.hasView) {
      this.viewContainer.clear();
      this.hasView = false;
    }
  }
}
// <div *appUnless="isLoggedIn">Show when NOT logged in</div>
```

---

## 6. Pipes

Transform data in templates. **Pure pipes** only run when the input reference changes (performant). **Impure pipes** run on every change detection cycle.

```html
<!-- Built-in pipes -->
{{ name | uppercase }}          <!-- ALICE -->
{{ name | lowercase }}          <!-- alice -->
{{ name | titlecase }}          <!-- Alice Bob -->
{{ price | currency:'USD':'symbol':'1.2-2' }}   <!-- $1,234.50 -->
{{ price | number:'1.2-2' }}    <!-- 1,234.50 -->
{{ date | date:'MM/dd/yyyy' }}  <!-- 01/15/2024 -->
{{ date | date:'shortDate' }}   <!-- 1/15/24 -->
{{ date | date:'fullDate' }}    <!-- Monday, January 15, 2024 -->
{{ bigObj | json }}             <!-- pretty JSON for debugging -->
{{ text | slice:0:50 }}         <!-- first 50 chars -->
{{ num | percent:'1.0-2' }}     <!-- 45.12% -->
{{ observable$ | async }}       <!-- subscribe + unsubscribe automatically -->

<!-- Chaining pipes -->
{{ name | slice:0:10 | uppercase }}

<!-- Passing arguments -->
{{ date | date:'yyyy-MM-dd HH:mm' }}
```

### Custom Pipe
```typescript
@Pipe({ name: "timeAgo", standalone: true })
export class TimeAgoPipe implements PipeTransform {
  transform(value: Date | string): string {
    const date = new Date(value);
    const now = new Date();
    const diffMs = now.getTime() - date.getTime();
    const diffSecs = Math.floor(diffMs / 1000);
    const diffMins = Math.floor(diffSecs / 60);
    const diffHours = Math.floor(diffMins / 60);
    const diffDays = Math.floor(diffHours / 24);

    if (diffSecs < 60) return "just now";
    if (diffMins < 60) return `${diffMins}m ago`;
    if (diffHours < 24) return `${diffHours}h ago`;
    return `${diffDays}d ago`;
  }
}

// Impure pipe (runs on every CD cycle — use sparingly)
@Pipe({ name: "filter", standalone: true, pure: false })
export class FilterPipe implements PipeTransform {
  transform(items: any[], searchText: string): any[] {
    if (!items || !searchText) return items;
    return items.filter(item =>
      item.name.toLowerCase().includes(searchText.toLowerCase())
    );
  }
}
```

### AsyncPipe
```html
<!-- Handles subscription and unsubscription automatically -->
<div *ngIf="users$ | async as users">
  <app-user-card *ngFor="let user of users" [user]="user">
</div>

<!-- With loading state -->
<ng-container *ngIf="users$ | async as users; else loading">
  <app-user-card *ngFor="let user of users" [user]="user">
</ng-container>
<ng-template #loading><app-spinner></app-spinner></ng-template>
```

---

## 7. Services & Dependency Injection

Services hold business logic, HTTP calls, and shared state. Angular's DI system manages their instances.

```typescript
// ─── Service ───
@Injectable({
  providedIn: "root"  // singleton across entire app (tree-shakeable)
  // Alternative: "platform", "any", or via NgModule providers array
})
export class UserService {
  private apiUrl = "https://api.example.com/users";

  constructor(private http: HttpClient) {}

  getUsers(page = 1, limit = 10): Observable<User[]> {
    const params = new HttpParams()
      .set("page", page.toString())
      .set("limit", limit.toString());
    return this.http.get<User[]>(this.apiUrl, { params });
  }

  getUserById(id: number): Observable<User> {
    return this.http.get<User>(`${this.apiUrl}/${id}`);
  }

  createUser(user: Partial<User>): Observable<User> {
    return this.http.post<User>(this.apiUrl, user);
  }

  updateUser(id: number, updates: Partial<User>): Observable<User> {
    return this.http.put<User>(`${this.apiUrl}/${id}`, updates);
  }

  deleteUser(id: number): Observable<void> {
    return this.http.delete<void>(`${this.apiUrl}/${id}`);
  }
}

// ─── Inject in component ───
@Component({ ... })
export class UserListComponent implements OnInit {
  users: User[] = [];

  constructor(private userService: UserService) {}
  // Modern inject() function (Angular 14+)
  // private userService = inject(UserService);

  ngOnInit(): void {
    this.userService.getUsers().subscribe({
      next: (users) => this.users = users,
      error: (err) => console.error(err)
    });
  }
}
```

### DI Providers

```typescript
// Value provider
{ provide: API_URL, useValue: "https://api.example.com" }

// Class provider
{ provide: UserService, useClass: MockUserService }  // swap implementation

// Factory provider
{
  provide: Logger,
  useFactory: (config: AppConfig) => new Logger(config.logLevel),
  deps: [AppConfig]
}

// Existing provider (alias)
{ provide: NewService, useExisting: OldService }

// Injection token (for non-class values)
export const API_URL = new InjectionToken<string>("API_URL");
// Inject:
const apiUrl = inject(API_URL);
// Or in constructor: constructor(@Inject(API_URL) private apiUrl: string) {}
```

### Provider Scope
```typescript
@Injectable({ providedIn: "root" })  // singleton app-wide

@Injectable()  // must be added to providers array
// In NgModule providers → module-level singleton
// In Component providers → new instance per component
@Component({ providers: [UserService] }) // component-level instance
```

---

## 8. Component Lifecycle Hooks

```typescript
@Component({ selector: "app-demo", template: "" })
export class DemoComponent implements OnInit, OnChanges, DoCheck,
  AfterContentInit, AfterContentChecked, AfterViewInit, AfterViewChecked, OnDestroy {

  @Input() data!: string;
  private subscription!: Subscription;

  // 1. constructor — DI happens here, no DOM yet
  constructor(private service: DataService) {
    console.log("1. Constructor");
  }

  // 2. Called when @Input() properties change (before ngOnInit on first change)
  ngOnChanges(changes: SimpleChanges): void {
    if (changes["data"]) {
      const { previousValue, currentValue, firstChange } = changes["data"];
      console.log("2. ngOnChanges — data:", previousValue, "→", currentValue, "first?", firstChange);
    }
  }

  // 3. Called once after first ngOnChanges — initialization logic here
  ngOnInit(): void {
    console.log("3. ngOnInit — component ready, inputs available");
    this.subscription = this.service.data$.subscribe(data => this.data = data);
  }

  // 4. Called on every change detection cycle — avoid heavy logic here
  ngDoCheck(): void {
    console.log("4. ngDoCheck");
  }

  // 5. After ng-content is projected
  ngAfterContentInit(): void {
    console.log("5. ngAfterContentInit");
  }

  // 6. After each change detection cycle for projected content
  ngAfterContentChecked(): void {
    console.log("6. ngAfterContentChecked");
  }

  // 7. After component's view (and children) is initialized — @ViewChild available
  ngAfterViewInit(): void {
    console.log("7. ngAfterViewInit — DOM ready");
  }

  // 8. After each change detection for the view
  ngAfterViewChecked(): void {
    console.log("8. ngAfterViewChecked");
  }

  // 9. Just before component is destroyed — cleanup here
  ngOnDestroy(): void {
    console.log("9. ngOnDestroy — cleanup");
    this.subscription.unsubscribe();
  }
}
```

### Lifecycle Order Summary
1. `constructor`
2. `ngOnChanges` (if @Input exists)
3. `ngOnInit`
4. `ngDoCheck`
5. `ngAfterContentInit`
6. `ngAfterContentChecked`
7. `ngAfterViewInit`
8. `ngAfterViewChecked`
9. **Repeat 4-8 on each change detection cycle**
10. `ngOnDestroy`

---

## 9. Routing & Navigation

```typescript
// app.routes.ts
import { Routes } from "@angular/router";

export const routes: Routes = [
  { path: "", redirectTo: "/home", pathMatch: "full" },
  { path: "home", component: HomeComponent },
  {
    path: "users",
    component: UsersLayoutComponent,
    children: [
      { path: "", component: UserListComponent },
      { path: ":id", component: UserDetailComponent },
      { path: ":id/edit", component: EditUserComponent, canActivate: [AuthGuard] }
    ]
  },
  {
    path: "admin",
    loadChildren: () => import("./admin/admin.routes").then(m => m.adminRoutes), // lazy load
    canActivate: [AuthGuard],
    canActivateChild: [RoleGuard],
    data: { roles: ["admin"] }   // pass data to route
  },
  { path: "**", component: NotFoundComponent }  // wildcard — must be last
];
```

### Router Outlet
```html
<!-- app.component.html -->
<nav>
  <a routerLink="/home" routerLinkActive="active">Home</a>
  <a routerLink="/users" routerLinkActive="active" [routerLinkActiveOptions]="{ exact: true }">Users</a>
  <a [routerLink]="['/users', user.id]">User Detail</a>
  <a [routerLink]="['/users', user.id]" [queryParams]="{ tab: 'orders' }">User Orders</a>
</nav>
<router-outlet></router-outlet>

<!-- Named outlets -->
<router-outlet name="sidebar"></router-outlet>
```

### Navigation in Components
```typescript
import { Router, ActivatedRoute } from "@angular/router";

@Component({ ... })
export class UserDetailComponent implements OnInit {
  constructor(
    private router: Router,
    private route: ActivatedRoute
  ) {}

  ngOnInit(): void {
    // Route params
    const id = this.route.snapshot.paramMap.get("id");

    // Observable params (reacts to changes)
    this.route.paramMap.subscribe(params => {
      const id = params.get("id");
      this.loadUser(+id!);
    });

    // Query params
    this.route.queryParamMap.subscribe(params => {
      const tab = params.get("tab") || "overview";
    });

    // Route data
    const roles = this.route.snapshot.data["roles"];
  }

  navigateToEdit(id: number): void {
    this.router.navigate(["/users", id, "edit"]);
    this.router.navigate(["edit"], { relativeTo: this.route });
    this.router.navigate(["/home"], {
      queryParams: { returnUrl: this.router.url }
    });
  }

  goBack(): void {
    this.router.navigate(["../"], { relativeTo: this.route });
  }
}
```

### Route Resolvers (pre-fetch data before navigation)
```typescript
@Injectable({ providedIn: "root" })
export class UserResolver implements Resolve<User> {
  constructor(private userService: UserService, private router: Router) {}

  resolve(route: ActivatedRouteSnapshot): Observable<User> {
    return this.userService.getUserById(+route.paramMap.get("id")!).pipe(
      catchError(() => {
        this.router.navigate(["/not-found"]);
        return EMPTY;
      })
    );
  }
}

// In routes:
{ path: ":id", component: UserDetailComponent, resolve: { user: UserResolver } }
// In component:
const user = this.route.snapshot.data["user"];
```

---

## 10. Forms — Template-Driven

Uses `FormsModule`. Good for simple forms.

```typescript
// Component
@Component({ ... })
export class LoginComponent {
  model = { email: "", password: "" };
  isSubmitting = false;

  onSubmit(form: NgForm): void {
    if (form.invalid) return;
    console.log(form.value); // { email: "...", password: "..." }
    console.log(this.model);
  }

  reset(form: NgForm): void {
    form.resetForm({ email: "", password: "" });
  }
}
```

```html
<!-- Template -->
<form #loginForm="ngForm" (ngSubmit)="onSubmit(loginForm)" novalidate>
  <div>
    <input
      type="email"
      name="email"
      [(ngModel)]="model.email"
      #email="ngModel"
      required
      email
      [class.is-invalid]="email.invalid && email.touched"
    />
    <div *ngIf="email.invalid && email.touched">
      <span *ngIf="email.errors?.['required']">Email is required</span>
      <span *ngIf="email.errors?.['email']">Invalid email format</span>
    </div>
  </div>

  <div>
    <input
      type="password"
      name="password"
      [(ngModel)]="model.password"
      #password="ngModel"
      required
      minlength="8"
    />
    <div *ngIf="password.errors?.['minlength'] && password.touched">
      At least 8 characters required
    </div>
  </div>

  <button type="submit" [disabled]="loginForm.invalid">Login</button>
</form>
```

---

## 11. Forms — Reactive

Uses `ReactiveFormsModule`. Programmatic, testable. Preferred for complex forms.

```typescript
import { FormBuilder, FormGroup, FormArray, Validators, AbstractControl } from "@angular/forms";

@Component({ ... })
export class RegistrationComponent implements OnInit {
  form!: FormGroup;

  constructor(private fb: FormBuilder) {}

  ngOnInit(): void {
    this.form = this.fb.group({
      name: ["", [Validators.required, Validators.minLength(2), Validators.maxLength(50)]],
      email: ["", [Validators.required, Validators.email]],
      password: ["", [Validators.required, Validators.minLength(8),
        Validators.pattern(/^(?=.*[a-z])(?=.*[A-Z])(?=.*\d)/)
      ]],
      confirmPassword: ["", Validators.required],
      address: this.fb.group({
        street: ["", Validators.required],
        city: ["", Validators.required],
        zip: ["", [Validators.required, Validators.pattern(/^\d{5}$/)]]
      }),
      phones: this.fb.array([this.fb.control("", Validators.required)])
    }, { validators: this.passwordMatchValidator });

    // Subscribe to value changes
    this.form.get("email")?.valueChanges
      .pipe(debounceTime(300), distinctUntilChanged())
      .subscribe(value => this.checkEmailAvailability(value));
  }

  // Custom validator
  passwordMatchValidator(group: AbstractControl) {
    const password = group.get("password")?.value;
    const confirm = group.get("confirmPassword")?.value;
    return password === confirm ? null : { passwordMismatch: true };
  }

  // FormArray helpers
  get phones(): FormArray { return this.form.get("phones") as FormArray; }

  addPhone(): void {
    this.phones.push(this.fb.control("", Validators.required));
  }

  removePhone(index: number): void {
    this.phones.removeAt(index);
  }

  onSubmit(): void {
    if (this.form.invalid) {
      this.form.markAllAsTouched(); // show all errors
      return;
    }
    console.log(this.form.value);
    console.log(this.form.getRawValue()); // includes disabled fields
  }

  // Dynamically control form state
  disableEmailField(): void { this.form.get("email")?.disable(); }
  patchUser(user: User): void { this.form.patchValue(user); }     // partial update
  setUser(user: User): void { this.form.setValue({ ...user, confirmPassword: "" }); } // full update
  resetForm(): void { this.form.reset(); }
}
```

```html
<form [formGroup]="form" (ngSubmit)="onSubmit()">
  <div>
    <input type="text" formControlName="name" />
    <div *ngIf="form.get('name')?.invalid && form.get('name')?.touched">
      <span *ngIf="form.get('name')?.errors?.['required']">Required</span>
      <span *ngIf="form.get('name')?.errors?.['minlength']">Too short</span>
    </div>
  </div>

  <!-- Nested FormGroup -->
  <div formGroupName="address">
    <input type="text" formControlName="street" />
    <input type="text" formControlName="city" />
  </div>

  <!-- FormArray -->
  <div formArrayName="phones">
    <div *ngFor="let phone of phones.controls; let i = index">
      <input [formControlName]="i" type="tel" />
      <button type="button" (click)="removePhone(i)">Remove</button>
    </div>
  </div>
  <button type="button" (click)="addPhone()">Add Phone</button>

  <p *ngIf="form.errors?.['passwordMismatch']">Passwords don't match</p>
  <button type="submit" [disabled]="form.invalid || form.pristine">Register</button>
</form>
```

### Custom Async Validator
```typescript
emailAvailableValidator(): AsyncValidatorFn {
  return (control: AbstractControl): Observable<ValidationErrors | null> => {
    return this.userService.checkEmail(control.value).pipe(
      debounceTime(300),
      map(isAvailable => isAvailable ? null : { emailTaken: true }),
      catchError(() => of(null))
    );
  };
}
```

---

## 12. HttpClient & HTTP Interceptors

```typescript
// ─── Service ───
@Injectable({ providedIn: "root" })
export class ApiService {
  constructor(private http: HttpClient) {}

  getUsers(): Observable<User[]> {
    return this.http.get<User[]>("/api/users");
  }

  createUser(user: Partial<User>): Observable<User> {
    return this.http.post<User>("/api/users", user);
  }

  updateUser(id: number, user: Partial<User>): Observable<User> {
    return this.http.patch<User>(`/api/users/${id}`, user);
  }

  deleteUser(id: number): Observable<void> {
    return this.http.delete<void>(`/api/users/${id}`);
  }

  // With query params
  searchUsers(query: string, page = 1): Observable<User[]> {
    const params = new HttpParams()
      .set("q", query)
      .set("page", page.toString());
    return this.http.get<User[]>("/api/users", { params });
  }

  // With custom headers
  uploadFile(file: File): Observable<any> {
    const formData = new FormData();
    formData.append("file", file);
    const headers = new HttpHeaders(); // don't set Content-Type for multipart
    return this.http.post("/api/upload", formData, {
      headers,
      reportProgress: true,
      observe: "events"
    }).pipe(
      filter(event => event.type === HttpEventType.UploadProgress),
      map((event: any) => Math.round(100 * event.loaded / event.total))
    );
  }
}
```

### HTTP Interceptors
```typescript
// ─── Auth Interceptor ───
@Injectable()
export class AuthInterceptor implements HttpInterceptor {
  constructor(private authService: AuthService) {}

  intercept(req: HttpRequest<unknown>, next: HttpHandler): Observable<HttpEvent<unknown>> {
    const token = this.authService.getToken();

    const authReq = token
      ? req.clone({ setHeaders: { Authorization: `Bearer ${token}` } })
      : req;

    return next.handle(authReq);
  }
}

// ─── Error Interceptor ───
@Injectable()
export class ErrorInterceptor implements HttpInterceptor {
  constructor(private router: Router, private authService: AuthService) {}

  intercept(req: HttpRequest<unknown>, next: HttpHandler): Observable<HttpEvent<unknown>> {
    return next.handle(req).pipe(
      catchError((error: HttpErrorResponse) => {
        if (error.status === 401) {
          this.authService.logout();
          this.router.navigate(["/login"]);
        }
        if (error.status === 403) {
          this.router.navigate(["/forbidden"]);
        }
        const message = error.error?.message || error.message || "Unknown error";
        return throwError(() => new Error(message));
      })
    );
  }
}

// ─── Loading Interceptor ───
@Injectable()
export class LoadingInterceptor implements HttpInterceptor {
  private activeRequests = 0;

  constructor(private loadingService: LoadingService) {}

  intercept(req: HttpRequest<unknown>, next: HttpHandler): Observable<HttpEvent<unknown>> {
    this.activeRequests++;
    this.loadingService.show();
    return next.handle(req).pipe(
      finalize(() => {
        this.activeRequests--;
        if (this.activeRequests === 0) this.loadingService.hide();
      })
    );
  }
}

// ─── Register interceptors ───
// In AppModule:
providers: [
  { provide: HTTP_INTERCEPTORS, useClass: AuthInterceptor, multi: true },
  { provide: HTTP_INTERCEPTORS, useClass: ErrorInterceptor, multi: true }
]

// With provideHttpClient (standalone):
bootstrapApplication(AppComponent, {
  providers: [
    provideHttpClient(withInterceptors([authInterceptor, errorInterceptor]))
  ]
});
```

---

## 13. RxJS & Observables in Angular

RxJS is the backbone of Angular's async programming.

### Key Operators Used in Angular

```typescript
import { Observable, Subject, BehaviorSubject, combineLatest, forkJoin, of, EMPTY } from "rxjs";
import { map, filter, switchMap, mergeMap, concatMap, exhaustMap,
         catchError, retry, retryWhen, debounceTime, distinctUntilChanged,
         takeUntil, take, tap, finalize, share, shareReplay } from "rxjs/operators";

// ─── Transformation ───
this.users$ = this.http.get<User[]>("/api/users").pipe(
  map(users => users.filter(u => u.active)),
  map(users => users.sort((a, b) => a.name.localeCompare(b.name)))
);

// ─── Search with debounce ───
this.searchControl.valueChanges.pipe(
  debounceTime(300),
  distinctUntilChanged(),
  filter(query => query.length >= 2),
  switchMap(query => this.userService.search(query)),  // cancels previous request
  catchError(err => { console.error(err); return of([]); })
).subscribe(results => this.results = results);

// ─── Flattening operators ───
// switchMap — cancels previous, use for search/navigation
// mergeMap  — runs all concurrently, use for parallel independent requests
// concatMap — queues, maintains order, use for sequential operations
// exhaustMap — ignores while current is running, use for login button

// ─── Combining ───
combineLatest([this.users$, this.departments$]).pipe(
  map(([users, departments]) => users.map(u => ({
    ...u,
    department: departments.find(d => d.id === u.deptId)
  })))
);

forkJoin([
  this.userService.getUser(1),
  this.orderService.getOrders(1)
]).subscribe(([user, orders]) => { });

// ─── Error handling ───
this.http.get("/api/data").pipe(
  retry(3),
  catchError(err => {
    console.error(err);
    return throwError(() => err); // re-throw
    // return of(defaultValue);   // replace with default
    // return EMPTY;              // complete without value
  })
);

// ─── Subjects ───
// Subject — multicast, no initial value, only gets new values
const subject = new Subject<string>();
subject.next("hello");
subject.subscribe(v => console.log(v)); // only gets future values

// BehaviorSubject — has initial value, new subscribers get current value
const userSubject = new BehaviorSubject<User | null>(null);
userSubject.getValue(); // get current value synchronously
userSubject.next(user);
userSubject.asObservable(); // expose as read-only

// ReplaySubject — replays last N values to new subscribers
const replay = new ReplaySubject<string>(3); // last 3 values

// ─── Memory leak prevention with takeUntil ───
@Component({ ... })
class MyComponent implements OnInit, OnDestroy {
  private destroy$ = new Subject<void>();

  ngOnInit() {
    this.service.data$.pipe(
      takeUntil(this.destroy$)  // unsubscribes when destroy$ emits
    ).subscribe(data => this.data = data);
  }

  ngOnDestroy() {
    this.destroy$.next();
    this.destroy$.complete();
  }
}

// ─── shareReplay — avoid duplicate HTTP requests ───
this.user$ = this.http.get<User>("/api/user").pipe(
  shareReplay(1)  // cache last value, share among subscribers
);
```

---

## 14. Route Guards

```typescript
// ─── CanActivate ───
@Injectable({ providedIn: "root" })
export class AuthGuard implements CanActivate {
  constructor(private authService: AuthService, private router: Router) {}

  canActivate(route: ActivatedRouteSnapshot, state: RouterStateSnapshot):
    boolean | UrlTree | Observable<boolean | UrlTree> {
    if (this.authService.isLoggedIn()) return true;
    return this.router.createUrlTree(["/login"], {
      queryParams: { returnUrl: state.url }
    });
  }
}

// ─── Functional guard (Angular 15+, preferred) ───
export const authGuard: CanActivateFn = (route, state) => {
  const authService = inject(AuthService);
  const router = inject(Router);
  if (authService.isLoggedIn()) return true;
  return router.createUrlTree(["/login"], { queryParams: { returnUrl: state.url } });
};

// ─── Role guard ───
export const roleGuard: CanActivateFn = (route) => {
  const authService = inject(AuthService);
  const requiredRoles: string[] = route.data["roles"] ?? [];
  return requiredRoles.some(role => authService.hasRole(role))
    ? true
    : inject(Router).createUrlTree(["/forbidden"]);
};

// ─── CanDeactivate (unsaved changes warning) ───
export interface HasUnsavedChanges {
  hasUnsavedChanges(): boolean;
}

export const unsavedChangesGuard: CanDeactivateFn<HasUnsavedChanges> = (component) => {
  if (component.hasUnsavedChanges()) {
    return confirm("You have unsaved changes. Leave?");
  }
  return true;
};

// Usage in routes
{ path: "edit/:id", component: EditComponent, canDeactivate: [unsavedChangesGuard] }
```

---

## 15. Lazy Loading

Split the app into feature chunks loaded on demand.

```typescript
// app.routes.ts
export const routes: Routes = [
  { path: "home", component: HomeComponent },
  {
    path: "users",
    loadChildren: () => import("./users/users.routes").then(m => m.usersRoutes)
  },
  {
    path: "admin",
    loadComponent: () => import("./admin/admin.component").then(m => m.AdminComponent)
    // Single component lazy loading (Angular 15+)
  }
];

// users/users.routes.ts
export const usersRoutes: Routes = [
  { path: "", component: UserListComponent },
  { path: ":id", component: UserDetailComponent }
];
```

### Preloading Strategies
```typescript
// Preload all lazy modules after initial load
provideRouter(routes, withPreloading(PreloadAllModules))

// Custom preloading — only preload routes marked with data.preload
export class SelectivePreloadingStrategy implements PreloadingStrategy {
  preload(route: Route, load: () => Observable<any>): Observable<any> {
    return route.data?.["preload"] ? load() : EMPTY;
  }
}
```

---

## 16. State Management

### Service with BehaviorSubject (simple)
```typescript
@Injectable({ providedIn: "root" })
export class UserStateService {
  private usersSubject = new BehaviorSubject<User[]>([]);
  private loadingSubject = new BehaviorSubject<boolean>(false);

  users$ = this.usersSubject.asObservable();
  loading$ = this.loadingSubject.asObservable();

  constructor(private http: HttpClient) {}

  loadUsers(): void {
    this.loadingSubject.next(true);
    this.http.get<User[]>("/api/users").pipe(
      finalize(() => this.loadingSubject.next(false))
    ).subscribe(users => this.usersSubject.next(users));
  }

  addUser(user: User): void {
    this.usersSubject.next([...this.usersSubject.getValue(), user]);
  }
}
```

### NgRx (Redux pattern for Angular — brief overview)
```typescript
// Actions
export const loadUsers = createAction("[Users] Load Users");
export const loadUsersSuccess = createAction("[Users] Load Success", props<{ users: User[] }>());

// Reducer
export const usersReducer = createReducer(
  { users: [], loading: false },
  on(loadUsers, state => ({ ...state, loading: true })),
  on(loadUsersSuccess, (state, { users }) => ({ ...state, users, loading: false }))
);

// Effect
@Injectable()
export class UsersEffects {
  loadUsers$ = createEffect(() =>
    this.actions$.pipe(
      ofType(loadUsers),
      switchMap(() => this.userService.getUsers().pipe(
        map(users => loadUsersSuccess({ users })),
        catchError(err => of(loadUsersFailure({ error: err.message })))
      ))
    )
  );
  constructor(private actions$: Actions, private userService: UserService) {}
}

// Selector
export const selectUsers = createSelector(selectUsersState, state => state.users);

// Component
export class UsersComponent {
  users$ = this.store.select(selectUsers);
  constructor(private store: Store) {
    this.store.dispatch(loadUsers());
  }
}
```

---

## 17. Performance Optimization

### ChangeDetectionStrategy.OnPush
```typescript
@Component({
  changeDetection: ChangeDetectionStrategy.OnPush
  // Only re-renders when:
  // 1. Input reference changes
  // 2. Event from this component or child fires
  // 3. async pipe emits
  // 4. markForCheck() is called
})
export class UserCardComponent {
  @Input() user!: User;
  // user.name = "new name" won't trigger re-render (mutation)
  // user = { ...user, name: "new name" } WILL trigger re-render (new reference)
}
```

### TrackBy in ngFor
```html
<li *ngFor="let user of users; trackBy: trackById">{{ user.name }}</li>
```
```typescript
trackById(index: number, user: User): number { return user.id; }
```

### Pure Pipes over Methods in Templates
```html
<!-- BAD — method called on every change detection cycle -->
{{ formatDate(user.createdAt) }}

<!-- GOOD — pure pipe only runs when input changes -->
{{ user.createdAt | date:'shortDate' }}
```

### async pipe over manual subscribe
```html
<!-- async pipe auto-unsubscribes on destroy -->
{{ user$ | async }}

<!-- vs manual (must remember to unsubscribe) -->
```

---

## 18. Angular CLI

```bash
# Create new project
ng new my-app --routing --style=scss

# Generate
ng generate component components/user-card
ng generate service services/user
ng generate module features/admin --routing
ng generate directive directives/highlight
ng generate pipe pipes/time-ago
ng generate guard guards/auth --implements CanActivate
ng generate interface models/user
ng generate enum models/status

# Shorter forms
ng g c user-card
ng g s user
ng g m admin --routing

# Build
ng build                  # development build
ng build --configuration production  # production build

# Serve
ng serve                  # http://localhost:4200
ng serve --port 4201
ng serve --open           # open browser

# Test
ng test                   # Karma + Jasmine unit tests
ng e2e                    # end-to-end tests

# Lint
ng lint

# Update
ng update                 # check for updates
ng update @angular/core @angular/cli
```

---

## 19. Miscellaneous Concepts

### Content Projection (ng-content)
```html
<!-- card.component.html -->
<div class="card">
  <div class="card-header"><ng-content select="[card-title]"></ng-content></div>
  <div class="card-body"><ng-content></ng-content></div>
  <div class="card-footer"><ng-content select="[card-footer]"></ng-content></div>
</div>

<!-- Usage -->
<app-card>
  <h2 card-title>User Profile</h2>
  <p>User content here</p>
  <button card-footer>Save</button>
</app-card>
```

### ViewEncapsulation
```typescript
@Component({
  encapsulation: ViewEncapsulation.Emulated, // default — scoped CSS (adds unique attrs)
  // ViewEncapsulation.None    — global CSS (no encapsulation)
  // ViewEncapsulation.ShadowDom — native Shadow DOM
})
```

### Signals (Angular 16+)
```typescript
import { signal, computed, effect } from "@angular/core";

@Component({ ... })
export class CounterComponent {
  count = signal(0);    // writable signal
  doubled = computed(() => this.count() * 2);  // derived signal

  constructor() {
    effect(() => {
      console.log("Count changed to:", this.count());
    });
  }

  increment(): void { this.count.update(c => c + 1); }
  reset(): void { this.count.set(0); }
}
```
```html
<p>{{ count() }}</p>   <!-- signals need () to read value -->
<p>{{ doubled() }}</p>
```

### Angular Animation
```typescript
import { trigger, state, style, transition, animate } from "@angular/animations";

@Component({
  animations: [
    trigger("fadeInOut", [
      state("void", style({ opacity: 0 })),
      transition(":enter", [animate("300ms ease-in", style({ opacity: 1 }))]),
      transition(":leave", [animate("300ms ease-out", style({ opacity: 0 }))])
    ])
  ]
})
class AnimatedComponent {
  show = true;
}
// Template: <div *ngIf="show" @fadeInOut>Content</div>
```

### Environment Configuration
```typescript
// environments/environment.ts (dev)
export const environment = { production: false, apiUrl: "http://localhost:3000/api" };

// environments/environment.prod.ts
export const environment = { production: true, apiUrl: "https://api.example.com" };

// Usage
import { environment } from "../environments/environment";
```

---

## 20. Interview Q&A

**Q: What is Angular and how does it differ from React?**
> Angular is a full TypeScript framework with built-in router, HTTP client, forms, DI container, and more. React is a UI library — you assemble your own stack. Angular enforces structure (components, services, modules); React is flexible. Angular uses two-way binding and RxJS; React uses one-way data flow and hooks.

**Q: What is the difference between NgModule and standalone components?**
> NgModules group related components/directives/pipes. Standalone components declare their own dependencies via `imports`, eliminating the need for NgModules. Angular 17+ encourages standalone components.

**Q: Explain Angular's Dependency Injection.**
> Angular's DI system maintains a tree of injectors. `providedIn: 'root'` creates an app-wide singleton. Components can have component-level providers (separate instance per component). When a component needs a service, Angular looks up the injector tree.

**Q: What is ChangeDetectionStrategy.OnPush?**
> OnPush tells Angular to skip change detection for a component unless: an @Input reference changes, an event fires from the component/children, an async pipe emits, or `markForCheck()` is called. Greatly improves performance.

**Q: What is the difference between template-driven and reactive forms?**
> Template-driven forms use directives in HTML and are simple but less flexible. Reactive forms are model-driven — defined in TypeScript, synchronous access to form state, more testable, better for complex validations and dynamic forms.

**Q: How do HTTP Interceptors work in Angular?**
> Interceptors implement `HttpInterceptor.intercept()`. They intercept every HTTP request/response. Common uses: adding auth headers, error handling, loading indicators. They form a chain via `next.handle(modifiedRequest)`.

**Q: What is the async pipe and why use it?**
> The async pipe subscribes to an Observable/Promise in a template and renders the emitted value. It automatically unsubscribes on component destruction, preventing memory leaks. Preferred over manual subscriptions.

**Q: What are Route Guards?**
> Guards control navigation. `CanActivate` — can user enter route? `CanDeactivate` — can user leave route? `Resolve` — pre-fetch data before activation. `CanLoad` — can lazy module be loaded? Modern Angular uses functional guards with `inject()`.

**Q: What is lazy loading and why use it?**
> Lazy loading splits the app into chunks loaded on demand using `loadChildren` or `loadComponent` in routes. Reduces initial bundle size, improving first load performance.

**Q: What are Signals in Angular?**
> Signals (Angular 16+) are reactive primitives that track dependencies automatically. Unlike RxJS, they're synchronous, don't need subscription management, and integrate with Angular's change detection for fine-grained updates without Zone.js.

Here's the Angular side in the order you'd write it live. It uses modern Angular (v17+): standalone components, `inject()`, signals, and the new `@for` / `@if` syntax.

**Step 0 – Enable CORS in the .NET API first**

```csharp
builder.Services.AddCors(o => o.AddPolicy("ng", p =>
    p.WithOrigins("http://localhost:4200").AllowAnyHeader().AllowAnyMethod()));
...
app.UseCors("ng");
```

Say: *"Without CORS the browser blocks calls from port 4200 to the API's port."*

**Step 1 – Environment config**

```ts
// src/environments/environment.ts
export const environment = {
  apiUrl: 'https://localhost:5001/api/v1'
};
```

**Step 2 – Models (match the API DTOs)**

```ts
// models/employee.ts
export interface Employee {
  id: number;
  name: string;
  email: string;
  salary: number;
  department: string;
}

export interface CreateEmployeeRequest {
  name: string;
  email: string;
  salary: number;
  departmentId: number;
}

export interface PagedResult<T> {
  items: T[];
  page: number;
  pageSize: number;
  totalCount: number;
}
```

Say: *"Typed models catch contract mismatches at compile time. In a big team I'd generate them from Swagger/OpenAPI."*

**Step 3 – Service (all HTTP calls in one place)**

```ts
// services/employee.service.ts
@Injectable({ providedIn: 'root' })
export class EmployeeService {
  private http = inject(HttpClient);
  private url = `${environment.apiUrl}/employees`;

  getAll(page = 1, size = 10): Observable<PagedResult<Employee>> {
    const params = new HttpParams().set('page', page).set('size', size);
    return this.http.get<PagedResult<Employee>>(this.url, { params });
  }

  getById(id: number): Observable<Employee> {
    return this.http.get<Employee>(`${this.url}/${id}`);
  }

  create(req: CreateEmployeeRequest): Observable<Employee> {
    return this.http.post<Employee>(this.url, req);
  }

  update(id: number, req: CreateEmployeeRequest): Observable<Employee> {
    return this.http.put<Employee>(`${this.url}/${id}`, req);
  }

  delete(id: number): Observable<void> {
    return this.http.delete<void>(`${this.url}/${id}`);
  }
}
```

Say: *"Components never call HttpClient directly. The service is a singleton, easy to mock in tests, and the single place to change the API URL."*

**Step 4 – Interceptors (auth token + global errors)**

```ts
// interceptors/auth.interceptor.ts
export const authInterceptor: HttpInterceptorFn = (req, next) => {
  const token = localStorage.getItem('token');
  if (token) {
    req = req.clone({ setHeaders: { Authorization: `Bearer ${token}` } });
  }
  return next(req);
};

// interceptors/error.interceptor.ts
export const errorInterceptor: HttpInterceptorFn = (req, next) => {
  const router = inject(Router);
  return next(req).pipe(
    catchError((err: HttpErrorResponse) => {
      if (err.status === 401) router.navigate(['/login']);
      const message = err.error?.title ?? 'Something went wrong';  // ProblemDetails
      alert(message);   // in a real app: a toast/snackbar service
      return throwError(() => err);
    })
  );
};
```

Say: *"The interceptor is the client-side mirror of the API's global exception handler. It reads `title` from ProblemDetails, so the error format is the same end to end."*

**Step 5 – app.config.ts (wire it all up)**

```ts
export const appConfig: ApplicationConfig = {
  providers: [
    provideRouter(routes),
    provideHttpClient(withInterceptors([authInterceptor, errorInterceptor]))
  ]
};
```

**Step 6 – Routes (lazy loaded)**

```ts
export const routes: Routes = [
  { path: '', redirectTo: 'employees', pathMatch: 'full' },
  { path: 'employees',
    loadComponent: () => import('./employees/employee-list.component')
                         .then(m => m.EmployeeListComponent) },
  { path: 'employees/new',
    loadComponent: () => import('./employees/employee-form.component')
                         .then(m => m.EmployeeFormComponent) },
  { path: 'employees/:id/edit',
    loadComponent: () => import('./employees/employee-form.component')
                         .then(m => m.EmployeeFormComponent) }
];
```

Say: *"Lazy loading keeps the initial bundle small."*

**Step 7 – List component (display, paging, delete)**

```ts
@Component({
  selector: 'app-employee-list',
  standalone: true,
  imports: [RouterLink, CurrencyPipe],
  template: `
    <h2>Employees</h2>
    <a routerLink="/employees/new">+ Add</a>

    @if (loading()) {
      <p>Loading...</p>
    } @else {
      <table>
        <tr><th>Name</th><th>Email</th><th>Salary</th><th>Dept</th><th></th></tr>
        @for (e of employees(); track e.id) {
          <tr>
            <td>{{ e.name }}</td>
            <td>{{ e.email }}</td>
            <td>{{ e.salary | currency:'INR' }}</td>
            <td>{{ e.department }}</td>
            <td>
              <a [routerLink]="['/employees', e.id, 'edit']">Edit</a>
              <button (click)="remove(e.id)">Delete</button>
            </td>
          </tr>
        } @empty {
          <tr><td colspan="5">No employees found</td></tr>
        }
      </table>

      <button (click)="load(page() - 1)" [disabled]="page() === 1">Prev</button>
      <span>Page {{ page() }}</span>
      <button (click)="load(page() + 1)" [disabled]="page() * 10 >= total()">Next</button>
    }
  `
})
export class EmployeeListComponent implements OnInit {
  private svc = inject(EmployeeService);

  employees = signal<Employee[]>([]);
  page = signal(1);
  total = signal(0);
  loading = signal(false);

  ngOnInit() { this.load(1); }

  load(page: number) {
    this.loading.set(true);
    this.svc.getAll(page).subscribe({
      next: res => {
        this.employees.set(res.items);
        this.total.set(res.totalCount);
        this.page.set(page);
      },
      complete: () => this.loading.set(false),
      error: () => this.loading.set(false)
    });
  }

  remove(id: number) {
    if (!confirm('Delete this employee?')) return;
    this.svc.delete(id).subscribe(() =>
      this.employees.update(list => list.filter(e => e.id !== id)));
  }
}
```

Say:
- *"Signals give fine-grained change detection, and they're simpler than BehaviorSubject for local state."*
- *"`track e.id` means Angular re-renders only the rows that changed."*
- *"After a delete I update local state instead of reloading the whole list."*

**Step 8 – Form component (create + edit, reactive form)**

```ts
@Component({
  selector: 'app-employee-form',
  standalone: true,
  imports: [ReactiveFormsModule],
  template: `
    <h2>{{ id ? 'Edit' : 'Add' }} Employee</h2>
    <form [formGroup]="form" (ngSubmit)="save()">
      <input formControlName="name" placeholder="Name" />
      @if (form.controls.name.touched && form.controls.name.invalid) {
        <small>Name is required</small>
      }

      <input formControlName="email" placeholder="Email" />
      @if (form.controls.email.touched && form.controls.email.errors?.['email']) {
        <small>Invalid email</small>
      }

      <input type="number" formControlName="salary" placeholder="Salary" />
      <input type="number" formControlName="departmentId" placeholder="Dept Id" />

      <button type="submit" [disabled]="form.invalid || saving()">Save</button>
    </form>
  `
})
export class EmployeeFormComponent implements OnInit {
  private fb = inject(NonNullableFormBuilder);
  private svc = inject(EmployeeService);
  private router = inject(Router);
  private route = inject(ActivatedRoute);

  id = Number(this.route.snapshot.paramMap.get('id')) || null;
  saving = signal(false);

  form = this.fb.group({
    name: ['', [Validators.required, Validators.maxLength(100)]],
    email: ['', [Validators.required, Validators.email]],
    salary: [0, [Validators.required, Validators.min(1)]],
    departmentId: [0, Validators.required]
  });

  ngOnInit() {
    if (this.id) {
      this.svc.getById(this.id).subscribe(e => this.form.patchValue(e));
    }
  }

  save() {
    if (this.form.invalid) return;
    this.saving.set(true);
    const req = this.form.getRawValue();

    const call$ = this.id ? this.svc.update(this.id, req) : this.svc.create(req);

    call$.subscribe({
      next: () => this.router.navigate(['/employees']),
      error: () => this.saving.set(false)
    });
  }
}
```

Say:
- *"Reactive forms are typed, testable, and keep the validation logic in TypeScript."*
- *"Client-side validation is for UX only. The API validates again, because the server is the source of truth."*
- *"One component handles both create and edit, based on whether the route has an id."*
- *"I disable the Save button while saving to prevent double submits."*

**Step 9 – Bonus: search with debounce (shows RxJS skill)**

```ts
search = new FormControl('', { nonNullable: true });

ngOnInit() {
  this.search.valueChanges.pipe(
    debounceTime(300),
    distinctUntilChanged(),
    switchMap(term => this.svc.search(term))
  ).subscribe(res => this.employees.set(res.items));
}
```

Say: *"`debounceTime` avoids firing an API call on every keystroke. `switchMap` cancels the previous in-flight request, so an old response can never overwrite a newer one."*

**Wrap-up talking points for Lead/SA:**
- **Structure:** `core/` (interceptors, guards, singleton services), `shared/` (reusable UI), `features/employees/` (lazy loaded).
- **Auth:** an `authGuard` on routes (`canActivate`), and MSAL for Azure AD/Entra ID instead of localStorage tokens.
- **Memory leaks:** use `takeUntilDestroyed()` or the `async` pipe for long-lived subscriptions. HTTP calls complete on their own.
- **Performance:** `ChangeDetectionStrategy.OnPush`, signals, `track` in loops, lazy routes.
- **State:** signals for local state, NgRx or a signal store only when several features share complex state.
- **Testing:** `HttpTestingController` for services, mocked services for components.
- **Contract:** generate TypeScript models from the OpenAPI spec (NSwag) so frontend and backend stay in sync.

**Order to remember:** CORS → env → model → service → interceptor → config → routes → list → form → (debounce bonus).

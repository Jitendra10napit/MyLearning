👍 For an interview, remember the **4-step Angular CRUD flow**:

**Component → Service → HttpClient → .NET API → Database**

Below is a deliberately short example you can explain on a whiteboard.

### 1. .NET API — Controller

```csharp
[ApiController]
[Route("api/[controller]")]
public class EmployeesController : ControllerBase
{
    private static readonly List<Employee> employees = new();

    [HttpGet]
    public IActionResult GetAll() => Ok(employees);

    [HttpGet("{id}")]
    public IActionResult Get(int id)
        => Ok(employees.FirstOrDefault(x => x.Id == id));

    [HttpPost]
    public IActionResult Create(Employee employee)
    {
        employee.Id = employees.Count + 1;
        employees.Add(employee);
        return Ok(employee);
    }

    [HttpPut("{id}")]
    public IActionResult Update(int id, Employee employee)
    {
        var existing = employees.First(x => x.Id == id);

        existing.Name = employee.Name;
        existing.Email = employee.Email;

        return Ok(existing);
    }

    [HttpDelete("{id}")]
    public IActionResult Delete(int id)
    {
        var employee = employees.First(x => x.Id == id);
        employees.Remove(employee);

        return NoContent();
    }
}

public class Employee
{
    public int Id { get; set; }
    public string Name { get; set; } = "";
    public string Email { get; set; } = "";
}
```

---

## 2. Angular Model

```typescript
export interface Employee {
  id: number;
  name: string;
  email: string;
}
```

---

## 3. Angular Service

This is the **most important part to remember**.

```typescript
@Injectable({
  providedIn: 'root'
})
export class EmployeeService {

  private apiUrl = 'https://localhost:5001/api/employees';

  constructor(private http: HttpClient) {}

  getAll() {
    return this.http.get<Employee[]>(this.apiUrl);
  }

  getById(id: number) {
    return this.http.get<Employee>(`${this.apiUrl}/${id}`);
  }

  create(employee: Employee) {
    return this.http.post<Employee>(this.apiUrl, employee);
  }

  update(employee: Employee) {
    return this.http.put<Employee>(
      `${this.apiUrl}/${employee.id}`,
      employee
    );
  }

  delete(id: number) {
    return this.http.delete(`${this.apiUrl}/${id}`);
  }
}
```

### Easy interview memory trick

```text
GET     → http.get()
POST    → http.post()
PUT     → http.put()
DELETE  → http.delete()
```

---

## 4. Angular Component

```typescript
@Component({
  selector: 'app-employees',
  templateUrl: './employees.component.html'
})
export class EmployeesComponent {

  employees: Employee[] = [];

  constructor(private service: EmployeeService) {}

  ngOnInit() {
    this.loadEmployees();
  }

  loadEmployees() {
    this.service.getAll()
      .subscribe(data => this.employees = data);
  }

  addEmployee() {
    const employee: Employee = {
      id: 0,
      name: 'John',
      email: 'john@test.com'
    };

    this.service.create(employee)
      .subscribe(() => this.loadEmployees());
  }

  updateEmployee(employee: Employee) {
    employee.name = 'Updated Name';

    this.service.update(employee)
      .subscribe(() => this.loadEmployees());
  }

  deleteEmployee(id: number) {
    this.service.delete(id)
      .subscribe(() => this.loadEmployees());
  }
}
```

---

## 5. HTML

```html
<button (click)="addEmployee()">Add</button>

<table>
  <tr *ngFor="let employee of employees">
    <td>{{ employee.id }}</td>
    <td>{{ employee.name }}</td>
    <td>{{ employee.email }}</td>

    <td>
      <button (click)="updateEmployee(employee)">
        Edit
      </button>

      <button (click)="deleteEmployee(employee.id)">
        Delete
      </button>
    </td>
  </tr>
</table>
```

---

# How to explain this in an interview

You can say:

> **"I follow a layered approach in Angular. The component handles UI events, while the Angular service is responsible for HTTP communication. The service uses HttpClient to call the REST API. The .NET API controller handles the request and delegates the business operation to the application/service layer, which then interacts with the repository or EF Core and database."**

Then draw this:

```text
┌─────────────────────┐
│   Angular Component │
│   employees.ts      │
└──────────┬──────────┘
           │
           ▼
┌─────────────────────┐
│  Angular Service    │
│  HttpClient         │
└──────────┬──────────┘
           │ HTTP
           ▼
┌─────────────────────┐
│   .NET Controller   │
│ GET/POST/PUT/DELETE │
└──────────┬──────────┘
           ▼
┌─────────────────────┐
│ Application/Service │
└──────────┬──────────┘
           ▼
┌─────────────────────┐
│ Repository / EF     │
└──────────┬──────────┘
           ▼
┌─────────────────────┐
│      SQL Server     │
└─────────────────────┘
```

### ⭐ One-line memory formula

**Angular Component → Angular Service → HttpClient → REST API → Service → Repository/EF → DB**

And for CRUD:

**C = POST | R = GET | U = PUT | D = DELETE**

For a **Senior/Architect interview**, also mention that in a real application you would add **DTOs, validation, authentication/JWT, global exception handling, logging, cancellation/timeouts, pagination, and an HTTP interceptor** rather than putting business logic directly in the controller.

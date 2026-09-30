# Part 29 — NgRx Entity

## NgRx Entity คืออะไร?

NgRx Entity เป็น Library ที่ช่วยจัดการ Collections ของ Objects ที่มี ID ให้ง่ายขึ้น โดยใช้ Normalized State Pattern ซึ่งเก็บข้อมูลเป็น Dictionary (Map) แทน Array ธรรมดา

### ทำไมต้องใช้ Entity?

**ปัญหาของ Array ธรรมดา:**
```typescript
// Array — ค้นหาช้า O(n)
const updateProduct = (state: ProductState, updated: Product) => ({
  ...state,
  products: state.products.map((p) =>
    p.id === updated.id ? updated : p  // ต้อง loop ทั้งหมด
  )
});

// Delete ก็ช้า O(n)
const deleteProduct = (state: ProductState, id: number) => ({
  ...state,
  products: state.products.filter((p) => p.id !== id)
});
```

**ข้อดีของ Entity (Normalized State):**
```typescript
// Normalized State — เข้าถึงเร็ว O(1)
const updateProduct = (state: EntityState<Product>, updated: Product) =>
  adapter.updateOne({ id: updated.id, changes: updated }, state);

const deleteProduct = (state: EntityState<Product>, id: number) =>
  adapter.removeOne(id, state);
```

---

## EntityState และ EntityAdapter

### EntityState Interface

```typescript
// EntityState<T> มี 2 Properties:
interface EntityState<T> {
  ids: string[] | number[];    // Array ของ ID ทุกตัว
  entities: { [id: string]: T };  // Dictionary: id → entity
}

// ตัวอย่างข้อมูลที่เก็บจริง:
const state: EntityState<Product> = {
  ids: [1, 2, 3],
  entities: {
    1: { id: 1, name: 'สินค้า A', price: 100 },
    2: { id: 2, name: 'สินค้า B', price: 200 },
    3: { id: 3, name: 'สินค้า C', price: 300 },
  }
};
```

### สร้าง EntityAdapter

```typescript
import { createEntityAdapter, EntityAdapter } from '@ngrx/entity';
import { Product } from './product.model';

// สร้าง Adapter โดยระบุ Sort Order (ไม่บังคับ)
export const productAdapter: EntityAdapter<Product> =
  createEntityAdapter<Product>({
    // กำหนด ID Field (ค่าเริ่มต้นคือ 'id')
    selectId: (product) => product.id,

    // กำหนดลำดับการเรียง
    sortComparer: (a, b) => a.name.localeCompare(b.name, 'th'),
  });

// สร้าง InitialState จาก Adapter
export const initialState: ProductEntityState =
  productAdapter.getInitialState({
    // เพิ่ม Properties พิเศษนอกเหนือจาก EntityState
    selectedId: null,
    loading: false,
    error: null,
  });
```

---

## CRUD Operations

### EntityAdapter Methods

```typescript
// เพิ่ม 1 รายการ (ถ้า ID ซ้ำ จะไม่ทำอะไร)
adapter.addOne(entity, state)

// เพิ่มหลายรายการ
adapter.addMany(entities, state)

// เพิ่มหรืออัปเดต (Upsert = Update + Insert)
adapter.upsertOne(entity, state)
adapter.upsertMany(entities, state)

// อัปเดตบางส่วน
adapter.updateOne({ id, changes }, state)
adapter.updateMany([{ id, changes }, ...], state)

// ลบ
adapter.removeOne(id, state)
adapter.removeMany(ids, state)
adapter.removeAll(state)

// แทนที่ทั้งหมด
adapter.setAll(entities, state)

// แทนที่ 1 รายการ (ถ้าไม่มี ID จะเพิ่มใหม่)
adapter.setOne(entity, state)
```

---

## Workshop: Users Entity แบบสมบูรณ์

### โมเดลและ State

```typescript
// user.model.ts
export interface User {
  id: number;
  username: string;
  email: string;
  firstName: string;
  lastName: string;
  role: 'admin' | 'editor' | 'viewer';
  isActive: boolean;
  createdAt: string;
  updatedAt: string;
}

// user.state.ts
import { EntityState } from '@ngrx/entity';
import { User } from './user.model';

export interface UserEntityState extends EntityState<User> {
  selectedUserId: number | null;
  loading: boolean;
  saving: boolean;
  error: string | null;
  searchTerm: string;
  filterRole: string | null;
}
```

### Adapter และ InitialState

```typescript
// user.reducer.ts
import { createEntityAdapter, EntityAdapter } from '@ngrx/entity';
import { createReducer, on } from '@ngrx/store';
import { User } from './user.model';
import { UserEntityState } from './user.state';
import * as UserActions from './user.actions';

// สร้าง Adapter
export const userAdapter: EntityAdapter<User> = createEntityAdapter<User>({
  selectId: (user) => user.id,
  sortComparer: (a, b) =>
    `${a.firstName} ${a.lastName}`.localeCompare(
      `${b.firstName} ${b.lastName}`,
      'th'
    ),
});

// InitialState
export const initialState: UserEntityState = userAdapter.getInitialState({
  selectedUserId: null,
  loading: false,
  saving: false,
  error: null,
  searchTerm: '',
  filterRole: null,
});

// Reducer
export const userReducer = createReducer(
  initialState,

  // Load
  on(UserActions.loadUsers, (state) => ({
    ...state,
    loading: true,
    error: null,
  })),

  on(UserActions.loadUsersSuccess, (state, { users }) =>
    userAdapter.setAll(users, { ...state, loading: false })
  ),

  on(UserActions.loadUsersFailure, (state, { error }) => ({
    ...state,
    loading: false,
    error,
  })),

  // Create
  on(UserActions.createUserSuccess, (state, { user }) =>
    userAdapter.addOne(user, { ...state, saving: false })
  ),

  on(UserActions.createUser, (state) => ({
    ...state,
    saving: true,
    error: null,
  })),

  // Update
  on(UserActions.updateUser, (state) => ({
    ...state,
    saving: true,
    error: null,
  })),

  on(UserActions.updateUserSuccess, (state, { user }) =>
    userAdapter.upsertOne(user, { ...state, saving: false })
  ),

  on(UserActions.updateUserFailure, (state, { error }) => ({
    ...state,
    saving: false,
    error,
  })),

  // Delete
  on(UserActions.deleteUserSuccess, (state, { id }) =>
    userAdapter.removeOne(id, {
      ...state,
      selectedUserId:
        state.selectedUserId === id ? null : state.selectedUserId,
    })
  ),

  // Partial Update (เช่น เปิด/ปิด Active)
  on(UserActions.toggleUserActive, (state, { id, isActive }) =>
    userAdapter.updateOne(
      { id, changes: { isActive, updatedAt: new Date().toISOString() } },
      state
    )
  ),

  // Bulk Update
  on(UserActions.bulkUpdateRole, (state, { ids, role }) =>
    userAdapter.updateMany(
      ids.map((id) => ({
        id,
        changes: { role, updatedAt: new Date().toISOString() },
      })),
      state
    )
  ),

  // Selection
  on(UserActions.selectUser, (state, { id }) => ({
    ...state,
    selectedUserId: id,
  })),

  on(UserActions.clearSelectedUser, (state) => ({
    ...state,
    selectedUserId: null,
  })),

  // Search & Filter
  on(UserActions.setSearchTerm, (state, { term }) => ({
    ...state,
    searchTerm: term,
  })),

  on(UserActions.setRoleFilter, (state, { role }) => ({
    ...state,
    filterRole: role,
  }))
);
```

### Actions

```typescript
// user.actions.ts
import { createAction, props } from '@ngrx/store';
import { User } from './user.model';

// Load
export const loadUsers = createAction('[User] Load Users');
export const loadUsersSuccess = createAction(
  '[User] Load Users Success',
  props<{ users: User[] }>()
);
export const loadUsersFailure = createAction(
  '[User] Load Users Failure',
  props<{ error: string }>()
);

// Create
export const createUser = createAction(
  '[User] Create User',
  props<{ user: Omit<User, 'id' | 'createdAt' | 'updatedAt'> }>()
);
export const createUserSuccess = createAction(
  '[User] Create User Success',
  props<{ user: User }>()
);
export const createUserFailure = createAction(
  '[User] Create User Failure',
  props<{ error: string }>()
);

// Update
export const updateUser = createAction(
  '[User] Update User',
  props<{ user: User }>()
);
export const updateUserSuccess = createAction(
  '[User] Update User Success',
  props<{ user: User }>()
);
export const updateUserFailure = createAction(
  '[User] Update User Failure',
  props<{ error: string }>()
);

// Delete
export const deleteUser = createAction(
  '[User] Delete User',
  props<{ id: number }>()
);
export const deleteUserSuccess = createAction(
  '[User] Delete User Success',
  props<{ id: number }>()
);
export const deleteUserFailure = createAction(
  '[User] Delete User Failure',
  props<{ error: string }>()
);

// Special
export const toggleUserActive = createAction(
  '[User] Toggle Active',
  props<{ id: number; isActive: boolean }>()
);
export const bulkUpdateRole = createAction(
  '[User] Bulk Update Role',
  props<{ ids: number[]; role: User['role'] }>()
);

// Selection
export const selectUser = createAction(
  '[User] Select User',
  props<{ id: number }>()
);
export const clearSelectedUser = createAction('[User] Clear Selected User');

// UI
export const setSearchTerm = createAction(
  '[User] Set Search Term',
  props<{ term: string }>()
);
export const setRoleFilter = createAction(
  '[User] Set Role Filter',
  props<{ role: string | null }>()
);
```

### Selectors สำหรับ Entity

```typescript
// user.selectors.ts
import { createSelector, createFeatureSelector } from '@ngrx/store';
import { UserEntityState } from './user.state';
import { userAdapter } from './user.reducer';

// Feature Selector
export const selectUserFeature =
  createFeatureSelector<UserEntityState>('users');

// Entity Selectors จาก Adapter (selectAll, selectIds, selectEntities, selectTotal)
const {
  selectAll: selectAllUsersRaw,
  selectIds: selectUserIds,
  selectEntities: selectUserEntities,
  selectTotal: selectTotalUsers,
} = userAdapter.getSelectors(selectUserFeature);

// Re-export
export { selectAllUsersRaw, selectUserIds, selectUserEntities, selectTotalUsers };

// Other Selectors
export const selectLoading = createSelector(
  selectUserFeature,
  (state) => state.loading
);

export const selectSaving = createSelector(
  selectUserFeature,
  (state) => state.saving
);

export const selectError = createSelector(
  selectUserFeature,
  (state) => state.error
);

export const selectSearchTerm = createSelector(
  selectUserFeature,
  (state) => state.searchTerm
);

export const selectFilterRole = createSelector(
  selectUserFeature,
  (state) => state.filterRole
);

export const selectSelectedUserId = createSelector(
  selectUserFeature,
  (state) => state.selectedUserId
);

// Derived: ผู้ใช้ที่เลือก
export const selectSelectedUser = createSelector(
  selectUserEntities,
  selectSelectedUserId,
  (entities, selectedId) =>
    selectedId !== null ? entities[selectedId] : null
);

// Derived: กรองตาม Search Term และ Role
export const selectFilteredUsers = createSelector(
  selectAllUsersRaw,
  selectSearchTerm,
  selectFilterRole,
  (users, term, role) => {
    let filtered = users;

    if (role) {
      filtered = filtered.filter((u) => u.role === role);
    }

    if (term) {
      const lower = term.toLowerCase();
      filtered = filtered.filter(
        (u) =>
          u.username.toLowerCase().includes(lower) ||
          u.email.toLowerCase().includes(lower) ||
          u.firstName.toLowerCase().includes(lower) ||
          u.lastName.toLowerCase().includes(lower)
      );
    }

    return filtered;
  }
);

// Derived: จำนวนผู้ใช้แต่ละ Role
export const selectUsersByRole = createSelector(
  selectAllUsersRaw,
  (users) => ({
    admin: users.filter((u) => u.role === 'admin').length,
    editor: users.filter((u) => u.role === 'editor').length,
    viewer: users.filter((u) => u.role === 'viewer').length,
  })
);

// Derived: ผู้ใช้ที่ Active
export const selectActiveUsers = createSelector(
  selectAllUsersRaw,
  (users) => users.filter((u) => u.isActive)
);

// Derived: ผู้ใช้ตาม ID
export const selectUserById = (id: number) =>
  createSelector(
    selectUserEntities,
    (entities) => entities[id]
  );
```

### User Management Component

```typescript
// user-management.component.ts
import { Component, OnInit } from '@angular/core';
import { Store } from '@ngrx/store';
import { Observable } from 'rxjs';
import { User } from '../store/user/user.model';
import * as UserActions from '../store/user/user.actions';
import * as UserSelectors from '../store/user/user.selectors';

@Component({
  selector: 'app-user-management',
  template: `
    <div class="user-management">
      <header>
        <h2>จัดการผู้ใช้</h2>
        <div class="stats" *ngIf="roleStats$ | async as stats">
          <span>Admin: {{ stats.admin }}</span>
          <span>Editor: {{ stats.editor }}</span>
          <span>Viewer: {{ stats.viewer }}</span>
          <span>รวม: {{ totalUsers$ | async }}</span>
        </div>
      </header>

      <div class="filters">
        <input
          type="text"
          placeholder="ค้นหาผู้ใช้..."
          (input)="onSearch($event)"
        />
        <select (change)="onRoleFilter($event)">
          <option value="">ทุก Role</option>
          <option value="admin">Admin</option>
          <option value="editor">Editor</option>
          <option value="viewer">Viewer</option>
        </select>
        <button (click)="addUser()">+ เพิ่มผู้ใช้</button>
      </div>

      <div *ngIf="loading$ | async" class="loading">กำลังโหลด...</div>
      <div *ngIf="error$ | async as err" class="error">{{ err }}</div>

      <table>
        <thead>
          <tr>
            <th>ชื่อ</th>
            <th>Email</th>
            <th>Username</th>
            <th>Role</th>
            <th>สถานะ</th>
            <th>การดำเนินการ</th>
          </tr>
        </thead>
        <tbody>
          <tr
            *ngFor="let user of filteredUsers$ | async"
            [class.selected]="(selectedUser$ | async)?.id === user.id"
            (click)="selectUser(user.id)"
          >
            <td>{{ user.firstName }} {{ user.lastName }}</td>
            <td>{{ user.email }}</td>
            <td>{{ user.username }}</td>
            <td>
              <select
                [value]="user.role"
                (change)="onRoleChange(user, $event)"
                (click)="$event.stopPropagation()"
              >
                <option value="admin">Admin</option>
                <option value="editor">Editor</option>
                <option value="viewer">Viewer</option>
              </select>
            </td>
            <td>
              <button
                [class.active]="user.isActive"
                (click)="toggleActive(user); $event.stopPropagation()"
              >
                {{ user.isActive ? 'Active' : 'Inactive' }}
              </button>
            </td>
            <td>
              <button (click)="editUser(user); $event.stopPropagation()">แก้ไข</button>
              <button (click)="deleteUser(user.id); $event.stopPropagation()" class="danger">ลบ</button>
            </td>
          </tr>
        </tbody>
      </table>
    </div>
  `
})
export class UserManagementComponent implements OnInit {
  filteredUsers$: Observable<User[]>;
  selectedUser$: Observable<User | null | undefined>;
  loading$: Observable<boolean>;
  error$: Observable<string | null>;
  totalUsers$: Observable<number>;
  roleStats$: Observable<{ admin: number; editor: number; viewer: number }>;

  constructor(private store: Store) {}

  ngOnInit(): void {
    this.filteredUsers$ = this.store.select(UserSelectors.selectFilteredUsers);
    this.selectedUser$ = this.store.select(UserSelectors.selectSelectedUser);
    this.loading$ = this.store.select(UserSelectors.selectLoading);
    this.error$ = this.store.select(UserSelectors.selectError);
    this.totalUsers$ = this.store.select(UserSelectors.selectTotalUsers);
    this.roleStats$ = this.store.select(UserSelectors.selectUsersByRole);

    this.store.dispatch(UserActions.loadUsers());
  }

  onSearch(event: Event): void {
    const term = (event.target as HTMLInputElement).value;
    this.store.dispatch(UserActions.setSearchTerm({ term }));
  }

  onRoleFilter(event: Event): void {
    const role = (event.target as HTMLSelectElement).value || null;
    this.store.dispatch(UserActions.setRoleFilter({ role }));
  }

  selectUser(id: number): void {
    this.store.dispatch(UserActions.selectUser({ id }));
  }

  toggleActive(user: User): void {
    this.store.dispatch(
      UserActions.toggleUserActive({ id: user.id, isActive: !user.isActive })
    );
  }

  onRoleChange(user: User, event: Event): void {
    const role = (event.target as HTMLSelectElement).value as User['role'];
    this.store.dispatch(
      UserActions.updateUser({ user: { ...user, role } })
    );
  }

  editUser(user: User): void {
    // เปิด Dialog
  }

  deleteUser(id: number): void {
    if (confirm('ยืนยันการลบผู้ใช้?')) {
      this.store.dispatch(UserActions.deleteUser({ id }));
    }
  }

  addUser(): void {
    // เปิด Dialog
  }
}
```

---

## Entity Selectors ที่ Adapter ให้มา

```typescript
// สร้าง Selectors จาก Adapter
const { selectAll, selectIds, selectEntities, selectTotal } =
  userAdapter.getSelectors(selectUserFeature);

// selectAll      → User[]         — Array ของทุก Entity
// selectIds      → number[]       — Array ของทุก ID
// selectEntities → Dictionary<User> — Object id→entity
// selectTotal    → number         — จำนวน Entity ทั้งหมด
```

---

## การทดสอบ Reducer กับ Entity

```typescript
// user.reducer.spec.ts
import { userReducer, initialState } from './user.reducer';
import * as UserActions from './user.actions';
import { userAdapter } from './user.reducer';
import { User } from './user.model';

const mockUser: User = {
  id: 1,
  username: 'john',
  email: 'john@example.com',
  firstName: 'John',
  lastName: 'Doe',
  role: 'viewer',
  isActive: true,
  createdAt: '2024-01-01',
  updatedAt: '2024-01-01',
};

describe('UserReducer', () => {
  it('ควร load users ลง state', () => {
    const action = UserActions.loadUsersSuccess({ users: [mockUser] });
    const state = userReducer(initialState, action);

    const allUsers = userAdapter.getSelectors().selectAll(state);
    expect(allUsers.length).toBe(1);
    expect(allUsers[0]).toEqual(mockUser);
  });

  it('ควร update user เฉพาะส่วน', () => {
    const loaded = userReducer(
      initialState,
      UserActions.loadUsersSuccess({ users: [mockUser] })
    );
    const updated = userReducer(
      loaded,
      UserActions.updateUserSuccess({
        user: { ...mockUser, role: 'admin' }
      })
    );

    const entities = userAdapter.getSelectors().selectEntities(updated);
    expect(entities[1]?.role).toBe('admin');
  });

  it('ควร delete user', () => {
    const loaded = userReducer(
      initialState,
      UserActions.loadUsersSuccess({ users: [mockUser] })
    );
    const state = userReducer(
      loaded,
      UserActions.deleteUserSuccess({ id: 1 })
    );

    expect(userAdapter.getSelectors().selectTotal(state)).toBe(0);
  });
});
```

---

## สรุป

| Method | การใช้งาน |
|--------|-----------|
| `setAll(entities, state)` | แทนที่ทุก Entity (ใช้หลัง Load) |
| `addOne(entity, state)` | เพิ่ม 1 Entity |
| `upsertOne(entity, state)` | Add หรือ Update 1 Entity |
| `updateOne({id, changes}, state)` | Update บางส่วน |
| `removeOne(id, state)` | ลบ 1 Entity |
| `removeAll(state)` | ลบทั้งหมด |

NgRx Entity ช่วยลดโค้ดซ้ำในการจัดการ Collections ได้มาก ใน Part ถัดไปจะเรียน Angular Signals ซึ่งเป็นระบบ Reactivity ใหม่ใน Angular 16+

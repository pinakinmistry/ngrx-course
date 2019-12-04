# NgRx and NgRx Data

## Installation

```cmd
ng add @ngrx/store @ngrx-devtools
```

### app.module.ts

```ts
const routes: Routes = [
    {
        path: 'courses',
        loadChildren: () => import('./courses/courses.module').then(m => m.CoursesModule),
        canActivate: [ AuthGuard ]
    }
];

@NgModule({
    imports: [
        StoreModule.forRoot(reducers, { metaReducers }),
        StoreDevToolsModule.instrument({ maxAge: 25, logOnly: environment.production }),
        EffectsModule.forRoot([]),
    ]
})
```

### app/reducers/index.ts

```ts
export interface AppState {
}

export const reducers: ActionReducerMap<AppState> = {
}

export const metaReducers: MetaReducer<AppState>[] = !environment.production ? [] : [];
```

## Generate Feature Store

```cmd
ng generate store auth/Auth --module auth.module.ts
```

### auth.module.ts

```ts
@NgModule({
    imports: [
        StoreModule.forFeature('auth', authReducer),
        EffectsModule.forFeature([])
    ]
})
export class AuthModule {
    static forRoot(): ModuleWithProviders {
        return {
            ngModule: AuthModule,
            providers: [
                AuthService,
                AuthGuard,
            ]
        }
    }
}
```

## Define Action

### auth.actions.ts

```ts
export const login = createAction(
    '[Login Page] User Login',
    props<{ user: User }>()
);

export const logout = createAction(
    '[Top Menu] Logout'
);
```

### auth/action-types.ts

```ts
import * as AuthActions from './auth.actions';

export { AuthActions };
```

## Dispatch Login Action

### login.component.ts

```ts
export class LoginComponent {
    // Observables
    constructor(
        private router: Router,
        private auth: AuthService,
        private store: Store<AppStore>,
    ) {}

    login() {
        const val = this.form.value;

        this.auth.login(val.email, val.password)
            .pipe(
                tap(user => {
                    this.store.dispatch(login({ user }));
                    this.router.navigateByUrl('/courses');
                })
            )
            .subscribe();
    }
}
```

### auth/reducers/index.ts

```ts
export interface AuthState {
    user: User;
}

export const initialAuthState: AuthState = {
    user: undefined,
}

export const authReducer = createReducer(
    initialAuthState,
    on(AuthActions.login, (state, action) => {
        return {
            user: action.user
        };
    }),
    on(AuthActions.logout, (state, action) => {
        return {
            user: undefined
        }
    })
);
```

## Selector functions

- Gives only distinct value from store, similar to `distinctUntilChanged()` operator, to avoid emitting same value
- Memoize the value being selected based on input
- Filters out action that doesn't change the state it selects

### auth.selectors.ts

```ts
export const selectAuthState = createFeatureSelector('auth');
export const isLoggedIn = createSelector(
    selectAuthState,
    auth => !!auth.user
);
export const isLoggedOut = createSelector(
    isLoggedIn,
    loggedIn => !loggedIn
);
```

### app.component.ts

```ts
export class AppComponent implements OnInit {
    isLoggedIn$: Observable<boolean>;
    isLoggedOut$: Observable<boolean>;

    constructor(
        private store: Store<AppStore>,
    ) {}

    ngOnInit() {

        const userProfile = localStorage.getItem('user');
        if (userProfile) {
            this.store.dispatch(login({ user: JSON.parse(userProfile) }));
        }

        this.isLoggedIn$ = this.store
            .pipe(
                select(isLoggedIn)
            );

        this.isLoggedOut$ = this.store
            .pipe(
                select(isLoggedOut)
            );
    }

    logout() {
        this.store.dispatch(logout());
    }
}
```

### auth.guard.ts

```ts
export class AuthGuard implements CanActivate {
    constructor(
        private store: Store<AppStore>,
        private router: Router,
    ) {}

    canActivate(
        route: ActivateRouteSnapshot,
        state: RouterStateSnapshot
    ): Observable<boolean> {
        return this.store
            .pipe(
                select(isLoggedIn),
                tap(loggedIn => {
                    if (!loggedIn) {
                        this.router.navigateByUrl('/login');
                    }
                })
            );
    }
}
```

## NgRx Effects

- Used to handle store side effects
- Effects service intercepts all actions dispatched in the app
- It can create action as side effect from specific actions
- Side effect can be async dispatching more actions or sync without no further action

### auth.effects.ts

```ts
export class AuthEffects {

    login$ = createEffect(() => this.actions$
        .pipe(
            ofType(AuthActions.login),
            tap(action => localStorage.setItem('user', JSON.stringify(action.user)))
        ),
        { dispatch: false }
    )

    logout$ = createEffect(() => this.action$
        .pipe(
            ofType(AuthActions.logout),
            tap(() => {
                localStorage.removeItem('user');
                this.router.navigateByUrl('/login');
            })
        ),
        { dispatch: false }
    )

    constructor(
        private actions$: Actions,
        private router: Router,
    ) {}
}
```

## NgRx Entity

- Call API once and get data from store thereafter

### course.action.ts

```ts
export const loadAllCourses = createAction(
    '[Courses Resolver] Load All Courses'
);

export const allCoursesLoaded = createAction(
    '[Load Courses Effect] All Courses Loaded',
    props<{courses: Course[]}>()
);
```

### action-types.ts

```ts
import * as CourseActions from './course.actions';

export {CourseActions};
```

### courses.resolver.ts

```ts
export class CoursesResolver implements Resolve<any> {
    loading: boolean = false;

    constructor(
        private store: Store<AppStore>
    ) {}

    resolve(
        route: ActivatedRouteSnapshot,
        state: RouterStateSnapshot
    ): Observable<any> {
        return this.store
            .pipe(
                tap(() => {
                    if (!this.loading) {
                        this.loading = true;
                        this.store.dispatch(loadAllCourses());
                    }
                }),
                first(),
                finalize(() => this.loading = false)
            )
    }
}
```

### courses.effects.ts

```ts
export class CoursesEffects {
    loadCourses$ = createEffect(
        () => this.actions$
            .pipe(
                ofType(CoursesActions.loadAllCourses),
                concatMap(() => this.coursesHttpService.findAllCourses()),
                map(courses => allCoursesLoaded({ courses }))
            )
    )

    constructor(
        private actions$: Actions,
        private coursesHttpService: CoursesHttpService,
    ) {}
}
```

app/reducers.index.ts:
interface AppState {}
reducers: ActionReducerMap<AppState> = {}
metaReducers: MetaReducer<AppState>[] = environment.production ? [] : [];
meta reducers run after all action reducers have run to create the final AppState
ng g action auth/Auth to generate action types
enum of action types. Eg:  AuthActionTypes { LoginAction = '[Login] Action', LogoutAction = '[Logout] Action', }
Action object creator class. eg: class Login implements Action { readonly type = AuthActionTypes.LoginAction; constructor(public payload: {user: User}) {}}
this.auth.login(username, password).pipe(tap(user => {this.store.dispatch(new Login({user})); thiz.router.navigateByUrl('/courses'); }).subscribe();
type AuthState = { loggedIn: boolean, user: User }
interface AppState {auth: AuthState, }
function authReducer(state: AuthState, action): AuthState { switch(action.type) { case AuthActionTypes.LoginAction: ... }}
export const reducers: ActionReducerMap<AppState> = { auth: authReducer, ... }
ng g reducer Auth --flat=false --module auth/auth.module.ts
@NgModule({..., StoreModule.forFeature('auth', fromAuth.authReducer)})
class Logout ...
type AuthActions = Login | Logout;
this.store.pipe(map(state => state.auth.loggedIn))
selectAuthState = state => state.auth;
IsLoggedIn = createSelector(selectAuthState, auth => auth.loggedIn)
this.store.pipe(select(IsLoggedIn))
Selectors are memoized function
Selector function also filters out action that doesn't change the state it selects
class AuthModule { static forRoot(): ModuleWithProviders { return { ngModule: AuthModule, providers: [AuthGuard, AuthService, ...]}}}
class AuthGaurd implements CanActivate { canActivate (route: ActivatedRouteSnapshot, state: RouterStateSnapshot): Observable<boolean> { this.state.pipe(select(isLoggedIn), tap(loggedIn => loggedIn ? noop : this.router.navigateByUrl('/login')))}}
route: canActivate: [AuthGaurd]
NgRx Effects are used to handle store side effects
Effects service intercepts all actions dispatched in the app to create action as side effect from specific actions. Side effect can be async dispatching more actions or sync without no further action
ng g effect auth/Auth --module auth/auth.module.ts

class AuthEffects ...
constructor (private actions$: Actions, private router: Router) {}}
tap to get the value out of observable and run side effect with that value
class AuthEffects { @Effect({dispatch: false}) login$ = this.actions$.pipe(ofType<Login>(AuthActionTypes.LoginAction), tap(action => localStorage.setItem('user', JSON.stringify(action.payload.user))));
@Effect({dispatch: false}) logout$ = this.actions$.pipe(ofType<Logout>(AuthActionTypes.LogoutAction), tap(action => { localStorage.removeItem('user'); this.router.navigateByUrl('/login'); ));
defer to wait for subscription before creating the observable. For eg: defer dispatching of action before NgRx store has been created
@Effect() init$ = defer(() => { const userData = localStorage.getItem('user'); if (userData) { return of(new Login(JSON.parse(userData)));} else { return of(new Logout()); }});
AppModule imports: Effects module.forRoot([])
Store freeze to avoid state mutation
reducers.index.ts: metaReducers: MetaReducer<AppState>[] = environment.production ? [] : [storeFreeze];
Time travel debugging with route changes. Add router state in NgRx store in app.module.ts:
StoreRouterConnectingModule.forRoot({stateKey: 'router'})
providers: [{ provide: RouterStateSerializer, useClass: CustomSerializer }]
reducers/index.ts:
export const reducers = ActionReducerMap<AppState> = {router: routerReducer() }

Sequence of constructs
Actions
State interface
Initial state
Reducers
Selectors
An Entity
entities as map of id vs value/object
Array of sorted ids
NgRx Entity
Actions classes, constructor with payload
State interface extends EntityState<T>
EntityAdapter<T>
createFeatureSelector for feature level state
createSelector to apply list of selectors and then project selected state using map or filter or specific value in entities array
Adapter makes writing reducer and selector easy
resolve() spstdff
return this.store.pipe(select(selectCourseById(courseId)), tap(course=> { if (! course) { this.store.dispatch(new CourseRequested(courseId))}}), filter(course => !!course), first())
Course.effects.ts apomm
@Effect() loadCourse$ = this.actions$.pipe(ofType<CourseRequested>(CourseActionTypes.CourseRequested), mergeMap(action => this.courseService.findCourseById(action.payload.courseId)), map(course => new CourseLoaded({ course }))),
Course.reducer.ts
initialState = adapter.getInitialState();
courses reducer(state = initialState, action: CoursesAction): CoursesState {switch(action.type) { case CourseActionTypes.CourseLoaded: return adapter.addOne(action.payload.course, state); default: { return state; }}}
courses.module.ts
Imports: [ StoreModule.forFeature('courses', coursesReducer)]
Get all courses: sps
Component: onNgInit { this.store.dispatch(new AllCoursesRequested()); const courses$ = this.store.pipe(select(selectAllCourses))}
course.reducers.ts
export const { selectAll } = adapter.getSelectors();
course.selectors.ts
selectAllCourses = createSelector(selectCoursesState, coursesState => fromCourse.selectAll)
Avoid multiple Api calls:
allCoursesLoaded = createSelector(selectCoursesState, coursesState => coursesState.allCoursesLoaded)
@Effect() loadAllCourses$ = this.actions$.pipe(ofType<AllCoursesRequested>(CourseActionTypes.AllCoursesRequested), withLatestFrom(this.store.pipe(select(allCoursesLoaded))), filter(([action, allCoursesLoaded]) => !allCoursesLoaded), mergeMap(action => this.courseService.findCourseById(action.payload.courseId)), map(course => new CourseLoaded({ course }))),
Update Entity:
save() { const changed = this.form.value; this.coursesService.saveCourse(this.courseId, changes).subscribe(() => { C nst course: Update<Course> = { id: this.courseId, changes}; this.store.dispatch(new CourseSaved({ course }))})}
















## NgRx Data

- To manage entities with minimal code with convention over configuration
- Covers actions, reducers, effects and selectors
- API needs to follow convention like `api/<entity>s` and send entity directly in response

### Using NgRx Data in Lazy Loaded Modules

```ts
// app.module.ts
// No entity at app level
EntityDataModule.forRoot({});
```

### courses.module.ts

- Define entity metadata with type `EntityMetadataMap`
- Register entity metadata using `EntityDefinitionService.registerMetadataMap()`
- Register custom data service using `EntityDataService.registerService()`
- Use resolver to resolve courses on route change
- `optimisticUpdate` will update the store without waiting for the PUT call to complete

```ts
const entityMetadata: EntityMetadataMap = {
    Course: {
        sortComparer: compareCourses,
        entityDispatcherOptions: {
            optimisticUpdate: true
        }
    },
    Lesson: {
        sortComparer: compareLessons
    }
};

const coursesRoutes: Routes = [
    {
        path: '',
        component: HomeComponent,
        resolve: {
            courses: CoursesResolver
        }
    },
    {
        path: ':courseUrl',
        component: CourseComponent
    }
];

@NgModule({
    providers: [
        CourseEntityService,
        CoursesResolver,
        CoursesDataService,
        LessonsEntityService
    ],
})
export class CoursesModule {
    constructor(
        private eds: EntityDefinitionService,
        private entityDataService: EntityDataService,
        private coursesDataService: CoursesDataService
        ) {
        eds.registerMetadataMap(entityMetadata);
        entityDataService.registerService('Course', coursesDataService);
    }
}
```

### course.ts

```ts
export interface Course {
    id: number;
    seqNo: number;
    // ...
}

export function compareCourses(c1: Course, c2: Course) {
    const compare = c1.seqNo - c2.seqNo;

    if (compare > 0) {
        return 1;
    } else if (compare < 0) {
        return -1;
    } else {
        return 0;
    }
}
```

### lesson.ts

```ts
export interface Lesson {
    id: number;
    description: string;
    seqNo: number;
    courseId: number;
}

export function compareLessons(l1: Lesson, l2: Lesson) {
    const compareCourses = l1.courseId - l2.courseId;

    if (compareCourses > 0) {
        return 1;
    } else if (compareCourses < 0) {
        return -1;
    } else {
        return l1.seqNo - l2.seqNo;
    }
}
```

```ts
// course-entity.service.ts
export class CourseEntityService extends EntityCollectionServiceBase<Course> {
    constructor(serviceElementsFactory: EntityCollectionServiceElementsFactory) {
        super('Course', serviceElementsFactory);
    }
}
```

```ts
// lesson-entity.service.ts
export class LessonEntityService extends EntityCollectionServiceBase<Lesson> {
    constructor(serviceElementsFactory: EntityCollectionServiceElementsFactory) {
        super('Lesson', serviceElementsFactory);
    }
}
```

### courses.resolver.ts

- Can use loaded$ observable to fetch data from backend first time and read it from NgRx store thereafter
- Avoid route transition by `filter`ing `!loaded` case
- Complete the returned observable using `first()` to complete the route transition

```ts
export class CoursesResolver implements Resolve<boolean> {
    constructor(private coursesService: CoursesEntityService) {}

    resolve(
        route: ActivatedRouteSnapshot,
        state: RouterStateSnapshot
    ): Observable<boolean> {
        return this.coursesService.loaded$
            .pipe(
                map(loaded => {
                    if (!loaded) {
                        this.coursesService.getAll();
                    }
                }),
                filter(loaded => !!loaded),
                first()
            );
    }
}
```

### courses-data.service.ts

- Customize entity data, endpoints, response, etc.

```ts
export class CoursesDataService extends DefaultDataService<Course> {
    constructor(
        http: HttpClient,
        httpUrlGenerator: HttpUrlGenerator
    ) {
        super('Course', http, httpUrlGenerator);
    }

    getAll(): Observable<Course[]> {
        return this.http.get('/api/courses')
            .pipe(
                map(res => res['payload'])
            );
    }
}
```

### home.component.ts

```ts
export class HomeComponent implements OnInit {
    // Observables

    constructor(
        private courseService: CourseEntityService
    ) {}

    ngOnInit() {
        this.reload();
    }

    reload() {
        this.beginnerCourses$ = this.courseService.entities$
            .pipe(
                map(courses => courses.filter(course => course.catogory === 'BEGINNER'))
            );
        // ...
    }
}
```

## CRUD actions

### edit-course-dialog.component.ts

```ts
export class EditCourseDialogComponent {
    // ...
    constructor(
        private courseService: CourseEntityService
    ) {}

    onSave() {
        const course: Course = {
            ...this.course,
            ...this.form.value
        };

        if (this.mode === 'UPDATE') {
            this.courseService.update(course);
            this.dialogRef.close();
        } else {
            this.courseService.add(course)
                .subscribe(() => {
                    this.dialogRef.close();
                });
        }
    }
}
```

### courses-card-list.component.ts

```ts
export class CoursesCardListComponent {
    constructor(
        private courseService: CourseEntityService
    ) {}

    onDeleteCourse(course: Course) {
        this.courseService.delete(course);
    }
}
```

### course.component.ts

```ts
export class CoureComponent implements OnInit {
    course$: Observable<Course>;
    lessons$: Observable<Lesson[]>;
    loading$: Observable<boolean>;
    nextPage: number = 0;

    constructor(
        private coursesService: CoursesEntityService,
        private lessonsService: LessonsEntityService,
        private route: ActivatedRoute,
    ) {}

    ngOnInit() {
        const courseUrl = this.route.snapshot.paramMap.get('courseUrl');

        this.course$ = this.coursesService.entities$
            .pipe(
                map(courses => courses.find(course => course.url === courseUrl))
            );

        this.lesssons$ = this.lessonsService.entities$
            .pipe(
                withLatestFrom(this.course$),
                tap(([lessons, course]) => {
                    if(this.nextPage === 0) {
                        this.loadLessonsPage(course);
                    }
                }),
                map(([lessons, course]) => lessons.filter(lesson => lesson.courseId === course.id))
            );

        this.loading$ = this.lessonsService.entities$.loading$.pipe(delay(0));
    }

    loadLessonsPage(course: Course) {
        this.lessonsService.entities$.getWithQuery({
            courseId: course.id.toString(),
            pageNumber: this.nextPage.toString(),
            pageSize: '5'
        });

        this.nextPage += 1;
    }
}
```

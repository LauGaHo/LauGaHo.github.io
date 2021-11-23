# RxJS：如何使用 RxJS 实现高效的 HTTP 请求

在项目中，经常碰到这样的需求：用户在输入框输入数据，需要实时调用后端 API，拿到结果显示在页面上。如果用传统方式一般实现方式是在输入框上绑定一个 `keydown` 或者 `keyup` 事件，然后每次输入值以后都调用一次后端 API，拿到返回数据。这样会有一个问题，比如我们输入 'liujiahao'，`l` `li` `liu` `liuj` `liuji` `liujia` `liujiah` `liujiaha` `liujiahao` 这五次会分别调用一次 API。这样就会有五个 Response，但是这个五个 Response 的顺序是不可控的，可能最后返回的是 `liu` 的结果，这种方法不仅效率低下，而且没有办法保证正确性。

这篇文章会介绍如何在 RxJS 中结合操作符 `debounceTime` `map` `filter` `distinctUtilChanged` 和 `switchMap` 实现：输入完 'liujiahao' 之后只调用一次 API，最后拿到这个 API 返回的输入显示在页面上。

我们来实现一个搜索 Github 用户的功能，页面上有一个输入框，在输入框中输入 Github 用户名，然后将搜索结果显示在页面上：

我们先来定义一个 Service 如下：

实例代码用的是 Angular + RxJS：

```typescript
export class RxjsSearchableInputService {

    constructor(private http: HttpClient) { }

    searchUser(val: any) {
        return this.http.get<Observable<SearchResult>>("https://api.github.com/search/users?q=" + val)
            .pipe(
                map(response => response),
                catchError((error) => {
                    console.log("something went wrong, " + error);
                    return of([]);
                })
            )
    }
}
```

定义一个 component 如下：

```typescript
export class RxjsSearchableInputComponent implements OnInit, OnDestroy {

    users: Array<User> = [];
    onSearchUser$ = new Subject<KeyboardEvent>();

    validSearch$: Observable<any>;
    emptySearch$: Observable<any>;

    subscription: Subscription;
    constructor(private rxjsSearchableInputService: RxjsSearchableInputService) { }

    ngOnInit() {
        this.validSearch$ = this.onSearchUser$
            .pipe(
                debounceTime(1000),
                map(event => (<HTMLInputElement>event.target).value),
                distinctUntilChanged(),
                filter(input => input !== ""),
                switchMap(data => this.rxjsSearchableInputService.searchUser(data))
            )

        this.emptySearch$ = this.onSearchUser$.pipe(
            debounceTime(1000),
            map(event => (<HTMLInputElement>event.target).value),
            filter(input => input === ""),
            switchMap(data => of([]))
        )

        this.subscription = merge(this.validSearch$, this.emptySearch$)
            .subscribe(resp => {
                if (resp && resp.items && resp.items.length) {
                    let result = resp as SearchResult;
                    this.users = result.items;
                } else {
                    this.users = [];
                }
            })
    }

    ngOnDestroy() {
        this.subscription.unsubscribe();
    }
}
```

component 对应的 html 文件如下：

```html
<div class="margin-large">
    <div class="container-flex margin-small">
        <div class="margin-right-small"><strong> Search Github user: </strong></div>
        <input type="text" (keyup)="onSearchUser$.next($event)" />
    </div>
    <div *ngFor="let user of users" style="margin: 20px">
        <div class="container-flex">
            <div style="font: 0.9em;width:20%"><strong>User Name: </strong></div>
            <div style="font:0.9em;width:10%"><strong>ID: </strong></div>
            <div style="font:0.9em;width:30%"><strong>GitHub URL: </strong></div>
        </div>
    </div>
    <div *ngIf="!users.length" class="margin-small">
        <label style="color:red">no search result!</label>
    </div>
</div>
```

在 component 中先定义一个名为 `onSearchUser$` 的 `Subject`，然后在 input 上绑定一个 `keyup` 事件 `(keyup)="onSearchUser$.next($event)"`，每次输入框输入有变化的时候，`onSearchUser$` 都会发送当前输入框的值。

然后在 component 中定义两个 `Observable` ：`validSearch$` 和 `emptySearch$`，`validSearch$` 是每隔 1 秒拿到 `input` 框中的非空值，并且是本次拿到的值和上次的值不一样的情况下调用对应 API 把搜索结果显示在页面上。`emptySearch$` 是每隔 1 秒拿到 `input` 框中的空值，并不调用 API，直接返回一个空的用户列表。在 `validSearch$` 中用 `switchMap` 的原因是：本次调用 API 的时候，上一次的 API 如果还没有返回，`switchMap` 会取消上一次的 API，这样就可以保证每次 API 返回的结果是正确的。

最后再用 `merge` 把 `emptySearch$`，`validSearch$` 两个 `Observable` 合并。
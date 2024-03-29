# RxJS：如何通过 RxJS 实现缓存

- Angular 中通过 HttpClient 执行 Http Request 返回的 Observables 是 Cold Observable。
- HttpClient Observable 每次被订阅都需要调用 http request，对于共用的 API 返回同样的值，在不同页面重复调用会浪费 http 资源降低性能。
- 如何通过 ReplaySubject 实现缓存效果，提高性能。

## HttpClient 返回的 Observables 是 Cold Observable

在 Angular 2.0 以上的版本，都是通过 HttpClient 跟后端 API 交互；所有的 Http 请求方法，比如 get、post、put、delete 都是返回一个 Observable。之前介绍了 Cold Observable 与 Hot Observable 的区别。

那么在 Angular 中通过 HttpClient 执行 Http Request 返回的 Observables 是 Hot Observables 还是 Cold Observables？先写一段代码测试一下，在页面中显示 30 个 GitHub user 的基本信息：

先定义一个 service：RxjsCacheService，在这个 service 中定义一个方法去拿 github 的用户信息：

```javascript
import { Injectable } from "@angular/core";
import { HttpClient } from "@angular/common/http";

import { User } from "../interface/rxjs-cache.interface";
import { map, catchError } from 'rxjs/operators';
import { of, Observable } from 'rxjs';

@Injectable()
export class RxjsCacheService {
    constructor(private http: HttpClient) { }

    get users() {
        return this.requestUsers();
    }

    private requestUsers() {
        // get the latest 30 github users: start from id =2
        return this.http.get<Array<User>>("https://api.github.com/users?since=1")
            .pipe(
                map(respone => respone),
                catchError(error => {
                    console.log("something went wrong " + error)
                    return of([]);
                })
            )
    }
}
```

接口 rxjs-cache.interface 定义如下：

```javascript
export interface User {
    login: string,
    id: number,
    node_id: string,
    avatar_url: string,
    gravatar_id: string,
    url: string,
    html_url: string,
    followers_url: string,
    following_url: string,
    gists_url: string,
    starred_url: string,
    subscriptions_url: string,
    organizations_url: string,
    repos_url: string,
    events_url: string,
    received_events_url: string,
    type: string,
    site_admin: boolean
}
```

然后定义一个 component：RxjsCacheComponent，代码如下：

```javascript
import { Component, OnInit } from "@angular/core";

import { RxjsCacheService } from "./service/rxjs-cache.service";
import { User } from "./interface/rxjs-cache.interface";
import { Observable } from 'rxjs';


@Component({
    templateUrl: "./rxjs-cache.component.html"
})
export class RxjsCacheComponent implements OnInit {

    private users$: Observable<Array<User>>;

    constructor(private rxjsCacheService: RxjsCacheService) { }

    ngOnInit() {
        this.users$ = this.rxjsCacheService.users;
    }

}
```

rxjs-cache.component.html 代码如下：

```html
<h3>here is the github user lists:</h3>

<div *ngFor="let user of users$ |async">
    <div style=" display: flex;flex-direction: row;">
        <div style="font-size: 0.9em;margin:10px;width: 10%"><strong>User Name:</strong>  { { user.login } } </div>
        <div style="font-size: 0.9em;margin:10px;width: 50%"><strong>GitHub URL:</strong> { { user.url } } </div>
    </div>
</div>
```

运行以上代码，在页面上列出了 30 位 Github 用户的用户名和相对应的 Github 地址，从上面的代码还是不能看出 HttpClient 返回的 Observable 是 Hot Observables 还是 Cold Observables。我们把 html 代码改一下：

```html
<h3>here is the github user lists:</h3>
<div *ngFor="let user of users$ |async">
    <div style=" display: flex;flex-direction: row;">
        <div style="font-size: 0.9em;margin:10px;width: 10%"><strong>User Name:</strong>  { { user.login } } </div>
        <div style="font-size: 0.9em;margin:10px;width: 50%"><strong>GitHub URL:</strong> { { user.url } } </div>
    </div>
</div>

<h3>here is the github user lists2:</h3>
<div *ngFor="let user of users$ |async">
    <div style=" display: flex;flex-direction: row;">
        <div style="font-size: 0.9em;margin:10px;width: 10%"><strong>User Name:</strong>  { { user.login } } </div>
        <div style="font-size: 0.9em;margin:10px;width: 50%"><strong>GitHub URL:</strong> { { user.url } } </div>
    </div>
</div>
```

F12 打开浏览器的 DevTools，当前页面会调用两次 GET API，并且两个 user list 列出的 30 位用户的信息一模一样：

![alt](https://cdn.jsdelivr.net/gh/LauGaHo/blog-img@master/uPic/scNswE.jpg)

两个 userList 订阅 users$，生成了两个 Observable 实例并且都是订阅开始之后才开始发送值，***也就是说 Angular 中通过 HttpClient 执行 Http Request 返回的 Observable 是 Cold Observable***。

## 性能问题？

每次调用 API ，都会生成一个新的 Observable 实例，有订阅之后才开始发送值，这也符合现在前端的开发需求。但在实际开发过程中，有时候后端会提供一些公用的常量 API，不同页面都需要用这些常量，按现在的调用 API 的方式，会导致常量 API 在不同的页面重复多次调用，这种方式显然性能不好。

## 通过 RxJS 实现缓存效果

ReplaySubject(size) 可以发送之前的旧值给新的订阅者，size 是定义发送具体多少个旧值给新的订阅者。那么在示例代码中可以用 ReplaySubject 实现缓存效果。

shareReplay 这个操作符会自动创建一个 ReplaySubject，一旦 http request 执行一次以后，就会在后续的订阅和源头 Observable 之间建立一个 ReplaySubject，ReplaySubject 是一个多播的 Hot Observable，后续订阅都是从这个中间的 ReplaySubject 拿到最后一个值，从而达到缓存效果。

我们把 RxjsCacheService 改成如下：

```javascript
import { Injectable } from "@angular/core";
import { HttpClient } from "@angular/common/http";

import { User } from "../interface/rxjs-cache.interface";
import { map, catchError, shareReplay } from 'rxjs/operators';
import { of, Observable } from 'rxjs';


const CACHE_SIZE = 1;

@Injectable()

export class RxjsCacheService {

    private cacheUsers$:Observable<Array<User>>;

    constructor(private http: HttpClient) { }

    get users() {
        if(!this.cacheUsers$){
            this.cacheUsers$ = this.requestUsers()
            .pipe(
                shareReplay(CACHE_SIZE)
            );
        }
       return this.cacheUsers$;
    }

    private requestUsers() {
        return this.http.get<Array<User>>("https://api.github.com/users?since=1")
            .pipe(
                map(respone => respone),
                catchError(error => {
                    console.log("something went wrong " + error)
                    return of([]);
                })
            )
    }
}
```

运行以上代码发现，页面里两个 user list 都是列出了相同的30位 Github 用户信息，但是只调用了一次 GET API，也就是说第二订阅不是从通过后端 API 拿到用户信息，而是从 ReplaySubject 中拿到的。

整个流程如下：

![alt](https://cdn.jsdelivr.net/gh/LauGaHo/blog-img@master/uPic/icGsXB.jpg)

页面的第一个 userlist 也就是第一个 consumer，是通过调用API拿到30个用户信息，第二个 userlist 也就是第二个 consumer，直接从 cacheUsers\$ 拿到这30个用户信息。cacheUsers\$ 是 ReplaySubject(1) 把最后一个旧值（30个用户信息）发送给新的订阅者（第二个 userlist ），从而实现了缓存效果。
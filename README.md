# inversify-koa

[![Build Status](https://secure.travis-ci.org/raman-nbg/inversify-koa-utils.svg?branch=master)](https://travis-ci.org/raman-nbg/inversify-koa-utils)
[![Test Coverage](https://codeclimate.com/github/raman-nbg/inversify-koa-utils/badges/coverage.svg)](https://codeclimate.com/github/raman-nbg/inversify-koa-utils/coverage)
[![npm version](https://badge.fury.io/js/inversify-koa.svg)](http://badge.fury.io/js/inversify-koa)
[![Dependencies](https://david-dm.org/raman-nbg/inversify-koa-utils.svg)](https://david-dm.org/raman-nbg/inversify-koa-utils#info=dependencies)
[![img](https://david-dm.org/raman-nbg/inversify-koa-utils/dev-status.svg)](https://david-dm.org/raman-nbg/inversify-koa-utils/#info=devDependencies)
[![img](https://david-dm.org/raman-nbg/inversify-koa-utils/peer-status.svg)](https://david-dm.org/raman-nbg/inversify-koa-utils/#info=peerDependenciess)
[![Known Vulnerabilities](https://snyk.io/test/github/raman-nbg/inversify-koa-utils/badge.svg?targetFile=package.json)](https://snyk.io/test/github/raman-nbg/inversify-koa-utils?targetFile=package.json)
[![NPM](https://nodei.co/npm/inversify-koa.png?downloads=true&downloadRank=true)](https://nodei.co/npm/inversify-koa/)
[![NPM](https://nodei.co/npm-dl/inversify-koa.png?months=9&height=3)](https://nodei.co/npm/inversify-koa/)

inversify-koa is a module based on [inversify-express-utils](https://github.com/inversify/inversify-express-utils). This module has utilities for koa 2 applications development using decorators and IoC Dependency Injection (with inversify)

## Installation
You can install `inversify-koa` using npm:

```
$ npm install inversify inversify-koa reflect-metadata --save
```

The `inversify-koa` type definitions are included in the npm module and require TypeScript 2.0.
Please refer to the [InversifyJS documentation](https://github.com/inversify/InversifyJS#installation) to learn more about the installation process.

## The Basics

### Step 1: Decorate your controllers
To use a class as a "controller" for your koa app, simply add the `@controller` decorator to the class. Similarly, decorate methods of the class to serve as request handlers.
The following example will declare a controller that responds to `GET /foo'.

```ts
import * as Koa from 'koa';
import { interfaces, Controller, Get, Post, Delete } from 'inversify-koa';
import { injectable, inject } from 'inversify';

@controller('/foo')
@injectable()
export class FooController implements interfaces.Controller {

    constructor( @inject('FooService') private fooService: FooService ) {}

    @httpGet('/')
    private index(ctx: Router.IRouterContext , next: () => Promise<any>): string {
        return this.fooService.get(ctx.query.id);
    }

    @httpGet('/basickoacascading')
    private koacascadingA(ctx: Router.IRouterContext, nextFunc: () => Promise<any>): string {
        const start = new Date();
        await nextFunc();
        const ms = new Date().valueOf() - start.valueOf();
        ctx.set("X-Response-Time", `${ms}ms`);
    }

    @httpGet('/basickoacascading')
    private koacascadingB(ctx: Router.IRouterContext , next: () => Promise<any>): string {
        ctx.body = "Hello World";
    }

    @httpGet('/')
    private list(@queryParams('start') start: number, @queryParams('count') cound: number): string {
        return this.fooService.get(start, count);
    }

    @httpPost('/')
    private async create(@response() res: Koa.Response) {
        try {
            await this.fooService.create(req.body)
            res.body = 201
        } catch (err) {
            res.status = 400 
            res.body = { error: err.message }
        }
    }

    @httpDelete('/:id')
    private delete(@requestParam("id") id: string, @response() res: Koa.Response): Promise<void> {
        return this.fooService.delete(id)
            .then(() => res.body = 204)
            .catch((err) => {
                res.status = 400
                res.body = { error: err.message }
            })
    }
}
```

### Step 2: Configure container and server
Configure the inversify container in your composition root as usual.

Then, pass the container to the InversifyKoaServer constructor. This will allow it to register all controllers and their dependencies from your container and attach them to the koa app.
Then just call server.build() to prepare your app.

In order for the InversifyKoaServer to find your controllers, you must bind them to the `TYPE.Controller` service identifier and tag the binding with the controller's name.
The `Controller` interface exported by inversify-koa is empty and solely for convenience, so feel free to implement your own if you want.

```ts
import * as bodyParser from 'koa-bodyparser';

import { Container } from 'inversify';
import { interfaces, InversifyKoaServer, TYPE } from 'inversify-koa';

// set up container
let container = new Container();

// note that you *must* bind your controllers to Controller
container.bind<interfaces.Controller>(TYPE.Controller).to(FooController).whenTargetNamed('FooController');
container.bind<FooService>('FooService').to(FooService);

// create server
let server = new InversifyKoaServer(container);
server.setConfig((app) => {
  // add body parser
  app.use(bodyParser());
});

let app = server.build();
app.listen(3000);
```

## InversifyKoaServer
A wrapper for an koa Application.

### `.setConfig(configFn)`
Optional - exposes the koa application object for convenient loading of server-level middleware.

```ts
import * as morgan from 'koa-morgan';
// ...
let server = new InversifyKoaServer(container);

server.setConfig((app) => {
    var logger = morgan('combined')
    app.use(logger);
});
```

### `.setErrorConfig(errorConfigFn)`
Optional - like `.setConfig()`, except this function is applied after registering all app middleware and controller routes.

```ts
let server = new InversifyKoaServer(container);
server.setErrorConfig((app) => {
    app.use((ctx, next) => {
        console.error(err.stack);
        ctx.status = 500
        ctx.body = 'Something broke!';
    });
});
```

### `.build()`
Attaches all registered controllers and middleware to the koa application. Returns the application instance.

```ts
// ...
let server = new InversifyKoaServer(container);
server
    .setConfig(configFn)
    .setErrorConfig(errorConfigFn)
    .build()
    .listen(3000, 'localhost', callback);
```

## Using a custom Router
It is possible to pass a custom `Router` instance to `InversifyKoaServer`:

```ts

import * as Router from 'koa-router';

let container = new Container();

let router = new Router({
    prefix: '/api',
});

let server = new InversifyKoaServer(container, router);
```

By default server will serve the API at `/` path, but sometimes you might need to use different root namespace, for
example all routes should start with `/api/v1`. It is possible to pass this setting via routing configuration to
`InversifyKoaServer`

```ts
let container = new Container();

let server = new InversifyKoaServer(container, null, { rootPath: "/api/v1" });
```

## Using a custom koa application
It is possible to pass a custom `koa.Application` instance to `InversifyKoaServer`:

```ts
let container = new Container();

let app = new Koa();
//Do stuff with app

let server = new InversifyKoaServer(container, null, null, app);
```

## Decorators

### `@controller(path, [middleware, ...])`

Registers the decorated class as a controller with a root path, and optionally registers any global middleware for this controller.

### `@httpMethod(method, path, [middleware, ...])`

Registers the decorated controller method as a request handler for a particular path and method, where the method name is a valid koa routing method.

### `@SHORTCUT(path, [middleware, ...])`

Shortcut decorators which are simply wrappers for `@httpMethod`. Right now these include `@httpGet`, `@httpPost`, `@httpPut`, `@httpPatch`, `@httpHead`, `@httpDelete`, and `@All`. For anything more obscure, use `@httpMethod` (Or make a PR :smile:).

### `@request()`
Binds a method parameter to the request object.

### `@response()`
Binds a method parameter to the response object.

### `@requestParam(name?: string)`
Binds a method parameter to request.params object or to a specific parameter if a name is passed.

### `@queryParam(name?: string)`
Binds a method parameter to request.query or to a specific query parameter if a name is passed.

### `@requestBody(name?: string)`
Binds a method parameter to request.body or to a specific body property if a name is passed. If the bodyParser middleware is not used on the koa app, this will bind the method parameter to the koa request object.

### `@requestHeaders(name?: string)`
Binds a method parameter to the request headers.

### `@cookies()`
Binds a method parameter to the request cookies.

### `@context()`
Binds a method parameter to the koa context object.

### `@next()`
Binds a method parameter to the next() function.

### `@authorize([role: string, ...])`
Ensures that only authenticated and authorized users can invoke this controller method. If authentication fails the status code 401 will be returned. If authorization fails the status code 403 will be returned. Even if authentication or authorization fail all middlewares (for router, controller and method) will be invoked. If you don't provide any roles only authentication will be ensured.

### `@authorizeAll([role: string, ...])`
Ensures that only authenticated and authorized users can invoke the entire controller. Behaviour is the same as in `@authorize`.

## AuthProvider

You can provide a custom `AuthProvider` to create and provida a principal for the current request.

```ts
const server = new InversifyKoaServer(
    container, null, null, null, CustomAuthProvider
);
```

We need to implement the `AuthProvider` interface.

The `AuthProvider` allow us to get an user (`Principal`):

```ts
import * as Router from "koa-router";
import { injectable, inject } from "inversify";
import { interfaces } from "inversify-koa";

const authService = inject("AuthService");

@injectable()
class CustomAuthProvider implements interfaces.AuthProvider {

    @authService private readonly _authService: AuthService;

    public async getUser(ctx: Router.IRouterContext): Promise<interfaces.Principal> {
        const token = req.headers["x-auth-token"]
        const user = await this._authService.getUser(token);
        const principal = new Principal(user);
        return principal;
    }

}
```

We also need to implement the Principal interface.
The `Principal` interface allow us to:

- Access the details of an user
- Check if it has access to certain resource
- Check if it is authenticated
- Check if it is in an user role

```ts
class Principal implements interfaces.Principal {
    public details: any;
    public constrcutor(details: any) {
        this.details = details;
    }
    public isAuthenticated(): Promise<boolean> {
        return Promise.resolve(true);
    }
    public isResourceOwner(resourceId: any): Promise<boolean> {
        return Promise.resolve(resourceId === 1111);
    }
    public isInRole(role: string): Promise<boolean> {
        return Promise.resolve(role === "admin");
    }
}
```

We can then access the current principal via the context state:

```ts
@controller("/")
class UserDetailsController extends BaseHttpController {

    @inject("AuthService") private readonly _authService: AuthService;

    @httpGet("/")
    public async getUserDetails(@context() ctx: Router.IRouterContext) {
        if (await this.context.principal.isAuthenticated()) {
            return this._authService.getUserDetails(this.context.principal.details.id);
        } else {
            throw new Error();
        }
    }
}
```

If you want to ensure that only authenticatied users can invoke a method you can use `@authorize`:

```ts
@controller("/")
class UserDetailsController extends BaseHttpController {

    @inject("AuthService") private readonly _authService: AuthService;

    @httpGet("/")
    @authorize()
    public async getUserDetails(@context() ctx: Router.IRouterContext) {
        let isAuthorized = await this.context.principal.isAuthenticated();
        // isAuthorized will always be true
    }
}
```

## BaseMiddleware

Extending `BaseMiddleware` allow us to inject dependencies 
in Koa middleware function.

```ts
import { BaseMiddleware } from "inversify-koa";

@injectable()
class LoggerMiddleware extends BaseMiddleware {
    
    @inject(TYPES.Logger) private readonly _logger: Logger;

    public handler(ctx: Route.IRouteContext, next: () => Promise<any>): any {
        this._logger.info(ctx);
        await next();
    }
}
```

We also need to declare some type bindings:

```ts
const container = new Container();

container.bind<Logger>(TYPES.Logger)
        .to(Logger);

container.bind<LoggerMiddleware>(TYPES.LoggerMiddleware)
         .to(LoggerMiddleware);

```

We can then inject `TYPES.LoggerMiddleware` into one of our controllers. 

```ts
@injectable()
@controller("/")
class MyController extends BaseHttpController {

    @httpGet("/", TYPES.LoggerMiddleware)
    public async getResource() {
        // controller logic
    }
}

container.bind<interfaces.Controller>(TYPE.Controller)
         .to(MyController)
         .whenTargetNamed("MyController");
```

## License

License under the MIT License (MIT)

Copyright © 2017 [Diego Plascencia](https://github.com/diego-d5000)

Permission is hereby granted, free of charge, to any person obtaining a copy of this software and associated documentation files (the "Software"), to deal in the Software without restriction, including without limitation the rights to use, copy, modify, merge, publish, distribute, sublicense, and/or sell copies of the Software, and to permit persons to whom the Software is furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY, FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT.

IN NO EVENT SHALL THE AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM, OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE SOFTWARE.

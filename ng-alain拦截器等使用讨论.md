#### 主要遇到了两个问题，有兴趣的朋友，欢迎讨论，也非常感谢给予的建议。

### 问题1: 具体描述为：[github地址](https://github.com/cipchk/ng-alain/issues/481)
简单说，就是对load()方法里没有返回值的处理。自己摸索出了一个表象上的解决方法，具体请查看链接。

### 问题2：JWT拦截器
1. [SimpleInterceptor](https://github.com/cipchk/delon/blob/master/packages/auth/token/simple/simple.interceptor.ts)
2. [JWTInterceptor](https://github.com/cipchk/delon/blob/master/packages/auth/token/jwt/jwt.interceptor.ts)

`SimpleInterceptor` 用起来真的是很方便，不论对token字符串怎么修改，都是直接获取token字符串，然后加载到header，后台也是直接验证这条字符串的值。后台也不会在验证字符串之前进行格式验证处理。

但是，如果使用`JWTInterceptor`，在处理过程中，前后端都会对token的字符串进行格式验证和处理。

前端的验证传递处理过程有：

1. 验证格式是否正确[（源码）](https://github.com/cipchk/delon/blob/master/packages/auth/token/jwt/jwt.model.ts)：JWT格式为：xxxx.xxxxxxx.xxxxx，
2. 验证过期期时间[（源码）](https://github.com/cipchk/delon/blob/master/packages/auth/token/jwt/jwt.model.ts)：key: `'exp'`

要进行这些这些验证，就必须符合对应条件，因为是加密字段，任何字符都不能有错误，错误了，前端就可能不能进行解密，然后运行就可能会出错。比如格式不符，或不能解析过期时间。

这些过程没走完，数据是不会传递到后台的。所以，基本与后台无关。后台保证之前返回的JWT格式的token是合法正确的，就行了。

比如，我之前用的是`SimpleInterceptor`，有缓存token值，换成`JWTInterceptor`之后，程序直接不能启动。

提示报错源码：提示格式不符。
```
if (parts.length !== 3) throw new Error('JWT must have 3 parts');

```
#### 为了使JWT拦截器能更好的执行，可能需要对JWT拦截器进行修改。

修改基类拦截器，参考文档地址[cipchk/delon](https://github.com/cipchk/delon/blob/master/packages/auth/token/base.interceptor.ts)，具体的，请比对作者源码。
```
import { environment } from '@env/environment';
export const WINDOW = new InjectionToken<any>('Window');

export abstract class SelfBaseInterceptor implements HttpInterceptor {
  constructor(@Optional() protected injector: Injector) {}

  protected model: ITokenModel;

  abstract isAuth(options: DelonAuthConfig): boolean;

  abstract setReq(
    req: HttpRequest<any>,
    options: DelonAuthConfig,
  ): HttpRequest<any>;

  intercept(
    req: HttpRequest<any>,
    next: HttpHandler,
  ): Observable<
    | HttpSentEvent
    | HttpHeaderResponse
    | HttpProgressEvent
    | HttpResponse<any>
    | HttpUserEvent<any>
  > {
    const options = Object.assign(
      new DelonAuthConfig(),
      this.injector.get(DelonAuthConfig, null),
    );
    if (options.ignores) {
      for (const item of options.ignores as RegExp[]) {
        if (item.test(req.url)) return next.handle(req);
      }
    }

    if (
      options.allow_anonymous_key &&
      req.params.has(options.allow_anonymous_key)
    ) {
      return next.handle(req);
    }

    if (this.isAuth(options)) {
      req = this.setReq(req, options);
    } else {
      // 根据值判断是否在没有token的时候实现跳转
      if (options.token_invalid_redirect === true) {
        if (/^https?:\/\//g.test(options.login_url)) {
          this.injector.get(WINDOW).location.href = options.login_url;
        } else {
          this.injector.get(Router).navigate([options.login_url]);
        }
      }
      // observer.error：会导倒后续拦截器无法触发，因此，需要处理 `_HttpClient` 状态问题
      const hc = this.injector.get(_HttpClient, null);
      if (hc) hc.end();
      return new Observable((observer: Observer<HttpEvent<any>>) => {
        if (environment.debug) {    // 测试环境下，打印拦截日志
          console.log(
            `%c ${req.method}_WITH_JWT_ERROR: ${req.url}`,
            `background:red;color:#fff`,
          );
        }
        const res = new HttpErrorResponse({
          status: 401,
          statusText: `From Simple Intercept --> http://ng-alain.com/docs/auth`,
        });
        observer.error(res);
      });
    }
    return next.handle(req);
  }
}
```
修改JWT拦截器
```
import { environment } from '@env/environment';
import { SelfBaseInterceptor } from '@core/net/self-base.interceptor';

@Injectable()
export class SelfJWTInterceptor extends SelfBaseInterceptor {
  isAuth(options: DelonAuthConfig): boolean {
    try {
      this.model = this.injector
        .get(DA_SERVICE_TOKEN)
        .get<JWTTokenModel>(JWTTokenModel);
      if (environment.debug) {  // 测试、开发环境下打印日志
        console.log(
          `%c JWT_START: JWT验证开始`,
          `background:green;color:#fff`,
          this.model.time,
        );
      }
      return (
        this.model &&
        this.model.token &&
        !this.model.isExpired(options.token_exp_offset)
      );
    } catch (event) {
      if (environment.debug) {  // 测试、开发环境下打印日志
        console.log(
          `%c JWT_ERROR: JWT验证失败`,
          `background:red;color:#fff`,
          this.model.time,
        );
      }
      return false;
    }
  }

  setReq(req: HttpRequest<any>, options: DelonAuthConfig): HttpRequest<any> {
    return req.clone({
      setHeaders: {
        Authorization: `Bearer ${this.model.token}`,
      },
    });
  }
}
```
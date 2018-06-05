### 说在前面
本整理针对的场景是直接从作者的[github](https://github.com/cipchk/ng-alain)项目上打包下载的，非cli操作。这样的好处是，可以直接在本地跑作者的完整项目，直接`copy`和简单修改作者源码，以便快速应熟悉此框架。
对于程序的开发，并非专业出身，是出于兴趣，代码难免不够简洁和优雅，欢迎提供修改意见，但请勿苛责，谢谢。

### 关于ng-alain官方文档

#### 请在正式使用此框架前，仔细阅读作者的说明文档。

1. 启动（[官方文档地址](https://ng-alain.com/docs/how-to-start)）
    对于动手能力（对angular）理解弱的朋友，如果要对代码进行修改，请量力而行。遵循以`微量改造`和`学习模仿`为主的思路来摸索。
    **个人理解：** 程序启动后首先执行的操作是初始化。如果顺利操作，就完成了对项目的初始化。但是，在某种情况下，这个代码逻辑是并没有完全执行的，可能执行了一部分，就被中断了。（中断的原理，简单说就是由于拦截器在验证过程中，对不符合条件的http请求进行的拦截，并通过路由重新导航）
2. 用户认证（[官方文档地址](https://ng-alain.com/auth/getting-started)）
    **个人理解：** 用户认证的方式有很多种，我自己采取的方式是登录，获取token，然后把token存下来，备用。当token没有的时候，也就不能登录了。

### app启动流程

首先启动`startup.service.ts`中的load()方法。
初始化过程代码解析：
```
// 原代码
this.httpClient.get(`assets/_/i18n/${this.i18n.defaultLang}.json`), // 请求本地数据
this.httpClient.get('assets/_/app-data.json'),  // 请求本地数据
```

代码修改的地方为`src/core/startup/startup.service.ts`
```
// 修改后
this.httpClient.get(`assets/_/i18n/${this.i18n.defaultLang}.json`), // 未做修改
this.httpClient.get(this.apiurl.app_startup_init), // this.apiurl.app_startup_init是后台的请求地址https://xxxxx.com/api/app
// ng-alain的拦截器会自动加载token，没有token，会跳转登录页
// 后端接口，根据是否有token和token是否合法，返回同的值
// 可以根据token，返回用户信息、权限信息、菜单信息等初始化时需要用到的数据
```
--------对返回结果的处理-----------
```
// 原代码
// application data
const res: any = appData;
// 应用信息：包括站点名、描述、年份
this.settingService.setApp(res.app);
// 用户信息：包括姓名、头像、邮箱地址
this.settingService.setUser(res.user);
// ACL：设置权限为全量
this.aclService.setFull(true);
// 初始化菜单
this.menuService.add(res.menu);
// 设置页面标题的后缀
this.titleService.suffix = res.app.name;
```
代码修改的地方为`src/core/startup/startup.service.ts`
```
// 个人修改
// application data
const res: any = appData.item;
// 应用信息：包括站点名、描述、年份
if (res.menu) {
    this.settingService.setApp(res.app);
    // 设置页面标题的后缀
    this.titleService.suffix = res.app.name;
}
// 用户信息：包括姓名、头像、邮箱地址
if (res.user) {
    this.settingService.setUser(res.user);
}
// 初始化菜单
if (res.menu) {
// ACL：设置权限为全量
    this.aclService.setFull(true);
    this.menuService.clear();
    this.menuService.add(res.menu);
}
```
后台服务端在获取token之后，一般会验证token，如果token不合法，可能会直接报错，提示错误信息，附带错误的状态码，如`401`、`404`等，另一种处理方式，是正常返回其他信息，如公开的APP信息等，为了不进入ng-alain的状态码拦截机制，返回的状态码就需要是正常的`200`了。

我采用的后者，如果没有token或token错误，正常返回APP的基础公共信息，所以，为了保证正常加载，我在程序启动的时候，添加了[路由守卫](https://www.angular.cn/guide/router#milestone-5-route-guards)，当toekn不正确的时候，会跳转到登录页。


```
代码修改的地方为`src/core/layout/passport/passport.component.ts`
// 登录页passport初始化 
constructor(
    public settingService: SettingsService,
    public passport: PassportService,
 ) {
    passport.appInit(); // 初始化方法，获取基本信息
 }
```
```
此处为自定义公`共服务中的公共方法`
// 直接访问
direct: any = { _allow_anonymous: true };
constructor(
    private http: _HttpClient,        
    public titleService: TitleService,
    public settingService: SettingsService,
    public log: ShowLogService,
    public msg: NzMessageService,
    private CommData: CommDataService,
    private apiurl: ApiUrlService,
    private router: Router
) { }
// 获取App的基本信息
getAppBasicInfo(): Observable<any> {
    return this.http.get(this.apiurl.app_startup_init, this.direct);
}
// 仿造startup.service.ts中的load()方法
appInit(): Promise<any> {
    return new Promise((resolve, reject) => {
        if (this.CommData.isNeedInitApp) {
            this.CommData.isNeedInitApp = false;
            this.getAppBasicInfo().subscribe(
                response => {
                    this.log.logObj(response, 'APP简单初始化信息', 'green');
                    // application data
                    const res: any = response.item;
                    // 应用信息：包括站点名、描述、年份
                    this.settingService.setApp(res.app);
                    // 设置页面标题的后缀
                    this.titleService.suffix = res.app.name;
                },
                () => { },
                () => {
                    resolve(null);
                }
            );
        }
        resolve(null);
    });
}
```

```
// 路由守卫核心代码
@Injectable()
export class AuthGuard implements CanActivate {

  constructor(
    private passport: PassportService,
    public msg: NzMessageService,
    private CommData: CommDataService,
    public log: ShowLogService,
    private router: Router,
    @Inject(DA_SERVICE_TOKEN) private tokenService: ITokenService
  ) { }

  canActivate(
    route: ActivatedRouteSnapshot,
    state: RouterStateSnapshot): boolean | Observable<boolean> | Promise<boolean> {

    const url: string = state.url;
    this.CommData.redirectUrl = url;
    this.log.logMsg('开始验证权限...', 'blue'); // 自定义log标记
    // 判断是否有Token
    const _token = this.tokenService.get();
    if (!_token.token) {
      this.log.logMsg('没有登录Token，权限验证失败...', 'red');
      this.passport.appInit().then(() => this.router.navigate(['/passport/login']));
      return false;
    }
    // 判断是否是锁屏状态
    const falg = this.CommData.isLockCcreen;
    if (falg) {
      this.log.logMsg('屏幕解锁验证失败！！！', 'red');
      this.msg.info('请输入正确登录密码，解锁屏幕...', { nzDuration: 2500 });
      this.router.navigate(['/passport/lock']);
      return false;
    }
    // 进行正常的权限验证
    return new Observable((observer) => {
      this.passport.checkToken().subscribe(
        (response: ApiResToComponent) => {
          if (!response.isComplete) observer.next(false);          
          if (response.isComplete && !response.items.needLogin) observer.next(true), 
          this.log.logMsg('成功通过验证权限！！！', 'green');       
          observer.complete();
        }
      );
    });
  }
}
```

```
// 原登录页模拟的登录代码
setTimeout(() => {
      this.loading = false;
      if (this.type === 0) {
        if (
          this.userName.value !== 'admin' ||
          this.password.value !== '888888'
        ) {
          this.error = `账户或密码错误`;
          return;
        }
      }
      // 清空路由复用信息
      this.reuseTabService.clear();
      // 设置Token信息
      this.tokenService.set({
        token: '123456789',
        name: this.userName.value,
        email: `cipchk@qq.com`,
        id: 10000,
        time: +new Date(),
      });
      // 重新获取 StartupService 内容，若其包括 User 有关的信息的话
      // this.startupSrv.load().then(() => this.router.navigate(['/']));
      // 否则直接跳转
      this.router.navigate(['/']);
    }, 1000);
```
```
// 个人修改
this.passport.login(body).subscribe(
    (response: ApiResToComponent) => {
        this.loading = false;
        if (response.isComplete) this.setUserInfoAndLoad(response.items);
    }
);

// 确认用户信息并重新加载
setUserInfoAndLoad(userInfo: UserInfoAndToken) {
    // 清空路由复用信息
    this.reuseTabService.clear();
    // 重新加载菜单信息                        
    this.tokenService.set({
        token: userInfo.token,
        name: userInfo.user.name,
        email: userInfo.user.email,
        avatar: userInfo.user.avatar,
        time: +new Date
    });
    // 标记已经登录
    this.CommData.isLoggedIn = true;
    // 重新获取 StartupService 内容，若其包括 User 有关的信息的话
    this.startupSrv.load().then(() => this.router.navigate([this.CommData.redirectUrl]));
    // 否则直接跳转
    // this.router.navigate([this.CommData.redirectUrl]);
}

```

整个项目流程如下
```flow
st=>start: StartupService.load()
co1=>condition: Token ?
op1=>operation: passport/login
op5=>operation: Login and SaveToken
op2=>operation: service API (checkToken / getAppInitInfo)
co2=>condition: Token is Right ?
op3=>operation: getAppInitInfo
op4=>operation: getAllBasicInfo
ed=>end: StartupService.load()


st->co1(no,right)->op3->op1
co1(yes)->op2->co2(no,right)->op3->op1
op1(right)->op5->ed
co2(yes)->op4->ed
```






















